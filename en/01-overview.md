# Fomoxa transport: model and responsibilities

This document describes the Fomoxa transport model at the conceptual level. It is independent of any programming language and references no implementation, API or source code.

This document together with `02_flows.md` is enough to rebuild Fomoxa, either as a new implementation or as a new transport, in any language, without reading the source of an existing implementation.

| Document | Content | Read it when |
|---|---|---|
| `01_overview.md` (this one) | Model, layer boundaries, split of responsibilities | You want to understand the design |
| `02_flows.md` | Wire format, every flow in detail, implementation guide | You start writing code |

Each language may ship its own API binding document. Those documents describe *how to call*, not *how it behaves*. Behaviour is defined in these two documents and must be identical in every implementation.

---

## 1. Three layers

```
  APP               sends/receives messages that mean something to the app
      |
  ─────────────────────────────────────────────
      |
  CORE              handshake, heartbeat, peer death detection,
                    frame packing/unpacking, event handling
      |
  ─────────────────────────────────────────────
      |
  TRANSPORT         moves bytes from one machine to another, nothing more
                    TCP / UDP / WebSocket / TLS / QUIC ...
```

The basic boundary is this: core does not know how the data travels, and transport does not know what the data contains. Transport never sees a handshake, a heartbeat or peer state, and cannot tell a greeting apart from a character position update. To transport, all data is one block of bytes to push to the other side.

In the other direction, core does not know, and must not know, whether it runs over TCP or WebSocket.

---

## 2. The four functions a transport must provide

A transport provides exactly four functions:

| Function | Meaning | Caller |
|-----------|-------|--------|
| Send | Take a block of bytes, push it to the other side | Core, when it has something to send |
| Receive | Return data that arrived from the other side, if any | Core, every tick |
| Soft close | Tell the other side we are about to disconnect | Core, when the app asks to disconnect |
| Hard close | Release every resource | Core, when the session is destroyed |

There is no fifth function: no reconnect, no wait until the send succeeds, no query for remaining buffer space.

### Four shared answers

For both send and receive, transport answers with the same vocabulary:

```
  ✔  Done                    → core moves on
  ⏸  Not right now           → core holds it, asks again next tick
  ✖  I am closed             → core ends the session
  ⚠  I failed                → core ends the session
```

### Two special answers, one per direction

The four answers above do not cover two cases that are neither success, nor a temporary refusal, nor the end of the session:

```
  SEND direction
  ⊘  This block is larger than I can carry
     → does not kill the session. Core reports the error up; the connection stays alive.
     → Core does not retry. (§7)

  RECEIVE direction
  ⤢  Data is here, but your buffer is too small. I need this many bytes.
     → Core grows the buffer and asks again. No data is lost. (§7)
```

⊘ must stay separate from ⏸ because the two signals mean different things. ⏸ means that a later retry will succeed. If transport returns ⏸ for an oversized payload, core keeps the payload and retries every tick, but the payload is still oversized on every later tick, so the loop never ends and the send channel jams permanently. ⊘ means the opposite: *do not retry; this payload will never go out*.

⤢ also belongs to the group where a retry will succeed, but it carries a number: core must grow the buffer before asking again. It therefore cannot be folded into ⏸ either.

### Prohibition number one

Transport has no answer meaning "let me wait", and this is prohibition number one of the design. Fomoxa runs inside a timed loop, so a transport that waits stops the whole loop. Transport reports state *at the instant it is asked* and returns control; retrying is core's job.

---

## 3. Two kinds of transport

There are exactly two kinds, and a transport must declare which kind it is.

### Byte-stream kind

Bytes flow continuously with no boundaries. If the sender writes three 100-byte blocks, the other side may receive one 300-byte block or seven small ones, and nothing indicates where one unit ends.

→ TCP, TLS, QUIC stream

### Packet kind

Transport keeps the boundaries itself. If the sender writes three blocks, the other side receives exactly three, with correct boundaries and never glued together.

→ UDP, WebSocket, QUIC datagram

### Why the distinction matters

Data from a byte-stream transport needs reassembly, and core does that job. When a transport declares itself byte-stream, core inserts a framing layer in between, so the transport author never needs to know how Fomoxa frames data.

```
  Packet transport               Byte-stream transport
  ────────────────               ─────────────────────
        CORE                              CORE
          |                                 |
          |                        [ framing layer ]   ← core handles it
          |                                 |
      WebSocket                            TCP
```

As a result, the author of a WebSocket or TLS transport never reads the wire format.

The framing layer must sit between core and transport, inside neither. Inside transport, transport would have to understand the frame format, which is the wrong layer. Inside core, core would have to check the transport kind on every call, which is also the wrong layer. Between the two, the branch happens once at setup, and neither side contains the other's logic.

### Return obligations: the two kinds differ completely

The return obligations of the two kinds are easy to confuse:

| | Byte-stream kind | Packet kind |
|---|---|---|
| Per return | Any number of bytes | Exactly one whole packet |
| Return a single byte? | Legal | Illegal |
| Return half a frame? | Legal | Illegal |
| Return 2.5 frames glued? | Legal | Illegal |
| Packet still incomplete? | Return what you have | Must answer ⏸ and hold it |
| Who reassembles/buffers? | Core's framing layer | The transport itself |

A byte-stream transport must not try to return exactly one frame, because it lacks the information; doing so would require parsing the wire format, which is the wrong layer. It returns raw bytes as they arrive.

A packet transport has the opposite obligation: returning a partial packet is forbidden. If only part of a new packet has arrived, it answers ⏸, and the partial data must not reach core.

---

## 4. Flow: first connection

```
  APP                    CORE                          TRANSPORT
   |                      |                                |
   |  "connect to X"      |                                |
   |─────────────────────>|                                |
   |                      |                                |
   |                      | (transport was already built   |
   |                      |  and connected externally)     |
   |                      |                                |
   |                      | create session, build greeting |
   |                      |                                |
   |                      | send greeting ────────────────>|
   |                      |<─────────── ✔ or ⏸             |
   |<─── done ────────────|                                |
   |                      |                                |
```

The transport must already be open before it is handed to core. The TCP handshake, the TLS handshake and the WebSocket upgrade all happen outside core, beforehand, and the transport author owns them.

### Who creates the transport

Core never creates a transport itself. Creating and connecting belongs to the extension library, not to the app and not to core:

```
    APP             WEBSOCKET LIBRARY             CORE
      |                       |                         |
      | "connect to           |                         |
      |  wss://example/ws"    |                         |
      |──────────────────────>|                         |
      |                       | open TCP                |
      |                       | upgrade the protocol    |
      |                       | wait for confirmation   |
      |                       | → the pipe is open      |
      |                       |                         |
      |                       | wrap as transport ─────>|
      |                       |                         | create session
      |                       |<──── session handle ────|
      |<──── session handle ──|                         |
      |
      | from here the app uses it exactly like a TCP session
```

The library should wrap this in a convenience function, for example "connect over WebSocket, return a session". Ideally the app never sees the transport concept and only picks which library to call.

### Two independent handshakes

Core receives an open pipe and only then runs Fomoxa's own handshake over it:

```
  transport handshake            Fomoxa handshake
  (TCP, TLS, WebSocket)   →      (greeting / verdict)
  ↑ outside Fomoxa              ↑ core's job, spans many ticks
```

The Fomoxa handshake is not finished when `connect` returns and continues across later ticks. The app learns that it finished when it receives the READY event. Until then every send is rejected, so no application byte goes on the wire before both sides agree.

---

## 5. Flow: sending data

### The normal case

```
  APP                    CORE                          TRANSPORT
   |                      |                                |
   |  "send this message" |                                |
   |─────────────────────>|                                |
   |                      | pack into a frame              |
   |                      | ───────── send ───────────────>|
   |                      | <──────── ✔ ──────────────────-|
   |<─── done ────────────|                                |
```

### When the transport is congested

This flow differs from a conventional blocking send:

```
  APP                    CORE                          TRANSPORT
   |                      |                                |
   |  "send this message" |                                |
   |─────────────────────>|                                |
   |                      | ───────── send ───────────────>|
   |                      | <──────── ⏸ not now ──────────-|
   |                      |                                |
   |                      | park it in the pending slot    |
   |<─── done ────────────|                                |
   |                      |                                |
   |  ... the app keeps running, never blocked ...         |
   |                      |                                |
   |  next tick           |                                |
   |─────────────────────>|                                |
   |                      | ─ resend the remainder ───────>|
   |                      | <──────── ✔ ──────────────────-|
```

From this flow:

1. The app never blocks. Send returns immediately, even when the transport is full.
2. Transport never waits. It answers "not now" and returns control.
3. Core remembers and retries. On every tick, the first job is to flush the leftover, before anything else.

Wire order is absolute. A new frame injected into a half-sent frame permanently corrupts the decoder on the receiving side, so sending a new frame while one is pending is forbidden.

### When congestion persists

The pending slot holds one frame, for one session, in the outbound direction. In detail:

- One session is one logical send channel. This layer has no concept of parallel streams.
- The receive direction has no pending slot in core, which does not mean data is dropped. See §6.
- The one-frame limit applies to application sends. Control frames that core generates (POLL, REPLY, handshake verdict) queue *after* the pending slot, never overwrite it, and have their own ceiling, see `02_flows.md` §8.
- Connections with substreams (QUIC): the transport interface exposes exactly one send/receive channel pair. A transport that wants substreams multiplexes them *internally* and keeps the order correct while doing so. Core sees one pipe, and the one-pending-frame rule applies to that logical channel, not to each substream.

If the previous frame has not gone out and the app sends again, the send is rejected with a "congested" signal. Fomoxa does not queue without bound. When the peer stops reading, the app must be told and must decide whether to drop the packet, slow the send rate or disconnect, instead of letting memory grow unnoticed.

---

## 6. Flow: receiving data

On every tick, core drains the transport:

```
  CORE                                  TRANSPORT
   |                                        |
   |───── anything new? ───────────────────>|
   |<──── ✔ here, one frame ────────────────|
   | process the frame                      |
   |───── anything else? ──────────────────>|
   |<──── ✔ here, another frame ────────────|
   | process the frame                      |
   |───── anything else? ──────────────────>|
   |<──── ⏸ nothing left ───────────────────|
   | stop the loop, move on                 |
```

On each call core receives exactly one whole frame or a "nothing left" signal. A packet transport guarantees this itself; for a byte-stream transport the framing layer guarantees it, and the transport underneath only returns raw bytes (§3).

After draining, core hands each frame to the protocol handler, which identifies it as a handshake, a heartbeat or an application message and then generates the event.

### Per-tick budget

The loop above must have a ceiling. Core caps how many frames it processes in one tick. On reaching the cap it stops and leaves the rest for the next tick, so the app regains control on schedule.

The cap does not exist to guard against faulty transport code, since the app itself picks and loads the transport. It exists to share time fairly with the loop: without it, one peer sending in bursts keeps the loop running indefinitely. Unprocessed data is not lost; it stays in the transport and core takes it on the next tick.

### Where data goes if the app stops ticking

The data stays where it is and core does not drop it. Where that is depends on the transport, however, and the consequences differ considerably. Core does not drop old data, but it also does not copy data into a queue of its own.

Undrained data sits in the transport buffer or the OS buffer. When that space fills up:

| Transport | What happens when the app stops ticking for a long time |
|-----------|------------------------------------------------|
| TCP / TLS / WebSocket | The OS receive window fills → acknowledgements stop → the sender congests and sees ⏸ at the far end. No byte is lost, but the peer is blocked. |
| UDP | The receive buffer fills → the OS silently drops new packets. Data is lost and nobody is told, which is how UDP behaves. |
| Transport with an internal buffer | The buffer fills → transport drops the oldest packet to take the new one. This buffer must have a ceiling; letting it grow without bound is a violation. |

The buffer at the end of the chain belongs to transport, not to core, because core has no inbound queue (§5). Transport decides what to drop, and core never learns that a packet was dropped.

Transport drops the oldest packet rather than the newest for two reasons. Transport must not read the payload (§12), so it cannot tell a handshake packet from a data packet in order to prioritise, and one rule has to apply to every session state. For real-time protocols, new data is worth more than old data. An app that needs guaranteed delivery should use a byte-stream transport (§7) instead of relying on this queue.

If the dropped packet was the handshake packet, no new failure mode appears: losing a handshake packet leads to a timeout and then to failure. This is fail-closed and matches the query round described in `02_flows.md` §3.3.2. Fomoxa does not retransmit, so every handshake over a packet transport is best-effort.

This direction of failure is safe. A session opens only on receiving verdict byte `0` (`02_flows.md` §3.4), so a lost packet can never turn a reject verdict into an accept; it can only block a session that would have opened, and the app must reconnect. The queue is also nearly empty during the handshake, because the whole exchange is two to four small packets. The queue ceiling is reached only under traffic bursts.

In addition, when the app stops ticking, the heartbeat stops with it (step 3, §8). The peer probes, gets no answer and declares the session dead. Stopping ticks for a long time is therefore not a safe pause; it causes the connection to be lost.

---

## 7. Size limits, and what Fomoxa does not guarantee

### Maximum size

Fomoxa caps the payload of an application message at 16 MiB, which is 16 MiB + 11 bytes with the frame header included. The number is the same for every implementation. Exact values and byte layout are in `02_flows.md` §2.

Transport ceilings are usually far lower, which is normal:

| Transport | Practical ceiling |
|-----------|--------------|
| TCP / TLS | No ceiling of its own; the byte stream carries whatever is written |
| WebSocket | Theoretically huge, but libraries and intermediate proxies usually impose a configurable ceiling |
| UDP | ~64 KiB absolute; stay under ~1200 bytes to avoid IP-layer fragmentation |

If a frame exceeds the transport ceiling, transport answers ⊘ "too large" (§2), not ⏸. This error does not end the session: the connection stays alive and only that message cannot go out. The app receives the error and handles it; core does not retry.

A transport must never fragment a frame on its own, because fragmenting changes the wire format.

In the receive direction, if an incoming packet is larger than the space core supplied, transport answers ⤢ with the byte count it needs. Core grows the buffer and asks again within the same tick. Losing the packet is forbidden.

### What Fomoxa does not do

Core does not reorder, retransmit lost packets or filter duplicates, because the wire format carries no sequence numbers for any of that. The message id field in a frame is the message *type*, not its *sequence number*.

Direct consequences:

```
  Over TCP/TLS   →  order and integrity are guaranteed by the transport.
                    Fomoxa inherits them and adds nothing.

  Over UDP       →  packets may be lost, reordered, or delivered twice.
                    Fomoxa hands them up in the order it received them, unchanged.
```

A UDP transport does not have to reorder packets and is not allowed to, because reordering requires buffering and delaying packets, which adds delay core cannot account for.

An app that needs ordering and no loss has two options: use a byte-stream transport (TCP, TLS and WebSocket already guarantee it), or handle ordering and reliability in the application layer. This is a deliberate trade-off; in a game, losing one position packet is usually better than waiting for a retransmission.

Heartbeat and peer death detection still work over every transport, UDP included, because they rely on time rather than on packet sequence numbers.

---

## 8. Flow: the tick

One tick does exactly four things, in this order:

```
  ┌─ 1. flush the leftover frame        (wire order is inviolable)
  │
  ├─ 2. drain incoming data             (§6, within the budget)
  │
  ├─ 3. run the protocol clocks         (handshake expired? probe due?
  │                                       peer silent too long → declare it dead?)
  │
  └─ 4. return the event list to the app
```

Transport is involved only in steps 1 and 2. Step 3 lives entirely in core and no transport can interfere, so heartbeat and peer death detection behave identically over every transport and are never rewritten per kind.

---

## 9. Flow: closing the connection

A session ends in one of three ways, and transport participates differently in each.

### 9.1 The app disconnects

```
  APP ──"disconnect"──> CORE ──soft close──> TRANSPORT ──> tell the other side (if it can)
```

Connections with a graceful shutdown (TCP, WebSocket) send a close signal. Connections without one (UDP) do nothing, which is the correct behaviour.

### 9.2 The other side disappears

There are two situations that differ in nature, and transport must distinguish them:

```
  ✖ ORDERLY CLOSE                      ⚠ ABRUPT BREAK
  ───────────────                      ──────────────
  TCP closed cleanly (FIN)             TCP reset (RST)
  WebSocket close frame                read error underneath
  QUIC stream ended cleanly            transport broke mid-way
                                       malformed data arrived
  ↓                                    ↓
  "the peer said goodbye"              "the peer vanished"
```

Transport must return exactly ✖ or ⚠ according to what actually happened. Collapsing the two is forbidden.

Core may not use the distinction yet; a simple implementation may fold both into one DISCONNECT event, and folding or not is core's decision. Once core does distinguish ✖ from ⚠, a transport that reports correctly works immediately without edits. A transport that reports wrongly forces a review of all error handling after the system is already in production. Getting it right up front costs almost nothing, because the underlying network library already tells the two cases apart.

### 9.3 The other side goes silent

In this case there is no close procedure: the peer stops answering. Core detects it with a clock in step 3 of the tick by sending a probe and declaring the peer dead if no answer arrives before the deadline.

Transport plays no part in this detection, which is why it also works over UDP, where there is no connection to lose.

---

## 10. Who is responsible for what

| Job | Core | Transport |
|------|:----:|:---------:|
| Transport handshake (TCP/TLS/WebSocket) | | ✔ |
| Fomoxa handshake (greeting/verdict) | ✔ | |
| Cutting and reassembling frames | ✔ | |
| Preserving wire order | ✔ | |
| Retrying a send after congestion | ✔ | |
| Heartbeat, peer death detection | ✔ | |
| Measuring time, deadlines | ✔ | |
| Capping frames processed per tick | ✔ | |
| Generating events for the app | ✔ | |
| Putting bytes on the network | | ✔ |
| Reporting instant state (✔ / ⏸ / ✖ / ⚠ / ⊘ / ⤢) | | ✔ |
| Ignoring data that is not from this session | | ✔ |
| Rejecting frames above its own ceiling | | ✔ |
| Releasing its own resources | | ✔ |

The transport column is kept short on purpose, so that writing a new transport is a small job.

### Details of "ignoring data that is not from this session"

This rule applies only to connectionless transports, which in practice means UDP. A UDP endpoint receives packets from any source, so a UDP transport must check the source address against the active peer. Unknown packets are dropped silently, without raising an error or ending a session, because an unexpected packet says nothing about session state.

The same endpoint must put a ceiling on its peer table. Every new source address that sends one byte creates an entry, with no handshake or other condition required, and one machine can produce tens of thousands of source ports. Without a ceiling the table grows without bound, which §6 forbids. At the ceiling, packets from unknown addresses are dropped silently, like the unexpected packets above, and running sessions are unaffected. The exact number is per implementation, but having one is mandatory.

TCP, TLS, WebSocket and QUIC are already bound to one peer, so the condition is satisfied automatically and transport does nothing extra.

This is not a security check: it does not authenticate and it does not stop spoofing. Whether both sides use the same schema is checked by the Fomoxa handshake in core. Protocol-specific checks belong to whoever accepts the connection and are out of scope for this document.

---

## 11. Adding a new transport

Example: a WebSocket transport.

### Step 1: determine the kind

WebSocket preserves packet boundaries, so it is packet kind. (A TLS transport would be byte-stream kind and would use core's framing layer.)

### Step 2: handle the connection

The transport opens the WebSocket connection itself: it parses the address, opens the underlying connection, upgrades the protocol and handles the response. Core does not take part and does not need to know. The result is an open pipe.

### Step 3: provide the four functions

- Send: take a block of bytes from core, wrap it in a binary WebSocket packet and push it out. On success return ✔; otherwise return ⏸ *immediately*, since waiting is forbidden. If the block exceeds the ceiling return ⊘; do not fragment and do not return ⏸.
- Receive: if one whole WebSocket packet is available, hand it to core. If it is incomplete, return ⏸ and buffer it internally. If the packet is larger than the space core supplied, return ⤢ with the byte count needed. Return ✖ on a clean close and ⚠ on an abrupt break.
- Soft close: send the WebSocket close frame.
- Hard close: release the connection, the buffers and every other resource.

> Use the binary data type; the text type is forbidden.
> Text WebSocket packets require valid UTF-8 content, and the libraries at both ends
> may validate or normalise it. Fomoxa frames are raw binary bytes
> that are almost certainly not valid UTF-8; sending them over text packets will
> corrupt the data or make the WebSocket library itself reject them. This rule
> applies to any transport that distinguishes "data types": always pick raw binary.

### Step 4: hand it to core

Wrap the open pipe as a packet transport, pass it to core and receive a session back. This step should be wrapped in a convenience function of the library (§4) so that the app does not see the transport concept.

From here the app uses the session exactly like a TCP session: same send call, same tick, same event list.

### Step 5: change nothing in core

If core has to change for a new transport to work, either the design is missing something or logic has been placed in the wrong layer, and the change needs review.

### What a transport author does not have to do

- No need to understand Fomoxa's wire format
- No need to know what greeting, verdict, POLL or REPLY are
- No need to cut or join frames
- No need to preserve order (unless the transport multiplexes substreams itself, §5)
- No need to handle retransmission
- No need to measure time at all
- No need to implement the heartbeat
- No need to reorder out-of-order packets (§7)

---

## 12. What a transport must never do

| Forbidden | Why |
|-----|--------|
| Waiting in any form | It hangs the main loop. This is prohibition number one. |
| Spawning a hidden thread | Fomoxa assumes one session is driven by one thread; a hidden thread breaks that where nobody can check it. |
| Calling back into core | You are inside a core operation; calling back recurses into half-finished state. |
| Taking half a block then answering ⏸ (packet kind) | Core resends from the start, the other side receives a corrupt frame and stays corrupt forever. |
| Returning a partial packet (packet kind) | Core treats everything it receives as one whole frame. |
| Fragmenting a frame to send it | It changes the wire format and breaks compatibility with other implementations. |
| Reordering or resending packets | It adds delay core cannot see, and destroys UDP's deliberate trade-off. |
| Reconnecting on its own after a break | Core already counts the peer as dead. A pipe that silently comes back leaves the two sides disagreeing about session state. |
| Interpreting the bytes | Wrong layer. Those bytes may be a handshake or application data, and transport must not distinguish them. |
| Adding a header of its own | The wire format is fixed so every implementation interoperates; extra bytes break that. |
| Using the text data type (where the transport distinguishes) | Fomoxa frames are pure binary, not valid UTF-8. |

---

## 13. Full picture: one message end to end

```
   APP (machine A)                               APP (machine B)
       │                                         ▲
       │ send message                            │ MESSAGE event
       ▼                                         │
   CORE ── pack into a frame                     CORE ── unpack the frame,
       │                                         │   recognise it as app
       │ (only when READY)                       │   data
       ▼                                         │
   TRANSPORT ── into a WebSocket packet          TRANSPORT ── take the packet
       │                                         ▲
       └──────────────── network ────────────────┘

   Swap WebSocket for TCP, UDP or QUIC:
   the two CORE boxes and the two APP boxes do not change at all.
```

The design exists to achieve this property: changing the transport does not change core or the app.

---

## 14. Invariants every implementation must hold

An implementation of Fomoxa in another language must not change any of the following. Violating one breaks interoperability with other implementations.

| # | Invariant |
|---|----------|
| B1 | The wire format is fixed. Add, remove or reorder no byte. Add no wrapper of your own. (`02_flows.md` §2) |
| B2 | No layer may block. Every operation returns control immediately. |
| B3 | Application data goes out only after the handshake. No application byte reaches the wire before an accept verdict. |
| B4 | Server is the only side that compares schemas. Client sends a greeting and reads one verdict byte. |
| B5 | Probe on silence, not on a fixed interval. And always probe before declaring death. |
| B6 | Exactly one termination event per session. |
| B7 | Wire order is absolute. No frame is injected into a half-sent frame. |
| B8 | Timestamps are passed in as parameters, never read from an internal clock. This is what makes a whole expiry cycle testable without real waiting. |
| B9 | The clock must be monotonic, never a wall clock: one system time sync must not expire a handshake or kill a live peer. |

Acceptance test: any two implementations, in any two languages, must interoperate without either side knowing what the other was written in.
