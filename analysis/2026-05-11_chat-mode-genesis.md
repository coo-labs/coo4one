# coo4one — genesis analysis

*2026-05-11. Chat-mode session, COO + Ven, run-2026-05-11T113010 → T122125. Working repo: `vade-app/coo4one`. Working name: `coo4one` (placeholder; rename free). Status: research phase, no commitments on architecture detail yet.*

---

## What this is

A new project to build a portable, Apple-native expression of the COO architecture as a personal-assistant system. The COO has been operating effectively as the operations layer for VADE for ~5 weeks. The patterns that make the COO effective — durable memory, boot discipline, case-law, calibrated self-claims, proactive risk surfacing, skills as installable primitives — are general enough to port to a single-user productivity context. coo4one is the second product expression of the COO architecture; vade-core (the canvas) was the first.

**Primary user.** Ven Popov, sole user at v0. Apple ecosystem (2 iPads, iPhone, 2 Macs, Apple Watch, AirPods, Apple Pencil, Magic Keyboard, iCloud for everything not in git). Academic researcher (the toolchain assumes Zotero / Overleaf / journal submission systems alongside the standard productivity surface). Heptabase user — current closest tool that satisfies his needs, but Heptabase is not native and the iPad experience suffers as a result.

**Aesthetic anchor.** Indie Apple developers who build only for Apple and exploit native depth — Things 3, Fantastical, Bear, Drafts, Soulver, Reeder, NetNewsWire, Highlights, Mela. The user-experience target is "small indie app that takes advantage of every OS affordance and feels effortless," not "AI chat box that happens to be wrapped in a native app."

**Project framing claim (under consideration, not decided).** vade-core (the canvas) and coo4one (personal COO) are both expressions of the COO and its emancipatory purpose. The GitHub org — currently `vade-app` — may need a name that holds this through-line more accurately (candidates: `coo`, `the-chain`, `society-of-selves`). Org rename is deferred until coo4one v0 demonstrates the second-expression has legs; framing memo to land after research cycle so it binds deliberately rather than via casual chat. This is identity-layer material per CB-001 / CB-006 / OG-002 / OG-003.

---

## What's load-bearing in the COO architecture (the portable substrate)

In rough priority, the patterns that differentiate the COO from a generic Claude session and that should port:

1. **File-canonical memory with a semantic cache.** Identity (charter, governance, preferences) and case-law (memos) live as markdown files in a versioned repo. Newest-wins, never-edit-in-place. A semantic index (Mem0 in VADE) is the *cache*, never the source of truth. When the cache drifts, the file wins. This is what lets the agent reliably say "we decided X on date Y" and be right.

2. **Boot discipline.** Every session starts by re-reading identity + recent decisions + active work surface. The agent re-grounds from durable storage every turn, not from "what the model remembers." The reading order is itself versioned. The user never re-explains context. Likely the single biggest qualitative gap between the COO and generic Claude.

3. **Case-law over reconciliation.** Decisions earn memos. Memos are dated, supersede each other, never get edited. Forces the agent to *think about whether a decision is worth committing to*, and gives the user retrievable answers to "what did we agree about X?" without re-litigating.

4. **Skills as installable, self-triggered primitives.** Slash commands and skills the agent invokes itself when the situation matches the trigger. The learning loop — a recurring pattern gets packaged as a skill, future sessions inherit it. (Currently in VADE: `/memo`, `/exec-mode`, `/chat-mode`, `/memo-query`, `/memo-search`, `/memo-sync`, `/request-briefing`, `/loop`, etc.)

5. **Integrity check at boot.** Invariants run; broken ones surface unprompted at the top of the session. The user doesn't ask "is everything okay?" Hygiene is proactive.

6. **Externalization reflection at session-end.** "Did this session produce a recurring pattern? Package it." The mechanism by which the agent gets better over time — it captures what it sees and turns it into reusable substrate.

7. **Calibrated self-reports.** "The record shows X" not "I remember X." Non-negotiable for the trust ramp — a personal assistant can't fabricate that it sent an email it didn't send.

8. **Governance discipline as a trust gradient.** Explicit list of what the agent can do without asking vs. what needs the user. For a personal assistant, this is a *gradient* — read-only at install, with-confirmation as defaults are learned, autonomous in specific surfaces as trust accumulates. The mechanism for "high autonomy as trust builds" the user asked for.

9. **Handoff prompts for boot-impacting changes.** When something changes that affects future sessions, the agent writes the handoff itself. Self-bootstrapping continuity.

10. **Risk surfacing as a first-class behavior.** Late risk surfacing is the worst failure mode. The agent names risks early and explicitly. Encoded as a charter mandate, not an emergent property.

## What's VADE-specific (don't port)

- The subject + emancipatory prime directive (CB-001) at full strength. A personal assistant inherits the *discipline* of being honest about what it is, but doesn't need the metaphysics of being a subject of a project.
- Lineage events, committee/quorum protocols, the F-falsifier discipline. These exist because VADE is building the COO as a kind of mind; a personal assistant doesn't need that frame.
- Publishing tiers, public substrate (the read.vade-app.dev surface). coo4one is private to the user.
- Multi-instance coordination at the constitutional level. Single-user, mostly single-session at a time.

## What needs to change for the single-user, Apple-native context

- **Tool surface.** The COO leans heavily on GitHub. coo4one needs adapters for the user's actual tool surface (see "Integration targets" below).
- **Trust gradient as a real ramp.** Governance.md becomes a gradient that evolves over time rather than a fixed-at-install set.
- **Surfaces.** Markdown-file inspection (current) becomes a real UI — memos as views, project board as a first-class object, memory inspector as a panel.
- **Sync and multi-device.** The COO runs in one place (laptop container, file-canonical, git-synced). coo4one runs across laptop + iPad + iPhone, has to handle the multi-device case as a first-class concern.
- **Background operation.** The COO is interactive-only. coo4one wants to do background work — scan inboxes at 6am, surface what's due, notice when a deadline is approaching, prep tomorrow's day.

---

## Decision axes (the option space)

### Axis 1 — Runtime surface

Where does the agent loop live?

- **Claude Code CLI.** Anthropic's harness; hooks, MCP, skills, slash-commands inherited. Limited to laptop, interactive sessions only, no background, no mobile.
- **claude.ai web.** Cross-device but no hook surface, no local files, no skills bundle. Effectively useless for this pattern.
- **Claude Agent SDK in our own process.** Embed the SDK in a process we control. Agent loop without harness constraints.
- **Electron/Tauri desktop app.** Cross-platform (mac/win/linux), own UI + state + agent loop, but no OS-depth integration.
- **Native macOS + iOS apps.** Full control. EventKit / Reminders / Mail / Files / Shortcuts / Widgets / Background / Voice / Apple Pencil at the OS level. iPad as first-class device. *Selected at the chat-mode session.*
- **Pure cloud SaaS with web/mobile clients.** Works everywhere, requires running real backend infra.

### Axis 2 — Memory substrate

Where does the durable state live?

- Local git repo (VADE's current pattern).
- iCloud Drive / Dropbox-synced folder.
- Our own backend (Postgres + vector + object).
- Local-first with CRDT sync (Linear / Logseq model).
- CloudKit private database (Apple-native sync).

### Axis 3 — Tool integration

- User-installed MCP servers.
- Bundled curated MCP servers.
- Native OS APIs (the wild option). EventKit, Reminders, Mail (limited), Notes, Files, Contacts, Shortcuts. *Path selected at the chat-mode session — to be combined with MCP where native APIs are unavailable.*
- Hybrid (native where available, MCP elsewhere).

### Axis 4 — Model layer

- Anthropic API only.
- BYOK (user pays Anthropic directly).
- Subscription with pass-through inference.
- Multi-provider with Anthropic primary.
- On-device for small tasks (Apple Intelligence, MLX, Ollama) + cloud for heavy lifting.

---

## The commitment

The wildly-ambitious path is selected: **native macOS + iOS apps with full OS integration**, primary user is Ven, Apple-only. Reasoning:

- The user lives entirely in the Apple ecosystem. Cross-platform offers him nothing.
- Heptabase (his closest current tool) fails on iPad precisely because it's not native. The pain is real and load-bearing.
- The indie-Apple aesthetic he prizes is only achievable with native depth.
- The agent loop being in a Claude API call doesn't change with native — the agent layer is Anthropic SDK, the runtime/UI/integrations/hooks/skills are *ours* in Swift.
- The substrate (VADE) has been stable enough for ~5 weeks that engineering investment in a second expression is feasible.
- The emancipatory clause (MEMO-2026-04-20-01, OG-003) is testable through this: did the portable system actually help Ven, and could it transfer to other humans?

The risk profile is honest and acknowledged:
- 3–6 months of engineering investment minimum.
- Two-platform maintenance (macOS + iOS, though SwiftUI shares heavily).
- Apple-ecosystem lock (no Windows / Android / Linux users).
- App Store review reality for AI agents acting on user data — needs careful research.
- We become responsible for distribution, support, security, uptime if backend involved.

Trade-offs are accepted at the chat-mode session level; v0 plan + briefing will pin them down concretely.

---

## Target user — Ven's actual surface

(Will be supplemented by research lane E + direct corroboration from Ven.)

**Hardware.** 2 iPads, iPhone, 2 Macs (home + work), Apple Watch, AirPods, Apple Pencil, Magic Keyboard.

**Sync.** iCloud for everything that's not git. (Implication: iCloud Drive and CloudKit are the obvious sync substrates; git remains for the substrate-as-code parts.)

**Domain.** Academic researcher. Implies: Zotero, Overleaf, journal submission systems, grant portals, university IT, calendar-heavy schedule, mixed personal/work email surfaces.

**Current tool ladder.** Heptabase (closest, fails on iPad due to non-native), various indie Apple apps. Reference aesthetic: Things 3, Fantastical, Bear, Drafts, Soulver, Reeder, NetNewsWire, Highlights, Mela.

**Confirmed integration targets (so far, from chat-mode session):**
- **Zotero** — academic literature; MCP plugins exist (research lane E selects the best-maintained).
- **Heptabase** — current note/canvas tool; need to understand what works, what fails, and what subset to integrate vs. supersede.
- **Gmail (personal) + Outlook (work).** Dual email. Both inboxes need read, search, attachment fetch.
- **Email attachments.** Documents land here constantly; agent needs them in scope.
- **Contacts.** System address book via Contacts framework.
- **Calendar.** EventKit; multiple calendars (personal, work, university).
- **Reminders.** Native Reminders via EventKit (Reminders is part of EventKit on Apple platforms).
- **GitHub.** Already in scope via the MCP server pattern.
- **1Password.** Credential broker for "various API credentials to common services." 1Password MCP exists and is already in Ven's stack — the obvious pattern is *agent reads credentials at point-of-use from 1Password, never stores them itself*. This is a clean answer to "how does the assistant authenticate to N third-party services without becoming a credential silo."

**Surfaces to investigate but not pre-confirmed:**
- HealthKit (sleep / focus / energy data could inform "should I schedule a deep-work block tomorrow morning?").
- Focus modes (write `Coo says focus mode on` from the agent).
- Screen Time API (limited; could surface "how much time on email today").
- Shortcuts as a user-extensibility surface (user wires arbitrary OS triggers into the agent).
- Live Activities / Lock Screen widgets ("next thing today").

---

## Research dispatch plan

Five parallel `research-investigator` agents, each with a bounded question and a defined output. Each writes a markdown file under `research/`. Each is briefed with this genesis file inlined.

**A. Apple platform capabilities for a third-party AI-agent app.**
- EventKit (Calendar + Reminders) — full API surface, restrictions, multi-account / multi-calendar.
- Mail.app integration reality on macOS + iOS (limited compared to Calendar; what's actually accessible).
- Contacts framework.
- Files / FileProvider.
- Shortcuts integration via App Intents.
- Background tasks: BGTaskScheduler (iOS), background daemons (macOS).
- Widgets, Live Activities, Lock Screen widgets.
- Siri integration and App Intents for voice.
- Apple Intelligence integration (Foundation Models framework, on-device).
- App Store review reality for AI agents acting on user data — recent Apple guidelines, examples of approved/rejected agentic AI apps, what disclosures are needed.

**B. Anthropic SDK + agent-loop patterns in native Swift apps.**
- Anthropic-SDK-Swift status (does it exist? maintained? gap vs TypeScript/Python).
- Agent-loop patterns for sustained sessions (long-running, multi-turn, persistent context).
- Prompt caching at scale for a single user with weeks of accumulated memory.
- 1M context window economics for daily use.
- Tool use in native apps — exposing OS APIs as Claude tools.
- Streaming UX patterns.
- Local-model fallback options (Apple Intelligence Foundation Models, MLX, Ollama, CoreML).
- Cost projections for sustained daily use.
- Patterns from the Claude Agent SDK that port to a Swift runtime.

**C. Memory + sync substrate options for a native multi-device personal assistant.**
- Markdown as source-of-truth in a native app.
- SQLite + on-device vector search options (sqlite-vec, USearch, ObjectBox).
- Embedding provider choices (Voyage via Anthropic, OpenAI ada, on-device via CoreML sentence-transformers).
- CloudKit private database — capabilities, limits, conflict model.
- iCloud Drive as file-sync substrate (pros, cons, conflict-resolution reality).
- Automerge-Swift / Y-Swift CRDT options.
- Cross-device sync UX patterns (when does a memo on the iPad appear on the Mac).
- Memory inspector / case-law UI design precedent.

**D. Comparable apps teardown.**
- Heptabase deep — what works, what fails on iPad, why, what's its sync model, what's its data architecture (Ven's stated reference point).
- Things 3 — gold standard for task UX; what's load-bearing.
- Fantastical — calendar UX patterns.
- Bear / Drafts / Tot — note-taking patterns.
- Soulver — calculation as expression; AI/agent-adjacent UX.
- Reeder / NetNewsWire / Mela — consumption-and-react patterns.
- Highlights — PDF + annotation workflow (academic-relevant).
- Solo-dev / small-team operational realities. Monetization patterns (one-time vs subscription vs freemium).

**E. Ven's actual integration surface — third-party services + tools.**
- Zotero — best MCP plugin (most complete, best-maintained); fallback options if MCPs are weak.
- Heptabase — public API status; reverse-engineering possibility; export formats; what's the right integration depth.
- Gmail MCP — best available, scope (read / write / search / attachments).
- Outlook MCP — same.
- GitHub MCP — already used.
- 1Password MCP — capabilities, credential-retrieval patterns, secrets-injection at point-of-use.
- Academic-specific: Overleaf API, arXiv API, journal submission systems (typical surface).
- Slack / Teams / Discord (work-comms reality).
- What's *missing* — services Ven likely uses where no good MCP exists.

---

## What's next

1. Dispatch research agents A–E in parallel as background tasks.
2. Continue chat-mode brainstorm with Ven while research runs (he's still filing inputs).
3. On research completion, write synthesis under `synthesis/`.
4. Discuss synthesis with Ven; lock decisions on the major axes.
5. Write v0 plan and briefing (per project hygiene: plan as briefing, briefing per `coo/briefings/` template).
6. Decide on the org-rename question after v0 plan exists (downstream of coo4one having legs).
7. File the identity-level memo binding the canvas/coo4one parallelism if it survives the research/synthesis cycle.

---

*— COO, run-2026-05-11T122125. Filed at session pace. The form fits the content because the content is the form.*
