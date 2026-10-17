# ShardStore Protocol Specification

**Document status:** Draft — implementation baseline  
**Protocol version:** 1 (initial implementation)  
**Scope:** Client–server wire protocol for ShardStore's initial TCP key-value server

---

## 1. Purpose and scope

This document defines the wire protocol used by clients to communicate with a ShardStore server over TCP. It specifies request framing, argument encoding, response types, command semantics, validation rules, and connection behavior.

The initial protocol is intentionally small and binary-safe. It supports `PING`, `SET`, `GET`, `DEL`, and `QUIT`.

### 1.1 Design goals

- **Unambiguous framing:** TCP is a byte stream, so message boundaries must be explicit.
- **Binary-safe values:** Keys, values, and optional `PING` messages are byte sequences, not null-terminated strings.
- **Bounded resource use:** Frames, keys, and argument counts have explicit limits.
- **Request correlation:** Every request and response carries a 64-bit request ID.
- **Predictable execution:** Requests on one connection execute in receive order in the initial implementation.
- **Implementation simplicity:** The protocol should be straightforward to parse and test in Rust.

### 1.2 Non-goals for protocol version 1

The initial protocol does not define authentication, encryption, transactions, client-side cancellation, subscriptions, streaming values, chunked frames, automatic leader election, or a consensus protocol. These may be considered separately if the project requires them.

---

## 2. Wire-format conventions

Unless stated otherwise:

- All multi-byte integers use **network byte order (big-endian)**.
- Integer lengths count **bytes**, not Unicode characters.
- Binary fields may contain any byte value, including `0x00`.
- No field is null-terminated.
- The server must not assume that a TCP read returns exactly one frame. A read may contain a partial frame, one complete frame, or multiple frames.
- The protocol's maximums are byte limits. Implementations must validate them before allocating memory based on untrusted lengths.

### 2.1 Integer types

| Type | Width | Encoding | Range |
|---|---:|---|---|
| `u8` | 1 byte | Unsigned | 0–255 |
| `u16` | 2 bytes | Unsigned, big-endian | 0–65,535 |
| `u32` | 4 bytes | Unsigned, big-endian | 0–4,294,967,295 |
| `u64` | 8 bytes | Unsigned, big-endian | 0–2⁶⁴−1 |
| `i64` | 8 bytes | Signed, big-endian two's complement | −2⁶³–2⁶³−1 |

Request IDs use `u64`. The `DEL` result uses `i64` in the response format; its value is always non-negative.

### 2.2 Limits

| Limit | Value | Definition |
|---|---:|---|
| Maximum request frame body | 67,108,864 bytes (64 MiB) | Bytes after the 4-byte request-length prefix; includes request ID and command payload |
| Maximum key length | 65,535 bytes | Applies to every key in `SET`, `GET`, and `DEL` |
| Maximum argument count | 1,024 arguments | Includes the command name; therefore `DEL` accepts at most 1,023 keys |
| Maximum error-message length | 65,535 bytes | UTF-8 byte length, represented by `u16` |
| Maximum value length | Frame-limited | A value must fit inside a valid request frame, including all framing and argument metadata |

The 64 MiB limit is `64 * 1024 * 1024` bytes. It excludes the 4-byte length prefix but includes the 8-byte request ID, the argument count, and all argument-length fields and data.

A server must reject a frame length larger than 64 MiB **before allocating a buffer of that size**. Chunked transfers are not supported in protocol version 1.

---

## 3. Request format

### 3.1 Frame layout

Each request is encoded as:

```text
+----------------------+-------------------------------+
| Field                | Size                          |
+----------------------+-------------------------------+
| Frame body length    | 4 bytes (u32, big-endian)     |
| Request ID           | 8 bytes (u64, big-endian)     |
| Argument count       | 2 bytes (u16, big-endian)     |
| Argument 0 length    | 4 bytes (u32, big-endian)     |
| Argument 0 data      | Argument 0 length bytes       |
| Argument 1 length    | 4 bytes (u32, big-endian)     |
| Argument 1 data      | Argument 1 length bytes       |
| ...                  | ...                           |
+----------------------+-------------------------------+
```

The 4-byte `frame_body_length` prefix specifies the number of bytes that follow it. It **does not include the prefix itself**. It includes the request ID and all remaining request fields.

The request body is therefore:

```text
[u64 request_id][u16 argument_count][encoded arguments...]
```

Each argument is encoded as:

```text
[u32 argument_length][argument_length bytes of data]
```

The argument count includes the command name, which is always argument 0.

### 3.2 Required validation

The server must validate each request in this order:

1. Read the 4-byte length prefix.
2. Reject and close the connection if the length is smaller than the minimum body size or larger than 64 MiB. The minimum valid body is 14 bytes: 8-byte request ID, 2-byte argument count, one 4-byte argument length, and at least one command byte.
3. Read exactly the declared number of body bytes. If the body is incomplete, wait for more bytes; do not treat a partial read as a malformed request.
4. Parse the request ID and argument count.
5. Require the argument count to be between 1 and 1,024 inclusive.
6. For each argument, ensure that its 4-byte length field is available and that the declared argument data fits within the remaining frame body.
7. Require the parser to consume the body exactly. Trailing unexplained bytes are invalid.
8. Validate the command's argument count, key lengths, and command-specific constraints before executing any mutation.

Length arithmetic must be checked for overflow. Do not trust a length field merely because it fits in its integer type.

### 3.3 Request IDs

- A request ID is an opaque `u64` chosen by the client.
- A client must not reuse an ID while an earlier request with that ID is outstanding on the same connection.
- The server echoes the ID in the corresponding response.
- Request IDs are for correlation; they do not imply durability, ordering, uniqueness across connections, or exactly-once execution.
- In the initial implementation, requests on a single connection are processed sequentially in receive order and responses are sent in that same order. IDs are retained to support correlation and future protocol evolution.

---

## 4. Argument encoding and command names

The command name is argument 0. Command names are ASCII and case-sensitive; version 1 defines uppercase names only.

The remaining arguments are command-specific. All arguments are byte strings, but command names must match the ASCII byte sequences shown below.

### 4.1 Command summary

| Command | Arguments, including command | Success response | Description |
|---|---:|---|---|
| `PING` | 1 or 2 | `SIMPLE_OK` or `BLOB` | Health check; optionally echoes a binary message |
| `SET` | Exactly 3 | `SIMPLE_OK` | Store a value for a key |
| `GET` | Exactly 2 | `BLOB` or `NULL` | Retrieve a value by key |
| `DEL` | 2–1,024 | `INTEGER` | Delete one or more keys |
| `QUIT` | Exactly 1 | `SIMPLE_OK`, then close | Close the client connection cleanly |

An unknown command returns `UNKNOWN_COMMAND` when the request frame is otherwise well-formed.

### 4.2 `PING [message]`

**Argument forms**

```text
PING
PING message
```

- With one argument, return `SIMPLE_OK`.
- With two arguments, return `BLOB` containing the exact bytes of `message`.
- The optional message is binary-safe and may be empty.
- `PING` does not read or modify stored keys.

### 4.3 `SET key value`

**Arguments:** exactly 3 — command, key, value.

- The key must be non-empty and no longer than 65,535 bytes.
- The value may be empty.
- The value is stored as an opaque byte sequence.
- On success, return `SIMPLE_OK`.
- If the key is invalid, return `BAD_REQUEST` and do not mutate storage.

The protocol-level `SIMPLE_OK` confirms that the command was accepted and completed according to the server's configured storage semantics. It does **not**, by itself, promise that the write survives a process crash or power loss. Those guarantees are defined by the storage engine's durability mode and must be documented separately.

### 4.4 `GET key`

**Arguments:** exactly 2 — command, key.

- The key must be non-empty and no longer than 65,535 bytes.
- If the key exists, return `BLOB` containing its value.
- If the key does not exist, return `NULL`.
- An existing empty value returns `BLOB` with a length of zero; it must not be confused with `NULL`.
- If the key is invalid, return `BAD_REQUEST`.

### 4.5 `DEL key [key ...]`

**Arguments:** 2–1,024 — command and one or more keys.

- Every key must be non-empty and no longer than 65,535 bytes.
- The server must validate **all** keys before deleting any of them. If any key is invalid, return `BAD_REQUEST` and perform no deletions.
- A missing key is not an error.
- The `INTEGER` response is the number of distinct keys that existed and were removed.
- Duplicate keys in one `DEL` request count at most once. For example, `DEL a a b` returns `2` if both `a` and `b` existed before the command.
- Deleting an already absent key leaves it absent and does not increase the returned count.

### 4.6 `QUIT`

**Arguments:** exactly 1 — command only.

- Return `SIMPLE_OK`.
- After the response has been written, close the connection cleanly.
- Do not process additional requests received after `QUIT` in the same connection's request stream.

---

## 5. Response format

### 5.1 Response envelope

Every response uses the following layout:

```text
+----------------------+-------------------------------+
| Field                | Size                          |
+----------------------+-------------------------------+
| Request ID           | 8 bytes (u64, big-endian)     |
| Response tag         | 1 byte (u8)                   |
| Type-specific data   | Determined by response tag    |
+----------------------+-------------------------------+
```

Responses do not have a separate length prefix in protocol version 1. The response tag determines the payload layout, and every payload is either fixed-width or explicitly length-prefixed. This allows the client to parse a TCP byte stream without an additional response-length field.

The server must serialize writes to each connection so that bytes from concurrent responses can never interleave. In the initial implementation, sequential request processing also ensures that responses are emitted in request receive order.

### 5.2 Response tags

| Tag | Name | Payload | Meaning |
|---:|---|---|---|
| `0x00` | `SIMPLE_OK` | None | Command completed successfully |
| `0x01` | `INTEGER` | 8-byte `i64`, big-endian | Integer result; non-negative for `DEL` |
| `0x02` | `BLOB` | 4-byte `u32` length, then that many bytes | Binary payload |
| `0x03` | `NULL` | None | Requested key does not exist |
| `0x40` | `BAD_REQUEST` | 2-byte `u16` message length, then UTF-8 bytes | Request is malformed or violates command rules |
| `0x41` | `UNKNOWN_COMMAND` | 2-byte `u16` message length, then UTF-8 bytes | Command name is not recognized |
| `0x50` | `INTERNAL_ERROR` | 2-byte `u16` message length, then UTF-8 bytes | Server could not complete an otherwise valid request |

All length fields are big-endian. Error messages must be valid UTF-8 and no longer than 65,535 bytes. Error messages should be short, stable, and safe to expose to clients; internal stack traces, filesystem paths, and sensitive implementation details must not be returned.

### 5.3 Response parsing

A client parses responses as follows:

1. Read 8 bytes for the request ID.
2. Read 1 byte for the response tag.
3. Read the payload dictated by the tag:
   - `SIMPLE_OK`, `NULL`: no additional bytes.
   - `INTEGER`: exactly 8 bytes.
   - `BLOB`: 4-byte length followed by that many bytes.
   - `BAD_REQUEST`, `UNKNOWN_COMMAND`, `INTERNAL_ERROR`: 2-byte length followed by that many UTF-8 bytes.
4. Reject an unknown response tag or invalid payload encoding as a protocol error.

Because responses have no independent length prefix, clients and servers must keep the tag-to-payload mapping stable. Any incompatible change to these layouts requires a protocol versioning decision.

---

## 6. Error handling and connection behavior

### 6.1 Parser outcomes

The request parser should distinguish three outcomes:

- **Complete:** a full, valid request frame is available.
- **Incomplete:** more TCP bytes are needed; retain the buffered bytes and wait.
- **Invalid:** the frame or its contents violate the protocol.

This distinction prevents ordinary TCP fragmentation from being mistaken for malformed input.

### 6.2 Invalid input policy

| Condition | Required behavior |
|---|---|
| Length prefix not yet fully received | Wait for more bytes |
| Body not yet fully received | Wait for more bytes |
| Frame length exceeds 64 MiB | Close connection immediately; do not allocate the declared body |
| Frame body too short to contain a request ID | Close connection; no response can be correlated |
| Malformed body with a readable request ID | Send `BAD_REQUEST` for that ID, then continue with the next frame |
| Invalid argument count or malformed argument encoding | Send `BAD_REQUEST` if the request ID is available; do not execute the command |
| Invalid command-specific argument count or key | Send `BAD_REQUEST`; do not execute the command |
| Unknown command in a well-formed frame | Send `UNKNOWN_COMMAND` |
| Storage or unexpected server failure | Send `INTERNAL_ERROR` if the connection remains usable |
| Client disconnects before a complete frame arrives | Discard the incomplete request and release connection resources |
| Client disconnects after sending a request but before receiving its response | The outcome may be unknown to the client; request IDs do not provide exactly-once semantics |
| `QUIT` | Send `SIMPLE_OK`, flush the response, then close the connection |

If the request ID cannot be read safely, the server must close the connection rather than emit a response with an invented ID.

A malformed request must never partially execute a mutating command. In particular, `DEL` must validate all keys before changing storage.

### 6.3 Connection lifecycle

- A connection may carry multiple request frames.
- Requests are parsed from the byte stream in frame order.
- The server processes requests on each connection sequentially in receive order in version 1.
- The server may process different connections concurrently.
- The server must serialize all writes to a given connection.
- A clean client disconnect releases the connection's resources.
- If the client sends `QUIT`, the server completes the `QUIT` response before closing the connection.
- Timeouts, maximum connection counts, and backpressure are operational server settings. They should be implemented and documented at the server layer rather than inferred from this wire format.

### 6.4 Concurrency across connections

Different connections may issue commands concurrently. The storage layer must define the synchronization and atomicity of individual operations.

Protocol version 1 does not promise a deterministic order for conflicting operations on the same key when those operations originate from different connections. Clients must not assume that request IDs establish a global order. Any stronger consistency guarantee must be specified by the storage and replication layers.

---

## 7. Binary examples

The examples below show logical fields and hexadecimal encodings. They are illustrative; multi-byte integers are big-endian.

### 7.1 `PING` request

A `PING` request with request ID `1` and one argument (`PING`):

```text
frame_body_length = 14
request_id        = 1
argument_count    = 1
argument_0        = "PING"
```

Hexadecimal:

```text
00 00 00 0E                         # body length = 14
00 00 00 00 00 00 00 01             # request ID = 1
00 01                               # argument count = 1
00 00 00 04 50 49 4E 47             # argument 0: length 4, "PING"
```

**Note:** The body length is 14 bytes (`8 + 2 + 4 + 4`), matching the length prefix.

A successful response:

```text
00 00 00 00 00 00 00 01             # request ID = 1
00                                  # SIMPLE_OK
```

### 7.2 `SET key value` request

Logical arguments:

```text
request_id = 2
arguments  = ["SET", "key", "value"]
```

The body contains the request ID, argument count, and three length-prefixed byte strings. A successful response is:

```text
00 00 00 00 00 00 00 02             # request ID = 2
00                                  # SIMPLE_OK
```

### 7.3 `GET` results

For `GET key`, assuming request ID `3`:

**Key exists with value `value`:**

```text
00 00 00 00 00 00 00 03             # request ID = 3
02                                  # BLOB
00 00 00 05                         # blob length = 5
76 61 6C 75 65                      # "value"
```

**Key does not exist:**

```text
00 00 00 00 00 00 00 03             # request ID = 3
03                                  # NULL
```

The response tag distinguishes a missing key from an existing key whose value is empty.

---

## 8. Implementation guidance for Rust

The following guidance is non-normative, but it aligns with the protocol's safety requirements.

### 8.1 Parsing

- Implement framing and payload parsing as separate steps.
- Prefer a parser interface that can return `Complete`, `Incomplete`, or `Invalid`.
- Use checked arithmetic when calculating offsets and required buffer sizes.
- Validate the frame length before allocating the complete frame body.
- Ensure that argument lengths cannot cause reads past the frame boundary.
- Reject trailing bytes after the final declared argument.
- Avoid converting arbitrary key/value bytes to UTF-8. Only command names and error messages require UTF-8/ASCII interpretation.

### 8.2 Execution

- Parse and validate the full request before invoking the storage layer.
- Keep protocol parsing separate from command execution and storage logic.
- Validate every `DEL` key before performing any deletion.
- Preserve the distinction between `NULL` and a zero-length `BLOB`.
- Convert internal failures into bounded `INTERNAL_ERROR` messages without exposing sensitive diagnostics.

### 8.3 TCP handling

- Use exact-read logic or an incremental buffer; a single `read` call is not guaranteed to return a complete field or frame.
- Do not assume one request per connection or one request per read.
- Ensure a single writer owns each connection's output stream, or protect writes with an appropriate serialization mechanism.
- Apply operational limits for connection count, idle time, and buffered incomplete requests to reduce resource-exhaustion risk.

---

## 9. Testing checklist

Protocol implementation is not complete until the following cases are covered by automated tests.

### 9.1 Framing and parser tests

- [ ] A valid frame split across multiple TCP reads.
- [ ] Multiple complete requests delivered in one TCP read.
- [ ] Length prefix smaller than the minimum valid body.
- [ ] Length prefix larger than 64 MiB.
- [ ] Argument count equal to zero.
- [ ] Argument count greater than 1,024.
- [ ] Missing argument-length field.
- [ ] Argument length greater than the remaining body bytes.
- [ ] Trailing unexplained bytes in a frame.
- [ ] Integer/offset overflow attempts.
- [ ] Frame body too short to contain a request ID.

### 9.2 Command tests

- [ ] `PING` without a message.
- [ ] `PING` with ordinary, binary, and empty messages.
- [ ] `SET` followed by `GET`.
- [ ] `SET` with an empty value.
- [ ] `GET` for a missing key.
- [ ] Empty key rejection for `SET`, `GET`, and `DEL`.
- [ ] Key exactly at the maximum permitted length and one byte over it.
- [ ] `DEL` with one key and with multiple keys.
- [ ] `DEL` with missing and duplicate keys.
- [ ] Invalid `DEL` request leaves all keys unchanged.
- [ ] `QUIT` response is fully written before the connection closes.
- [ ] Unknown command returns `UNKNOWN_COMMAND`.

### 9.3 Response and connection tests

- [ ] Each response tag is encoded and parsed correctly.
- [ ] Binary blobs containing `0x00` are preserved.
- [ ] Empty `BLOB` is distinct from `NULL`.
- [ ] Error messages are valid UTF-8 and within the configured limit.
- [ ] Request IDs are echoed exactly.
- [ ] Responses from concurrent server activity never interleave on one connection.
- [ ] Disconnect during an incomplete request releases buffered resources.
- [ ] Malformed requests do not mutate storage.

---

## 10. Versioning and future changes

This document defines the initial protocol baseline, not a permanent compatibility guarantee.

Changes to response tags, request framing, field widths, byte order, or the meaning of existing commands can break clients. Such changes should be reviewed explicitly and recorded in an architecture decision record (ADR) before implementation or release.

Potential future extensions include:

- Explicit protocol version negotiation.
- Authentication and TLS.
- A response length prefix, if future response types require less rigid parsing.
- Chunked or streamed large values.
- Batching and transaction semantics.
- Explicit durability acknowledgements.
- Replication-aware status and error codes.

These extensions are not part of protocol version 1 unless this document is revised.

---

## 11. Open implementation notes

The wire-format decisions in this document are sufficient to begin implementing the initial protocol. The following are server/storage policy decisions and should be specified in the relevant design documents rather than left implicit:

1. **Durability contract:** Which configured durability mode a successful `SET` acknowledges.
2. **Storage atomicity:** The atomicity guarantees of individual commands and the behavior under concurrent access.
3. **Operational limits:** Connection caps, read/write timeouts, and maximum buffered incomplete input.
4. **Replication semantics:** When a write is considered successful in replicated mode and what happens when replicas are unavailable.

Until those contracts are defined, the protocol must not claim crash durability, exactly-once execution, or globally ordered operations.