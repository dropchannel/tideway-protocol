# Tideway Protocol

Tideway is a turn-passing, store-and-forward protocol for two-party messaging over
shared storage. A payload deposited by the initiating endpoint accumulates at every hop
during a forward pass; the terminating endpoint's read triggers a deletion cascade
backward through the chain, confirming delivery and unblocking the next send.

Tideway is one protocol in the [DropChannel](https://github.com/dropchannel) runtime.
System-level concerns — the [DockProvider interface](https://github.com/dropchannel/spec/blob/main/channel-provider.md),
[encryption](https://github.com/dropchannel/spec/blob/main/encryption.md),
[observability](https://github.com/dropchannel/spec/blob/main/observability.md), and
[Agent/Worker runtime](https://github.com/dropchannel/spec/blob/main/agent.md) — are
specified in [`dropchannel/spec`](https://github.com/dropchannel/spec).

---

## Contents

- [Participants](#participants)
- [Propagation protocol](#propagation-protocol)
- [Raft lifecycle](#raft-lifecycle)
- [Backward compatibility](#backward-compatibility)
- [Version history](#version-history)
- [Out of scope](#out-of-scope)

---

## Participants

### Channel

A **Channel** is a named path between exactly two endpoints. All Waterways within a
Channel connect the same two endpoints.

### Waterway

A **Waterway** is a directed flow path within a Channel — a sequence of one or more
Docks carrying payloads from one endpoint to the other. A single-hop Waterway is two
endpoints sharing a Dock directly; a multi-hop Waterway inserts one or more Rafts
between two adjacent Docks.

The Waterway name is invariant across all Docks in the hop sequence.

### Endpoint

An **Endpoint** is the terminating participant — it originates or consumes messages.
Endpoints are the only participants that encrypt or decrypt payload content. Endpoints
have no knowledge of how many hops their Waterway contains.

See [`dropchannel/spec/encryption.md`](https://github.com/dropchannel/spec/blob/main/encryption.md)
for the encryption standard.

### Raft

A **Raft** is a forwarding transport. Rafts carry payloads between Docks without
decrypting or inspecting content — all payloads are opaque bytes to a Raft. A Raft
operates against exactly two Docks: the `upper_dock` it reads from and the `lower_dock`
it writes to. A Raft belongs to exactly one Waterway and has no knowledge of channel
semantics.

---

## Propagation protocol

### File lifecycle invariant

**File present in Dock = payload pending. File absent = handled.**

A file in a Dock means a payload is in flight at that position. An empty Dock means
either nothing has been sent or the ACK cascade has completed. No separate queue, log,
or delivery state is maintained.

### Forward pass

Each participant — originating Endpoint and every intermediate Raft — reads its
`upper_dock` using `peek()` (non-consuming) and writes the payload forward to its
`lower_dock`. No participant deletes its `upper_dock` during the forward pass. The
payload accumulates at every hop simultaneously.

```
After full forward pass (two-hop example):

  Endpoint A ──► Dock A ──► Raft ──► Dock B ──► Endpoint B
                 [blob]              [blob]
```

### ACK cascade

1. **Terminating Endpoint** calls `read()` on its `upper_dock` — the only consuming
   read in the entire protocol. The file is deleted and the payload delivered.

2. **Final Raft** polls its `lower_dock` with `exists()`. Deletion by the terminating
   Endpoint propagates as `exists()` → false. This is the ACK signal. The Raft calls
   `delete()` on its `upper_dock`.

3. **Each preceding Raft** likewise polls its `lower_dock`, detects the downstream
   deletion, and calls `delete()` on its own `upper_dock`.

4. **Originating Endpoint** polls its `lower_dock`. `exists()` → false: the ACK
   cascade has reached the origin. Delivery is confirmed; the next payload may be sent.

```
After full ACK cascade (two-hop example):

  Endpoint A ◄── Dock A ◄── Raft ◄── Dock B ◄── Endpoint B
                 [    ]              [    ]
```

### Key properties

**Crash safety.** If any participant crashes mid-propagation, the payload remains
durably in every Dock already written. On restart, each participant reconstructs its
state by inspecting its two Docks. No external coordination is required.

**Structural backpressure.** The originating Endpoint's `lower_dock` remains occupied
for the entire round-trip. A second payload cannot be sent until the ACK cascade
completes.

**Delivery confirmation.** The originating Endpoint receives implicit confirmation that
the terminating Endpoint consumed the payload. The signal is the clearing of its
`lower_dock`, caused deterministically by the cascade.

**No in-memory buffering.** No participant holds a payload in memory between poll
cycles. A Raft that has peeked but not yet written simply retries on the next poll.

### Write guard

`write()` returns `False` if the target filename is already present. The
check-before-write sequence has an inherent race window.

---

## Raft lifecycle

### Startup: inspect both Docks

On startup the Raft inspects both Docks to determine starting state.

| `upper_dock` | `lower_dock` | Condition | Action |
|---|---|---|---|
| Empty | Empty | Idle | → Recv-polling |
| Occupied | Empty | Unforwarded payload | `peek()` upper → `write()` lower → Send-polling |
| Occupied | Occupied (match) | Forwarded; awaiting ACK | → Send-polling |
| Occupied | Occupied (differ) | State corruption | Log payload sizes and hashes, halt |
| Empty | Occupied | Impossible state | Log, halt |

Payload comparison is raw byte equality — no decryption needed.

File discovery within a Dock requires a `list()` operation, which is deferred (see
[Out of scope](#out-of-scope)).

### Recv-polling (steady state: idle)

```
Loop:
  exists(upper_dock, <filename>)?
    No  → sleep(poll_interval), repeat
    Yes → payload = peek(upper_dock, <filename>)
          write(lower_dock, <filename>, payload)
          → Send-polling
```

### Send-polling (steady state: in-flight)

```
Loop:
  exists(lower_dock, <filename>)?
    Yes → sleep(poll_interval), repeat
    No  → delete(upper_dock, <filename>)
          → Recv-polling
```

### Polling cost

| Phase | Operations per cycle |
|---|---|
| Startup | Two `peek()` calls, once only |
| Recv-polling | One `exists()` |
| Recv→Send transition | One `peek()` + one `write()` |
| Send-polling | One `exists()` |
| Send→Recv transition | One `delete()` |

---

## Backward compatibility

v0.6 and earlier participants used `ChannelProvider(channel_id, waterway)`. v0.7
changed to `DockProvider(channel, waterway, filename)`. Mixed v0.6/v0.7 deployments
are not supported.

---

## Version history

| Version | Summary |
|---------|---------|
| [v0.1](history/v0.1.md) | Initial protocol: asymmetric Sender/Receiver roles, two fixed Waterways (`request`/`response`), delete-on-read, AES-256-GCM encryption |
| [v0.2](history/v0.2.md) | Symmetric peers, generic Waterway names, write guard |
| [v0.3](history/v0.3.md) | `ChannelProvider` abstraction; storage medium becomes pluggable |
| [v0.4](history/v0.4.md) | Channel and Node concepts formalised; propagation protocol and ACK cascade; `peek()` added to interface |
| [v0.5](history/v0.5.md) | `http` provider renamed to `httprelay`; no protocol behavior changes |
| [v0.6](history/v0.6.md) | Heartbeat protocol; meta Waterways; persistent participant identity |
| [v0.7](history/v0.7.md) | Unified vocabulary: Winch → Tideway, Node → Raft, ChannelProvider → DockProvider; `(channel, waterway, filename)` address model |

---

## Out of scope

- Automatic recovery from Occupied/Occupied payload-mismatch error state
- Atomic write guards (compare-and-swap) for concurrent writers
- Raft fan-out (one inbound, multiple outbound)
- More than two parties per Channel
- Multiple payloads in flight per Waterway
- `list()` enumeration operation on DockProvider (deferred; needed for Raft startup inspection)
- Raft startup role determination without `list()`
