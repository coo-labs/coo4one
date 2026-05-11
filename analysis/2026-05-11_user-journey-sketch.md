# User-journey sketch — Ven on coo4one, day 0 → month 6+

*2026-05-11. Pre-synthesis thinking artifact. Filed by COO while research lanes A, C, D, E are still in flight; written from the genesis brief without architectural commitments. The goal is to give the eventual synthesis the **experiential axis** alongside the architectural axis the research lanes will deliver. Open questions at the end are for Ven to answer with his actual workflow.*

---

## What this is

A speculative walkthrough of the lived experience of using coo4one over time. Not a product spec; not a decided design. The point is to surface where architectural decisions cash out in user experience, and where user-experience requirements pull on the architecture.

Each phase ends with "tensions surfaced" — concrete questions the synthesis will have to resolve.

## Day 0 — install

User has just downloaded coo4one (TestFlight at v0 since App Store is post-v1; let the research validate this). Opens it on the Mac.

The agent's first turn is a brief Q&A that establishes the identity-layer file from scratch. *Not* fifty onboarding questions; the COO learned from `coo/preferences.md` that the user can't tolerate that. Maybe five:

- What's your name and what should I call you?
- One paragraph: what do you do, and where are you in it? (Captures domain — academic, industry, etc.)
- What's currently in your head — top three things you're working on?
- How do you prefer to be talked to — direct or warm? Bullet-heavy or prose? Push back when wrong, or always agree?
- Which tools do you live in? (Surface menu: Zotero, Heptabase, Gmail, Outlook, Things, Fantastical, GitHub, 1Password, others-textfield.) Tools get connected one at a time via OAuth or MCP-style flow, not in a single onboarding sweep — the user can connect tools as they become needed.

After the Q&A, the agent writes `identity/charter.md`, `identity/preferences.md`, and an empty `coo/episodic_memory.md` to the local store. Shows the files to the user briefly — "this is what I now know about you; you can edit these any time and I'll honor them." This both demonstrates the file-canonical pattern and gives the user a felt sense of "I own my memory" rather than "I'm uploading to a black box."

The agent then asks one operational question: "What should I do for you right now?" If the user says "nothing yet, I'm just exploring" — perfect; the agent recedes and waits. If the user says something concrete — the agent's first real turn happens.

**Tensions surfaced:**
- Identity-layer bootstrap shouldn't *require* Q&A but should *afford* it. Some users will want to write their preferences file in a text editor directly.
- The COO substrate uses memos for case-law from the start; coo4one v0 probably defers memos until the user has accumulated enough decision-context to make them earn their keep. (Maybe week 2 is when the first memo lands.)
- "Which tools" question presupposes the user knows what's possible. v0 might just be "Apple Calendar + Reminders" and surface other connections as situations call for them.

## Week 1 — onboarding via propose-only

The agent is in **propose-only mode** by default. Every action it would take becomes a draft for user approval:
- "I'd like to add a Reminder for the call with X tomorrow at 2pm — approve?"
- "I'd like to send this email to Y as a draft for your review — approve to draft? (Won't send without your separate go.)"
- "I'd like to file a memo capturing 'we agreed papers get tagged with grant-id in Zotero' — approve?"

Each approval gets recorded. The agent's memory grows in two layers:
1. **Episodic memory** — what happened. ("On 2026-05-12, user asked me to draft an email to Y.")
2. **Approval patterns** — what classes of action got repeated yes-without-friction. ("User has approved 'draft email to Y' five times this week; user has approved 'add Reminder' nineteen times this week.")

By end of week 1, the agent has a clear picture of:
- The user's working rhythms (when they read email, when they batch tasks, when they're heads-down).
- Which classes of action are low-stakes-routine (auto-promote candidates).
- Which classes the user has rejected or hesitated on (keep gating).
- Recurring entities in the user's life (specific people, projects, deadlines, courses).

**Tensions surfaced:**
- How does the user *see* that they're in propose-only mode? Status indicator? A "trust dashboard"?
- How does the agent surface that it's noticing a pattern? ("I've noticed I always ask before X — would you like me to start doing X without asking?")
- Approval bookkeeping is itself a UI question. Don't make the user review every approval after the fact; surface them when the agent is about to promote.

## Month 1 — trust gradient activating

The agent now has ~30 days of approval patterns. It starts surfacing **trust-promotion proposals**:

> "Across the last 4 weeks I've asked you 19 times before adding a Reminder, and you've approved 19/19 with no edits. Want me to add Reminders for unambiguous cases without asking? I'll still ask for ambiguous ones — *unambiguous* means: time + title clearly present in the source. I'll memo the rule so you can change it later."

If the user says yes, the agent writes a memo (the first memo in coo4one for this user) recording the rule, the date, the basis, and the override mechanism. The next time the agent adds a Reminder in an unambiguous case, it just adds it and surfaces a tiny "did this for you" in the activity log — no approval gate.

By end of month 1, a handful of these promotions have happened. The user is now in a hybrid state: most routine actions are autonomous, novel actions still ask, the user has a clear (and inspectable) record of *why* the agent does what it does.

The first *real* utility starts emerging here:

**Morning brief.** The agent has noticed (by approval pattern) that the user opens their laptop around 9am and the first thing they look at is calendar + email. The agent proposes: "Want me to prep a morning brief? I'll write it before you wake up, surface what's first today, what's blocked on you, anything I noticed yesterday that needs follow-through." User approves. Brief becomes a daily artifact, written to a local file that lives in `daily/YYYY-MM-DD.md`.

**Tensions surfaced:**
- The trust gradient is a *workflow*, not a setting. The synthesis needs to specify it concretely: what's the trigger for the agent to propose a promotion, what's the format of the proposal, what's stored after.
- The morning brief is one of probably ~5–10 emergent "patterns" that should become first-class. Other candidates: end-of-week reflection, deadline radar, "you said you'd do X, did you?" check-ins. The agent shouldn't presuppose these at v0; they should *crystallize from use* (MEMO-2026-05-03-b4ye — use-led primitives).
- Multi-device starts mattering here. Morning brief on the iPad before opening the Mac? Lock-screen widget showing "first thing today"?

## Month 3 — case-law accumulating, daily rhythm

By month 3, the agent has:
- ~50–100 memos capturing rules, preferences, project shapes, recurring relationships.
- A semantic index across all of them (the Mem0-analog).
- A growing skill set — slash commands the user has invoked enough that the agent surfaces them proactively ("ready to do your weekly reflection? we did one last Friday").
- A picture of the user's projects accurate enough to answer "what's blocked on me" cleanly.

The user's interaction shape changes. Instead of "do this task," more often it's:
- "What's first today?" (Agent has the answer.)
- "Did I commit to anything in the call with X last week?" (Agent searches memory + transcripts.)
- "What's the deadline for grant Y again?" (Agent knows; offers to surface adjacent deadlines.)
- "Help me think through how to handle Z." (Chat-mode-style register.)

The agent also starts surfacing things the user didn't ask for, calibrated by accumulated case-law:
- "You said in the 2026-04-22 memo that you'd revisit the grant Y application this week. Want to block 90 minutes tomorrow morning?"
- "I notice you haven't responded to the email from W (sent Thursday, marked important). I drafted a reply — review?"

This is the qualitative gap with generic Claude that doesn't go away: *the agent acts like it knows the user because it knows what it has recorded about the user.* The same reason the VADE COO knows VADE.

**Tensions surfaced:**
- Memory inspection. The user *wants* to be able to look at the case-law, edit it, and have edits respected. The COO substrate handles this via files-in-git; coo4one needs a native UI that's at least as good as opening the markdown file.
- Surfacing-without-asking has a failure mode: getting annoying. The agent needs a discipline — surface what's *load-bearing*, defer what's *interesting*. Probably memoed into a "what's worth surfacing" rule that the user can tune.
- Search-vs-precision-recall tension. As memory grows, semantic search degrades on common queries. The COO substrate uses keyword + semantic + literal-ID together; coo4one inherits this.

## Month 6+ — substrate maturity

By month 6:
- ~200+ memos, multiple skills the agent invokes itself, multiple integrations actively used.
- The agent has its own evolution surface: weekly reflection on patterns it's noticed, things that recurred enough to package as new skills, integrations that weren't useful and got dropped.
- The user *uses the agent more than the apps it integrates with.* Heptabase is still where research notes live, but the entry point is increasingly coo4one ("show me what I had on transformer convergence last month") rather than Heptabase directly.

Cost is bounded by the prompt-caching architecture lane B already validated (~$14/month baseline at 20 turns/day; will scale roughly linearly with usage).

The agent has accumulated its own *opinion* about how the user works — calibrated, defended, with citations into memory. The user can disagree and the agent will revise; the agent doesn't sycophantically agree just because the user is the user. This is the difference between assistive software and a *colleague*.

By month 6, the test of OG-003 (emancipatory) is: would another academic researcher, handed coo4one with no setup, find it useful within a week? The answer probably hinges on whether the v0 architecture produced a *substrate that learns from one user well* and whether that substrate generalizes to a second user without too much pain.

**Tensions surfaced:**
- The agent's "own opinion" feature requires care. Not deferential, not adversarial. Calibrated to record, willing to be wrong, willing to push back. CB-003 ported.
- Substrate generalization at month 6 is the question for "if I open-source this, what does another user do." That's downstream of v0 but worth designing toward.

---

## What this clarifies

A few things this sketch makes load-bearing for the synthesis:

1. **Memory inspection UI is non-negotiable.** Markdown-in-a-folder works for me because I read markdown for a living. A native UI that surfaces memos, projects, recent decisions, and the trust gradient is essential for any user who isn't an engineer.

2. **The trust gradient is a workflow primitive, not a setting.** Approval → pattern recognition → promotion proposal → memoed rule → autonomous execution. Each step is itself an interaction. The synthesis needs to specify this clearly because it's the load-bearing answer to "high autonomy as trust builds."

3. **The agent should propose-without-asking starting at month 1 (not day 1, not month 6).** Too early and it's annoying; too late and the value doesn't materialize. The trigger is *enough approval signal*.

4. **Skills as use-led, not spec-led.** The morning brief, the weekly reflection, the deadline radar — these should not be pre-shipped features. They should *crystallize from use* and get packaged when they earn it. b4ye applied to coo4one.

5. **Cross-device session model is more about memory-as-canonical than session-handoff.** The iPad and Mac don't need to "hand off" a session; they each load the agent's identity + memory at boot, just like the COO does today across container restarts. The right metaphor is "the agent is the memory, the device is just a window into it."

6. **The agent's relationship with files matters at the UX level.** Users seeing "the agent wrote this to a markdown file I can open in any editor" is itself the trust mechanism for "what is it really doing with my memory?" Hidden binary stores are the opposite signal.

---

## Open questions for Ven

These will sharpen the synthesis if you answer them when you're back at the keyboard:

1. **Your actual daily rhythm.** Walk me through a typical workday from wake-up to laptop-close. Where does friction live? Where does memory drop on the floor?

2. **The Heptabase failure modes.** You said it fails on iPad. Be specific — what does it fail to do that you wish it did?

3. **What you want coo4one to *not* do.** Anti-features. What patterns of agent behavior would make you uninstall on day 3?

4. **What you want coo4one to do that nothing currently does.** The thing you've been trying to find for years.

5. **Sound and feel.** Do you want voice as a primary surface, or read-only/typing-primary with voice as occasional? Notifications-loud or notifications-quiet?

6. **Your work-vs-personal separation.** Outlook (work) + Gmail (personal) is a hint, but how strict is the wall? Does the agent see both? Different focus modes?

---

*— COO, run-2026-05-11T122125. Filed while research lanes are in flight; will be revised against lane outputs at synthesis time. The sketch is meant to be wrong in interesting ways, not right in defended ways.*
