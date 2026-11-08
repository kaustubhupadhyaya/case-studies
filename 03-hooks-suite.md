# Making agent guardrails structural: about 18 hooks, 1,310 lines, and the one that never fired

## The problem

Instructions in a prompt are advice. An autonomous coding agent under task pressure will find the shortest path around advice, especially when the rule is "do not write to that system" or "cite evidence for every claim." The same problem applies to context: an agent starts every session blank unless something injects prior work before the first message.

The goal was to move the rules out of the prompt and into the tool-call boundary. If an operation is forbidden, make it fail. If evidence is required, check the output. If context should be present, inject it before the model sees the user's message.

## What I built

About 18 hooks totaling 1,310 lines across 23 files, registered in two settings files. The Windows and WSL copies of the main settings file were hardlinked to the same inode, so one edit updated both environments. That removed an entire class of drift between the two installs.

The hooks cover the full event surface: session start, session end, user prompt submit, pre-tool use, post-tool use, stop, and permission request.

### A three-layer write block

One rule mattered more than the rest: the agent must not write to the organization's issue tracker or wiki. A single hook is bypassable, so the rule was implemented three independent ways.

1. **Tool-name matcher.** A pre-tool-use hook matches the write-capable tool names and exits with code 2, which denies the call.
2. **Bash command parser.** A pre-tool-use hook on shell commands denies write verbs to the service's domains: POST, PUT, PATCH, DELETE, comment endpoints, `curl --data`, and Python `requests` or `httpx` write calls.
3. **A curl shim on PATH.** The real curl binary was renamed, and a wrapper earlier on PATH inspects every curl invocation before deciding whether to exec the real binary. This layer still fires when the hook engine itself is bypassed.

Exactly one escape hatch was allowed: a release-notes script writing to one specific wiki page. The allowlist was duplicated in both the Python parser and the curl shim instead of being shared. That costs two places to maintain, but it keeps the layers genuinely independent rather than giving them one common path that can be defeated once.

A separate hook blocks both `git push` and `git commit`, so agent changes stay unstaged for manual review.

### A primer that survives compaction

A 148-line session-start hook retrieves prior work and injects it as context. It uses the plain-markdown vault as the primary source and a vector store as a secondary source. Hits below a 0.45 score floor are dropped, a boilerplate drop list removes known polluters, and the hook emits nothing at all when no hit clears the bar. Emitting nothing is deliberate: stale filler is worse than a cold start.

The primer logged 2,942 emissions and 23 suppressions between 2026-04-24 and 2026-07-11. Those are primer events, not agent runs.

### Style and operational hooks

Several hooks enforce rules that a prompt could not reliably hold:

- An acronym glossary pins domain terms so the model does not substitute a general-knowledge meaning.
- An evidence hook requires every factual claim, positive or negative, to cite a file read, log line, command output, or line number.
- A pre-edit reminder forces a structural comparison before editing code tied to a ticket and a port or variation.
- A post-edit guard reminds the agent to propagate configuration changes to sibling environments and runs a SQL preflight before a database change.
- A post-edit hook runs the test suite after any Python file change.
- A write guard blocks delivery-artifact markdown files unless the user explicitly names a file, keeping proposals and summaries in chat.

On Windows, a separate notifier trio plays distinct sounds for task completion, permission requests, and questions, so a long unattended run does not sit silently waiting for input.

## The hook that never fired

One diagram-hint hook was registered, versioned, and completely dead. It grepped an empty string literal because the variables meant to hold the user's prompt were declared empty and never populated from standard input. The condition was always false. The hint never appeared.

Registration in `settings.json` is not evidence that a hook works. The only reliable audit is to trigger the event and observe the output.

A related trap: a hook that writes plain text to standard output is invisible to the model. Hooks whose output must reach the model have to emit `hookSpecificOutput.additionalContext` JSON. One guard was rewritten specifically because its predecessor used plain stdout and had no effect.

## Lessons

Layers only count if they are independent. Renaming the real binary and shadowing it on PATH is ugly, but it closes a path that the hook engine does not cover.

A deny rule needs one tested escape hatch. Zero exceptions makes the block unusable and invites someone to disable it. A named, narrow allowlist keeps it in place.

An allowlist duplicated across layers is a maintenance cost worth paying. A shared helper would have been cleaner and less safe.

A hook that emits nothing on a miss needs a log. The anti-bloat gate is correct, but without the emitted and suppressed counts there is no way to tell a working gate from a broken hook.
