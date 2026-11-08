# Semantic memory over 318 AI coding sessions, and the 87-second salvage that saved it

## The problem

Work was spread across four AI coding tools. Each one kept its own transcripts in its own format: JSONL from one, JSONL with a different schema from another, IDE chat files from a third, markdown plans from a fourth. There was no way to ask "what did I do about this three months ago" across all of them.

Grep over transcript files answers exact-string questions. It does not answer "where did I solve the retry problem" when the note used different words. A searchable index needed a common record format, chunking that preserved enough context to be useful, and a retrieval path that ran automatically instead of depending on the user remembering to search.

## What I built

A FastMCP server exposing six tools over one vector collection: search sessions, read session detail, list recent sessions, reindex everything, summarize work, and report index stats.

**Parsing.** Each source gets its own parser, and all of them produce the same `ParsedSession` record. The parsers handle the ugly parts per tool: deduplicating streamed assistant chunks by message id, stripping injected IDE tags, skipping encrypted reasoning blocks, deriving a session id from a parent directory when the file has none, and falling back to file modification time when the format has no timestamps.

**Chunking and embedding.** Each session becomes one summary document plus overlapping chunks of 1,500 characters with 200 characters of overlap. The summary document is what search returns first; chunks carry the detail. Embeddings come from the vector store's bundled small sentence model, and the collection uses cosine distance.

**Indexing.** A session-end hook reads the transcript path from its input and forks a background process. The hook returns immediately, so indexing never delays session exit. It skips sessions that are already indexed.

**Retrieval.** A session-start hook queries the collection, drops boilerplate hits, applies a 0.45 relevance floor, keeps the top two results, and injects them as context. If nothing clears the floor, it emits nothing rather than stale filler.

There is no fixed run count. Indexing is event-driven: one session ends, one session is indexed. The only scheduled job in the service directory handles a separate markdown index and never touches this collection.

## Scale

- 14,473 documents across 318 sessions.
- 14,155 chunk documents plus 318 summary documents.
- Sessions by source: 222 Claude Code, 63 Codex, 30 Cursor, 3 plan files.
- Indexed content spans 2026-01-02 to 2026-07-10.
- About 230 MB of metadata and embeddings in SQLite, plus a 28.8 MB HNSW segment.

## Recovery one: a corrupted vector index

The index stores metadata in SQLite and vectors in an HNSW segment. The two fell out of sync. Queries started failing with `Error finding id`, and the normal repair path could not open the collection cleanly.

The recovery tool did not trust the index as a whole. It enumerated every id cheaply, then fetched records one at a time, copying stored vectors into a fresh collection and rebuilding a clean HNSW graph. Records that failed individually were skipped instead of aborting the run. Unrecoverable session ids were written to a separate file so they could be re-indexed from source later.

The numbers from the salvage log:

- 14,385 ids enumerated.
- 14,337 recovered.
- 48 documents dropped across 3 unrecoverable sessions.
- 87 seconds total.
- Three probe queries returned hits against the rebuilt collection.

The old database was kept as a dated backup. Nothing was destroyed to complete the recovery.

## Recovery two: a semantic attractor in the source data

A second failure was quieter. Codex transcripts begin with a large, near-identical block of agent instructions and IDE context. That block was being included in the session summary. Because it was identical across sessions, it embedded to nearly the same vector every time and matched almost any query. It polluted 57 of 63 Codex summaries and capped their relevance near 0.49, so the boilerplate outranked the actual work.

The fix had two independent halves. The parser strips the instruction and IDE-context blocks before summarization, in the right order so removing one anchor does not orphan the next. The retrieval hook also keeps a boilerplate drop list and a score floor, so even a bad document that slips through cannot be injected.

Re-indexing all 63 Codex sessions took 168 seconds. The reindex ran against a copy of the database, verified that zero summaries still contained the boilerplate, and only then replaced the live file. The copy-then-swap step exists because the session-end hook can be writing to the live database at any moment.

## Lessons

A vector database is not one file. Metadata and the graph can desynchronize, and a repair that operates record by record is slower but survives partial corruption.

Never write to a live index from two processes. Every bulk repair ran against a copy, was verified, and was swapped in.

The retrieval layer should distrust its own vector index when a cleaner source exists. This system treats plain markdown as primary and the embedding store as secondary. That ordering is what kept a single polluted document class from poisoning every query.
