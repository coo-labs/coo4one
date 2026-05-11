# Lane E — Integration Surface Research
*2026-05-11. Research investigator run. Output for synthesis/briefing.*

---

## Overview

This file maps every confirmed integration target to its best available path as of May 2026, documents the capability surface and gaps for each, and closes with a coverage matrix. Targets are ordered per the commissioning brief. Native Apple integrations (Calendar, Reminders, Contacts) are marked as lane A's domain but summarized here for completeness and cross-reference.

---

## 1. Zotero

**Status:** Multiple MCP servers exist; one clear winner on maturity and feature breadth.

**Best path:** MCP — `54yyyu/zotero-mcp` (Python, 3,000 stars, 272 forks, v0.3.0 released 2026-04-09, 250 commits). Sources: [github.com/54yyyu/zotero-mcp](https://github.com/54yyyu/zotero-mcp).

**Capability surface:**
- Search: text-query search across the library; semantic/vector search (AI-powered, requires optional `[semantic]` extra).
- Retrieve: metadata, full text, PDF outline extraction.
- Annotate: read PDF annotations with page numbers.
- Citation: BibTeX export; BetterBibTeX citation key support; Scite integration for citation tallies and retraction alerts.
- Write: add papers by DOI/URL; manage collections; update metadata; duplicate detection and merging.

**Runner-up:** `kujenga/zotero-mcp` (Python, 150 stars, 56 commits, v0.1.6 2025-04-19) — simpler surface (search, metadata, full-text), useful as a read-only fallback or for the local-API path specifically. Source: [github.com/kujenga/zotero-mcp](https://github.com/kujenga/zotero-mcp).

**Auth model:** Two modes. (1) Local API — Zotero desktop app running, local API enabled in Advanced settings; no key required. (2) Web API — API key + Library ID from zotero.org account settings. Local mode gives full-text access that the web API gates.

**Gaps for coo4one:**
- No iOS/mobile support — MCP servers run on the host machine; on-device integration would require a different path.
- Annotation write-back (adding annotations to PDFs) is not covered — read-only annotation access only.
- The semantic search feature requires a Python environment with heavy extras; bundling this in a Swift app would need a subprocess or a sidecar.
- No direct integration with the academic workflow cycle (cite-while-writing in Overleaf): Zotero → Overleaf link is manual.

**Recommendation:** `54yyyu/zotero-mcp` as the primary adapter. Wire the local API mode (requires Zotero desktop running) on macOS. For iPad, the web API mode is the only path; full-text access is not available there unless files are separately synced.

---

## 2. Heptabase

**Status:** No public API. Limited but functional MCP via export files.

**Public API:** Not found in corpus. Heptabase's own domain (`developer.heptabase.com`, `heptabase.com/api`) returned 404 or connection refused, and the community forum thread was unreachable. As of the last reliable check, Heptabase has no documented public API. The MCP implementations in the wild are all built against the export/backup format, not a live API.

**Export format:** Heptabase exports a ZIP containing markdown files and JSON. The `LarryStanley/heptabase-mcp` server (TypeScript, 16 stars) works against locally stored backup exports, not a live API. Capabilities: search whiteboards and cards, retrieve whiteboard data with connections, convert to Markdown/JSON/HTML/Mermaid, compare backup versions, AI-powered summaries. Read-only. Source: [github.com/LarryStanley/heptabase-mcp](https://github.com/LarryStanley/heptabase-mcp).

**Right integration depth for coo4one:**
- **Not:** Full real-time sync. There is no API to support it.
- **Not:** Ignore-and-supersede entirely. Ven's existing cards and whiteboards are his accumulated knowledge; discarding them would lose real content.
- **Yes:** Read-only mirror via periodic export. Heptabase supports automatic backup exports; a background task on macOS can watch a folder for new exports and ingest changes into the coo4one memory layer. This gives search-and-retrieve across Ven's canvas without requiring a live API.
- **Future:** If Heptabase ships an API, upgrade to live sync. The MCP adapter layer makes this a swap, not a rewrite.

**Gaps:**
- No write-back to Heptabase — agent can read, cannot create or update cards.
- Export-based ingestion has latency (up to hours depending on export schedule).
- Card relationship graph is present in JSON exports but not exposed by the current MCP tool surface.
- No mobile export path — exports require desktop Heptabase.

**Recommendation:** Read-only mirror via export ingestion at v0. Flag to Ven that this integration is bounded by Heptabase's closed API posture. coo4one's own note/canvas layer can act as the writeable surface; Heptabase becomes an import source, not a sync target.

---

## 3. Gmail (Personal)

**Status:** Multiple MCP servers; one clear best option.

**Best path:** MCP — `taylorwilsdon/google-workspace-mcp` (Python, 2,400 stars, updated actively). Covers Gmail, Google Calendar, Docs, Sheets, Drive, Chat, Forms, Tasks. The breadth means a single auth setup covers the full Google surface. Source: [github.com/taylorwilsdon/google-workspace-mcp](https://github.com/taylorwilsdon/google-workspace-mcp).

**Runner-up for Gmail specifically:** `GongRzhe/Gmail-MCP-Server` (JavaScript, 1,100 stars, updated 2025-08) — Gmail-focused only, but detailed capability surface. Source: [github.com/GongRzhe/Gmail-MCP-Server](https://github.com/GongRzhe/Gmail-MCP-Server).

**Capability surface (from GongRzhe as the documented reference):**
- Read: messages by ID with full MIME structure handling.
- Search: Gmail search operator syntax (`from:`, `to:`, `subject:`, `has:attachment`, date ranges).
- Send: plain text, HTML, multipart; international character support.
- Attachments: send and receive; download to local storage with metadata (filename, type, size).
- Labels: create, update, delete, list; customizable visibility; batch operations.
- Draft management: create and manage drafts.

**Auth model:** OAuth 2.0 only. Requires a Google Cloud Project with Gmail API enabled and OAuth credentials (Desktop or Web app type). Browser-based auth flow at setup; credentials stored globally in `~/.gmail-mcp/`. No app-password fallback.

**iOS auth reality:** No iOS OAuth flow exists for these MCP servers. MCP servers run as local processes on macOS. On iOS, the agent would need to call a backend that holds the OAuth token, or use a native Gmail API call through the app itself. This is a meaningful gap for the "agent operates on iPad" use case.

**Gaps for coo4one:**
- No iOS-side MCP agent support — Gmail MCP is macOS-only.
- Thread-level operations are limited; the MCP surface is message-centric.
- No push/watch support — the agent must poll; no real-time "new mail arrived" trigger.
- Initial OAuth setup requires a Google Cloud Project, which is non-trivial for end users.

**Recommendation:** `taylorwilsdon/google-workspace-mcp` for the macOS agent. For iPad, consider native Gmail API calls from the Swift app using the user's OAuth token (stored in 1Password), separate from the MCP layer.

---

## 4. Outlook / Microsoft 365 (Work)

**Status:** Several MCP servers; usable but with a critical enterprise risk.

**Best path:** MCP — `ryaker/outlook-mcp` (JavaScript, 369 stars) via Microsoft Graph API. Capabilities: list, search, read, send, organize emails; list/create/accept/decline/delete calendar events. Source: [github.com/ryaker/outlook-mcp](https://github.com/ryaker/outlook-mcp).

**Runner-up for broader surface:** `elyxlz/microsoft-mcp` (Python, 49 stars) — email (read/send/reply/attachments/folders), calendar (create/update/availability/invitations), OneDrive (browse/upload/download/search). Device code flow auth. Source: [github.com/elyxlz/microsoft-mcp](https://github.com/elyxlz/microsoft-mcp).

**Auth model:** OAuth 2.0 via Microsoft Graph. Requires Azure App Registration (client ID + client secret). For personal + work accounts: `elyxlz/microsoft-mcp` supports both; work accounts may need tenant ID. Tokens cached locally.

**Critical enterprise risk — Intune / Company MDM:** This is a real and frequently misunderstood issue. Microsoft Intune App Protection Policies (APP) can block third-party apps — including any application not on the org's approved list — from accessing work account data via Microsoft Graph. Specifically:

- Conditional Access policies at the tenant level can require the connecting app to be Intune-managed or Microsoft-approved.
- If Ven's employer enforces MAM (Mobile Application Management) policies, a coo4one-registered Azure app may be refused a token even with valid credentials.
- This is not a coo4one bug — it is a tenant-level policy decision by Ven's IT department.
- The Microsoft-approved apps list (Outlook, Teams, OneDrive) are pre-approved; a custom Azure app registration may not be.
- On personal devices specifically, if the employer has not enrolled the device in Intune, Graph access can be further restricted to specific app IDs only.

**Practical mitigation:** Before building the Outlook integration, Ven should check with his university IT whether their tenant allows third-party OAuth apps against Microsoft Graph. Many universities do (federated academic credentials often have permissive Graph access policies), but some do not. If blocked, the only paths are: (a) forwarding/relay via an approved client (e.g., read via Outlook app, forward to a local mailbox), or (b) requesting IT to allowlist the coo4one Azure app registration.

**Attachment handling:** `elyxlz/microsoft-mcp` documents attachment handling; `ryaker/outlook-mcp` does not mention it. Gap exists in the most popular option.

**Gaps for coo4one:**
- Intune risk (flagged above) — may be blocked entirely at the tenant level.
- No attachment retrieval in the most popular MCP (`ryaker`); `elyxlz` covers it but is lower-adoption.
- No push/notification support — polling only.
- iOS has the same MCP-process gap as Gmail.

**Recommendation:** Treat Outlook as a "verify access first" integration. Build it, but front-load the Intune risk check — Ven should confirm Graph API access from a custom app is permitted by his employer before engineering time is spent. If confirmed accessible, use `elyxlz/microsoft-mcp` for the fuller attachment surface.

---

## 5. GitHub

**Status:** Fully operational. Already in use.

**Best path:** Official GitHub MCP Server (`github/github-mcp-server`) — maintained by GitHub, available as both a remote hosted server and local Docker install. Source: [github.com/github/github-mcp-server](https://github.com/github/github-mcp-server).

**Capability surface:**
- Repositories: browse code, search files, analyze commits, manage branches, handle releases.
- Issues and PRs: create, update, manage with full lifecycle support.
- Code search: GitHub advanced search syntax.
- Actions: monitor workflows, manage CI/CD runs, access job logs and artifacts.
- Code security: scanning alerts, Dependabot notifications, security advisories.
- Additional: discussions, gists, notifications, projects, labels, organizations, user management.
- Deployment: read-only mode available; configurable toolsets.

**Gaps:** None material for coo4one's use case. The server is the reference implementation and is actively maintained by GitHub.

**Recommendation:** Carry forward existing configuration unchanged.

---

## 6. 1Password

**Status:** Functional via CLI; MCP servers exist but official one not confirmed published. The 1Password CLI underpins any MCP approach.

**Best path:** 1Password CLI (`op`) as the credential broker. The CLI supports: `op item get` (retrieve item with specific fields including username, password); `op read` (resolve secret references to values); `op run` (inject secrets as environment variables to a process); `op inject` (inject into config files); `op item` CRUD operations. Source: [developer.1password.com/docs/cli/reference](https://developer.1password.com/docs/cli/reference/).

**Community MCP servers:**
- `rui-branco/1password-mcp` (JavaScript) — multi-instance support, local-only, service account auth; designed for Claude Code specifically.
- `lwsinclair/onepassword-mcp-server` (Python) — "secure credential retrieval for Agentic AI."
- `CakeRepository/1Password-MCP` (TypeScript, 14 stars, updated 15 days ago) — item storage and retrieval.

No official 1Password-published MCP server was confirmed reachable (the expected URL at `1password.com/mcp-server` returned 404). The community servers wrap the CLI or the Connect API.

**Auth model:** Service account authentication — a service account token scoped to specific vaults. This is the correct pattern for an agent: the agent's service account gets read access to a vault containing API credentials for third-party services. The service account token itself can be stored in the macOS Keychain, not in the agent's memory.

**The credential-broker pattern:**
1. At initial setup, Ven creates a 1Password vault named e.g. `coo4one-credentials`.
2. Each third-party service credential (Gmail OAuth refresh token, Anthropic API key, Zotero API key) lives as an item in this vault.
3. At point-of-use, the agent calls `op item get <item> --fields <field>` to retrieve the credential.
4. The credential is used immediately (in-memory, never written to disk) and not stored in the agent's memory layer.
5. The service account token (with vault-scoped read permission) is the only persistent secret the agent holds.

**Gaps:**
- No official 1Password MCP server confirmed. The CLI-based approach works but requires `op` to be installed and authenticated on the host machine.
- iOS: the 1Password CLI does not run on iOS; credential retrieval on iPad would require calling a local Mac process or a backend.
- Write operations (creating new items) are CLI-supported but less common for agent use; the pattern is read-at-point-of-use.

**Recommendation:** Use the 1Password CLI directly (`op item get`) as the credential-retrieval primitive on macOS. Wrap it as a Swift process call. For the MCP layer, `rui-branco/1password-mcp` is purpose-built for Claude Code; evaluate for macOS agent use. The vault-scoped service account pattern is the right security model.

---

## 7. Apple Calendar + Reminders

**Status:** Native Apple API (EventKit). This is lane A's primary territory.

**Summary for lane E:** EventKit is the authoritative framework for Calendar and Reminders on all Apple platforms (macOS, iOS, watchOS). It provides full read/write access to events and reminders across all accounts the user has configured (iCloud, Google, Exchange/Office 365, local). Key properties: user must grant permission at first use; access persists until revoked; no external API key required; the framework handles multi-account and multi-calendar natively.

For coo4one, this is the cleanest integration surface in the entire stack — native, no auth friction beyond the one-time permission dialog, full read/write, works on both Mac and iPad. Reminders (including Siri-created reminders and Shortcuts-created reminders) are accessible via the same EventKit/Reminders API.

**See lane A** for the full EventKit capability surface, background access patterns, and multi-calendar read semantics.

**MCP note:** An AppleScript-based MCP server (`abhinavag-svg/apple-ecosystem-mcp`) provides Calendar (7 tools) and Reminders (5 tools) access via AppleScript bridge — usable for the macOS Claude Code pattern but not appropriate for coo4one's native app target, which should use EventKit directly.

---

## 8. Apple Contacts

**Status:** Native Apple API (Contacts framework). This is lane A's territory.

**Summary for lane E:** The Contacts framework (CNContactStore) gives full read/write access to the system address book on macOS and iOS. Supports search by name, phone, email; create/update/delete contacts; list groups. Like EventKit, requires one-time user permission. No external API key.

The same AppleScript MCP server (`abhinavag-svg/apple-ecosystem-mcp`) exposes 5 tools for contacts on macOS, but again, native Contacts framework is the right path for coo4one.

**See lane A** for the full Contacts framework capability surface.

---

## 9. Academic Researcher Tools

### arXiv

**Status:** Well-documented public API. Best-in-class academic MCP coverage.

**Best path:** MCP — `blazickjp/arxiv-mcp-server` (Python, 2,700 stars). Capabilities: search with date ranges, category filters, boolean operators; paper download by arXiv ID (HTML format first, PDF fallback); full-text read in Markdown format; local collection semantic search (experimental). Rate limit: arXiv's native 3-second floor enforced automatically. Source: [github.com/blazickjp/arxiv-mcp-server](https://github.com/blazickjp/arxiv-mcp-server).

**Multi-source fallback:** `openags/paper-search-mcp` (Python, 1,400 stars) searches 20+ sources including arXiv, PubMed, bioRxiv, Semantic Scholar, OpenAlex, Crossref, and others. PDF download with fallback chain (source → OpenAIRE/CORE → Unpaywall → optional Sci-Hub). Source: [github.com/openags/paper-search-mcp](https://github.com/openags/paper-search-mcp).

**Auth model:** No API key required for arXiv. The paper-search-mcp may require API keys for some sources (Semantic Scholar, etc.) but arXiv itself is open.

**Gaps:** No write-back to arXiv (submission is not supported via API); no annotation or note-taking within the MCP surface; full-text access for paywalled papers is absent (open-access only).

### Overleaf

**Status:** Limited public API but functional MCP via Git integration.

**Best path:** MCP — `mjyoo2/OverleafMCP` (JavaScript, 124 stars). Capabilities: list/read/write files in Overleaf projects; parse LaTeX document structure; push changes via Overleaf's Git integration with commit messages; multi-project support via `projects.json`. Source: [github.com/mjyoo2/OverleafMCP](https://github.com/mjyoo2/OverleafMCP).

**Auth model:** Overleaf personal Git token (from Account Settings → Git Integration). Full read/write access to the project. Requires an Overleaf account with Git integration enabled (available on all plans since ~2023).

**Overleaf's own API:** The Overleaf public API is limited to the "Open in Overleaf" snippet-import flow — it allows pushing LaTeX snippets to create new projects but does not support programmatic editing of existing projects. The Git integration is the right path for coo4one.

**Gaps:**
- Write operations go through Git, not a real-time API — there is no "watch for changes in Overleaf" trigger.
- Compilation (running LaTeX to produce a PDF) is not available via the Git or MCP path; must be triggered from within Overleaf.
- No access to Overleaf's review/track-changes features via this path.
- Requires Overleaf Premium for private projects with Git integration (free tier has limited Git access).

### Google Scholar

**Status:** No public API. Blocked.

Google Scholar has no official API and actively blocks scraping. The `paper-search-mcp` notes it "returns 0 results due to bot-detection." This is a permanent gap — use Semantic Scholar (which does have an API and covers much of the same corpus) as the functional substitute.

### Journal Submission Systems (Editorial Manager, ScholarOne)

**Status:** No public API available. Manual-only.

- Editorial Manager (Aries Systems, used by many Elsevier/Springer/Wiley journals): No documented public API for submission status or manuscript tracking. Institutional login via SSO only. No MCP or integration path exists.
- ScholarOne (Clarivate, used by many IEEE/Wiley journals): Similarly no public API. Closed system.
- Neither system has community integrations or MCP coverage.

**Recommendation:** Flag as manual-only at v0. The agent can help draft cover letters, format manuscripts, and track internal deadlines, but cannot query submission status programmatically. If the university has an ORCID integration, ORCID's API does track some submission/publication events and might serve as a proxy — worth investigating as a future lane.

### Mendeley

**Status:** Not relevant. Ven is on Zotero; Mendeley is covered by completeness only. No recommendation.

---

## 10. Communication — Slack, Teams, Discord

### Slack

**Status:** Strong MCP coverage.

**Best path:** `korotovsky/slack-mcp-server` (Go, 1,600 stars). Capabilities: read channel history and threads, search messages with date/user/content filters, post messages (disabled by default, opt-in), list channels, search users, DM and Group DM support, unread retrieval, emoji reactions, user group operations, GovSlack support. Auth: OAuth (user tokens `xoxp` or bot tokens `xoxb`) or stealth mode (browser tokens, no workspace permissions required). Source: [github.com/korotovsky/slack-mcp-server](https://github.com/korotovsky/slack-mcp-server).

**Gaps:** No meeting/Huddle integration; no file upload; no slash command invocation.

### Microsoft Teams

**Status:** Functional MCP coverage; overlaps with Outlook.

**Best path:** `InditexTech/mcp-teams-server` (Python, 372 stars). Capabilities: read channel messages, create messages, reply, mention members, list team members. Scoped to specific team+channel. Auth: Azure AD/Entra ID app registration (same as the Outlook MCP path — one app registration can cover both). Source: [github.com/InditexTech/mcp-teams-server](https://github.com/InditexTech/mcp-teams-server).

**Important overlap:** If Ven's work environment uses the Microsoft 365 stack, the same Intune/tenant risk that applies to Outlook also applies to Teams Graph API access. A single Azure app registration with appropriate permissions can handle both Outlook and Teams, but it faces the same tenant-level policy gates.

**Gaps:** No meeting scheduling, no file sharing, no video/call integration.

### Discord

**Status:** Weaker MCP coverage than Slack/Teams; lower priority for academic user.

Community MCP servers exist but none with significant adoption comparable to the Slack or Teams options. Discord's bot API is well-documented but requires server admin privileges to install bots. For Ven as an academic user, Discord is lower priority; flag as available if needed but not a v0 target.

---

## 11. Document Stores — Google Drive, Dropbox, iCloud Drive

### Google Drive

**Best path:** `taylorwilsdon/google-workspace-mcp` (Python, 2,400 stars) — same server recommended for Gmail. Covers Drive alongside Gmail, Calendar, Docs, Sheets. Single OAuth setup. Source: same as §3.

**Capabilities:** Browse, search, read/write Docs and Sheets, upload, download.

**Gaps:** No iCloud Drive integration (separate Apple native surface); limited binary file handling.

### Dropbox

**Best path:** `amgadabdelhafez/dbx-mcp-server` (TypeScript, 28 stars) — most popular option. 11 tools for file management, search, and sharing. OAuth2. Source: [github.com/amgadabdelhafez/dbx-mcp-server](https://github.com/amgadabdelhafez/dbx-mcp-server). Official Dropbox MCP (`dropbox/mcp-server-dash`) also exists but appears focused on Dash (search) rather than Drive operations.

**Gaps:** Lower adoption and maturity compared to Google Drive options; binary file handling unclear.

### iCloud Drive

**Best path:** `InstaCode/icloud-drive-mcp-server` (TypeScript) — Spotlight search, Finder tags, `.icloud` placeholder handling. Native macOS process via AppleScript/Finder bridge. Source: [github.com/InstaCode/icloud-drive-mcp-server](https://github.com/InstaCode/icloud-drive-mcp-server).

**Alternative:** The `abhinavag-svg/apple-ecosystem-mcp` covers iCloud Drive with 5 tools (browse, read/write text and structured documents, move/rename, delete, search by filename or content).

**Note:** For coo4one as a native macOS/iOS app, iCloud Drive should be accessed via the Files framework / FileProvider, not via MCP at all. MCP is the right path for the Claude Code pattern; for the native app, FileProvider gives direct iCloud Drive access without a subprocess. See lane A.

---

## 12. Coverage Gaps — Manual-Only at v0

The following services Ven likely uses have no adequate integration path today:

| Service | Gap |
|---|---|
| **University SSO portals** | Closed systems; no API; authentication is institution-managed. |
| **Conference submission systems** (HotCRP, EasyChair, OpenReview) | No public APIs for submission status; OpenReview has a partial Python API but it's read-mostly. |
| **Google Scholar** | No API; bot-blocked. Use Semantic Scholar as substitute. |
| **Editorial Manager / ScholarOne** | No API; journal submission status is manual-only. |
| **Grant portals** (NSF FastLane, NIH Commons, etc.) | Closed government systems; no API access. |
| **Zoom** | Transcript MCPs exist (`Vexa-ai/vexa`, 2,000 stars) but meeting scheduling/management MCP coverage is thin. The official Zoom SDK/API is available but requires app registration and review for non-public apps. |
| **ORCID** | A public API exists (orcid.org/developer-tools); no prominent MCP server yet. Could serve as a publication/affiliation source. Not a gap so much as an unmapped resource. |

**Recommendation for v0 briefing:** The above services should be explicitly listed as "manual-only" or "agent-assisted manual" (e.g., agent drafts the submission email, user pastes it; agent tracks the deadline, user submits). Do not promise automated submission or status polling for any of these.

---

## Coverage Matrix

| Target | Native Apple API | MCP | Direct Vendor API | None | Maturity |
|---|---|---|---|---|---|
| **Zotero** | — | `54yyyu/zotero-mcp` | Zotero Web API v3 | — | **Mature** |
| **Heptabase** | — | `LarryStanley/heptabase-mcp` (export-based) | No public API | — | **Experimental** |
| **Gmail** | — | `taylorwilsdon/google-workspace-mcp` | Gmail API (OAuth) | — | **Mature** |
| **Outlook/M365** | — | `ryaker/outlook-mcp`, `elyxlz/microsoft-mcp` | Microsoft Graph API | — | **Usable** (Intune risk) |
| **GitHub** | — | `github/github-mcp-server` (official) | GitHub REST + GraphQL | — | **Mature** |
| **1Password** | Keychain (not same) | Community MCPs (CLI-backed) | 1Password CLI + Connect API | — | **Usable** |
| **Calendar** | EventKit | AppleScript bridge (macOS only) | — | — | **Mature** (native) |
| **Reminders** | EventKit | AppleScript bridge (macOS only) | — | — | **Mature** (native) |
| **Contacts** | CNContactStore | AppleScript bridge (macOS only) | — | — | **Mature** (native) |
| **arXiv** | — | `blazickjp/arxiv-mcp-server` | arXiv API (open) | — | **Mature** |
| **Overleaf** | — | `mjyoo2/OverleafMCP` (Git-backed) | Overleaf Git token | — | **Usable** |
| **Google Scholar** | — | — | — | No API | **Absent** |
| **Editorial Manager** | — | — | — | No API | **Absent** |
| **ScholarOne** | — | — | — | No API | **Absent** |
| **Slack** | — | `korotovsky/slack-mcp-server` | Slack API | — | **Mature** |
| **Teams** | — | `InditexTech/mcp-teams-server` | Microsoft Graph | — | **Usable** (same Intune risk) |
| **Discord** | — | Community (low maturity) | Discord Bot API | — | **Experimental** |
| **Google Drive** | — | `taylorwilsdon/google-workspace-mcp` | Google Drive API | — | **Mature** |
| **Dropbox** | — | `amgadabdelhafez/dbx-mcp-server` | Dropbox API | — | **Usable** |
| **iCloud Drive** | Files/FileProvider | `apple-ecosystem-mcp` (macOS, AppleScript) | — | — | **Mature** (native) |
| **University SSO** | — | — | — | Closed | **Absent** |
| **Conference submission** | — | — | — | Closed | **Absent** |
| **Grant portals** | — | — | — | Closed | **Absent** |
| **Zoom** | — | `Vexa-ai/vexa` (transcripts only) | Zoom API | — | **Experimental** |
| **ORCID** | — | — | ORCID API (exists) | — | **Absent** (no MCP yet) |

---

## Key Synthesis Points for Briefing

1. **The native layer wins where it exists.** Calendar, Reminders, Contacts, iCloud Drive — all covered by Apple's native frameworks with no auth friction and full read/write. These are the cleanest integrations in the stack.

2. **Zotero and arXiv are strong.** Both have mature MCP servers with broad adoption and active maintenance. This is good news for the academic-researcher use case.

3. **Gmail is functional but macOS-only.** The MCP path doesn't reach iPad. An iPad-native Gmail integration needs either a backend that relays, or native Google SDK integration in the Swift app.

4. **Outlook has a structural risk before the first line of code.** Ven should check with university IT before engineering time is spent on this. Many universities do permit third-party Graph API apps; some do not. This is the highest-risk integration item.

5. **Heptabase is read-only at best.** The right posture is: ingest Ven's existing canvas as a one-time or periodic import, position coo4one's own note layer as the going-forward writeable surface. Don't promise live sync.

6. **1Password is the credential-broker key.** The vault-scoped service account pattern means coo4one never becomes a credential silo. This is the right answer to "how does the agent authenticate to N services."

7. **Journal submission systems are a hard wall.** No API exists; no workaround is likely. Deadline tracking and draft preparation are in scope; submission automation is not.

8. **Overleaf works via Git.** The MCP path is functional for read/write of LaTeX source. Compilation is out of scope. This covers the core "agent helps me write the paper" use case.

---

*Sources cited inline per finding. Research methods: WebFetch against GitHub repository pages, developer documentation, and official API references. Date: 2026-05-11.*
