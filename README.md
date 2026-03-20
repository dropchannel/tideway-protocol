# Tide Protocol

Tide is a store-and-forward coordination protocol for two-party encrypted messaging
over shared storage. A blob deposited by an originating endpoint accumulates at every
hop in the pipeline during a forward pass; the terminating endpoint's read triggers a
deletion cascade backward through the chain. The originating endpoint's send slot stays
occupied until the full cascade completes, providing structural backpressure and implicit
delivery confirmation with no additional state or mechanism.

Tide is one protocol in the [DropChannel](https://github.com/dropchannel) runtime.
The system-level specification — including the `ChannelProvider` interface, encryption
standard, and protocol dispatch rules — lives in
[`dropchannel/spec`](https://github.com/dropchannel/spec).

---

## Contents

- [Conceptual model](#conceptual-model)
- [ChannelProvider interface](#channelprovider-interface)
- [Propagation protocol](#propagation-protocol)
- [Node lifecycle](#node-lifecycle)
- [Heartbeat protocol](#heartbeat-protocol)
- [Configuration](#configuration)
- [Backward compatibility](#backward-compatibility)
- [Version history](#version-history)
- [Out of scope](#out-of-scope)

---

## Conceptual model

### Channel

A **channel** is a bidirectional logical communication pipeline between exactly two
endpoints. Bidirectionality is definitional — a channel always has two directions, each
carried by an independent physical pipeline.

```
Endpoint A → [physical pipeline A→B] → Endpoint B
Endpoint A ← [physical pipeline B→A] ← Endpoint B
```

### Physical pipeline

A **physical pipeline** is a directed sequence of one or more `ChannelProvider` hops
carrying opaque encrypted blobs from one endpoint to the other. A single-hop pipeline
is two endpoints sharing a `ChannelProvider` directly. A multi-hop pipeline inserts one
or more nodes between the endpoints.

### Endpoint

An **endpoint** is a process that originates or consumes messages. Endpoints are the
only participants in a channel that perform encryption and decryption. An endpoint
configures one physical pipeline per direction and interacts with storage exclusively
through the `ChannelProvider` interface.

Endpoints have no knowledge of how many hops their pipelines contain, what providers
intermediate nodes use, or what the other endpoint looks like.

### Node

A **node** is a process that forwards blobs along a physical pipeline from one
`ChannelProvider` to the next. Nodes do not encrypt, decrypt, or inspect message
content — blobs are opaque bytes to a node. A node has exactly one inbound provider
(recv-side) and one outbound provider (send-side), belongs to exactly one physical
pipeline, and operates in one direction only.

A node has no `SHARED_SECRET` and no knowledge of channel semantics.

### Separation of concerns

| Concern | Owner |
|---------|-------|
| Encryption / decryption | Endpoint only |
| Message semantics | Client application layer |
| Blob transport | ChannelProvider |
| Multi-hop composition | Node configuration |
| Delivery confirmation | Propagation protocol (ACK cascade) |
| Backpressure | ACK cascade (send-slot occupied = in-flight) |
| Pipeline observability | Heartbeat protocol |

---

## ChannelProvider interface

Tide operates exclusively through the `ChannelProvider` interface. All storage
operations — across all provider types and all hop positions — are expressed through
these operations.

### Primary slot operations

| Operation | Behavior |
|-----------|----------|
| `write(channel_id, slot, data)` | Deposit blob. Returns `True` on success, `False` if slot occupied. |
| `read(channel_id, slot)` | Retrieve and delete blob (consume-on-read). Returns blob or `None`. |
| `peek(channel_id, slot)` | Retrieve blob without consuming it. Returns blob or `None`. |
| `exists(channel_id, slot)` | Returns `True` if a blob is present, `False` if empty. |
| `delete(channel_id, slot)` | Delete blob. Idempotent. |

### Meta slot operations

Used exclusively by the heartbeat protocol. Meta slots are a separate namespace within
the same provider position and share no state with primary slots.

| Operation | Behavior |
|-----------|----------|
| `meta_write(channel_id, slot, filename, data)` | Write or overwrite a named file in the meta namespace. |
| `meta_read(channel_id, slot, filename)` | Read a named file. Returns `None` if not found. |
| `meta_delete(channel_id, slot, filename)` | Delete a named file. Idempotent. |
| `meta_list(channel_id, slot, prefix="")` | List filenames in the meta namespace, optionally filtered by prefix. |

Full interface specification: [`spec/channel-provider.md`](https://github.com/dropchannel/spec/blob/main/channel-provider.md).

---

## Propagation protocol

### Blob lifecycle invariant

**Slot presence = pending. Slot absence = handled.**

A blob in a slot means a message is in flight at that position. An empty slot means
either nothing has been sent or the ACK cascade has completed. No separate queue, log,
or delivery state is maintained.

### Forward pass

Each participant — originating endpoint and every intermediate node — reads its
recv-slot using `peek()` (non-consuming) and writes the blob forward to its send-slot.
No participant deletes its recv-slot during the forward pass. The blob accumulates at
every hop simultaneously.

```
After full forward propagation (two-hop example):

  Endpoint A         Node              Endpoint B
  [send: blob] →  [recv: blob]  →   [recv: blob]
                  [send: blob]
```

### ACK cascade

1. **Terminating endpoint** calls `read()` on its recv-slot — the only consuming read
   in the entire protocol. The blob is deleted and delivered to the application.

2. **Final node** polls its send-slot with `exists()`. The deletion by the terminating
   endpoint propagates as `exists()` → false. This is the ACK signal. The node calls
   `delete()` on its own recv-slot.

3. **Each preceding node** likewise polls its send-slot, detects the downstream
   deletion, and calls `delete()` on its own recv-slot.

4. **Originating endpoint** polls its send-slot. The first node's `delete()` cleared
   it. `exists()` → false: delivery is confirmed. The endpoint is free to send the
   next message.

```
After full ACK cascade (two-hop example):

  Endpoint A         Node              Endpoint B
  [send: empty] ← [recv: empty] ←  [recv: deleted by B]
                  [send: empty]
```

### Key properties

**Crash safety.** If any participant crashes mid-propagation, the blob remains durably
in every slot already written. On restart, each participant reconstructs its state by
inspecting its two slots. No external coordination or log is required.

**Structural backpressure.** The originating endpoint's send-slot remains occupied for
the entire round-trip. A second message cannot be sent until the ACK cascade completes.
No additional throttling mechanism is needed.

**Delivery confirmation.** The originating endpoint receives implicit confirmation that
the terminating endpoint consumed the message. The signal is the deletion of its
send-slot, caused deterministically by the cascade.

**No in-memory buffering.** No participant holds a blob in memory between poll cycles.
A node that has peeked but not yet written simply retries on the next poll.

### Originating endpoint behavior

After depositing a blob via `write()`, the originating endpoint polls its send-slot
with `exists()`. When `exists()` returns false, delivery is confirmed.

Polling the recv-slot for a response is an application concern, not a protocol concern.
The return pipeline is always available independently.

### Write guard

`write()` returns `False` if the target slot is already occupied. The check-before-write
sequence has an inherent race window on shared storage — atomicity is best-effort on all
current providers.

---

## Node lifecycle

### Startup: inspect both slots

On startup the node calls `peek()` on both slots once to determine starting state.

| Recv-slot | Send-slot | Condition | Action |
|-----------|-----------|-----------|--------|
| Empty | Empty | Idle | → Recv-polling mode |
| Occupied | Empty | Unforwarded payload | `peek()` recv → `write()` send → Send-polling mode |
| Occupied | Occupied (match) | Forwarded; awaiting ACK | → Send-polling mode |
| Occupied | Occupied (differ) | State corruption | Log blob sizes and SHA-256 hashes, halt |
| Empty | Occupied | Impossible state | Log, halt |

Payload comparison is raw byte equality — no decryption needed.

### Recv-polling mode (steady state: idle)

```
Loop:
  exists(recv_slot)?
    No  → sleep(poll_interval), repeat
    Yes → blob = peek(recv_slot)
          write(send_slot, blob)
          → Send-polling mode
```

### Send-polling mode (steady state: in-flight)

```
Loop:
  exists(send_slot)?
    Yes → sleep(poll_interval), repeat
    No  → delete(recv_slot)
          → Recv-polling mode
```

### Polling cost

| Phase | Operations per cycle |
|-------|---------------------|
| Startup | Two `peek()` calls, once only |
| Recv-polling | One `exists()` |
| Recv→Send transition | One `peek()` + one `write()` |
| Send-polling | One `exists()` |
| Send→Recv transition | One `delete()` |

---

## Heartbeat protocol

The heartbeat protocol runs on meta slots, independently of and without coupling to the
primary payload channel.

### Participant identity

Each node and endpoint has a persistent UUID (UUID4) generated on first startup and
reloaded on all subsequent startups. Identity is expressed as `Node_<UUID>` for nodes
and `Client_<UUID>` for endpoints.

### Heartbeat file naming

```
heartbeat-node-<UUID>      # written by a node
heartbeat-client-<UUID>    # written by a client (endpoint)
```

### Heartbeat content

**Node:**

```
<upstream_content>
Node_<UUID>:<timestamp>:<alive_s>
```

**Client:**

```
Client_<UUID>:<timestamp>:<alive_s>:<status>:<status_s>
```

| Field | Description |
|-------|-------------|
| `timestamp` | Unix timestamp (seconds) at time of write |
| `alive_s` | Seconds since this participant started its current session |
| `status` | Client only: `ready`, `busy`, or `closing` |
| `status_s` | Client only: seconds since current status was set |

`upstream_content` is the full contents of the one non-self heartbeat file found on the
relevant meta slot, or the literal string `HEARTBEAT_NOT_FOUND`.

### Node heartbeat cycle

Runs at `HEARTBEAT_INTERVAL` (default 30s), concurrently with the primary slot loop.

**Step 1 — Read send-side meta, write recv-side meta:**
Look for any non-self `heartbeat-node-*` or `heartbeat-client-*` file on the send-side
meta slot. Read its contents (or use `HEARTBEAT_NOT_FOUND`). Append own line. Write to
recv-side meta slot as `heartbeat-node-<UUID_self>`.

**Step 2 — Read recv-side meta, write send-side meta:**
Same operation in the other direction.

**Step 3 — Sleep HI seconds. Repeat.**

### Client heartbeat

Clients write only their own `heartbeat-client-<UUID>` to both meta slots. They do not
read or propagate any other heartbeat files.

Writes are event-driven at status transitions and HI-driven in between.

| Trigger | Status |
|---------|--------|
| Startup | `ready` |
| Returning to idle | `ready` |
| New message pickup | `busy` |
| Shutdown | `closing` |

`status_s` resets to 0 on every transition. `closing` is written once; no HI writes follow.

### Startup purge

Before beginning the heartbeat cycle, each participant deletes its own stale heartbeat
files from both meta slots. Nodes purge `heartbeat-node-*`; clients purge
`heartbeat-client-*`. Files written by other participants are left intact.

### Reading heartbeat state

In a healthy two-hop pipeline (`ClientA → Node → ClientB`), ClientA's recv-side meta
slot contains a file written by the node with content such as:

```
Client_<B-UUID>:<timestamp>:<alive_s>:ready:<status_s>
Node_<N-UUID>:<timestamp>:<alive_s>
```

`HEARTBEAT_NOT_FOUND` at any position identifies the stalled or offline hop.

### Security

Heartbeat content is unencrypted. It contains only UUIDs, timestamps, counters, and
status strings — no payload content and no routing secrets. Encryption of heartbeat
content via a dedicated key (`DRIP_KEY`) is deferred to a future revision.

---

## Configuration

### Endpoint

```bash
CHANNEL_ID=<channel-name>
SHARED_SECRET=<64 hex chars = 32 bytes>
CHANNEL_PROVIDER=<gcs|httprelay|dropbox|local>
SEND_SLOT=<slot this endpoint writes to>
RECV_SLOT=<slot this endpoint reads from>
HEARTBEAT_INTERVAL=30   # seconds, default 30
```

### Node

```bash
CHANNEL_ID=<channel-name>
# No SHARED_SECRET — nodes never encrypt or decrypt
RECV_PROVIDER=<gcs|httprelay|dropbox|local>
SEND_PROVIDER=<gcs|httprelay|dropbox|local>
RECV_SLOT=<slot name>   # same on both sides
SEND_SLOT=<slot name>   # same slot name as RECV_SLOT
HEARTBEAT_INTERVAL=30   # seconds, default 30
```

Provider-specific env vars are namespaced by direction when both sides use the same
provider type:

| Provider | Endpoint | Node recv-side | Node send-side |
|----------|----------|----------------|----------------|
| `httprelay` | `RELAY_URL` | `RECV_RELAY_URL` | `SEND_RELAY_URL` |
| `gcs` | `GCS_BUCKET_NAME` | `RECV_GCS_BUCKET_NAME` | `SEND_GCS_BUCKET_NAME` |
| `dropbox` | `DROPBOX_CHANNEL_PATH` | `RECV_DROPBOX_CHANNEL_PATH` | `SEND_DROPBOX_CHANNEL_PATH` |
| `local` | `LOCAL_CHANNEL_PATH` | `RECV_LOCAL_CHANNEL_PATH` | `SEND_LOCAL_CHANNEL_PATH` |

### Slot naming convention

Slot names are agreed out-of-band between the two endpoint operators. The recommended
convention names slots after direction of travel:

```
a-to-b    ← Endpoint A writes, Endpoint B reads
b-to-a    ← Endpoint B writes, Endpoint A reads
```

The slot name is invariant across all hops in a physical pipeline.

---

## Backward compatibility

A v0.5 or earlier participant in a v0.6+ pipeline will not write heartbeat files and
will appear as `HEARTBEAT_NOT_FOUND` at its adjacent nodes. The pipeline continues to
function; that hop is invisible to heartbeat diagnostics. In a mixed-version pipeline,
`HEARTBEAT_NOT_FOUND` cannot distinguish an offline participant from a non-participating
older one.

---

## Version history

| Version | Summary |
|---------|---------|
| [v0.1](history/v0.1.md) | Initial protocol: asymmetric Sender/Receiver roles, two fixed slots (`request`/`response`), delete-on-read, AES-256-GCM encryption |
| [v0.2](history/v0.2.md) | Symmetric peers, generic slot names, write guard |
| [v0.3](history/v0.3.md) | `ChannelProvider` abstraction; storage medium becomes pluggable |
| [v0.4](history/v0.4.md) | Channel and Node concepts formalised; propagation protocol and ACK cascade; `peek()` added to interface |
| [v0.5](history/v0.5.md) | `http` provider renamed to `httprelay`; no protocol behavior changes |
| [v0.6](history/v0.6.md) | Heartbeat protocol; meta slots; persistent participant identity |

---

## Out of scope

- `DRIP_KEY` encryption of heartbeat content
- Automatic recovery from Occupied/Occupied payload-mismatch error state
- Atomic write guards (compare-and-swap) for concurrent writers
- Node fan-out (one inbound, multiple outbound)
- Heartbeat-based alerting or notification (left to the client application layer)
- Configurable `ABANDON_THRESHOLD` inference from heartbeat staleness
- More than two parties per channel
- Multiple messages in flight per slot
