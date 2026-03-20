# tide-protocol — Working Notes

## What this repo is

The specification for the Tide coordination protocol. Tide is one protocol in the
DropChannel runtime. System-level concerns (ChannelProvider interface, encryption,
protocol dispatch) live in `dropchannel/spec`.

## Current state

Canonical spec (`README.md`) is complete through v0.6. Version history is complete:
`history/v0.1.md` through `history/v0.6.md` all exist and are protocol-only (no
Python implementation content).

The canonical README covers:

- Conceptual model (Channel, physical pipeline, Endpoint, Node)
- ChannelProvider interface (primary + meta slot operations)
- Propagation protocol (forward pass, ACK cascade, key properties)
- Node lifecycle state machine (startup inspection, recv-polling, send-polling)
- Heartbeat protocol (meta slots, node cycle, client heartbeat, startup purge)
- Configuration reference (endpoint and node env vars, directional namespacing)
- Backward compatibility notes
- Version history table with links
- Out of scope list

## What does not exist yet

- `history/v0.7.md` and beyond — no v0.7 spec has been written

## How to start a new session here

Paste this file and `README.md` into a new conversation. For protocol changes, describe
what you want to add or modify and why. The output will be a new `history/v0.x.md` and
an updated `README.md`.

Typical prompt:
> "Read WORKING.md and README.md. I want to spec the following change to the Tide
> protocol: [description]. Generate history/v0.x.md and an updated README.md."
