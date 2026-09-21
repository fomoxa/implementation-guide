# Fomoxa implementation guide

A document set for building a transport or a Fomoxa SDK in any language. It is described at the conceptual level, without an SDK, an API or source code.

The documents are meant for lookup, and reading both end to end is not a prerequisite for starting.
Pick a reading path for the task at hand in [§ Read by task](#read-by-task), then come back to look things up.

## Document map

| Document | Content | Length |
|---|---|---|
| [en/01-overview.md](en/01-overview.md) | The three-layer model, the boundaries, the core ↔ transport split, the prohibitions | ~550 lines |
| [en/02-flows.md](en/02-flows.md) | Wire format, every flow in detail, pseudocode for core and transport, the test checklist | ~990 lines |

Internally the two documents refer to each other by the concept names `01_overview.md` and `02_flows.md`. Those map to `en/01-overview.md` and `en/02-flows.md`.

The `vi/` directory holds the Vietnamese version. Other translations, if any, live in their own language-code directory.

## Read by task

| What you want | Sections to read |
|---|---|
| Understand how Fomoxa splits its layers | `01` §1–§3 |
| Write a new transport (WebSocket, TLS, QUIC, UDP…) | `01` §2, §3, §11, §12 → `02` §9, plus the self-check table §9.5 |
| Only need the bytes on the wire (writing a codec) | `02` §2 |
| Understand the handshake and the schema check | `02` §3 |
| Understand the heartbeat and peer death detection | `02` §4 |
| Write a new Fomoxa SDK in another language | Both, in the order `01` → `02` |
| Verify an existing implementation | `02` §11 → `01` §14 |
| Debug interoperability between two implementations | `02` §2 and §10 |

A transport author does not read the wire format. This is a design condition: a transport only carries bytes, so a transport that needs to know what a byte means has logic in the wrong layer.

## One-page summary

Fomoxa has three layers, and most of the rules concern the boundaries between them:

```
  APP          messages that mean something to the application
  CORE         handshake, heartbeat, frame packing/unpacking, events
  TRANSPORT    carries bytes only  (TCP / UDP / WebSocket / TLS / QUIC)
```

A transport provides exactly four functions (send, receive, soft close, hard close) and answers with exactly six signals:

```
  ✔ done    ⏸ not now    ✖ closed    ⚠ error    ⊘ too large    ⤢ buffer too small
```

A transport has no fifth function, no blocking wait and no automatic reconnect. Remembering and retrying belong to core.

Fixed numbers: a DATA frame is at most 16 MiB + 11 bytes; a HANDSHAKE body is at most 1 MiB; the default handshake deadline is 5 seconds, the heartbeat interval 5 seconds, the heartbeat deadline 15 seconds.

The nine invariants every implementation must hold are in `01` §14. The one that applies everywhere is that no layer may block.

## Out of scope for this document set

- Schema fingerprint computation and message data encoding belong to the Fomoxa
  schema specification.
- Per-language API bindings. Each implementation picks its own names and its own call
  shapes. Only *behaviour* is defined here, and behaviour must be identical everywhere.
- The transport's own handshake (TCP, TLS, the WebSocket upgrade). It happens first,
  outside Fomoxa, and the transport author owns it.

## License

CC BY 4.0, see [LICENSE](LICENSE). Software implementations are independent projects and pick whatever license their authors choose.
