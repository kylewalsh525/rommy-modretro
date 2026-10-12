# Cartridge pack and save protocol v1

> This reference describes the access-controlled Rommy source kit. See the [starting guide](../README.md) for source access. File paths and build commands refer to that source checkout. Documentation snapshot: 2026-10-12.

## Assignment envelope and content hash

The exporter accepts the offline-assignment API response envelope (other response metadata is not compiled):

```json
{
  "assignmentId": "0123456789abcdef0123456789abcdef",
  "packHash": "64 lowercase SHA256 hexadecimal characters",
  "pack": {
    "schemaVersion": 1,
    "assignmentId": "0123456789abcdef0123456789abcdef",
    "questions": [
      {
        "questionId": "question-identifier",
        "questionIndex": 0,
        "questionText": "What is 2 plus 2?",
        "choices": ["3", "4"],
        "correctChoiceIndex": 1
      }
    ]
  }
}
```

`packHash` is SHA256 of the UTF-8 bytes of compact `JSON.stringify(pack)` with properties reconstructed in exactly the displayed order: `schemaVersion,assignmentId,questions`; each question `questionId,questionIndex,questionText,choices,correctChoiceIndex`. It does not hash the envelope or the `packHash` field itself. JSON insertion order from a storage system must not be trusted. Strings use normal JSON encoding, and there is no trailing newline in the hashed bytes.

Assignment IDs are exactly 32 lowercase hexadecimal characters. Hashes are exactly 64. There are 1–32 unique question IDs (printable ASCII 1–256 characters, no slash), and indexes are consecutive zero-based array positions. `correctChoiceIndex` and `choiceIndex` are zero-based. Text wrapping is deterministic: split on single spaces, append each word to the current line when it fits 18 columns including a separator, otherwise start a line. Reject nonprintable/non-ASCII text, leading/trailing/consecutive spaces, any word longer than 18, more than 5 prompt lines/90 prompt characters, or more than 2 choice lines/36 choice characters. Choices number 2–4. No truncation or silent transliteration is permitted.

## Save layout

Exactly 8192 bytes; all multibyte integers use **little endian**. The save stores no key, bearer token or learner name. The assignment ID resolves learner ownership on the server.

| Offset | Bytes | Meaning |
| --- | ---: | --- |
| 0 | 8 | ASCII `RMYGBSV1` |
| 8 | 1 | Format version 1 |
| 9 | 1 | Question count 1–32 |
| 10 | 2 | Record size 24 |
| 12 | 16 | Assignment ID decoded from hex, in byte-string order |
| 28 | 32 | Pack hash decoded from hex, in byte-string order |
| 60 | 4 | CRC32 of bytes 0–59 |
| 64 | 8112 | 338 consecutive 24-byte record slots |
| 8176 | 16 | Reserved zero padding |

Each record has this layout relative to its slot:

| Offset | Bytes | Meaning |
| --- | ---: | --- |
| 0 | 4 | Sequence; first is 1, each next is previous+1 |
| 4 | 1 | Question index |
| 5 | 1 | Choice index |
| 6 | 2 | Reserved zero |
| 8 | 4 | Response time in milliseconds, unsigned 32-bit |
| 12 | 4 | CRC32 of record bytes 0–11 |
| 16 | 4 | ASCII commit marker `CMT1` |
| 20 | 4 | Reserved zero |

CRC32 is the reflected IEEE algorithm: polynomial 0xEDB88320, initial 0xFFFFFFFF, final XOR 0xFFFFFFFF. Test vector ASCII `123456789` gives 0xCBF43926. Both C and JavaScript implementations are tested against the same save bytes.

Initialization is offered only when every byte of the save is 00 or FF. It clears the region to 00 and writes the header. A partial header or unrelated save is not erased automatically. For an answer, write record bytes 0–15 and 20–23, then commit bytes 16–19 last. Read back and validate the whole journal and expected record count before acknowledging the answer. No previously occupied slot is reused.

A valid record needs its commit marker, CRC, zero reserved fields, valid question/choice indexes and expected sequence. The first all-zero slot ends the journal; all later bytes must be zero. A nonzero slot without the complete marker is an incomplete tail only when all subsequent bytes are zero. Ordinary import stops with an error; an explicit recovery flag can import the already-committed prefix. A committed record with a bad CRC, unexpected sequence, impossible index or later hidden data is corruption and blocks import. The cartridge blocks further answers on either torn or corrupt state. A full journal is readable but cannot accept more answers.

CRC detects accidental corruption and incomplete writes. It is **not** authentication or proof of a student's identity. The authenticated import API validates project ownership and the immutable server-side pack. Offline records do not contain a reliable wall-clock time; the reference renderer measures approximate 60 Hz elapsed frames and saturates uint32 milliseconds. Server processing time is distinct from time spent answering.

## Import API

`POST /api/public/v1/offline/assignments/{assignmentId}/attempts` uses `X-Rommy-Api-Key` on the trusted computer. Body:

```json
{
  "packHash": "64 lowercase hexadecimal characters",
  "attempts": [
    { "sequence": 1, "questionIndex": 0, "choiceIndex": 1, "responseTimeMs": 1500 }
  ]
}
```

A batch contains 1–8 ordered consecutive attempts. Sequences and response times are unsigned 32-bit; sequence 0 is forbidden. The server expects new sequences in order and uses stable `assignmentId + sequence` identity, independent of runtime sessions. Repeated identical records return receipts; changed content for an existing sequence conflicts. The server grades against its stored immutable pack and atomically commits the learning update, import receipt, usage and webhook delivery. A batch may partially commit before a later record fails; retrying the original batch is safe.

The response contains `assignmentId`, `packHash`, `nextSequence` and one ordered receipt per submitted record: `sequence,questionIndex,choiceIndex,responseTimeMs,isCorrect,attemptId,processedAt`. `processedAt` is a UTC ISO8601 string with millisecond precision. The importer validates answer/time identities and local grading against the immutable pack, requires nonempty unique attempt IDs and a cursor greater than the submitted final sequence, and rejects error-bearing success responses. An older replay can legitimately return a cursor beyond the submitted batch. Responses are limited to 64 KiB.

The importer does not alter or compact the save and sends all confirmed records on every invocation. A failed or malformed response may follow a committed write; retain the save and replay the same records. Never resolve a conflict by silently changing the assignment or sequence. A new lesson/assignment, not a local journal reset, starts a new identity space.
