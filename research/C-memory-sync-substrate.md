# C — Memory + Sync Substrate for coo4one

*Research lane C. Investigator: coo4one research agent. Date: 2026-05-11.*
*Question: What is the right memory + sync substrate for coo4one that preserves the file-canonical-markdown-with-semantic-cache pattern while delivering reliable cross-device continuity in the Apple ecosystem?*

---

## 1. Markdown as Source of Truth in a Native App

The core tension: markdown files are human-readable and version-control-friendly, but native apps need structured, fast, indexed access. Three models exist in production:

**File-canonical with derived index (Obsidian model).** The vault lives as `.md` files on disk. Obsidian syncs the files (via its own sync service or iCloud Drive) and maintains a derived SQLite index for backlinks, tags, and search. The index is treated as a cache — it can be rebuilt from files at any time. On boot, Obsidian indexes the vault; on mobile, iCloud Drive handles file sync, and the index rebuilds locally. This is the closest existing analogy to the COO pattern: files win, index serves. Failure mode: index can drift from files during rapid sync events; the fix is always "re-index from source."

**SQLite-canonical with markdown serialization (Bear model).** Bear stores everything in SQLite (CloudKit-backed). Markdown export is a derived format, not the source. The benefit is transactional consistency and fast query. The cost: the user's data is not in files they control; export is lossy if tags are in a proprietary format. This is the wrong model for coo4one, which needs files to be the durable, inspectable, version-controlled substrate.

**Block-canonical (Logseq/Roam model).** The unit of storage is the block (a paragraph or bullet), not the file. Logseq started file-canonical and is moving to a database model (Logseq DB) because file-canonical block sync is extremely fragile under concurrent edits. Heptabase uses its own card-and-board data model. These are both wrong for coo4one: the memo is the unit, and memos should map cleanly to files.

**Recommended pattern for coo4one:** File-canonical with a derived local SQLite index per device, exactly the Obsidian model. Each memo, each identity file, each session log is a `.md` file. The derived index (built on `FTS5` in SQLite) serves full-text search. The vector index (discussed in §2) is a second derived layer. Both layers rebuild from files if corrupt. The authoritative answer to "what did we decide?" is always the file, never the index.

*The index-rebuilds-from-files guarantee is non-negotiable.* It is what makes the system trustworthy under the "calibrated self-reports" charter requirement.

---

## 2. On-Device Vector Search

Four viable options for ~10K–100K documents on Apple-class hardware:

**sqlite-vec.** A pure-C SQLite extension enabling vector search via `vec0` virtual tables. Runs on Linux, macOS, Windows, WASM, and embedded hardware. Pre-v1 (API may break), but 7,600+ GitHub stars and sponsored by Mozilla Builders and Fly.io suggest production-adjacent maturity. iOS deployment requires bundling the `.dylib` or building from source as a static library — doable but not zero-friction. At 10K documents with 1024-dimensional float32 vectors, the in-memory index is ~40 MB; at 100K, ~400 MB, which strains iPhone RAM. sqlite-vec does brute-force scan by default (no HNSW), so query time scales linearly with corpus size. Fine for 10K; potentially slow for 100K. Source: [github.com/asg017/sqlite-vec](https://github.com/asg017/sqlite-vec).

**USearch.** HNSW-based vector index. Claims 10x faster than FAISS on comparable benchmarks; explicit iOS and Swift support listed in the official repo. The HNSW algorithm delivers sub-linear query time at 100K scale (typically <10 ms on phone-class hardware for 1024-dim vectors). Memory scales with the `M` parameter — at M=16, a 100K document index is roughly 200–400 MB, manageable on modern iPhones (6–8 GB RAM). Source: [github.com/unum-cloud/usearch](https://github.com/unum-cloud/usearch). USearch indexes must be saved to disk and loaded on boot; they cannot be stored inside SQLite, so the query path is separate from the metadata query path.

**ObjectBox.** Commercial iOS/macOS database with on-device vector search, marketed as "First On-device Vector Search on iOS." Provides ACID transactions + ANN search in one library with native Swift API. Proprietary license (free tier available). Source: [objectbox.io/swift](https://objectbox.io/swift/). Suitable if the team prefers a single-library answer that bundles object storage + vector search; less suitable if the markdown-canonical pattern requires keeping files as the source and the database as pure index.

**Apple NaturalLanguage framework (NLEmbedding).** On-device, no API key, strong privacy. However, NLEmbedding is a word-level embedding trained for linguistic tasks (NER, sentiment), not a sentence transformer. Dimensions are much lower than modern retrieval embeddings, and semantic similarity quality is substantially below Voyage or OpenAI models on MTEB benchmarks. Not recommended as the primary retrieval embedding — acceptable as a cheap fast-path trie or keyword boost layer.

**Recommendation for coo4one at ~10K documents:** sqlite-vec is sufficient and keeps everything in one SQLite file alongside the FTS5 index, simplifying backup and migration. At ~100K documents (unlikely for a personal assistant at v0 but possible after multi-year accumulation), switch to USearch for the vector layer while keeping sqlite-vec's `vec0` or FTS5 for keyword search. The two can coexist: FTS5 for keyword recall, USearch for semantic recall, RRF (reciprocal rank fusion) to merge results.

---

## 3. Embedding Provider

**Anthropic's recommendation.** Anthropic does not offer its own embedding model. Their documentation explicitly states: "One embeddings provider that has a wide variety of options and capabilities encompassing all of the above considerations is Voyage AI." This is a partnership-level endorsement, not a casual suggestion. Source: [platform.claude.com/docs/en/docs/build-with-claude/embeddings](https://platform.claude.com/docs/en/docs/build-with-claude/embeddings).

**Voyage AI models (2026 generation).**
- `voyage-4-large`: Best general-purpose quality. 1024-dim default (supports 256–2048). 32K token context. ~$0.00012/1K tokens.
- `voyage-4`: Balanced quality/cost. Same dimensions. ~$0.00006/1K tokens.
- `voyage-4-lite`: Lowest latency and cost. ~$0.00002/1K tokens.
- `voyage-4-nano`: Open-weight (Apache 2.0), available on Hugging Face. Can be run on-device via CoreML conversion.
- Free tier: 200M tokens per account — sufficient for months of personal assistant use before billing begins.

Source: [docs.voyageai.com/docs/embeddings](https://docs.voyageai.com/docs/embeddings), [docs.voyageai.com/docs/pricing](https://docs.voyageai.com/docs/pricing).

**OpenAI embeddings.**
- `text-embedding-3-small`: 1536-dim. $0.00002/1K tokens. MTEB scores competitive but below Voyage 4-series.
- `text-embedding-3-large`: 3072-dim. $0.00013/1K tokens.
- Default used by Mem0's open-source version: `text-embedding-3-small`. Source: [github.com/mem0ai/mem0](https://github.com/mem0ai/mem0).

**On-device via CoreML / swift-transformers.** The `voyage-4-nano` open-weight model can theoretically be converted to CoreML and run entirely on-device. Apple's `swift-transformers` package supports downloading and running HuggingFace transformer models in Swift, though primary examples are generative models, not embedding models. Embedding inference on an iPhone 15 Pro with CoreML Neural Engine should run at ~5–20ms per document for a 256M-param model — acceptable for incremental indexing (one memo at a time), too slow for bulk re-indexing (100K documents would take minutes). Privacy is maximized: no data leaves the device.

**Cost analysis for coo4one personal use.** A personal assistant accumulating 10K memos at ~500 tokens each = 5M tokens total for initial indexing. At `voyage-4` pricing ($0.00006/1K), that's $0.30. Re-embedding on edits adds trivial ongoing cost. The 200M free-token threshold covers all of v0 and likely v1 without billing. Cost is not a decision constraint at personal scale.

**Quality vs. privacy trade-off.** Voyage API calls mean memo content leaves the device to Voyage's servers. For a personal assistant holding sensitive professional and personal data, this is a material privacy consideration. The `voyage-4-nano` CoreML path offers the privacy of on-device embedding with meaningfully lower quality (as an open-weight model trained for efficiency rather than peak MTEB performance). A reasonable default: use Voyage API for indexing on Mac (background task, less latency-sensitive) and either replicate the index to iOS or use NLEmbedding as a fast approximate path on iOS. Never send iOS-device-local context (in-progress conversations) to the embedding API without user consent.

**Recommended embedding provider:** `voyage-4` for quality/cost balance, with `voyage-4-nano` as the privacy-preserving fallback for users who opt out of cloud embedding. Mem0's open-source choice of `text-embedding-3-small` is acceptable but Anthropic's documented partnership with Voyage makes it the natural default for a Claude-backed system.

---

## 4. CloudKit Private Database

CloudKit is Apple's managed cloud backend, free to use up to iCloud account storage quotas, requiring no developer server infrastructure.

**Capabilities.** Records (CKRecord) store typed fields: String, Int, Double, Date, Data, CLLocation, CKAsset (binary blob), and arrays of the above. Private database stores data in the user's iCloud account, not the developer's. CKAsset handles large blobs (up to 4 GB per asset). Subscriptions deliver push notifications when data changes. NSPersistentCloudKitContainer provides CoreData-to-CloudKit bridging with automatic sync. Source: WWDC19 session "Using Core Data with CloudKit."

**Limits.**
- Record body (excluding assets): **1 MB per record.** A long memo as a CKRecord field fits; the entire vault does not.
- CKReference (relationships): limited to **750 objects per reference chain** in practice.
- Free tier storage: uses the user's iCloud storage quota (5 GB free, scales with their subscription).
- Query power: predicate-based (NSPredicate), not SQL. No joins. No full-text search natively. Queries must use indexed fields. Source: NSHipster CloudKit overview.

**Conflict model.** CloudKit's default is **last-write-wins** by record. When two devices write to the same CKRecord, the server timestamps determine the winner. No three-way merge. For structured data (project board, memo index entries), this is usually acceptable. For text content, it means silently losing one device's edit if they conflict. The WWDC19 recommendation for avoiding conflicts is to model data as relationships (many PostContent objects rather than one flat `content` string) or to use CRDT-like patterns in the data model.

**Real-world use by reference apps.** Things 3 uses a custom CloudKit sync engine for task sync. Bear 2 uses CloudKit for its SQLite-canonical store. Drafts uses CloudKit for its draft database. These are all structured-data CloudKit uses — the structured record maps naturally to a task or a note. None of them sync raw markdown files through CloudKit records (that would be iCloud Drive, discussed in §5).

**What fails.** Query flexibility is the most common indie-dev complaint: you cannot do `WHERE content CONTAINS 'keyword'` — full-text search is not a CloudKit feature. Offline behavior requires manual implementation of a local cache (NSPersistentCloudKitContainer handles this if using CoreData, but raw CKRecord usage requires manual caching). Debugging sync issues is notoriously difficult (no server-side logs, opaque error codes). Rate limiting is per-device, not per-account, and limits are undocumented.

**Verdict for coo4one.** CloudKit private database is the right substrate for the **structured derived layer**: the memo index (metadata: id, date, title, status, tags), the project board (tasks, states, owners), and user preferences/settings. It is the wrong substrate for the markdown source files themselves (size limits, no file semantics) and for the vector index (binary blobs at scale exceed practical record sizes).

---

## 5. iCloud Drive as File-Sync Substrate

iCloud Drive presents files as a normal POSIX filesystem with transparent cloud sync. Apps access files via the standard FileManager API; iCloud handles upload, download, and conflict resolution.

**What works.** For a single active device at a time, iCloud Drive is reliable. Files created on Mac appear on iPhone within seconds if both devices are online. The user sees their files in the Files app, can inspect them, can take them to other apps. This transparency is exactly what the COO pattern requires: the user should always be able to see and audit their memory files.

**Conflict resolution model.** When two devices edit the same file offline and then reconnect, iCloud Drive creates a conflict version. The NSFileVersion API exposes `unresolvedConflictVersionsOfItemAtURL()` — the app receives both versions and must resolve. iCloud's default is to keep the most recently modified version and rename the other with a conflict suffix (e.g., `memo-2026-05-11-abc.md` becomes `memo-2026-05-11-abc (conflict on 2026-05-11 at 10:30 AM).md`). The user can then inspect both. Apps that don't handle conflicts at all will silently diverge or produce named conflict files that clutter the folder.

**Failure modes.**
- **Simultaneous edits from two devices:** the most common case where the user edits the same file on Mac and iPad before either syncs. iCloud creates a conflict; the app must resolve it or present it to the user.
- **Eviction from local storage:** on iPhone with storage pressure, iCloud Drive files can be evicted to cloud-only status. An app that tries to read an evicted file gets an error until download completes. For a boot-time reading sequence (the COO's "read identity files at session start"), this is a critical failure: the agent tries to read its charter and gets a "downloading..." error.
- **Sync latency:** not instant. Background sync on iOS runs on Apple's schedule, not the app's. A file written on Mac may take 30 seconds to 5 minutes to appear on iPhone if the iPhone is not actively running the app.

**Apps that handle iCloud Drive well.** Obsidian (with caveats — they warn against using iCloud Drive alongside Obsidian Sync due to double-sync conflicts), Tot (tiny app, tiny files, low conflict probability), Drafts (uses CloudKit for its database, not iCloud Drive for raw files). Bear 2 uses CloudKit for its core database and iCloud Drive for exported files — it separates the two concerns explicitly.

**Apps that fail.** Heavy text editors that auto-save frequently can generate spurious conflicts when two devices are both online and both saving. Logseq explicitly warns against iCloud Drive for its file-canonical version due to block-level conflicts across its many small files.

**Verdict for coo4one.** iCloud Drive is the right substrate for the **markdown source files**, with the following mitigations:
1. Memos are append-only (never edit in place). New content means a new file. This eliminates the conflict case for memos entirely — two devices cannot conflict on a file that only one device ever creates.
2. Identity files (charter, governance, preferences) change infrequently. A conflict on these files is rare and can be surfaced to the user as "two versions exist; please review."
3. The agent must handle file-not-downloaded errors gracefully at boot — either wait for download or degrade gracefully (skip the unavailable file and note it in the session report).
4. Store the markdown files in a dedicated app container folder, not in the user's top-level iCloud Drive, to reduce accidental modification by other apps.

---

## 6. Automerge-Swift / Y-Swift CRDT Options

**Automerge.** A CRDT library implementing the Automerge algorithm, which provides conflict-free replicated JSON documents with rich text support added in v2.2. Production-ready as of v2.0 (January 2023); v3.0 released July 2025 with 10x memory reduction. Swift bindings exist (automerge-swift, 316 stars, last release December 2025, 31 total releases — active). The Swift binding wraps a Rust core compiled to a static library, which can be linked into iOS apps. Source: [github.com/automerge/automerge-swift](https://github.com/automerge/automerge-swift).

**What Automerge is good at.**
- Collaborative real-time editing of text (uses a sequence CRDT for characters).
- Offline-first: edits accumulate locally and merge when devices reconnect.
- Rich text: block elements, inline formatting supported as of v2.2.
- No central server required for peer-to-peer sync (works over any transport).

**What Automerge is bad at.**
- Memory overhead: even at v3.0 (10x reduction), a 100KB document can require several MB of CRDT metadata in-memory. A vault of 10K memos would have significant aggregate overhead.
- Not a file format: Automerge documents are binary (not human-readable). To maintain markdown files as the source, you would need to serialize Automerge → markdown on write and parse markdown → Automerge on read. This round-trip is lossy for rich CRDT metadata (change history, peer attributions).
- Complexity: CRDT sync requires a sync protocol. Automerge provides the data structure; the transport and storage are the developer's problem. Storing Automerge documents in iCloud Drive or CloudKit requires additional plumbing.

**Would a CRDT-based markdown sync work for coo4one?** For memos (append-only, one memo per file, not concurrently edited from two devices): CRDTs are overkill and add complexity without benefit. For ongoing conversation context (the active session, which might be written from Mac while the iPad reads it): CRDTs could eliminate conflicts, but the cost is the binary format and the round-trip complexity.

**Y-CRDT / y-swift.** Y.js is the other major CRDT library (Yjs for JavaScript). A Rust port (y-crdt) exists, with Swift bindings under active development but less mature than automerge-swift. Not recommended for v0 given the lower ecosystem maturity on Apple platforms.

**Verdict.** CRDTs are the right answer for collaborative text editors where two humans concurrently edit the same document. coo4one is a single-user system: the agent writes, the user reads (or vice versa), but they are not simultaneously editing the same memo from two devices. The concurrency problem is device-to-device sync lag, not collaborative editing. **iCloud Drive with append-only memos eliminates the conflict problem without CRDT complexity.** Reserve CRDT consideration for the active session state (the "current conversation" buffer), if that use case emerges.

---

## 7. The Hybrid Pattern

The natural architecture for coo4one is two-layer:

**Layer 1: Markdown files via iCloud Drive.** Source of truth. Each memo, each identity file, each session log is a `.md` file in `~/Library/Mobile Documents/iCloud~com~yourapp~coo4one/Documents/`. Append-only for memos (new file per decision). iCloud Drive handles cross-device sync with eventual consistency. The agent reads from this layer on boot and writes to it on session-end commits.

**Layer 2: Derived structured data via CloudKit + local SQLite.** The memo index (id, date, title, status, tags, file path — JSON-serializable per-memo metadata), the project board (tasks, states), and user settings. Stored in CloudKit private database for cross-device availability. Also cached in a local SQLite database (built from CloudKit on first sync, updated incrementally). The FTS5 full-text index and the vector index (USearch or sqlite-vec) live in this local SQLite only — they are per-device, rebuilt from the markdown files locally, not synced through CloudKit (binary vector indexes are large, frequently updated, and don't need to be on CloudKit since they can be rebuilt from the files).

**Conflict resolution between layers.**
- File edits (Layer 1) trigger a re-index (Layer 2 update). The agent detects new/changed `.md` files via FileSystemWatcher (FSEvents on macOS, no equivalent on iOS — use a poll on app-foreground and background-task wake).
- CloudKit updates (Layer 2) are consumed on sync and materialized into the local SQLite cache.
- Divergence: if a CloudKit record references a memo file that doesn't exist locally (not yet downloaded from iCloud Drive), the agent defers the reference until the file arrives. Never invent data from the index when the source file is unavailable.

**Cache invalidation strategy.**
- Each memo file is hashed (SHA-256 of content) on index. Re-index fires only when hash changes. This avoids re-embedding stable files.
- Vector index entries are keyed by (file_path, content_hash). On memo update, the old vector is deleted and a new one inserted.
- On device boot: scan iCloud Drive directory, compare to local index, re-index any missing or changed files. This boot scan is the equivalent of the COO's session-start reading sequence — it grounds the derived layer in the file reality.

---

## 8. Memory-Inspector UI Design Precedent

Reference apps and their approaches to surfacing memory/notes as first-class objects:

**Obsidian (vault + graph).** The graph view is the memory inspector: nodes are files, edges are links. Users navigate their knowledge through the graph. The key insight: files are the atoms, but the *relationships between files* are the UI layer. The vault browser is a flat file list; the graph is the meaning layer. coo4one's equivalent: the memo list is the flat view; a "decisions graph" (which memos supersede which, which decisions link to which projects) is the meaning view.

**Bear (notes + tags).** The left sidebar is a tag hierarchy; the main panel is the note list filtered by tag; the note content is the rightmost panel. Tags are applied inline in the note (`#tag`). Bear's UI teaches: the tag/filter surface reduces the memory problem from "find anything" to "browse by category." coo4one's memo statuses (active, superseded, archived) and memo topics (CB-*, OG-*, operational) map naturally to Bear's tag pattern.

**Heptabase (cards on canvas).** Cards are the atom; boards are the spatial arrangement. The user manually places cards on the whiteboard to see relationships. This is the highest-fidelity spatial memory model but requires the most active maintenance from the user. For coo4one, the auto-generated "today's active decisions" board (a curated subset of memos relevant to current tasks) could be the Heptabase-inspired surface without requiring the user to manually arrange cards.

**Logseq (blocks + pages).** The daily journal page is the primary entry point; everything else is linked from it. The memory inspector is the graph view (same as Obsidian) plus a queries panel (Datalog queries over the block graph). The daily-journal-as-spine is load-bearing for coo4one: each session log is the day's journal entry, and the agent generates it automatically.

**Design direction for coo4one.** Three panels:
1. **Inbox / Active** — decisions pending, tasks in flight, items that need attention. Auto-surfaced by the agent.
2. **Case-law browser** — searchable, filterable memo list. Status filter (active/superseded). Topic filter. Full-text search. Semantic search ("decisions about email" → retrieves relevant memos even if the word "email" doesn't appear).
3. **Today's brief** — the session log entry for today: what the agent did, what was decided, what's next. Handoff in readable form.

The memory inspector is not a graph view at v0. The graph emerges from the data (linked memos) but the primary interaction is list + search. Add the graph view as a power-user feature in v1 when the corpus is large enough to make it meaningful.

---

## 9. Multi-Device Session Model

**What "the iPad picks up where the Mac left off" means technically.** The session state that needs to transfer is: (a) what was the last open context (the active project or conversation), (b) what was the last thing the agent was doing, (c) what memos/files were recently read. This is fundamentally different from transferring an agent *conversation* — the model context cannot be transferred, only the *inputs* to reconstruct context.

**Apple Handoff API.** NSUserActivity encodes lightweight state (a dictionary, a URL, a search string) that Handoff transmits to nearby devices via Bluetooth/iCloud. The receiving device launches the app and calls `continue(userActivity:)`. Handoff state must be small (the `userInfo` dictionary is limited in size). Appropriate for: "open memo X", "open project Y", "resume search query Z." Not appropriate for: "here is the last 10,000 tokens of agent context."

**NSUbiquitousKeyValueStore.** iCloud key-value store. 1 MB total limit per app. Syncs within seconds across devices. Appropriate for: last-open memo ID, last-open project ID, user preferences. Not appropriate for: conversation history or large session state.

**CloudKit for session continuity.** A lightweight "session handoff" record in CloudKit: `{ last_memo_id, last_project_id, last_active_at, brief_context_hint }`. When the iPad opens the app, it reads this record, opens the referenced memo/project, and the agent reads the boot files + the referenced context to reconstruct the session. This is load-from-store, not state-transfer — the agent re-derives context from durable memory, not from transferred model state. This matches the COO pattern: the agent never relies on "what the model remembers" but on "what the files say."

**Always-load-from-store pattern.** The right session model for a Claude-backed agent:
1. Device A finishes a session: writes session log to iCloud Drive file, updates CloudKit session record with `last_active_context`.
2. Device B opens app: reads session log file (boot sequence), reads CloudKit session record for "where were we," opens that context.
3. Agent greets user with status recall based on what it read — not on what it "remembers."

This eliminates the need for context transfer and is robust to long inter-session gaps (weeks without use). The COO's boot discipline is exactly this pattern — and it's the right model for coo4one.

**Siri/Handoff integration.** Register common agent tasks as App Intents (e.g., "add a decision," "what's my next task," "brief me on project X"). These become Siri shortcuts and Handoff activities automatically. The agent doesn't need to be running on the receiving device for Handoff to open the right view.

---

## Recommended Substrate

### Primary recommendation: iCloud Drive files + CloudKit structured index + local SQLite per-device

**Architecture in brief:**
- Markdown source files: iCloud Drive, app container, append-only for memos.
- Structured derived data (memo index, project board, settings): CloudKit private database, locally cached in SQLite.
- Full-text index: SQLite FTS5, per-device, rebuilt from files.
- Vector index: sqlite-vec (10K corpus) or USearch (100K corpus), per-device, rebuilt from files.
- Embedding provider: `voyage-4` API (cloud, Anthropic's recommended partner), with `voyage-4-nano` CoreML on-device as privacy-preserving alternative.
- Session continuity: always-load-from-store (boot discipline), lightweight CloudKit session record for "last active context," NSUserActivity for in-session Handoff.

**Why this wins:**
- Markdown files are visible, auditable, version-controllable. The user can always see and export their data.
- CloudKit is zero-infra, zero-cost at personal scale, natively integrated with Apple's sync stack.
- Local SQLite indexes are fast, offline-capable, and trivially rebuildable.
- Append-only memos eliminate iCloud Drive conflicts for the most important content class.
- The COO's "file wins on divergence" invariant is structurally enforced: no path writes to the index first and propagates to files.

**Trade-offs:**
- iCloud Drive sync latency (seconds to minutes, not instant). Acceptable for a personal assistant where the user is not simultaneously editing from two devices.
- CloudKit query limitations (no full-text search natively). Mitigated by keeping FTS5 local.
- Embedding API calls send memo content to Voyage's servers. Mitigated by the on-device fallback option and by user consent at indexing time.
- sqlite-vec is pre-v1 (API instability). Mitigated by the USearch fallback and by the fact that the vector index is a derived, rebuildable layer.

### Fallback: iCloud Drive files + local SQLite only (no CloudKit)

If CloudKit proves too limiting (query constraints, debugging opacity, or App Store review issues with CloudKit entitlements), the structured derived data can live entirely in a per-device SQLite database, with iCloud Drive syncing the database file itself. This is the "SQLite file in iCloud Drive" pattern used by some indie apps. Conflict risk is higher (two devices writing the same SQLite file can corrupt it), mitigated by WAL mode and careful write coordination (only one device writes at a time, others read). Obsidian uses this pattern for its local index. The fallback degrades gracefully: no CloudKit dependency, entirely offline-capable, but cross-device index consistency depends on iCloud Drive file sync reliability.

Do not adopt the fallback unless CloudKit proves specifically problematic during prototyping. The primary recommendation is well-validated by Things 3, Bear 2, and Drafts — three of coo4one's reference apps — all of which use CloudKit for their structured data layer.

---

## Open Questions

1. **CoreML conversion of voyage-4-nano.** The model is Apache 2.0 on HuggingFace but the CoreML conversion pipeline (via `coremltools`) has not been validated in this research. Lane B (Anthropic SDK + Swift) may have partial coverage; the question of "which embedding model can run on-device via Apple Neural Engine at acceptable latency" needs a prototype, not just research.

2. **FileSystemWatcher on iOS.** FSEvents (macOS) provides real-time file change notifications. iOS has no equivalent for iCloud Drive containers — the app must poll on foreground and use BGTaskScheduler wake for background indexing. The latency between file write and index update on iOS needs empirical measurement.

3. **CloudKit + App Store review for AI agent apps.** The genesis analysis flagged App Store review as a risk. Lane A covers this, but the intersection of CloudKit entitlements + AI agent behavior + user data needs explicit research.

4. **voyage-4 availability from Swift.** Voyage AI provides a Python SDK and HTTP API. A Swift wrapper must be written by the coo4one team (a thin URLSession client). No official Swift SDK exists as of the research date.

5. **Obsidian iCloud Drive failure mode depth.** Obsidian's warnings about iCloud Drive conflicts are widely cited but the specific triggering conditions (concurrent saves? rapid file creation? large vaults?) are not fully characterized in available documentation. A controlled test with the target vault size would resolve this before committing to the iCloud Drive file substrate.

---

*Sources cited inline. Research conducted 2026-05-11. Do not write any files beyond this report.*
