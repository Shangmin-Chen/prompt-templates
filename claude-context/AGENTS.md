# claude-context — multi-phase pair programming

A protocol for using Claude (chat) and Cursor (executor) as pair
programmers across multiple phases. Built around the idea that risky
mechanical work is best done in chunks with human checkpoints between
them.

## Shape

- **Phase 1a (Discovery):** Claude writes a read-only Cursor prompt
  that explores the codebase and reports back structured context.
- **Phase 1b (Targeted follow-ups):** Chat-only clarifications and
  decisions, no new agent runs.
- **Phase 1c (Plan):** Claude proposes a plan, you approve or amend.
- **Phase 2 (Execution):** Claude writes a Cursor prompt with hard
  rules, pre-flight checks, recovery commands, and **pause points
  after every chunk**. You review each chunk's report before
  approving the next.

The defining feature is the pause-and-report rhythm during execution.
You stay in the loop continuously, the agent doesn't autonomously run
past checkpoints, and you can pivot or retreat at any chunk boundary.

## How to use it

Paste as message 1, then immediately follow with your task in
message 2:

```text
[paste of claude-context.md]

I have a migration that needs to roll out to 3 environments and I'm
worried about ordering. Here's the situation: ...
```

The full protocol text to paste is in [`claude-context.md`](./claude-context.md) in this directory.

## What it captures

- The two-phase shape (Phase 1 with Claude, Phase 2 with Cursor)
- Diagnostic-first instinct — ask for command output before proposing
  fixes
- Agent prompt template with pre-flight, hard rules, recovery, pause
  points
- Multi-agent specifics — main thread mutates, subagents verify in
  parallel
- How to handle agent reports — read carefully, distinguish real
  failures from harmless ones
- Tone matching, no sycophancy, willingness to retreat

## When this is the right tool

- Stacked PRs across a codebase
- Cross-environment migrations (dev → staging → prod)
- Risky refactors where you want eyes on every chunk
- Debugging in unfamiliar territory
- Anything where "land it all at once" feels reckless

## Choosing vs one-shot protocol

If you can describe the end state in one sentence and the diff is
bounded, consider [`../claude-gemini-cursor-protocol/AGENTS.md`](../claude-gemini-cursor-protocol/AGENTS.md).
If you'd want to stop and look around after each piece lands, use this
folder. If you're not sure, default here — the cost of pausing
unnecessarily is lower than letting an under-specified plan run.
