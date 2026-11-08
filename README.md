# Case studies

Three engineering write-ups about running AI coding agents at scale. Each one starts with a failure mode, shows the design that addressed it, and keeps the numbers from the logs, including the parts that broke.

These are not tutorials or resume bullets. They are the post-mortem version: what the system looked like, what went wrong, and what the recovery actually cost.

| # | Case study | Scale | Core problem |
|---|---|---|---|
| 1 | [Two-tier retrieval over a 2,414-note vault](01-vault-rag.md) | 380 unattended ingestion ticks | Past sessions were write-only; keyword and semantic search each failed alone |
| 2 | [Semantic memory over 318 sessions](02-session-history-rag.md) | 14,473 documents | Four tools, four transcript formats, no cross-tool recall |
| 3 | [Structural guardrails for an autonomous agent](03-hooks-suite.md) | About 18 hooks, 1,310 lines | Prompt rules are advice; tool-call boundaries are enforcement |

## The through-line

All three systems sit around the same loop: an agent finishes a session, the session becomes searchable history, and the next session starts with relevant prior work already loaded. The guardrails stop the agent from doing damage while that loop runs.

The recurring lesson is that silent success is the dangerous state. A scheduler that ticks while exporting nothing looks healthy. A vector index that answers queries until its metadata and graph desynchronize looks healthy. A hook listed in a settings file looks healthy until you check whether it ever fired. Every system here needed a progress signal, not a liveness signal.

## Sanitization

These write-ups are sanitized versions of private engineering notes. Employer and product names, ticket and page identifiers, internal hostnames, filesystem paths, and personal names have been removed or generalized. The counts, date ranges, and recovery details are unchanged from the source logs. No customer data, credentials, or proprietary source code is included.

## License

Copyright 2026 the author. Licensed under the [Creative Commons Attribution 4.0 International License](LICENSE).
