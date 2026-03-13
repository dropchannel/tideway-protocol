# Winch Protocol

**Winch** is a pull-then-hold coordination protocol with deletion-as-ACK cascade. It
defines how a blob moves from an originating endpoint to a terminating endpoint through
a chain of storage-backed forwarding hops, and how confirmed delivery propagates back
to the originator without a dedicated acknowledgment channel.

Winch is one of the coordination protocols in the [DropChannel system](../spec). It
operates on channels whose names begin with the `winch-` prefix.

The name reflects the physical mechanic: a winch pulls a load across a distance by
winding in cable, one deliberate crank at a time. Each crank is a hop. The load is held
at position between cranks. When the destination releases, the tension unwinds back
through the cable — the ACK cascade.

---

## The Core Idea

Most messaging protocols are built on a **push** model: a sender actively transmits data
toward a receiver, hop by hop, until it arrives. Winch is built on a **pull-then-hold**
model: each participant independently notices that a blob is available upstream, pulls a
copy without removing it, and deposits a copy at the next location. The blob accumulates
at every hop simultaneously. No participant pushes anything toward the next — the payload
*materializes* at each location rather than being thrown across.

This distinction has deep consequences for crash recovery, delivery confirmation, and the
nature of backpressure — all of which are addressed below.

---

## Concepts

For definitions of Channel, Physical Pipeline, Slot, Endpoint, Node, and ChannelProvider,
see the [DropChannel System Specification](../spec). Winch-specific concepts are defined
here.

### Recv Slot / Send Slot

Each Node in a Winch pipeline operates on exactly two slots:

- **Recv slot** — the slot it reads from (upstream)
- **Send slot** — the slot it writes to (downstream)

These are the only two slots a node knows about. The node has no visibility into the
rest of the pipeline.

### Forward Pass

The phase during which a blob propagates from the originating endpoint toward the
terminating endpoint. During the forward pass, no participant deletes its recv slot.
The blob accumulates at every slot in the pipeline simultaneously.

### ACK Cascade

The phase during which confirmed delivery propagates back toward the originating endpoint.
The cascade begins when the terminating endpoint's destructive `read()` clears the final
slot. Each upstream node detects the clearing of its send slot and responds by deleting
its own recv slot, triggering the next step in the cascade.

---

## The Propagation Protocol

### Forward Pass

The originating endpoint writes an encrypted blob to its send slot via `write()`. Each
node polls its recv slot via `exists()`. On detecting a blob, the node reads it
non-destructively via `peek()` and writes a copy to its send slot via `write()`. **No
participant deletes its recv slot during the forward pass.**

After full forward propagation, the pipeline looks like this (two-hop example):

```
  Endpoint A         Node 1            Node 2          Endpoint B
  [send: ●] ──────▶ [recv: ●]  ─────▶ [recv: ●] ────▶ [recv: ●]
                    [send: ●]          [send: ●]
```

Every slot is occupied. All copies are byte-for-byte identical.

### ACK Cascade

The terminating endpoint reads its recv slot via `read()`, which consumes and deletes it.
This is the **only destructive read in the entire protocol**. The endpoint decrypts and
processes the message.

Each upstream node polls its send slot via `exists()`. On detecting that the slot has
been cleared, the node calls `delete()` on its own recv slot. This clears the slot
observed by the next upstream node, propagating the cascade.

```
  Endpoint A         Node 1            Node 2          Endpoint B
  [send: ○] ◀────── [recv: ○]  ◀───── [recv: ○] ◀──── [recv: consumed by B]
                    [send: ○]          [send: ○]
```

All slots empty. The originating endpoint, detecting its send slot cleared, knows
delivery is confirmed and may send the next message.

### Key Properties

**Crash safety.** If any participant crashes mid-propagation, the blob remains durably
stored at every slot already written. On restart, each participant reconstructs its exact
state by inspecting its two slots. No external coordination or log is required.

**Delivery confirmation without a dedicated ACK channel.** The ACK cascade travels over
the same pipeline as the data, using deletions rather than messages. No additional
channel capacity is consumed by acknowledgment traffic.

**Structural backpressure.** The originating endpoint's send slot remains occupied for
the full round-trip duration. A second message cannot be sent until the ACK cascade
completes. This is a protocol property, not a configuration option.

**Asynchronous node availability.** Because blobs are durably stored at each hop, nodes
do not need to be online simultaneously. A node that is offline when a blob arrives will
process it on restart. The pipeline stalls gracefully and resumes from exact state.

---

## Node Lifecycle

A Winch node's complete behavior is determined by the observable state of its two slots.

### Startup: Reconstruct State

On startup, a node peeks both slots once to determine its starting state:

| Recv slot | Send slot | Action |
|-----------|-----------|--------|
| Empty | Empty | Enter recv-polling mode |
| Occupied | Empty | peek recv → write send → enter send-polling mode |
| Occupied | Occupied (payloads match) | Enter send-polling mode |
| Occupied | Occupied (payloads differ) | **Error** — log SHA-256 of each, halt |
| Empty | Occupied | **Impossible state** — log, halt |

Payload comparison is byte-for-byte on raw (encrypted) bytes. No decryption is needed
or performed. The SHA-256 hashes logged on mismatch allow post-hoc diagnosis without
revealing plaintext.

### Recv-Polling Mode

Poll recv slot `exists()` only. Send slot is empty.

On `exists()` → `true`: `peek()` recv, `write()` to send, transition to send-polling mode.

### Send-Polling Mode

Poll send slot `exists()` only. Recv slot is occupied and must not be touched.

On `exists()` → `false`: `delete()` recv slot, transition to recv-polling mode.

### Polling Cost

Each steady-state mode requires exactly **one `exists()` call per poll cycle**. The only
moments requiring additional operations are:

- Forward transition: one `peek()` + one `write()`
- ACK transition: one `delete()`

This minimizes both operation count and cost against metered storage backends.

---

## Comparison with Other Protocols

| Property | UDP | TCP | SMTP | Winch |
|----------|-----|-----|------|-------|
| Propagation model | Push (fire and forget) | Push (reliable stream) | Push (store and forward) | Pull-then-hold (materialization) |
| Delivery guarantee | None | Yes (retransmit) | Best effort | Yes (ACK cascade) |
| ACK mechanism | None | Receiver sends ACK packet | None (delivery reports optional) | Deletion propagates backward |
| In-flight copies | One (in transit) | One (buffered) | One (per hop) | N (one per hop, all durable simultaneously) |
| Crash recovery | None | Retransmit from buffer | Resume from spool | Restart and inspect two slots |
| Backpressure | None | Sliding window | None | Structural (send slot occupied) |
| Simultaneous availability | Required | Required | Not required | Not required |
| Latency profile | Microseconds | Milliseconds | Seconds–hours | Seconds–minutes (poll-interval bound) |
| Throughput | Many packets in flight | Sliding window | One message per queue slot | One message at a time per channel |

Winch is not a replacement for TCP, UDP, or SMTP. It operates at a different point in
the design space: lower throughput, higher latency, but with asynchronous node
availability, provider-agnostic hop composition, and structural delivery confirmation
that other protocols require additional application-layer work to achieve.

The closest historical analogues are SMTP/UUCP store-and-forward for the durable-hop
property, and tuple spaces (Linda) for the materialization-at-every-location model.
The combination of pull-then-hold propagation, deletion-as-ACK cascade,
provider-agnostic hops, and structural backpressure is, as far as is known, novel as a
unified protocol design.

---

## What Winch Is Not

**Winch is not a transport protocol.** It does not define how bytes move between
machines. That is the responsibility of the ChannelProvider implementation.

**Winch is not an encryption scheme.** The protocol assumes blobs are opaque and
encrypted end-to-end by the endpoints. Nodes perform no cryptographic operations.

**Winch is not a high-throughput protocol.** One message in flight per channel at a time
is a hard structural property. Winch is the right choice when confirmed single-payload
delivery matters more than throughput.

For use cases where the producer cannot stall on consumer availability — log shipping,
telemetry, continuous data streams — see the
[Ring protocol](../ring-protocol).

---

## Specification Versions

| Version | Summary |
|---------|---------|
| v0.1 | Initial: asymmetric ClientA/ClientB, GCS via Cloud Run, AES-256-GCM PSK |
| v0.2 | Symmetric peers, generic slot names, write guard on occupied slot |
| v0.3 | ChannelProvider abstraction, pluggable backends (GCS, Dropbox, local, httprelay) |
| v0.4 | Channel as first-class concept, Node primitive, Winch propagation protocol, peek() |
| v0.5 | Monorepo package structure, crypto isolation enforcement at package dependency level |

---

## Contributing

This repository contains the Winch protocol specification only. Implementation
contributions belong in the relevant implementation repository (e.g.,
[dropchannel/dropchannel-py](../dropchannel-py)).

When proposing changes to this specification, please distinguish between:

- **Clarifications** — making the existing protocol more precisely described
- **Extensions** — adding new capabilities while preserving backward compatibility
- **Revisions** — changing existing behavior (requires a new spec version)

---

## License

Protocol specification: [Creative Commons CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/)
(public domain dedication — implement freely without restriction)
