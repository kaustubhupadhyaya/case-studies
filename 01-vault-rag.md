# Two-tier retrieval over 2,414 notes of AI-session history

## The problem

Every AI coding session ended as a file nobody read again. The transcript was there, but the next session started with no memory of the last one. Restarting meant re-explaining the project, re-discovering decisions, and sometimes repeating a mistake that had already been fixed.

Plain search was not enough. Keyword search missed paraphrases. Embedding search missed exact identifiers such as file paths, ticket keys, and function names. Running either one alone produced the same result: a long list of vaguely related sessions, most of them useless at the moment a new session started.

## What I built

A pipeline that turns finished sessions into a linked markdown vault, then retrieves from two different representations of the same history at session start.

**Ingestion.** An exporter normalizes session logs from several coding agents into one frontmatter-tagged markdown note per session. A second stage extracts entities from those notes and builds a knowledge graph, emitting wiki-linked hub notes for projects, tools, paths, and tags. The vault is the durable artifact. Any editor can read it, and deleting the retrieval layer does not destroy the history.

**Tier 1: hybrid search over cleaned markdown.** A markdown index combines SQLite FTS5 BM25 with 384-dimensional dense vectors from a small embedding model. The two result lists are fused with Reciprocal Rank Fusion at k=60. This tier sees truncated, frontmatter-tagged notes, which makes it good at project and topic recall.

**Tier 2: semantic search over raw session data.** A separate vector store indexes the original session records. It keeps detail that the markdown export truncates, and it gives a second, independent view of the same corpus.

**Retrieval.** A session-start hook builds a query from the working directory, the current git branch, and the last commit. It asks both tiers, then injects the top hits as context before the first user message. If nothing clears the relevance bar, it injects nothing.

## Scale

- 2,414 markdown notes: 271 Claude Code session notes, 195 Codex exports, and 1,944 generated hub and entity notes.
- 380 unattended ingestion ticks, every two hours, from 2026-05-13 to 2026-07-12.
- The primary index holds 468 documents and 1,778 chunks, with a 1,626 x 384 float32 vector store.
- The knowledge graph holds 468 notes, 1,475 entities, and 8,908 co-occurrence edges.

## What broke, and what fixed it

**Overlapping ticks corrupted the vector store.** Two ingestion runs could overlap and race on the same index files. The result was a wiped embedding file and a forced full rebuild that took about 13 minutes. The fix was a non-blocking single-instance lock, per-phase timeouts of 600 seconds for export and 1,800 seconds for reindex, low CPU and I/O priority, and log rotation at 50 MB. The lock is not decorative. It is the component that keeps the index from destroying itself.

**A silent workflow move froze the entire pipeline.** The exporter originally globbed one project directory. Work moved to a different workspace, the glob matched nothing, and exports stopped. Because the graph stage only runs after new exports, the graph stopped too. Nothing failed loudly. The scheduler kept ticking and reporting success. The fix was to glob all project subdirectories while deliberately excluding subagent transcripts, which are not standalone sessions. The broader lesson was to monitor for artifacts produced, not for process liveness.

**The two tiers index different things on purpose.** Tier 1 indexes cleaned and truncated markdown; Tier 2 indexes raw records. A full rebuild skipped 1,946 chunks that were too short to be useful, against 1,778 that were kept. That filtering is a feature of the cleaned tier, not a bug in the raw tier.

**A second pipeline reused the engine without forking it.** A separate workspace needed its own vault, index, lock, log, and kill switch, but not a second copy of the code. The pipeline overrode the vault and session directories at runtime. Its schedule was offset by 30 minutes from the main pipeline, so the two could never contend for the same files.

## Lessons

A concurrency guard is only as good as its failure mode. The first version had no lock and worked until it did not. Once a corrupted index costs 13 minutes of rebuild time, the lock pays for itself.

Silent success is worse than a crash. A scheduler that ticks every two hours while exporting nothing looks healthy from the outside. The pipeline needed a progress check, not a liveness check.
