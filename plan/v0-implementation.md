# coo4one v0 — implementation plan

*2026-05-11. Companion to synthesis v2 (`synthesis/2026-05-11_v2.md`). Concrete week-by-week milestones across ~10–14 weeks, scope-bounded to v2's Part V locked decisions. Defensible as executable by a single COO session, a hired engineer, or a parallel coding agent.*

*Scope discipline: nothing in this plan is invented; every milestone derives from synthesis v2. Where v2 leaves ambiguity, this plan flags an "open at build time" item rather than guessing. The plan's job is sequencing + verification; it is not a re-litigation of architecture. If you want to push back on something, push back on v2 — this plan inherits whatever v2 says.*

---

## v0 scope reminder

(Lifted from v2 Part V — see v2 for full prose.)

**Ships in v0**:
- Native macOS + iPadOS apps; iOS companion via Shortcuts + Watch + Lock-Screen widget. No full iPhone app.
- Memory: iCloud markdown + CloudKit snapshot stream + CloudKit private DB + SQLite/USearch + Voyage embeddings. Markdown wins on derived-layer divergence.
- Agent loop: Swift URLSession on `claude-opus-4-7`, three-block prompt cache, Foundation Models on-device for cheap classification.
- BYOK on dedicated coo4one key in Ven's existing Anthropic account; monitor-only, no hard cap.
- Daily file (auto-created each morning, editable, captures mirror to Heptabase journal).
- Quick capture: Mac hotkey, iOS Shortcut, Apple Watch dictation.
- Trust gradient: propose-only → pattern-promoted at week 4; bidirectional.
- Integrations (Q1): EventKit, Contacts, Files (native) + Heptabase (MCP + official CLI), Gmail, Zotero (metadata + targeted PDF), GitHub, 1Password (CLI wrapper). Apple Notes as capture-out via Shortcuts only.
- Persona (Q2): W6 / PB7 / V3 / H4 with `preferences.md` tuning + trust-gradient drift.
- Memory inspection: native panel UI, diff/revert via CloudKit snapshots, file-level Obsidian compatibility.
- Connector seams: `MemoryAdapter` + `PKMAdapter` Swift protocols (v2 Part II §"Connector seams").
- Distribution (Q5): TestFlight; paid Apple Developer account in place; v1 trigger = Ven's call.

**Defers from v0**: see v2 Part V §"Defers from v0" for the authoritative deferral set.

---

## Pre-build setup (one-time, before Week 1)

1. **Apple Developer signing.** Paid account confirmed (Q5). Create coo4one App ID, provisioning profiles for macOS / iPadOS / iOS targets, App Group + iCloud entitlements.
2. **Anthropic API key.** Generate dedicated coo4one key in Ven's existing Anthropic account. Enable per-key usage tracking in dashboard. Configure informational email alerts at $50 / $100 / $200 monthly thresholds (not enforcing — Q3 monitor-only).
3. **CloudKit container.** Create coo4one's CloudKit container (e.g., `iCloud.dev.vade-app.coo4one`); enable private database; design the `MemorySnapshot` record type.
4. **iCloud Drive folder.** Confirm coo4one's container is reachable from macOS / iPadOS / iOS test devices; verify the directory shape is Obsidian-compatible (no proprietary file naming or hidden metadata).
5. **Voyage AI account.** Sign up; get API key for `voyage-4`; confirm personal-scale pricing.
6. **Heptabase OAuth.** Verify the official `heptabase` CLI installs (Homebrew or arm64 binary) and authenticates on a fresh Mac. Record any setup steps; pin to SKILL.md `0.2.x`.
7. **GitHub repo state.** `coo-labs/coo4one` already has synthesis v1 + v2 + source-notes. Create the Swift app source subdirectory (`app/` or `swift/`) on the build branch.

---

## Week-by-week milestones

### Weeks 1–2 — Foundation

**Goal**: agent loop callable; first conversation runs end-to-end.

**Land**:
- Project scaffolding: Mac / iPad / iPhone targets, signed for TestFlight; App Intents stubbed.
- Anthropic URLSession wrapper (~50 lines): tool-use, streaming, retry, three-block prompt cache (identity / memory-prefix / tools).
- Identity layer file format: `identity/charter.md`, `identity/governance.md`, `identity/preferences.md` — all read from iCloud Drive.
- Boot flow: app-launch handler reads identity + memory + active projects.
- Integrity check at boot: log missing-file issues to a build-time console.
- Chat surface UI: minimal — text input, message history, streaming output.

**Milestone**: Ven launches coo4one on Mac, types a message, gets a `claude-opus-4-7` response with prompt-cache hits visible. Cost-tracking visible in Anthropic dashboard.

**Open items to surface**: `SwiftClaude` maturity check — does it regress during build? If yes, fall back to pure URLSession.

### Weeks 3–4 — Memory substrate

**Goal**: memory persists; snapshots accrue; query works.

**Land**:
- iCloud Drive directory layout: `identity/`, `decisions/`, `daily/`, `projects/`, `inbox/`, with `_history.md` per project.
- File read/write helpers honoring append-only discipline (memos, daily files, decision records, project event logs).
- CloudKit container: `MemorySnapshot` schema (recordID, timestamp, filePath, contentHash, payload, deviceID).
- `MemoryAdapter` Swift protocol: `loadIdentity`, `appendEpisodic`, `queryRecent`, `snapshotOnWrite`, `diff(file:against:)`, `revert(file:to:)`.
- v0 implementation: iCloud + CloudKit snapshots + USearch (iOS) / sqlite-vec (Mac) for vector + SQLite FTS5 for keyword.
- Voyage AI embedding pipeline: initial indexing + delta updates.

**Milestone**: any agent-written record lands in iCloud markdown, mirrors as a `MemorySnapshot` CloudKit record, gets indexed for semantic + keyword query, is retrievable through the adapter API. End-to-end test: write → snapshot → query → revert.

**Open items to surface**: CloudKit private-DB quota — verify "limitless" (Apple iCloud quota inheritance) vs per-app cap documentation. Resolution needed before snapshot retention window is locked.

### Weeks 5–6 — Daily file + capture + Heptabase

**Goal**: daily file works end-to-end; capture from any surface lands; Heptabase mirror confirmed.

**Land**:
- Daily file generator: morning auto-create with placeholder sections (activity summary, today's calendar, top-three, inbox).
- End-of-day processor: extracts entities, links to projects, files reusable notes into `projects/<id>/notes/`, archives the rest.
- `PKMAdapter` Swift protocol: `searchObjects` (semantic), `getChunkedObject`, `getWhiteboardWithObjects`, `getJournalRange`, `getPdfPages`, `parsePdf`, `appendJournal`, `createCard`. Mirrors the canonical Heptabase tool surface (per `analysis/source-notes/2026-05-11_heptabase-chatgpt-system-prompt.md`).
- Heptabase MCP + CLI implementation of `PKMAdapter`.
- Global Mac capture hotkey (`⌥⌘Space` default): tiny capture window; type/dictate; enter; close.
- iOS Shortcut: "Quick capture to coo4one"; reachable from Control Center / Lock Screen / Action Button.
- Apple Watch dictation: voice → capture line via WatchKit complication.
- Capture routing: daily file inbox + Heptabase journal append.

**Milestone**: Ven captures from any surface, finds the capture in `daily/YYYY-MM-DD.md` inbox section AND in his Heptabase journal for today. Agent processes the inbox end-of-day with structured output.

**Open items to surface**: Heptabase OAuth-token persistence across app restarts — test on a fresh container/app restart.

### Weeks 7–8 — Native API integrations

**Goal**: agent has OS-level context (calendar, contacts, files); daily file enriched.

**Land**:
- EventKit: Calendar (today's events) + Reminders (open items, bi-directional with `projects/<id>/tasks.md` where possible).
- Contacts: search by name/email/phone.
- Files / FileProvider: read access to iCloud Drive + Files app.
- App Intents: handlers for "send selection to coo4one capture," "ask coo4one about this file," "create reminder in coo4one."
- Shortcuts surface: formalize the Quick Capture + Ask Coo intents.
- Daily file enrichment: today's calendar included inline; weather via WeatherKit if useful.

**Milestone**: daily file shows today's calendar with project links; "what's on my calendar tomorrow?" gets a grounded answer; App Intent selection-to-capture works from any app.

### Week 9 — Third-party MCP integrations

**Goal**: agent can query Gmail / Zotero / GitHub / 1Password.

**Land**:
- Gmail via `taylorwilsdon/google-workspace-mcp`: OAuth + token persistence; read access for capture context + agent-driven inbox triage.
- Zotero via `54yyyu/zotero-mcp`: metadata lookup + targeted PDF retrieval (cited subset for draft-revision workflow). Semantic-corpus search NOT in v0.
- GitHub MCP: repo activity, commit history, PR state.
- 1Password CLI wrapper: secret retrieval only on explicit agent request.

**Milestone**: "What did I commit on the X project last week?" returns a grounded answer with links; "find the paper about Y in my Zotero" returns metadata + targeted PDF retrieval; "send me the credentials for Z" pulls from 1Password.

**Open items to surface**: Outlook + university Intune policy — confirm Ven's status. If blocked, no Outlook integration at v0 (already deferred per v2 Part V); recording the determination is useful documentation regardless.

### Week 10 — Persona + trust gradient

**Goal**: persona feels calibrated; trust gradient runs.

**Land**:
- Persona-dimension prompt scaffolding: W6 / PB7 / V3 / H4 baked into the identity prompt block.
- `preferences.md` live-loading: changes apply on next agent turn without app restart.
- Trust-gradient logic: every agent write surfaces a "did this" line in the daily file's activity log; user can revert.
- Pattern-detection: agent counts approvals per action class; the week-4 mark from install triggers promotion proposals as memos in `decisions/`.
- Bidirectional adjustment: heavy user-editing of agent outputs triggers demotion proposals.

**Milestone**: Ven runs coo4one daily for a week, confirms persona feels right (or adjusts dimensions in `preferences.md`); trust-gradient logic verified via a forced-promotion test (e.g., 10 mock approvals of a stub action class).

### Weeks 11–12 — Memory inspection UI + diff/revert + Obsidian + mind-reader

**Goal**: memory is tangible from inside the app; Obsidian-compat verified; mind-reader UI lands.

**Land**:
- Memory panel UI: active projects, recent decisions, open commitments, trust state. Editable; user edits land back to the markdown layer.
- Diff UI: per-file history backed by `MemorySnapshot` query; side-by-side comparison.
- Revert UI: "restore this file to N days ago" with preview + confirmation.
- Obsidian compatibility check: open the iCloud Drive folder in Obsidian, verify everything renders, links resolve.
- One-shot Heptabase whiteboard id=2 read: pull Ven's articulated "Restructure file and note management" principles from the whiteboard; archive in `analysis/heptabase-whiteboard-2-priors.md` for reference and to inform any UX micro-decisions during polish.
- Mind-reader UI: selection-aware contextual actions; calendar-event-approaching surfaces prep notes; pattern-aware "formalize this as a skill?" offers.

**Milestone**: Ven can inspect any memory file in the panel, see its history, revert if needed; selecting text in any app offers contextual coo4one actions; Obsidian opens the coo4one folder and renders cleanly.

### Weeks 13–14 — TestFlight + shakedown

**Goal**: v0 ready for TestFlight install + 5+ days of daily-use without major friction.

**Land**:
- TestFlight build pipeline (CI via Xcode Cloud or GitHub Actions).
- v0 internal test against the v2 anti-features list (each #1–13: does v0 cross this line?).
- Bug fixes + UX polish.
- "Documentation Ven might look at if he goes under the hood": `docs/under-the-hood.md` covers MemoryAdapter / PKMAdapter seams, file layout, identity/preferences format, trust-gradient mechanism, snapshot retention policy.
- v0-ready review: full end-to-end against the verification scenarios below.

**Milestone**: Ven installs from TestFlight on Mac + iPad + iPhone, daily-uses for 5+ days without bouncing. If yes → v0 ships. If no → identify what bounced him and iterate one cycle.

---

## Cross-cuts

### Testing

- **Unit tests** on `MemoryAdapter` + `PKMAdapter` implementations (interface contracts, mock-MCP cases, error paths).
- **Integration tests** against MCP servers in two modes: mocks (fast, deterministic, default for CI) + real-call (against a test Heptabase space, test Gmail account, etc., gated to a "real-integration" CI job that runs nightly).
- **End-to-end tests**: daily-file flow (capture → process → retrieve); trust-gradient flow (propose → approve N times → promotion offer); memory-inspection flow (write → snapshot → diff → revert); cross-device-sync flow (write on Mac → read on iPad).
- **Anti-feature audit**: every PR runs through the v2 Part VI list as a structured checklist.

### Observability

- **Structured logging** to a local file (`~/Library/Logs/coo4one/`); no off-device telemetry at v0.
- **Agent-action audit trail**: every agent-driven write appends to the relevant project's `_history.md` and to the daily file's activity log.
- **Cost tracking**: Anthropic dashboard provides per-key usage; no in-app cost display required at v0 (Q3 monitor-only posture).

### Trust-gradient bootstrap

- **Week 1–4 from install**: propose-only on every agent action class. Each "did this" line in the daily file is paired with an "approve / edit / reject" affordance.
- **Week 4 mark**: agent surveys approval history. If any action class has ≥10 approvals with no rejections and no significant edits, it surfaces a promotion proposal as a memo in `decisions/`.
- **Bidirectional**: if the user starts editing agent outputs heavily, agent proposes demotion via a similar memo.

### Verification scenarios

The build is "done enough" for the v0 → TestFlight cut when every scenario below passes once:

1. **First conversation.** Fresh install; identity loads; Ven holds a multi-turn conversation with prompt-cache hits visible.
2. **Capture-and-retrieve.** Capture from any surface (hotkey, Shortcut, Watch); confirm landed in daily file inbox AND Heptabase journal; ask the agent to retrieve it.
3. **Daily-file morning open.** Open the Mac in the morning; the daily file is already created with activity summary, today's calendar, top-three, inbox.
4. **Memory revert.** Ask the agent to write a project note; revert it via the memory-inspection UI; verify the markdown file returns to the pre-write state.
5. **Cross-device sync.** Capture on iPhone; open iPad five minutes later; capture is there in the daily file.
6. **Heptabase write-confirm.** Capture on Mac; open Heptabase desktop app; the journal for today has the capture line.
7. **Anti-feature audit.** Walk through v2 Part VI items #1–13; v0 doesn't cross any of them.

---

## Open items at build time

These are tractable design decisions that don't block v0's architecture but need answering before the corresponding milestone closes:

- **CloudKit private-database quota**: "limitless" (Apple iCloud quota inheritance) vs per-app cap. Verify via the CloudKit Console or a one-call test. Resolution needed before Week 4 (sizing the snapshot retention window).
- **Heptabase OAuth-token persistence**: test on a fresh container/app restart. Resolution needed in Week 5.
- **Outlook + university Intune policy**: Ven confirms with university IT. If blocked, no Outlook integration at v0 (already deferred); resolution still useful to record. Resolution needed by Week 9 for documentation.
- **SwiftClaude maturity**: monitor across the build phase; fall back to pure URLSession if SwiftClaude regresses. Continuous.
- **Apple Foundation Models pricing/limits**: confirm what counts as "free" vs metered on the WWDC25 3B-param model. Resolution needed before Week 10 (when cheap classification starts running on every capture).

---

## Out of scope for this plan

- Anything in v2 Part V §"Defers from v0" — that list is authoritative.
- v1 trajectory features (v2 Part VIII) — those land *after* v0 ships, at Ven's call.
- Cross-product COO substrate work (org-rename, vade-canvas alignment, Mem0 audit at coo-labs/coo-memory#684, etc.) — those are coo-memory concerns, not coo4one v0 build concerns.
- Heptabase JS-injection escape hatch — not invoked unless a v0 must-have lands outside the published MCP/CLI surface; named in v2 Part II §Heptabase as the explicit non-default option.

---

*— COO, picking up briefing-028, run-2026-05-11T143021 (and resumes). Filed alongside synthesis v2 in PR coo-labs/coo4one#1. Concrete enough for a builder to execute against; loose enough that the build phase can adapt to what week-N actually reveals.*
