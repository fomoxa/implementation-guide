# Fomoxa Transport  -  Model and Responsibilities

This document describes the Fomoxa transport model at the conceptual level. It is independent of any programming language. It references no implementation, no API, no specific source code.

Read this document together with `02_flows.md` and you can rebuild Fomoxa: a new implementation, or a new transport, in any language. You never need to read the source of an existing implementation.

| Document | Content | Read it when |
|---|---|---|
| `01_overview.md` (this one) | Model, layer boundaries, split of responsibilities | You want to understand the design |
| `02_flows.md` | Wire format, every flow in detail, implementation guide | You start writing code |

Each language may ship its own API binding document. Those documents describe *how to call*, not *how it behaves*. Behaviour lives in these two documents. Every implementation must behave identically.

---

## 1. Three Layers

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

The boundary that matters most: core does not know how the data travels, transport does not know what the data contains. Transport never sees a handshake, a heartbeat, or peer state. It cannot tell a greeting apart from a character position update. To transport, everything is one block of bytes to push to the other side.

The reverse also holds: core does not know whether it runs over TCP or WebSocket. It must not know.

---

## 2. The Four Functions a Transport Must Provide

Exactly four. No more:

| Function | Meaning | Caller |
|-----------|-------|--------|
| Send | Take a block of bytes, push it to the other side | Core, when it has something to send |
| Receive | Return data that arrived from the other side, if any | Core, every tick |
| Soft close | Tell the other side we are about to disconnect | Core, when the app asks to disconnect |
| Hard close | Release every resource | Core, when the session is destroyed |

There is no fifth function. No "reconnect". No "wait until the send succeeds". No "tell me how much room is left".

### Four Shared Answers

Asked to send or asked to receive, transport answers with one vocabulary:

```
  ✔  Done                    → core moves on
  ⏸  Not right now           → core holds it, asks again next tick
  ✖  I am closed             → core ends the session
  ⚠  I failed                → core ends the session
```

### Two Special Answers, One Per Direction

Those four do not cover everything. Two cases are left: not success, not "wait a moment", and not death either:

```
  SEND direction
  ⊘  This block is larger than I can carry
     → does not kill the session. Core reports the error up; the connection stays alive.
     → Core does not retry. (§7)

  RECEIVE direction
  ⤢  Data is here, but your buffer is too small. I need this many bytes.
     → Core grows the buffer and asks again. No data is lost. (§7)
```

⊘ must stay separate from ⏸ because the two say different things. ⏸ means "retry later and it will work". If transport returns ⏸ for an oversized payload, core keeps it and retries every tick. The payload is still oversized next tick. The loop never ends and the send channel jams forever. ⊘ says the opposite: *do not retry; this payload will never go out*.

⤢ also belongs to the "retry and it will work" group. It carries a number: core must grow the buffer before it asks again. So it cannot be folded into ⏸ either.

### The Most Important Prohibition

There is no "let me wait" answer. This is prohibition number one of the design. Fomoxa runs inside a timed loop. A transport that sits and waits stops the whole loop. Transport reports state *at the instant it is asked* and returns control. Retrying is core's job.

---

## 3. Two Kinds of Transport

Exactly two kinds. A transport must declare which kind it is:

### Byte-Stream Kind

Bytes flow continuously, with no dividers. Send three 100-byte blocks and the other side may receive one 300-byte block. Or seven small ones. Nobody can tell where one unit ends.

→ TCP, TLS, QUIC stream

### Packet Kind

Transport keeps the dividers itself. Send three blocks and the other side receives exactly three. Correct boundaries, never glued together.

→ UDP, WebSocket, QUIC datagram

### Why the Distinction Matters

A byte-stream transport needs its data reassembled. Core does that job. When a transport declares itself byte-stream, core inserts a framing layer in between. The transport author never needs to know how Fomoxa frames data.

```
  Packet transport               Byte-stream transport
  ────────────────               ─────────────────────
        CORE                              CORE
          |                                 |
          |                        [ framing layer ]   ← core handles it
          |                                 |
      WebSocket                            TCP
```

That is why a WebSocket or TLS author never reads the wire format.

The framing layer must sit between core and transport, inside neither. Put it inside transport and transport must understand the frame format - wrong layer. Put it inside core and core must check the transport kind on every call. Also wrong layer. Put it in between and the branch happens once, at setup. Both ends stay clean.

### Return Obligations  -  the Two Kinds Differ Completely

This is the easiest part to get wrong. Stated plainly:

| | Byte-stream kind | Packet kind |
|---|---|---|
| Per return | Any number of bytes | Exactly one whole packet |
| Return a single byte? | Legal | Illegal |
| Return half a frame? | Legal | Illegal |
| Return 2.5 frames glued? | Legal | Illegal |
| Packet still incomplete? | Return what you have | Must answer ⏸ and hold it |
| Who reassembles/buffers? | Core's framing layer | The transport itself |

A byte-stream transport must not try to return "exactly one frame". It lacks the information. Doing it would require parsing the wire format - wrong layer. Return raw bytes as they arrive.

A packet transport is the opposite. Returning a partial packet is forbidden. If only part of a new packet has arrived, answer ⏸. The partial data must not reach core.

---

## 4. Flow: First Connection

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

The key point: the transport must already be open before it is handed to core. TCP handshake, TLS handshake, WebSocket upgrade - all of that happens outside, beforehand. The transport author owns it.

### Who Creates the Transport

Core never creates a transport itself. Creating and connecting belongs to the extension library. Not to the app, and not to core:

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

The library should wrap that in a convenience function. For example: "connect over WebSocket, return a session". Ideally the app never sees the transport concept. It only picks which library to call.

### Two Handshakes, Fully Independent

Core receives an open pipe. Only then does it run Fomoxa's own handshake over that pipe:

```
  transport handshake            Fomoxa handshake
  (TCP, TLS, WebSocket)   →      (greeting / verdict)
  ↑ outside Fomoxa              ↑ core's job, spans many ticks
```

The Fomoxa handshake is not finished when `connect` returns. It continues across later ticks. The app learns it finished when it receives the READY event. Until then every send is rejected. No application byte goes on the wire before both sides agree.

---

## 5. Flow: Sending Data

### The Happy Path

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

### When the Transport Is Congested

This is the flow that matters. It is where this design departs from the naive one:

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

Three things follow:

1. The app never blocks. Send returns immediately, even when the transport is full.
2. Transport never waits. It says "not now" and stops there.
3. Core remembers and retries. Every tick, the first job is to flush the leftover - before anything else.

Wire order is absolute. A new frame injected into a half-sent frame corrupts the decoder on the receiving side. Permanently. While a frame is pending, sending a new frame is forbidden.

### When Congestion Persists

The pending slot holds one frame, for one session, in the outbound direction. Stated fully:

- One session is one logical send channel. This layer has no concept of parallel streams.
- The receive direction has no pending slot in core. That does not mean data is dropped. See §6.
- The "one frame" limit applies to application sends. Control frames that core generates - POLL, REPLY, handshake verdict - queue *after* the pending slot and never overwrite it. They have their own ceiling, see `02_flows.md` §8.
- Connections with substreams (QUIC): the transport interface exposes exactly one send/receive channel pair. A transport that wants substreams multiplexes them *internally* and keeps the order correct while doing so. Core sees one pipe. The "one pending frame" rule applies to that logical channel, not to each substream.

If the previous frame has not gone out and the app sends again, the send is rejected with a "congested" signal. Fomoxa does not queue without bound. If the peer stops reading, the app must know it and decide for itself: drop the packet, slow the send rate, or disconnect. Memory must not grow silently.

---

## 6. Flow: Receiving Data

Every tick, core drains the transport:

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

On each call core receives exactly one whole frame, or a "nothing left" signal. A packet transport handles this itself. For a byte-stream transport the framing layer handles it. The transport underneath only returns raw bytes (§3).

Only after draining does core hand each frame to the protocol handler. There the frame is identified: handshake, heartbeat, or application message. Then the event is generated.

### Per-Tick Budget

The loop above must have a ceiling. Core caps how many frames it processes in one tick. On reaching the cap it stops and leaves the rest for the next tick. The app regains control on schedule.

This cap does not exist to guard against sloppy transport code. The app itself picks and loads the transport; it is not a foreign component. The cap exists to share time fairly with the loop. Without it, one peer sending in bursts keeps the loop running forever. Unprocessed data is not lost. It stays in the transport and core takes it next tick.

### Where Data Goes If the App Stops Ticking

The data stays exactly where it is. Nobody drops it. But *where* "there" is depends on the transport, and the consequences differ sharply. Core does not drop old data. It simply does not copy data into a queue of its own.

Undrained data sits in the transport buffer or the OS buffer. When that space fills up:

| Transport | What happens when the app stops ticking for a long time |
|-----------|------------------------------------------------|
| TCP / TLS / WebSocket | The OS receive window fills → acknowledgements stop → the sender congests and sees ⏸ at the far end. No byte is lost, but the peer is blocked. |
| UDP | The receive buffer fills → the OS silently drops new packets. Data is lost and nobody is told  -  exactly what UDP is. |
| Transport with an internal buffer | The buffer fills → transport drops the oldest packet to take the new one. This buffer must have a ceiling; letting it grow without bound is a violation. |

The buffer at the end of the chain belongs to transport, not to core. Core has no inbound queue (§5). Transport decides what to drop. Core never learns that a packet was dropped.

Why drop the oldest packet and not the newest? Transport must not read the payload (§12). It cannot tell a handshake packet from a data packet in order to prioritise. One rule applies to every session state, and for real-time protocols new data is worth more than old data. If you need guaranteed delivery, use a byte-stream transport (§7). Do not rely on this queue.

What if the dropped packet was the handshake packet? No new failure mode appears. Losing a handshake packet leads to a timeout and then to failure. That is fail-closed, and it matches the query round described in `02_flows.md` §3.3.2. Fomoxa does not retransmit. Every handshake over a packet transport is best-effort.

The safety lies in the direction of failure. A session opens only on receiving verdict byte `0` (`02_flows.md` §3.4). A lost packet can never turn a reject verdict into an accept. It only blocks a session that would have opened, and the app must reconnect. The queue is also nearly empty during the handshake: the whole exchange is two to four small packets. The queue ceiling is reached only under traffic bursts.

One more consequence. If the app stops ticking, the heartbeat stops with it (step 3, §8). The peer probes, gets no answer, and declares the session dead. Stopping ticks for a long time is not a "safe pause". It is how you lose the connection.

---

## 7. Size Limits, and What Fomoxa Does Not Guarantee

### Maximum Size

Fomoxa caps the payload of an application message hard: 16 MiB. With the frame header included that is 16 MiB + 11 bytes. The number is the same for every implementation. Exact values and byte layout: `02_flows.md` §2.

Transport ceilings are usually far lower. That is normal:

| Transport | Practical ceiling |
|-----------|--------------|
| TCP / TLS | No ceiling of its own  -  the byte stream carries whatever you write |
| WebSocket | Theoretically huge, but libraries and intermediate proxies usually impose a configurable ceiling |
| UDP | ~64 KiB absolute; stay under ~1200 bytes to avoid IP-layer fragmentation |

If a frame exceeds the transport ceiling, transport answers ⊘ "too large" (§2), not ⏸. This error does not kill the session: the connection stays alive, only that message cannot go out. The app receives the error and handles it. Core does not retry.

A transport must never fragment a frame on its own. Fragmenting changes the wire format.

The reverse direction: if an incoming packet is larger than the space core supplied, transport answers ⤢ with the byte count it needs. Core grows the buffer and asks again within the same tick. Losing the packet is forbidden.

### What Fomoxa Does Not Do

Core does not reorder, does not retransmit lost packets, does not filter duplicates. The wire format carries no sequence numbers for any of that. The message id field in a frame is the message *type*, not its *sequence number*.

Direct consequences:

```
  Over TCP/TLS   →  order and integrity are guaranteed by the transport.
                    Fomoxa inherits them and adds nothing.

  Over UDP       →  packets may be lost, reordered, or delivered twice.
                    Fomoxa hands them up in the order it received them, unchanged.
```

A UDP transport does not have to reorder packets. It is also not allowed to. Reordering means buffering and delaying packets. Core cannot account for that delay.

An app that needs ordering and no loss has two options. One: use a byte-stream transport - TCP, TLS, WebSocket already guarantee it. Two: handle ordering and reliability in the application layer. This is a deliberate trade-off. In a game, losing one position packet usually beats waiting for a retransmission.

Heartbeat and peer death detection still work over every transport, UDP included. They rely on time, not on packet sequence numbers.

---

## 8. Flow: The Tick

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

Transport is involved only in steps 1 and 2. Step 3 lives entirely in core and no transport can interfere. That is why heartbeat and peer death detection behave identically over every transport. You never rewrite them per kind.

---

## 9. Flow: Closing the Connection

A session ends in three ways. Transport participates differently in each:

### 9.1 The App Disconnects

```
  APP ──"disconnect"──> CORE ──soft close──> TRANSPORT ──> tell the other side (if it can)
```

Connections with a graceful shutdown - TCP, WebSocket - send a close signal. Connections without one - UDP - do nothing at all. That is correct behaviour, not an omission.

### 9.2 The Other Side Disappears

Two situations, different in nature. Transport must distinguish them:

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

Transport must return exactly ✖ or ⚠ according to what really happened. Collapsing the two is forbidden.

Core may not use the distinction yet. A simple implementation may fold both into one DISCONNECT event. Folding or not is core's call. Once core does distinguish ✖ from ⚠, a transport that reports correctly works immediately. No edits. A transport that reports wrongly forces a review of all error handling at the worst possible moment: when the system is already in production. Getting it right up front costs almost nothing. The underlying network library already tells the two cases apart.

### 9.3 The Other Side Goes Silent

There is no close procedure at all. The peer stops answering. Core detects it with a clock in step 3 of the tick: it sends a probe, and if no answer arrives before the deadline it declares the peer dead.

Transport plays no part in this detection. That is why it works over UDP too - where there is no "connection" to lose.

---

## 10. Who Is Responsible for What

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

The right column is very short. That is deliberate. Writing a new transport must be a small job.

### On "Ignoring Data That Is Not From This Session"

This applies only to connectionless transports, which in practice means UDP alone. A UDP endpoint receives packets from any source. A UDP transport must check the source address against the active peer. Unknown packets are dropped silently. No error is raised, no session is ended. An unexpected packet says nothing about session state.

That same endpoint must put a ceiling on its peer table. Every new source address that sends one byte creates an entry. No handshake is required, nothing else is either. One machine can produce tens of thousands of source ports. Without a ceiling the table grows without bound, which is what §6 forbids. At the ceiling, packets from unknown addresses are dropped silently, handled exactly like the unexpected packets above. Running sessions are unaffected. The exact number is per implementation. Having a number is mandatory.

TCP, TLS, WebSocket and QUIC are already bound to one peer. The condition is satisfied automatically and transport does nothing extra.

This is not a security check. It does not authenticate and it does not stop spoofing. Whether both sides speak the same language is decided by the Fomoxa handshake, in core. Protocol-specific checks belong to whoever accepts the connection. Out of scope for this document.

---

## 11. How to Add a New Transport

Example: build a WebSocket transport.

### Step 1  -  Determine the Kind

WebSocket preserves packet boundaries → packet kind. (For TLS it would be byte-stream, and you get the framing layer for free.)

### Step 2  -  Handle the Connection Yourself

Open the WebSocket connection yourself: parse the address, open the underlying connection, upgrade the protocol, handle the response. Core does not help and does not need to know. The result is an open pipe.

### Step 3  -  Provide the Four Functions

- Send: take a block of bytes from core, wrap it in a binary WebSocket packet, push it out. On success return ✔. On failure return ⏸ *immediately*; waiting is forbidden. If the block exceeds the ceiling return ⊘; do not fragment and do not return ⏸.
- Receive: if one whole WebSocket packet is available, hand it to core. If it is incomplete, return ⏸ and buffer it yourself. If the packet is larger than the space core supplied, return ⤢ with the byte count needed. Return ✖ on a clean close and ⚠ on an abrupt break.
- Soft close: send the WebSocket close frame.
- Hard close: release the connection, the buffers, every resource.

> Use the binary data type; the text type is forbidden.
> Text WebSocket packets require valid UTF-8 content, and the libraries at both ends
> may validate or normalise it. Fomoxa frames are raw binary bytes
> that are almost certainly not valid UTF-8; sending them over text packets will
> corrupt the data or make the WebSocket library itself reject them. This rule
> applies to any transport that distinguishes "data types": always pick raw binary.

### Step 4  -  Hand It to Core

Wrap the open pipe as a packet transport, pass it to core, receive a session back. Wrap this step in a convenience function of your library (§4). The app must not have to see the transport concept.

From here the app uses it exactly like a TCP session: same send call, same tick, same event list.

### Step 5  -  Change Nothing in Core

Needing to change core to make a new transport work is a bad sign. Either this design is missing something, or you put logic in the wrong layer. Stop and review.

### What You Do Not Have to Do

- No need to understand Fomoxa's wire format
- No need to know what greeting, verdict, POLL or REPLY are
- No need to cut or join frames
- No need to preserve order (unless you multiplex substreams yourself - §5)
- No need to handle retransmission
- No need to measure time at all
- No need to implement the heartbeat
- No need to reorder out-of-order packets (§7)

---

## 12. What a Transport Must Never Do

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
| Interpreting the bytes | Wrong layer. Those bytes may be a handshake or application data  -  transport must not care. |
| Adding a header of its own | The wire format is fixed so every implementation interoperates; extra bytes break that. |
| Using the text data type (where the transport distinguishes) | Fomoxa frames are pure binary, not valid UTF-8. |

---

## 13. Full Picture: One Message End to End

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

That is the entire point of this design.

---

## 14. Invariants Every Implementation Must Hold

Rebuild Fomoxa in another language and these are the things you must not change. Violate one and you lose the ability to talk to other implementations.

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

Acceptance test: any two implementations, in any two languages, must talk to each other. Neither side needs to know what the other was written in.
