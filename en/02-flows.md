# Fomoxa: wire format, flows, implementation guide

This document describes the behaviour every Fomoxa implementation must have. It is independent of any programming language and references no implementation, API or source code.

This document together with `01_overview.md` is enough to rebuild Fomoxa, either as a new implementation or as a new transport, in any language.

- `01_overview.md`: the model, layer boundaries, split of responsibilities
- `02_flows.md` (this one): byte formats, every flow, implementation guide

Conventions used here:

- **LE** = little-endian. Every multi-byte number in Fomoxa is little-endian.
- The six symbols ✔ ⏸ ✖ ⚠ ⊘ ⤢ are the six transport answers, defined in `01_overview.md` §2.
- Protocol terms have exactly one name each: client, server, peer, transport, frame,
  handshake, greeting, verdict, query, schema, fingerprint, session, message, heartbeat,
  probe, tick. Frame, state and event names are capitalised: DATA, POLL, REPLY, HANDSHAKE,
  HANDSHAKING, READY, CLOSED, CONNECT, DISCONNECT, MESSAGE, HANDSHAKE FAILED.
- Capitalised state and event names (READY, DISCONNECT) are abstract concepts bound to no language. Each implementation names them by its own conventions.

---

## 1. The whole lifecycle

```
   ┌────────────────────────────────────────────────────────────────┐
   │ PHASE 1 - BUILD THE PIPE (outside Fomoxa)                      │
   │ TCP connect / TLS handshake / WebSocket upgrade                │
   └────────────────────────────────────────────────────────────────┘
                              ↓ the pipe is open
   ┌────────────────────────────────────────────────────────────────┐
   │ PHASE 2 - FOMOXA HANDSHAKE §3                                  │
   │ greeting (carries the schema fingerprint) → 1-byte verdict     │
   │ (sometimes one query round in between - §3.3.2)                │
   │ state: HANDSHAKING                                             │
   │ the app has sent nothing yet                                   │
   └────────────────────────────────────────────────────────────────┘
                              ↓ verdict = accept
   ┌────────────────────────────────────────────────────────────────┐
   │ PHASE 3 - OPERATION §4 §5 §6                                   │
   │ state: READY                                                   │
   │ the app sends/receives messages · heartbeat runs in background │
   └────────────────────────────────────────────────────────────────┘
                              ↓ one of three
   ┌────────────────────────────────────────────────────────────────┐
   │ PHASE 4 - TERMINATION §7                                       │
   │ the app shuts down · the peer closes · the peer stays silent   │
   │ state: CLOSED                                                  │
   └────────────────────────────────────────────────────────────────┘
```

There are three session states, traversed in one direction with no way back:

```
   HANDSHAKE ──accept──> READY ──────────> CLOSED
       │                                     ↑
       └──reject / expiry────────────────────┘
```

---

## 2. Wire format

Nothing in this section may change. Any two implementations must produce and read exactly these bytes.

### 2.1 The frame envelope

Everything on the wire is a frame. Every frame starts with a type byte:

```
   ┌───────────────┬─────────────────────────────────┐
   │ 1 type byte   │ body, depends on the type       │
   └───────────────┴─────────────────────────────────┘

   Type byte values:
     0 DATA → the body is a data record (§2.2)
     1 POLL → no body. One byte is the whole frame.
     2 REPLY → no body. One byte is the whole frame.
     3 HANDSHAKE → body carries a length + opaque bytes (§2.3)

   Any type byte other than 0-3 is a violation.
```

### 2.2 The DATA frame

```
   ┌────┬─────┬─────┬──────────────┬───────────────┬─────────────────────┐
   │ 00 │ 46  │ 4F  │ message      │ payload       │ payload             │
   │    │ 'F' │ 'O' │ id           │ length        │                     │
   │ 1B │ 1B  │ 1B  │ 4B u32 LE    │ 4B u32 LE     │ = length bytes      │
   └────┴─────┴─────┴──────────────┴───────────────┴─────────────────────┘
    ^0   ^1    ^2    ^3-6           ^7-10           ^11 onward

   Total size = 11 + payload length
```

- The two bytes `0x46 0x4F` are a fixed marker. Either one wrong → the frame is corrupt.
- The message id is the message *type*, assigned by the application schema. It is not sequential and does not increase. Never use it to order or to deduplicate.
- The payload is opaque to transport and to the framing layer. Interpreting it belongs to the data-encoding layer, outside this document.
- Maximum payload length: 16 MiB (16 777 216 bytes). More than that must not be encoded, and must not be accepted while decoding.

### 2.3 The HANDSHAKE frame

```
   ┌────┬──────────────┬─────────────────────────────────┐
   │ 03 │ length       │ handshake body                  │
   │ 1B │ 4B u32 LE    │ = length bytes                  │
   └────┴──────────────┴─────────────────────────────────┘
    ^0   ^1-4           ^5 onward

   Total size = 5 + length
```

The body is opaque here; §3 explains it. Maximum length: 1 MiB. That is an allocation guard, not a semantic limit. A handshake body is control data and never needs to be as large as application data.

### 2.4 The POLL and REPLY frames

Exactly one byte: `0x01` and `0x02`. No body, no length, no id. They are transport control data. They are never application messages and they carry no message id. The whole id space belongs to the application.

### 2.5 Framing on a byte stream

On a byte stream, frames follow one another with no separator:

```
   ... │ frame │ frame │ frame │ ...
```

A frame is self-describing: read the type byte and you know how much more to read. The decoder must run incrementally: accept bytes in arbitrary chunks, retain the partial tail, and return a frame only when it is complete.

Any framing violation on a byte stream is fatal. There is no way to resynchronise. Every valid byte starts with a valid type byte. Anything else means the other side can no longer be trusted to delimit frames. Close the session.

Once the decoder reports an error it is permanently poisoned: every later call returns that same error. That makes "just read and see" impossible to write by accident.

### 2.6 Framing on packets

On a packet transport, one packet holds exactly one frame.

- The packet ends before the frame it declared → corrupt.
- Bytes remain after one frame has been read → corrupt.

A framing violation on a packet is not fatal: drop that packet and move on. Packets share no parse state, so one bad packet cannot corrupt later ones the way a bad chunk does on a byte stream.

The asymmetry is deliberate, and both sides must implement it correctly.

### 2.7 Ceiling table

| Quantity | Ceiling | Note |
|---|---|---|
| Message payload | 16 MiB | The same for every implementation |
| One DATA frame | 16 MiB + 11 bytes | |
| HANDSHAKE frame body | 1 MiB | Allocation guard |
| Messages declared in a greeting | 1 000 000 | Arithmetic guard, see §3.3 |
| Entries in one query | 1 000 000 | Never exceeds the greeting message count, see §3.3.2 |
| Query rounds per session | 1 | The handshake never runs more than twice |

---

## 3. Flow: greeting and fingerprint check

### 3.1 Overview

```
   CLIENT                                          SERVER
     │                                                │
     │  the transport is now open                     │ the transport is now open
     │  build the greeting at once                    │
     │                                                │
     │ ── HANDSHAKE (greeting) ─────────────────────> │
     │                                                │ ① check the framing
     │                                                │ ② check the version
     │                                                │ ③ compare fingerprints
     │                                                │
     │                            can you decide now? ┤
     │                                                │
     │ <── HANDSHAKE( verdict, 1 byte ) ───────────── │ Good - done here
     │                                                │
     │ <── HANDSHAKE( 4 = NEED MORE + list ) ──────── │ Not yet - §3.3.2
     │ ── HANDSHAKE (query answer) ─────────────────> │
     │                                                │ ③′ fill in the gap
     │ <── HANDSHAKE( verdict, 1 byte ) ───────────── │
     │                                                │
     │ 0 → READY                                      │ 0 → peer READY
     │ ≠0 → HANDSHAKE FAILED, close                   │ ≠0 → send the verdict, then close
```

Four facts shape this flow:

1. The client always greets and the server always rules; the client never rules. The exchange usually finishes in one round. Exactly one case forces the server to ask again before ruling (§3.3.2), and the exchange never takes more than two rounds.
2. The server is the only side that compares. The client sends its own schema and never sees the server's. Even when asked again, the client only answers the question and does not rule.
3. No byte separates the greeting from the query answer. Both ride in a HANDSHAKE frame; *role plus state* decides how to read it. Server reads the first payload as the greeting, the second one (if any) as the query answer. In the other direction the client reads the first byte: `0-3` is a verdict, `4` is a query.
4. The verdict is one byte, and that byte value is the rejection reason. There is no second lookup table. The value `4` is not a verdict but a request, so it does not end the session.

### 3.2 Greeting body

This is what sits inside the HANDSHAKE frame (§2.3):

```
   ┌────────────────────────────────────────────────────────────┐
   │ protocol version      4B u32 LE   always = 2               │
   │ schema fingerprint    8B u64 LE   of the whole message set │
   │ message count         4B u32 LE                            │
   ├────────────────────────────────────────────────────────────┤
   │ repeat "message count" times, 14 bytes per entry:          │
   │   message id          4B u32 LE                            │
   │   field count n       2B u16 LE                            │
   │   fingerprint h_n     8B u64 LE                            │
   └────────────────────────────────────────────────────────────┘

   The length must be exactly 16 + 14 × message count.
   One byte off = failure, without exception.
```

The three quantities have one role each:

- schema fingerprint: stands for the whole message set. A match means both sides were built from the same schema. Server accepts immediately and reads no entry.
- `n`: the field count. Without `n`, `k = min(n_client, n_server)` cannot be computed, and the condition of RFC-0002 §9.1 cannot be tested.
- `h_n`: the fingerprint of the whole message, that is the prefix fingerprint at `k = n`. The full prefix fingerprint chain still exists, but it stays in each side's static memory and never goes on the wire. Each side keeps its own chain, and only one value per message is sent. §3.3.1 explains why that is enough.

Fingerprint computation is defined by the Fomoxa schema specification, outside this document. For every message the application supplies a triple: message id, `n`, and the prefix fingerprint chain. Transport only compares; it never computes. Transport requires exactly two properties of that computation:

```
1. The kth prefix fingerprint depends only on the first k fields,
   in declaration order. Field k+1 onward must not
   influence it.

2. Two messages whose first k fields are identical must have the same
   kth prefix fingerprint - across every language, every implementation.
```

### 3.3 How the server decides

The server applies three gates in this order and stops at the first one that fails:

```
   receive the greeting body
        │
        ▼
   ① IS THE FRAMING CORRECT?
        │ shorter than 16 bytes?
        │ or length ≠ 16 + 14 × count?
        │ or count > 1 000 000?
        ├── no ──> verdict = 3 (BROKEN) ──> close
        ▼ yes
   ② IS THE VERSION 2?
        ├── no ──> verdict = 1 (WRONG VERSION) ──> close
        ▼ yes
   ③ COMPARE FINGERPRINTS
        │
        ├─ does the schema fingerprint match the server's?
        │  └── MATCH ──────────────────> verdict = 0, ACCEPTED
        │                                   (no entry was read)
        │
        └─ DIFFERENT → walk every entry in the greeting.
                  For each id the server also has, set
                      n_c = field count the client declared
                      n_s = field count the server has

                  ⓐ are both h_n equal?
                        └── YES ──> this message is identical, continue

                  ⓑ n_c == n_s (but h_n differs)?
                        └── YES ──> same length, different content
                                   → not a prefix
                                   → verdict = 2 ──> close

                  ⓒ n_c < n_s?
                        └── YES ──> compare the client's h_n against h_{n_c}
                                   from the server's local prefix chain
                                     ├── DIFFERENT ──> verdict = 2 ──> close
                                     └── MATCH ─────> prefix matches, continue

                  ⓓ n_c > n_s
                        ├── n_s == 0 ──> empty prefix, always matches, continue
                        └── n_s > 0 ──> undecided.
                                        Write into the query list: (id, n_s)

                  After the walk:
                     query list empty ──> verdict = 0, ACCEPTED
                     query list filled ──> send 4 = NEED MORE (§3.3.2)
```

Gate ③ is the prefix test of RFC-0002 §9.1, written in bytes. `k = min(n_c, n_s)` is the field count of the shorter side. Two equal kth prefix fingerprints mean the first `k` fields of both sides match in type and in order. That is exactly the definition of "the shorter sequence is an exact prefix of the longer one".

Three of the four branches resolve on the spot, because either the `h_n` the client sent is already the `h_k` needed, or the server can look up its own `h_k` in the local chain. Only branch ⓓ needs a value that only the client can produce.

The "count > 1 000 000" guard at gate ① must be checked before the `14 × count` multiplication, not after. On 32-bit integers a declared count near 2³² overflows the multiplication, the length check then passes, and a hostile greeting gets through.

Note: `n` takes no part in frame delimiting. Every entry is exactly 14 bytes, whatever `n` says. A wrong `n` only makes a comparison fail or produces an empty query. It never leads to an allocation sized by the number the client declared.

Different schemas can still be accepted.

The accept condition does not require identical schemas. It requires that, in every message both sides know, the fields both sides carry match:

| Situation | Result | Why |
|---|---|---|
| Schema fingerprints match | ✅ Accept | Same schema, nothing more to weigh |
| Client has one message the server does not know | ✅ Accept | The server never receives that message |
| Server has one message the client does not know | ✅ Accept | The client never sends that message |
| Shared message, one side appended a field at the end | ✅ Accept | Prefix match, a valid version difference (RFC-0002 §9.1) |
| Shared message, one side has `n = 0` | ✅ Accept | The empty prefix is a prefix of everything |
| Shared message, drift at an index inside the shared part | ❌ Reject (2) | The field at that index changed, which is a protocol change (RFC-0002 §5.1) |
| Shared message, one field renamed (type and order unchanged) | ❌ Reject (2) | A rename = delete + add at the same index; the fingerprint hashes field names, §3.3.1 |

Two sides running different schemas therefore interoperate as long as the intersection matches. It must match at the message-set level, and at the field-list level inside every shared message.

This lets each side upgrade gradually, with no system-wide stop and no dependency on the transport. RFC-0002 §9.1 lets both sides read each other when the shorter type sequence is an exact prefix of the longer one. RFC-0003 §8.6 pins that case as vectors V-001 and V-002: "must not be rejected". Those two vectors bind the decoder (RFC-0003 §1.3, the compatibility test). At the handshake layer, gate ③ deliberately preserves the append-at-the-end case through the prefix chain. That is the only reason the prefix chain exists (§3.3.1).

Interoperability is blocked only when both sides put two different fields at the same index inside the shared part. One position then carries two meanings, and since position is the only identifier on the wire, there is no safe way to interoperate. One mismatched message is enough to reject the whole session, even when every other message matches. There is no "partial accept": the session opens fully, or not at all.

One note on the nature of gate ③: it trusts the list the client declares about itself. A client that declares `message count = 0` passes gate ③ untouched. This is a compatibility check between two cooperating sides, not a security mechanism. Do not use it as one.

### 3.3.1 Why this is enough, and why branch ⓓ must ask

RFC-0002 §9.1 asks *whether the shorter sequence is an exact prefix of the longer one*, not merely whether two messages differ. One fingerprint for the whole message cannot answer that question:

```
     Item { id, name } → 0xA3F1…
     Item { id, name, hp } → 0x77B0… unrelated to the line above

   After hashing, the prefix relationship is gone.
```

The prefix fingerprint chain is needed for this reason. `h_k` commits to exactly the first `k` fields. `k = min(n_c, n_s)` is the §9.1 condition translated into bytes. Only one point needs comparing: drift at any index below `k` already makes `h_k` differ.

But the chain does not need to go on the wire. Each side keeps its own chain in static memory, and three of the four branches need nothing more from the other side:

```
   ⓐ h_n equal → identical. No k needed.
   ⓑ n_c == n_s → k = n_c = n_s, and h_n differs → reject. Nothing more needed.
   ⓒ n_c < n_s → k = n_c, and h^client_{n_c} is the h_n the client just sent.
                 The server looks up its own h_{n_c} in the local chain. Enough.
   ⓓ n_c > n_s → k = n_s. Needs h^client_{n_s} - only the client can produce it.
```

Branch ⓓ is unavoidable because of an information limit, not a design flaw. The client must answer at an index `n_s` it did not know when it sent the greeting. Covering every possible index in one round costs `O(field count)`, which means shipping the whole chain. No shortcut exists:

- a hash is one-way: you cannot derive `h_{n_s}` from `h_{n_c}`;
- switching to XOR or a running sum changes nothing: reversing it still needs exactly the fields you did not send;
- a Merkle tree gives a small commitment, but the *opening proof* is what must be sent. Covering every `k` costs more than the chain itself.

Exactly two options remain: ship the whole chain every time, or ship one value and ask again when branch ⓓ hits. A 14-byte-per-message greeting takes the second.

Branch ⓓ fires when the server has fewer fields than the client in a message that changed, regardless of which side upgraded first. Two paths lead there:

| | `n_c` vs `n_s` | Query needed? |
|---|---|---|
| Client moved first, appended a field | `n_c > n_s` | ✅ |
| Server moved first, deleted a field | `n_c > n_s` | ✅ |
| Server moved first, appended a field | `n_c < n_s` | ❌ branch ⓒ |
| Client moved first, deleted a field | `n_c < n_s` | ❌ branch ⓒ |

This is stricter than §9.1, and legitimately so. The §9.1 criteria look only at type and order; field names are not on the wire. The schema-layer fingerprint does hash field names (`fomoxa-fingerprint/2` §5; this is the mechanism that catches two same-typed fields swapped). Gate ③ therefore rejects a pair of peers that differ only by one rename, even though both sides produce identical bytes.

This is not a bug. Renaming a field is deleting the old field and adding a different one at the same index. It ranks with a mid-list insert or delete, which §9.1 classifies as a protocol change, not a version difference. §9.1 does not require accepting this case; it cannot see the case, because it measures drift by type, and two `f32` fields read out the same. That exact blind spot is handed upward by the "Scope of this section" clause: "Detecting and blocking such cases belongs to a higher schema layer, where field names and full declarations are available for comparison." Gate ③ performs that job.

Keep this boundary in mind, because the two layers follow different rules. The decoder must never reject a byte stream that §9.1 calls valid. RFC-0003 §1.3 files V-001/V-002 under the decoder's compatibility test. The handshake is allowed to be stricter, because it sees what the decoder cannot: field names and full declarations.

The price is written down in `SPEC-FINGERPRINT.md` §5, and it is a deliberate trade-off. A pure rename is blocked too, but that failure is immediate, visible and harmless. In exchange, two swapped same-typed fields never slip through, a failure that would be silent and would corrupt data permanently. Since `fomoxa-fingerprint/2`, field names are canonicalised before hashing: underscores, hyphens and spaces removed, uppercase folded down. Naming-convention differences between languages (`id`/`ID`/`Id`, `player_id`/`PlayerID`) therefore do not change the fingerprint. Only a real rename (`x` → `position_x`) does.

### 3.3.2 Query: verdict 4 and the answer round

Used only for branch ⓓ. The body rides in a HANDSHAKE frame, like everything else in this section.

Server → client. The first byte is `4`, so the client tells it apart from a verdict immediately:

```
   ┌────────────────────────────────────────────────┐
   │ 04                    1B          NEED MORE    │
   │ entry count           4B u32 LE                │
   ├────────────────────────────────────────────────┤
   │ repeat, 6 bytes per entry:                     │
   │   message id          4B u32 LE                │
   │   requested index n_s 2B u16 LE   (always ≥ 1) │
   └────────────────────────────────────────────────┘

   The length must be exactly 5 + 6 × entry count.
```

The server may only ask about ids the client declared in the greeting, and only when `1 ≤ n_s < n_c`. A query that breaks this rule is a server bug.

Client → server. No distinguishing byte: the server expects exactly one payload and reads it as the query answer (§3.1, item 3).

```
   ┌──────────────────────────────────────────────────────────────────┐
   │ entry count           4B u32 LE                                  │
   ├──────────────────────────────────────────────────────────────────┤
   │ repeat, 12 bytes per entry:                                      │
   │   message id          4B u32 LE                                  │
   │   prefix fingerprint  8B u64 LE   at exactly the requested index │
   └──────────────────────────────────────────────────────────────────┘

   The length must be exactly 4 + 12 × entry count.
   Answer every requested entry, in full and in order.
```

The server then compares: for each entry, the fingerprint the client sent must equal `h_{n_s}` in the server's local chain. A missing entry → verdict 2. All entries match → verdict 0.

Three rules prevent any additional state:

```
1. At most one query per session. After the answer arrives, the server must rule; no further questions.
2. Client receives a second byte 4 → treat as HANDSHAKE FAILED.
3. The client handshake deadline (§3.5) covers the whole process. It must not reset per round. Two rounds share one deadline.
```

On a packet transport those two extra query packets can be lost. If they are, the handshake expires and fails: the session does not open and the app retries. This is not a new failure mode. Fomoxa does not retransmit (RFC-0001 §2), so every handshake over an unreliable transport is best-effort. In exchange, a 14-byte-per-message greeting usually costs two packets, and no large greeting has to be split on every connection.

To drop the extra round trip, try optimistically first: if the handshake fails, the next connection sends the prefix chain directly instead of waiting for a query. That is an implementation choice, not a requirement of this document.

### 3.4 How the client reads the verdict

```
   receive the payload from the server
        │
        ▼
   first byte == 4?
        │
        ├── yes ──> QUERY (§3.3.2)
        │            │ has a query arrived once already?
        │            │ └── yes ─────> treat as HANDSHAKE FAILED
        │            │ is the query frame valid?  are the ids in the greeting?
        │            │ └── no ──────> treat as a corrupt frame, fail
        │            └──> send the query answer, then keep waiting
        │                 (the deadline does not reset)
        │
        └── no ───> VERDICT
                       │ exactly 1 byte?  and the value ≤ 3?
                       │ └── no ──────> treat as HANDSHAKE FAILED
                       ▼ yes
                    byte == 0?
                       ├── yes ──> READY.  From here the app may send.
                       └── no ───> HANDSHAKE FAILED, reason = the byte value itself
                                   → the session closes at once, the app gets no extra event
```

The client never judges a schema itself, not even when asked. It only looks up its local prefix chain at the requested index and sends the value back. The final decision still belongs to the server (§3.1, item 2).

Reason table (byte value = reason):

| Byte | Meaning | Real cause |
|---|---|---|
| 0 | Accept | |
| 1 | Wrong protocol version | The two sides run different versions of the handshake frame: the greeting layout changed, for example one side still sends v1. Unrelated to the schema, and not "two versions of Fomoxa" |
| 2 | Schema conflict | In a message both sides know, the two put different fields at the same index inside the shared part: a mid-list insert, a mid-list delete, a reorder, a type change, or a rename (a rename = delete + add at the same index, §3.3.1). Fields appended at the end do not belong here (§3.3.1) |
| 3 | Broken greeting | A bug on the sending side, or something that is not Fomoxa knocking on this port |

The value `4` is absent from the table: it is not a verdict but a request (§3.3.2), and it never ends the session. Any value from `5` upward is corrupt.

### 3.5 Handshake deadline: the two roles count differently

The two roles use completely different mechanisms:

```
   CLIENT                                           SERVER
   ──────                                           ──────
   HARD DEADLINE
   The clock runs from session creation.            No absolute deadline.
   The handshake deadline passed unfinished.        Only counts "how long has it been silent".
   → fail, terminate.                               Is the peer still answering probes?
                                                    → it has not expired yet.
```

In practice, a client that keeps answering probes without ever sending a valid greeting holds a slot on the server indefinitely. This behaviour is deliberate. The specification sets no absolute ceiling; adding one departs from the design.

The client deadline covers the whole handshake, query rounds included (§3.3.2). The clock does not reset per round; two rounds share one deadline. Resetting it opens the door for a hostile server to stretch the session by asking forever.

### 3.6 Other frames arriving during the handshake

Frames other than HANDSHAKE can arrive before the handshake finishes. None of them is a violation:

| Incoming frame | Client (HANDSHAKING) | Server (peer HANDSHAKING) |
|---|---|---|
| DATA | Drop it, never hand it to the app | Drop it, but count it as activity |
| POLL | Still answer REPLY, raise no event | Answer REPLY, count it as activity |
| REPLY | Ignore | Count it as activity |
| A second HANDSHAKE (after the verdict) | Ignore | Ignore |

Do not confuse the last row with the query round. During HANDSHAKING a second HANDSHAKE frame is normal and meaningful: for the client it is a query, for the server it is the query answer (§3.3.2). Only HANDSHAKE frames arriving after the verdict are dropped. Each session allows at most one query round; a third HANDSHAKE frame during the handshake is corrupt.

Why must the client still answer REPLY before READY? The server may probe the client before it answers a valid greeting. If the client stays silent because "the handshake is not done", the server concludes it is dead and cuts it off while it is still waiting for the server's answer. Answering REPLY here is what gives the process a chance to finish.

---

## 4. Flow: when the heartbeat runs

### 4.1 Principle: probe on silence, not on a fixed interval

This differs from the usual approach:

```
   ❌ THE USUAL WAY                    ✅ FOMOXA
   send 1 probe every 5 seconds        probe only after 5 seconds of silence
   even while data is flowing          data flowing = the peer is known alive
```

A peer that is sending receives no extra probe. On a system running 60 ticks per second the heartbeat produces almost no packets.

### 4.2 The heartbeat state machine

Each side measures its own listening direction: the client measures how long the server has been silent, the server measures how long the client has been silent. Never share one clock across both directions.

```
        ┌────────────────────────────────────────────────┐
        │ NORMAL                                         │
        │ count: how long it has been silent             │
        └────────────────────────────────────────────────┘
             │                                    ▲
             │ silence ≥ silence window           │ any valid frame
             │ → send exactly one probe           │ arrives
             ▼                                    │ → clear it, go back
        ┌────────────────────────────────────────────────┐
        │ PROBING                                        │
        │ count: how long since the probe, still nothing │
        └────────────────────────────────────────────────┘
             │
             │ ≥ reply deadline and still nothing back
             ▼
        ┌────────────────────────────────────────────────┐
        │ PEER DEAD                                      │
        └────────────────────────────────────────────────┘
```

Two things follow from the diagram:
- Nobody cuts at the end of the silence window. A probe always comes first. Worst-case death detection time = `silence window + reply deadline`.
- Every valid frame counts as a sign of life: DATA, POLL, REPLY, HANDSHAKE. It does not have to be a REPLY.

### 4.3 When the heartbeat starts: the two roles differ

```
   CLIENT
   ──────
   HANDSHAKE: no heartbeat at all.
                  the handshake has one hard deadline and nothing else (§3.5).
                  During this phase the server is the side that watches
                  whether the connection is alive.
                     ↓
   READY: the heartbeat starts exactly here.
                  silence window = heartbeat interval
                  reply deadline = heartbeat deadline


   SERVER (per peer)
   ─────────────────
   HANDSHAKE: the heartbeat runs from the moment the peer appears.
                  silence window = handshake deadline ← wider
                  reply deadline = heartbeat deadline
                     ↓
   READY: only the silence window changes, nothing else is reset.
                  silence window = heartbeat interval ← narrower
                  reply deadline = heartbeat deadline ← unchanged
```

Why does the client handshake carry no heartbeat? The client just sent the greeting and waits for exactly one thing. Nothing needs periodic refreshing, and it already has a hard deadline. A second mechanism is redundant.

Why does the server carry one? A server may hold many half-open peers and must tell "a slow client" apart from "a client that evaporated". Only a probe tells them apart.

Note how the server moves to READY: it only changes the silence window. It does not touch the last-activity mark and does not cancel a probe in flight. A probe sent just before the greeting arrived stays valid and is still waiting for an answer.

### 4.4 The three time parameters

| Parameter | Default | Meaning |
|---|---|---|
| Handshake deadline | 5 s | Client: the deadline for the whole handshake. Server: the silence window while the peer has not finished the handshake |
| Heartbeat interval | 5 s | How long of silence before one probe goes out |
| Heartbeat deadline | 15 s | After a probe, how long without an answer counts as dead |

Worst-case time to detect a dead peer in READY: 5 + 15 = 20 seconds.

These are the recommended defaults, and the app may reconfigure them, but the relationship between them must not be inverted. The reply deadline must be large compared to the round-trip latency. Otherwise a healthy peer is declared dead.

### 4.5 The whole probe cycle

```
   A                                               B
   │                                               │
   │ ... both sides exchange DATA normally         │
   │ every frame received resets A's clock         │
   │                                               │
   │ ... B stops sending ...                       │
   │                                               │
   │ A's silence clock runs: 1s 2s 3s 4s 5s        │
   │                                               │
   │ ── POLL ────────────────────────────────────> │ ① exactly once,
   │ A moves to the PROBING state                  │    never repeated
   │                                               │
   │ <──────────────────────────── REPLY ───────── │ ② B is alive
   │ A clears the probe state, back to normal      │
   │ A's app receives the REPLY event              │
   │                                               │
   │ ... or B really is dead ...                   │
   │                                               │
   │ 15 seconds pass, nothing comes back           │
   │ A declares B dead → DISCONNECT event          │ ③
```

At ① there is exactly one probe per window, not one per tick. At ②, any frame from B is enough to clear the probe. It does not have to be a REPLY; DATA from B works too.

---

## 5. Flow: sending a message

### 5.1 The full path

```
   APP
    │ send(message id = 42, payload = 100 bytes)
    ▼
   ① CHECK THE STATE
    │ not READY? → return "not ready" at once, send nothing
    ▼
   ② PACK INTO A FRAME (§2.2)
    │ [00] ['F'] ['O'] [42 u32 LE] [100 u32 LE] [100 payload bytes]
    │  1    1     1     4           4            100 = 111 bytes
    │ the payload is copied here → the app may release it as soon as
    │ the send call returns
    ▼
   ③ IS THE OLD FRAME STILL STUCK?
    │ yes → return "congested", do not queue
    ▼ no
   ④ HAND IT TO THE TRANSPORT
    │
    ├── ✔ done, nothing is retained
    ├── ⏸ copy into the pending slot, return success to the app
    ├── ⊘ return "too large" to the app, the session stays alive
    └── ✖⚠ mark the transport dead
```

### 5.2 Why sending before READY is forbidden

Before the verdict, nobody has confirmed that both sides read the same bytes. A message sent early may reach a server that rejects the schema a moment later. That server interprets it against its own schema and reads out a value different from what the client sent. This check is the only point where such a message can be stopped.

### 5.3 When a stuck frame goes out

```
   tick N   send frame ──────────────────────> transport: ⏸ → into the slot
              the app keeps running normally

   tick N+1 first job: flush the slot ───────> transport: ⏸ → still stuck
              (if the app sends again in this tick → "congested" error)

   tick N+2 first job: flush the slot ───────> transport: ✔ → through
              sending is normal again from here
```

"Old work first" is mandatory, not an optimisation. A new frame reaching the wire before the remainder of the old one makes the decoder on the other side read two frames glued together and corrupts it permanently (§2.5).

---

## 6. Flow: receiving a message

### 6.1 The full path

```
   TRANSPORT
    │ hands up one complete frame
    ▼
   ① UNPACK (§2)
    │ read the first byte → frame type
    │ DATA, then check the two marker bytes 'F' 'O'
    │ wrong → corrupt frame (handle per §2.5 or §2.6, by transport kind)
    ▼
   ② FEED THE SESSION STATE MACHINE
    │
    ├─ HANDSHAKE → handle the verdict (§3.4) or the greeting (§3.3), by role
    │
    ├─ POLL → answer REPLY at once + (if READY) raise a POLL event
    │
    ├─ REPLY → (if READY) clear the probe state + raise a REPLY event
    │
    └─ DATA → not READY? drop it, never hand it to the app
              READY?     reset the silence clock
                         raise a MESSAGE event
    ▼
   ③ APPEND TO THE EVENT LIST
    ▼
   THE APP reads the list after tick returns
```

Note at ②: the REPLY to a POLL goes out on the spot, without waiting for the app. The app may never read a POLL event and the heartbeat still works correctly.

### 6.2 Lifetime of event data

```
   tick N
     ├── frame received, the event references an internal buffer
     ├── the app reads the event, uses the data ← valid here
     └── tick ends

   tick N+1
     └── the buffer is overwritten ← the old reference now points at garbage
```

Rule: *data inside an event lives only until the next tick of the same session*. Data needed for longer must be copied out. This is the rule for languages that hand out non-owning references.

An implementation that copies data into the app, or hands over an owned object, is not bound by this restriction, but must state that clearly in its own documentation. Programmers moving between implementations always carry the old assumption with them.

---

## 7. Flow: ending a session

A session can end by three paths, which differ in who notices first:

```
   ① THE APP SHUTS DOWN
      the app calls disconnect → core marks itself closed → asks the transport to soft close
      → session CLOSED. No extra event is produced.

   ② THE PEER CLOSES OR FAILS
      the transport returns ✖ or ⚠ when core asks
      → core marks the transport dead
      → if the session is not CLOSED: raise DISCONNECT

   ③ THE PEER GOES SILENT
      the transport is perfectly healthy and reports nothing
      → the heartbeat clock expires during the tick
      → raise DISCONNECT
```

Exactly one termination event per session. If the handshake failed and already raised HANDSHAKE FAILED, no DISCONNECT follows, even when the transport dies right afterwards. A flag is needed to guarantee this. Apps usually clean up inside the termination handler, and cleaning up twice is a bug.

---

## 8. Implementation guide: the core side

The shape of one tick. This order is mandatory, not advisory.

```
   tick(current_time):

       ── 0. CONNECT event, once per session lifetime ──
       if the connection has not been reported:
           push the CONNECT event
           mark it reported

       ── 1. flush the stuck frame ── must run before anything else
       while the pending slot holds data:
           result = transport.send(the remainder)
           ✔ → clear the slot, exit the loop
           ⏸ → exit the loop, leave it for the next tick
           error → transport_dead = true, exit the loop

       ── 2. drain incoming data, within the budget ──
       if the transport is not dead:
           budget = LIMIT
           while budget remains:
               result = transport.receive(buffer)
               ⏸ → exit the loop
               ⤢ → grow the buffer, do not spend budget, retry
               ✖⚠ → transport_dead = true, exit the loop
               ✔ → unpack into a frame
                     feed the session state machine
                     apply the result (see below)
                     spend one unit of budget

       ── 3. run the protocol clocks ──
       if the transport is not dead:
           result = session.tick(time)   handshake deadline · probe · declare dead
           apply the result

       ── 4. the transport died but the session is not closed ──
       if the transport is dead and the session is not CLOSED:
           result = session.report_transport_close()
           apply the result

       ── 5. hand the event list to the app ──
       return the list
```

`apply the result` - the session state machine returns at most one frame to send and at most one event:

```
   apply(result):
       if there is a frame to send:
           if the pending slot holds data:
               queue this frame after it ← never overwrite
               if the pending queue exceeds its ceiling → transport_dead = true
           else:
               write that frame to the transport
               ⏸ → park it in the pending slot
               ✖⚠ → transport_dead = true

       if there is an event:
           push it into the list
           if HANDSHAKE FAILED:
               transport_dead = true
               ask the transport to soft close
```

"At most one frame and at most one event" reflects exactly what the protocol produces. The most complex case is "answer, then end": the server sends a reject verdict and closes the session. The query round (§3.3.2) does not break the invariant: both the query and the query answer are frames with no event attached. The app sees nothing until the final verdict. To the app, a two-round handshake and a one-round handshake are the same thing.

Three points in the block above need explicit mention, because missing any one of them causes a silent bug.

The pending slot must never be overwritten. The frames involved (POLL, REPLY, the handshake verdict) are generated by core itself and do not come through an application send (unlike the scenario in `01_overview.md` §5). There is nobody to report "congested" to. Overwriting the slot kills the reject verdict: the peer never learns why it was rejected and sits waiting until its deadline.

The pending queue must have a ceiling, and that ceiling must be low. The protocol itself caps how many control frames can live at once: exactly one probe per silence window (§4.5), one REPLY per POLL received, at most one query round per session (§3.3.2). Exceeding a small number means an assumption broke, not that load is high. The exact number is per implementation, but having one is mandatory.

Hitting the ceiling ends the session; it does not throw. Invariant B6 requires exactly one termination event per session, and DISCONNECT fits that shape. Throwing an exception out of the tick does not: it breaks the promise that "tick returns a list of events". A server iterating many peers would lose the remaining peers in that same tick.

Core has four fixed rules:

1. Never wait. Core asks the transport, takes the answer and moves on. No loop here may go around again just to "retry immediately". Retrying is the next tick's job.
2. The session state machine must not know the transport exists. It takes frames and returns frames and events, with no connections, no OS errors and no clock reads of its own.
3. Every timestamp is passed in from outside. This is what makes a whole expiry cycle testable without real waiting. It must hold at every layer.
4. One termination event, no more. The "termination reported" flag must be checked on every path that leads to a termination event.

### Clocks

Time must come from a monotonic clock: increasing only, never adjusted. Wall clocks are forbidden, because one backward time sync is enough to expire a handshake or to kill a live peer.

---

## 9. Implementation guide: the transport side

### 9.1 Minimum state to keep

```
   transport state:
       connection      the connection object of the library you use
       inbound_queue   what was read but not yet handed to core
       outbound_queue  the part that could not be pushed out (if any)
       closed          so every later call consistently returns ✖
```

### 9.2 The SEND function

```
   send(bytes, length):
       if closed: return ✖
       if length > my_ceiling: return ⊘ ← not ⏸
       try to push into the underlying connection, do not wait
           pushed everything → return ✔
           could not push → return ⏸ ← having taken no byte at all
           peer closed cleanly → return ✖
           error → return ⚠
```

When the transport returns ⏸, it must have taken no byte. Taking half and then returning ⏸ means core resends from the start, and the other side receives the first half twice.

If the underlying library can take a partial write (a raw connection), there are two options: declare the transport as byte-stream kind and let core handle it, or buffer the remainder into `outbound_queue` and return ✔. Returning ⏸ halfway is forbidden.

### 9.3 The RECEIVE function

```
   receive(buffer, capacity):
       if closed and inbound_queue is empty: return ✖

       if inbound_queue is empty:
           read from the underlying connection, do not wait
               nothing → return ⏸
               peer closed cleanly → closed = true, return ✖
               error → closed = true, return ⚠
               data → store it in inbound_queue

       (packet transports only)
       if the head packet has not arrived in full: return ⏸

       if the head packet > capacity:
           return ⤢ with the byte count needed
           ← the packet stays queued, do not discard it

       copy the head packet into the buffer
       remove it from the queue
       return ✔
```

On the ⤢ branch, losing the packet is forbidden. Core grows the buffer and asks again immediately. Discarding it loses that data permanently and nobody finds out.

### 9.4 The SOFT CLOSE and HARD CLOSE functions

```
   soft_close():
       if already closed: fine, do nothing
       send the underlying protocol's close signal
       DO NOT wait for the other side to answer
       DO NOT release anything - data may still be arriving and waiting to be read

   hard_close():
       close the underlying connection
       release the queues, the buffers, everything you own
       ← called exactly once. After this call no other function is called.
```

The two functions differ: soft close is *being polite to the peer*, hard close is *cleaning house*. Between the two calls, core may still ask for data still in flight.

### 9.5 Self-check before closing the review

- [ ] No function waits, sleeps, or blocks with a timeout > 0
- [ ] No function spawns a hidden thread
- [ ] No function calls back into core
- [ ] Send: returns ⏸ only when no byte was taken
- [ ] Send: returns ⊘ (not ⏸) above the ceiling
- [ ] Receive (packet kind): never returns a partial packet
- [ ] Receive: the ⤢ branch keeps the packet, never discards it
- [ ] ✖ (clean close) and ⚠ (break/error) are reported correctly
- [ ] Hard close releases everything, and is safe on a half-built state
- [ ] Calling close twice does not fail (even though core promises to call it once)
- [ ] Transport with a data-type distinction: use binary, never text
- [ ] No byte of your own is added to the data

---

## 10. Quick reference: where each event comes from

| Event the app receives | Raised when | At which tick step |
|---|---|---|
| CONNECT | On the very first tick, always | 0 |
| READY | Verdict = 0 came back | 2 |
| HANDSHAKE FAILED | Verdict is 1, 2, 3; or the verdict is corrupt; or the handshake expired | 2 or 3 |
| MESSAGE | A DATA frame arrived while READY | 2 |
| POLL | A POLL frame arrived while READY (the REPLY was already sent automatically) | 2 |
| REPLY | A REPLY frame arrived while READY | 2 |
| DISCONNECT | The transport reported ✖/⚠, or the heartbeat expired | 3 or 4 |

CONNECT is always the first event, even over UDP, where there is no connection to report. There it is generated artificially, so the app learns one vocabulary for every transport.

The server side has a matching event set per peer: PEER CONNECTED, PEER READY, PEER HANDSHAKE FAILED, PEER DISCONNECTED. They follow the rules above, with the only difference that each event carries the peer identity.

---

## 11. Minimum test checklist

An implementation counts as correct when it passes all of these. They are language-independent and must exist in every implementation.

### Wire format
- Encode and decode every frame kind, byte for byte against the tables in §2
- Decode a byte stream fed one byte at a time; the same frames must come out
- Several frames glued into one feed must split correctly
- An invalid type byte: fatal on a byte stream, drop the packet only on a packet transport
- Packets missing bytes and packets with extra bytes are both corrupt
- Data exactly at the ceiling passes; one byte over the ceiling is rejected

### Handshake
- Valid greeting, identical schemas → accept, no entry is read
- Different schemas but a matching intersection → accept

The four branches of gate ③, at least one case each:

- ⓐ both `h_n` equal → accept
- ⓑ `n` equal but `h_n` differs → reject with reason 2
- ⓒ client has fewer fields, prefix matches → accept, no query sent
- ⓒ client has fewer fields, wrong prefix → reject with reason 2, no query sent
- ⓓ client has more fields → the server must send verdict 4
- ⓓ with `n_s = 0` → accept, no query sent

Two cases are forgotten most often, and they are the reason for this design:

- Client appended a field at the end → branch ⓓ → query → accept
- Server deleted a field at the end → also branch ⓓ → query → accept

Both are an append or a truncation at the end, the two cases pinned as V-001/V-002 in RFC-0003 §8.6. Gate ③ preserves that behaviour at the handshake layer through the prefix chain. Rejecting this case is an error, not extra strictness. The rename case is the opposite: rejected on purpose (§3.3.1).

Query round:

- Client answers the right fingerprint at the requested entry → verdict 0
- Client answers a wrong fingerprint → verdict 2
- Client answers with entries missing, extra, or out of order → verdict 3
- Server sends a second query → the client treats it as HANDSHAKE FAILED
- A query asks for an id absent from the greeting → the client treats it as HANDSHAKE FAILED
- The client handshake deadline does not reset across the query round

Framing and limits:

- Shared message, two same-typed fields swapped → reject with reason 2. This is the case a type-and-order-only fingerprint would miss; it is caught because field names are inside the fingerprint (§3.3.1)
- Shared message, one field renamed, type and order unchanged → reject with reason 2 (§3.3.1)
- Shared message, only the naming convention differs (`player_id` ↔ `PlayerID`) → accept, the fingerprint is unchanged (`fomoxa-fingerprint/2` §3.2)
- Wrong version (a v1 greeting meets a v2 server) → reject with reason 1
- Length off by one byte → reject with reason 3
- Length ≠ `16 + 14 × message count` → reject with reason 3
- A very large declared message count → reject, no arithmetic overflow, no large allocation
- A verdict value ≥ 5 → the client treats it as HANDSHAKE FAILED
- Client: the handshake expires → failure
- Server: the peer keeps answering probes without sending a greeting → no expiry

### Heartbeat
- Continuous traffic → no probe is sent
- Silence for a full window → exactly one probe, not one per tick
- An answer arrives → back to normal
- No answer within the deadline → declare it dead
- Any frame, not just REPLY, clears the probe state
- Run a whole expiry cycle without real waiting, which proves time is a parameter

### Transport and flow
- A fake transport that always returns ⏸ → the frame sits in the slot, resent in order, never duplicated
- Sending again while stuck → "congested", never queued
- ⤢ → grow the buffer and get that exact packet back, nothing lost
- ⊘ → error to the app, the session stays alive, core does not retry
- A burst of incoming data → stop at the ceiling, take the rest next tick
- Handshake failed and the transport died → exactly one termination event
- The receive queue of a packet transport fills → the oldest packet is gone, the newest is kept
- A fake transport that always returns ⏸, plus a peer sending POLL every tick → the pending queue stops at its ceiling, exactly one termination event, nothing thrown out of the tick
- Many unknown source addresses sending to one UDP endpoint → the peer table stops at its ceiling, running sessions are unaffected

### Interoperability
- Two implementations in two different languages interoperate in both directions (the primary criterion)
- Over a byte-stream transport and over a packet transport alike
