# Fomoxa UDP transport: session identifier

This document defines how a UDP transport tells one peer from another. It applies to UDP only. Core does not know about it, and TCP, TLS, WebSocket and QUIC are unaffected, because each of those is already bound to one peer (`01_overview.md` §10).

A transport author reads this document in addition to `01_overview.md` §2, §3, §11 and §12. The author still does not read the wire format of `02_flows.md` §2: the session identifier sits outside the frame, and the transport never looks inside the bytes after it.

---

## 1. Why the source address is not enough

A UDP transport that identifies a peer by its source address loses the session whenever that address changes:

- A phone moves from wifi to cellular, or a NAT assigns a new port after an idle period. The packets keep coming, from an address the transport has never seen.
- A relay or load balancer in front of several servers has nothing in the packet to route by.

A session identifier carried in every datagram separates who the peer is from where it currently sends from.

---

## 2. Datagram layout

```
   ┌────────────────────────┬──────────────────────────────────┐
   │ session id             │ one frame                        │
   │ 8B u64 LE              │ exactly as core handed it over   │
   └────────────────────────┴──────────────────────────────────┘
    ^0-7                     ^8 onward

   Total size = 8 + frame size
```

- The frame is the bytes core passed to `send`, unchanged. The transport does not read, split or join it.
- A datagram shorter than 9 bytes is dropped.
- The value `0` means "no identifier assigned yet". The server never assigns it.
- The transport's own ceiling counts the 8 bytes of the header. A frame that fits a ceiling of C bytes per datagram must be at most C − 8 bytes, and the transport answers ⊘ above that, as in `01_overview.md` §7. The guideline of staying under about 1200 bytes also counts the header.

---

## 3. How an identifier is assigned and used

### 3.1 Server

```
   on a datagram from source address A with session id S:

       S = 0:
           find the peer registered at address A
               found     → accept the frame for that peer
               not found → if the peer table is at its ceiling: drop silently
                           otherwise create a peer at A, assign it a new id, accept the frame

       S ≠ 0:
           find the peer that owns S
               found     → accept the frame for that peer;
                           if A differs from the peer's recorded address,
                           record A as the peer's new address
               not found → drop silently
```

- Every datagram the server sends to a peer carries that peer's id.
- A new id is a random 64-bit value from a cryptographically secure generator. It is non-zero and differs from every id of a peer that currently exists.
- The recorded address changes only after a datagram has been accepted. A dropped datagram never moves a peer.
- The peer table keeps the ceiling from `01_overview.md` §10. That ceiling applies to the creation step above.
- When a peer is released by hard close, its id is released with it. A later datagram that carries the id is dropped as an unknown id.

### 3.2 Client

- The client sends `0` as the session id until it has received a datagram from the server.
- It adopts the id of the first datagram it receives and uses it for every datagram after that.
- A later datagram with a different non-zero id is dropped. The id of a session never changes.

### 3.3 What core sees

Core sees no change. It receives frames in the same order as before, from the same peer object. A change of address does not raise an event, and heartbeat keeps working as `02_flows.md` §4 defines it. If the old address was still reachable and the new one is not, the peer goes silent and core handles it as it handles any silent peer.

---

## 4. What this does not do

- It does not authenticate and it does not encrypt. The identifier is visible to anyone who sees the datagrams, and anyone who learns it can send datagrams that move the peer to another address. Confidentiality and authentication belong to a layer below, such as DTLS, or to whoever accepts the connection. They are outside this document.
- It does not verify that the new address is reachable before the server sends to it. A holder of a captured id can steer the server's traffic to a third address. An implementation should limit what it sends to a new address until that address has sent a datagram of its own, as QUIC does with path validation.
- It does not apply to a server restart. After a restart the old ids are unknown, the datagrams of old peers are dropped, and those sessions end through the heartbeat deadline.

---

## 5. Compatibility

This is the only datagram format of the UDP transport. A datagram without the 8-byte header is not a valid datagram, so a UDP transport that does not implement this document cannot talk to one that does.

---

## 6. Self-check

- [ ] Every datagram sent starts with 8 bytes of session id, followed by exactly one frame
- [ ] A datagram under 9 bytes is dropped without an error
- [ ] The server assigns a non-zero random id from a cryptographically secure generator, unique among live peers
- [ ] The server never sends a datagram with id `0`
- [ ] Id `0` creates a peer only when the peer table is below its ceiling
- [ ] An unknown non-zero id is dropped silently
- [ ] The recorded address changes only after an accepted datagram
- [ ] The client keeps the first id it receives and drops datagrams that carry another
- [ ] The ceiling for a frame is the datagram ceiling minus 8
- [ ] Core is told nothing about an address change
