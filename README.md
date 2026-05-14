# prompt-templates

A collection of prompt templates by Simon.

This repo holds two distinct working protocols. They are not
interchangeable — pick based on what kind of work you're doing.

| File | Style | When to use |
|---|---|---|
| `claude-context.md` | **Multi-phase, iterative** | Risky, exploratory, or staged work where each step's output feeds the next decision. Stacked PRs, migrations across environments, refactors where you want a human in the loop after every chunk. |
| `claude-gemini-cursor-protocol.md` | **One-shot implementation** | Tasks that should land as a single, complete diff. Features, bug fixes, mechanical changes where the planning loop produces a fully specified plan and the executor follows it without judgment. |

Both files are pasted into a fresh chat as the first message, before
you describe your actual task.

---

## `claude-context.md` — multi-phase pair programming

A working protocol for using Claude (chat) and Cursor (executor) as
pair programmers across multiple phases. Built around the idea that
risky mechanical work is best done in chunks with human checkpoints
between them.

### Shape

- **Phase 1a (Discovery):** Claude writes a read-only Cursor prompt that
  explores the codebase and reports back structured context.
- **Phase 1b (Targeted follow-ups):** Chat-only clarifications and
  decisions, no new agent runs.
- **Phase 1c (Plan):** Claude proposes a plan, you approve or amend.
- **Phase 2 (Execution):** Claude writes a Cursor prompt with hard
  rules, pre-flight checks, recovery commands, and **pause points after
  every chunk**. You review each chunk's report before approving the
  next.

The defining feature is the pause-and-report rhythm during execution.
You stay in the loop continuously, the agent doesn't autonomously run
past checkpoints, and you can pivot or retreat at any chunk boundary.

### How to use it

Paste as message 1, then immediately follow with your actual task in
message 2:

```text
[paste of claude-context.md]

I have a migration that needs to roll out to 3 environments and I'm
worried about ordering. Here's the situation: ...
```

The context prompt establishes the working style. Your task
description supplies the situation. From there, the new chat should
naturally drop into diagnostic-first mode.

### What it captures

- The two-phase shape (Phase 1 with Claude, Phase 2 with Cursor)
- Diagnostic-first instinct — ask for command output before proposing
  fixes
- Specific patterns — "run these, paste output, then we'll know X"
- Agent prompt template with all the structural pieces (pre-flight,
  hard rules, recovery, pause points)
- Multi-agent specifics — main thread mutates, subagents verify in
  parallel
- How to handle agent reports — read carefully, distinguish real
  failures from harmless ones
- Tone matching, no sycophancy, willingness to retreat

### When this is the right tool

- Stacked PRs across a codebase
- Cross-environment migrations (dev → staging → prod)
- Risky refactors where you want eyes on every chunk
- Debugging in unfamiliar territory
- Anything where "land it all at once" feels reckless

---

## `claude-gemini-cursor-protocol.md` — one-shot implementation

A four-role handoff protocol for tasks that should land as a single
diff. Trades the per-chunk human pause points for an **independent
reviewer (Gemini CLI)** that gates execution — and post-execution
conformance verification.

### Roles

| Role | Tool | Job |
|---|---|---|
| Thinker | Claude (chat) | Writes prompts for the other three, interprets every artifact, decides next move |
| Auditor / Reviewer | Gemini CLI | Grounds Claude in the codebase, reviews the plan before execution, reviews the diff after |
| Planner | Cursor (single comprehensive plan) | Produces one complete `planner.md` |
| Writer | Cursor multi-agent | Executes `planner.md` concurrently, no pauses |

### Shape

The full loop, end to end:

```
Task description in chat
  → Gemini audit (codebase context)
  → Cursor planner produces planner.md
  → Gemini plan review (APPROVE / REVISE)
    → REVISE: Claude writes edit prompt, planner revises, re-review
    → APPROVE: Cursor multi-agent executes
  → Gemini conformance review (MATCH / SMALL_DRIFT / BIG_DRIFT)
    → MATCH: done
    → SMALL_DRIFT: Claude writes fix prompt, executor patches
    → BIG_DRIFT: git rollback, re-enter planner loop
```

Two principles do the load-bearing work:

1. **Every agent starts with zero memory of every other agent.** The
   protocol is a context-relay race. Claude's central job is to be the
   relay coordinator.
2. **Executor purity.** The writer is dumb on purpose. Smart logic
   lives in the planner. If `planner.md` ever asks the executor to
   "figure out" or "handle appropriately," that's a blocking review
   issue — even if Gemini missed it.

### How to use it

Same pattern as `claude-context.md` — paste as message 1, follow with
your task in message 2:

```text
[paste of claude-gemini-cursor-protocol.md]

I need to add OAuth login (Google + GitHub) to my Next.js app. Currently
using email+password via Auth.js.
```

Claude responds with the Gemini audit prompt. You run it, paste the
report back, and the loop unfolds from there.

### Human role during the loop

You're watching the whole protocol stream by. There is no dedicated
"human checkpoint" step — you interject whenever something looks off,
and Claude folds your note into the artifact currently in flight
without restarting the loop. The protocol assumes you're ambient.

### When this is the right tool

- A single feature that lands as one diff
- A bug fix where the root cause is understood
- Mechanical refactors with a clear endpoint
- Anything where the plan can be fully specified up front and the
  executor doesn't need to make decisions

### When it's the *wrong* tool

- Exploratory work where you don't know what the right shape is yet
- Multi-PR stacks
- Work that should be staged across environments with verification
  between each stage

For those, use `claude-context.md`.

---

## Choosing between them

A rough heuristic:

- If you can describe the end state in one sentence and the diff is
  bounded → `claude-gemini-cursor-protocol.md`.
- If you'd want to stop and look around after each piece lands →
  `claude-context.md`.
- If you're not sure → `claude-context.md`. The cost of pausing
  unnecessarily is lower than the cost of letting an under-specified
  plan run to completion.

---

## Other templates

Other files in this repo are agent prompts — discovery prompts,
execution prompts, audit prompts, review prompts — that the two
protocols above generate or reference. They live here as reusable
starting points when a task fits a known shape.

---

## Things to add over time

As you do more sessions, you'll probably notice patterns. Consider
adding:

- Repo-specific conventions (branch naming, commit message scope tags
  your team uses, CI requirements)
- Stack-specific knowledge (your TypeScript/Python conventions, what
  your typecheck/lint scripts are called)
- Personal preferences that evolved (preferred phase size, how
  aggressive about subagent parallelism, when you'd want
  `claude-context.md` over the protocol or vice versa)
- New protocol variants if you find a workflow shape neither file
  captures

Treat both files as living docs. Edit them after each session where
you notice "I wish Claude had defaulted to X."