# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## What this repo is

The specification for the **Tideway protocol** — one protocol in the DropChannel
waterway system. No source code, build system, or tests. Markdown specs only.

System-level concerns (DockProvider interface, encryption standard, security model,
Agent/Worker runtime, observability) live in `dropchannel/spec` and are not owned here.

**Naming note:** This repo is being renamed. The protocol was previously called
*Winch* (prefix `winch-`), and is now called *Tideway* (prefix `tideway-`).
History files (v0.1.md -v0.6.md) use old names — do not update them.

## Scope

This repo owns exactly one thing: the **Tideway state machine specification**.

### What belongs here

- Participant roles and behaviors (Endpoint, Raft)
- Waterway file model — how many files, naming convention, what presence/absence means
- Forward pass mechanics — which Dock a Raft reads from and writes to, and when
- Return pass and ACK mechanics — what triggers reversal or cascade, and how
- Head/tail Dock role semantics — how a Raft resolves direction on startup or restart
- Protocol-specific error conditions and halt rules
- Version history

### What does not belong here

- DockProvider interface definition — see `dropchannel/spec/channel-provider.md`
- Encryption details — see `dropchannel/spec/encryption.md`
- Security model — see `dropchannel/spec/security-model.md`
- Agent/Worker config schema — see `dropchannel/spec/agent.md`
- Heartbeat/telemetry/observability — see `dropchannel/spec/`
- Dock backend specifics (GCS, Dropbox, local, httprelay)
- Implementation guidance, configuration, or operational concerns

**The clean test:** a Raft implementor should need only this repo and
`dropchannel/spec/channel-provider.md` to implement Tideway correctly.

## Working with this repo

The canonical spec is `README.md`. Version history snapshots live in `history/v0.x.md`.

**Current state:** `README.md` is current through v0.7. `history/v0.1.md` through
`history/v0.7.md` exist.

**To propose a protocol change:** Read `README.md`, then produce a new
`history/v0.x.md` and an updated `README.md`.

Typical prompt:
> "Read README.md. I want to spec the following change to Tideway: [description].
> Generate history/v0.x.md and an updated README.md."

**History files record changes only — not the full protocol.** A history file must
contain: the version, what changed from the prior version, the design rationale, and
any compatibility notes. It must not restate unchanged mechanics. The full protocol
definition at any point in time is `README.md`.

## Vocabulary

This repo uses DropChannel waterway terminology throughout. Do not use retired terms.

| Term | Retired term |
|---|---|
| Raft | Node |
| Dock / DockProvider | ChannelProvider |
| Waterway | slot, reach |
| upper_dock / lower_dock | recv_backend / send_backend |
| Channel | channel_id (as namespace) |
