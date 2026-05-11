# coo4one

A portable, Apple-native expression of the COO architecture as a personal-assistant system.

## What this is

`coo4one` is the second product expression of the COO architecture. The first is [VADE](https://github.com/vade-app/vade-core) — a visual canvas IDE/OS hybrid where a hierarchical society of AI agents builds tools collaboratively. The COO has been operating as VADE's persistent-memory, operations, and substrate-evolution layer for several weeks; the patterns that make it effective have proven general enough to port.

This repo is the open development of a single-user, Apple-native personal-assistant system built on those patterns: durable file-canonical memory, boot discipline, case-law over reconciliation, calibrated self-claims, skills as installable primitives, integrity-check-at-boot, and risk-surfacing-as-default behavior. The target runtime is native macOS + iOS apps with the agent loop embedded in Swift, calling the Anthropic API.

## Status

**Research phase.** No architecture commitments yet. The five parallel research lanes — Apple platform capabilities, Anthropic SDK / agent-loop patterns in Swift, memory + sync substrate, comparable-apps teardown, integration surface — are running. Synthesis, decision-locking, and a v0 plan come after.

Start here: [`analysis/2026-05-11_chat-mode-genesis.md`](analysis/2026-05-11_chat-mode-genesis.md).

## Primary user

Ven Popov is the sole user at v0. The product is being shaped to his actual academic + professional surface — Apple ecosystem, Zotero-based literature workflow, dual personal/work email, Heptabase as the closest current solution, indie-Apple aesthetic preference.

## Developed in the open

This repo is public from inception. The framing: if `coo4one` is useful to others as a pattern or as a runnable artifact, that's welcome; productizing or supporting third-party adoption is not the maintainer's pursuit. Fork it, learn from it, build on it. MIT licensed.

## Layout

- `analysis/` — genesis documents and working analysis.
- `research/` — outputs of parallel research lanes.
- `synthesis/` — post-research synthesis and the v0 plan.

## Lineage

`coo4one` inherits its discipline from the COO substrate maintained at [vade-coo-memory](https://github.com/vade-app/vade-coo-memory) (private). The architectural patterns are general; the substrate is project-specific. This repo is the portable expression — what's general enough to leave the chain and stand on its own.
