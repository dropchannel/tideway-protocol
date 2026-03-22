# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A protocol specification repository for the **Tide coordination protocol** — a store-and-forward messaging protocol for two-party encrypted communication over shared storage. Tide is one of several protocols in the DropChannel runtime. There is no source code, build system, or tests — only Markdown specifications.

System-level concerns (ChannelProvider interface, encryption, protocol dispatch) live in the separate `dropchannel/spec` repository.

## Working with this repo

The canonical spec is `README.md`. Version history snapshots live in `history/v0.x.md`. The `WORKING.md` file contains session instructions and current state.

**Current state:** Canonical spec (`README.md`) is complete through v0.6. `history/v0.1.md` through `history/v0.6.md` all exist. No v0.7 has been written yet.

**To propose a protocol change:** Read `README.md`, then produce a new `history/v0.x.md` and an updated `README.md`. History files are protocol-only — no implementation content.

Typical prompt pattern (from WORKING.md):
> "Read README.md. I want to spec the following change to the Tide protocol: [description]. Generate history/v0.x.md and an updated README.md."

## Architecture overview

**Core participants:**

- **Endpoint** — Originates or consumes messages; the only participant that encrypts/decrypts. Configured with `SHARED_SECRET`.
- **Node** — Intermediate forwarder; handles opaque blobs, never sees plaintext. No `SHARED_SECRET`.
- **Channel** — Bidirectional logical pipeline between exactly two endpoints.
- **ChannelProvider** — Abstraction over storage backends (gcs, httprelay, dropbox, local).

**Protocol layers (in README order):**

1. **ChannelProvider interface** — Primary slots (`write`, `read`, `peek`, `exists`, `delete`) and meta slots (`meta_write`, `meta_read`, `meta_delete`, `meta_list`).
2. **Propagation protocol** — Forward pass (peek-and-write blob chain) + ACK cascade (deletion cascade backward confirming delivery).
3. **Node lifecycle state machine** — Startup inspection → recv-polling → send-polling.
4. **Heartbeat protocol** (v0.6+) — Meta slot layer for liveness; per-hop visibility; participant UUIDs; statuses: `ready`, `busy`, `closing`.

**Configuration env vars** (documented in README, not used here):

- Endpoints: `CHANNEL_ID`, `SHARED_SECRET`, `CHANNEL_PROVIDER`, `SEND_SLOT`, `RECV_SLOT`, `HEARTBEAT_INTERVAL`
- Nodes: same minus `SHARED_SECRET`; directional namespacing for multi-provider hops
