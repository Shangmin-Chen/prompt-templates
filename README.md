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

A protocol for using Claude (chat) and Cursor (executor) as pair
programmers across multiple phases. Built around the idea that risky
mechanical work is best done in chunks with human checkpoints between
them.

### Shape

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

### How to use it

Paste as message 1, then immediately follow with your task in
message 2:

```text
[paste of claude-context.md]

I have a migration that needs to roll out to 3 environments and I'm
worried about ordering. Here's the situation: ...
```

### What it captures

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

### When this is the right tool

- Stacked PRs across a codebase
- Cross-environment migrations (dev → staging → prod)
- Risky refactors where you want eyes on every chunk
- Debugging in unfamiliar territory
- Anything where "land it all at once" feels reckless

---

## `claude-gemini-cursor-protocol.md` — one-shot implementation

A four-role handoff protocol for tasks that should land as a single
diff. Trades per-chunk human pause points for an **independent
reviewer (Gemini CLI)** that gates execution and reviews the diff
afterward.

### Roles

| Role | Tool | Job |
|---|---|---|
| Thinker | Claude (chat) | Writes prompts for the other three, interprets every artifact, triages reviewer rejections |
| Auditor / Reviewer | Gemini CLI | Grounds Claude with audit, reviews plan, reviews diff. Every verdict binary. |
| Planner | Cursor (single comprehensive plan) | Produces one complete `planner.md` |
| Writer | Cursor multi-agent | Executes `planner.md` concurrently, no pauses |

Every prompt Claude writes for an agent opens with a `## Your role`
block. Each agent only sees the one prompt — so the role is declared
every time.

### Shape

The full loop:

```
Task description in chat
  → Gemini audit (codebase context)
  → Cursor planner produces planner.md
  → Gemini plan review (APPROVE / REJECT)
    → REJECT: Claude writes planner edit prompt, planner revises,
              re-review
    → APPROVE: Cursor multi-agent executes, then Gemini conformance
               review
  → Gemini conformance review (APPROVE / REJECT)
    → APPROVE: done. Protocol ends. No paste-back.
    → REJECT: Claude triages →
        Path A — Small fix: Claude writes fix prompt + re-review
                 prompt in one response. Same Cursor context fixes,
                 Gemini re-reviews.
        Path B — Rollback: git rollback, fresh Cursor context, new
                 planner prompt. Loop restarts at planning.
```

### Five load-bearing principles

1. **Every agent starts with zero memory of every other agent.** The
   protocol is a context-relay race. Claude is the relay coordinator.
2. **Executor purity.** The Cursor writer is dumb on purpose. Smart
   logic lives in the planner. If `planner.md` ever asks the executor
   to "figure out" or "handle appropriately," that's grounds for
   REJECT.
3. **Trust between agents.** No chaperoning. Cursor knows how to
   write its own files. The user pastes `planner.md` into Gemini
   directly. No `.agent/` directory ceremony.
4. **Phases run concurrently by default.** Sequential ordering is
   the exception and must be justified.
5. **Reviewers commit. Claude triages.** Gemini returns binary
   verdicts — APPROVE or REJECT, no hedging. Claude decides what to
   do on REJECT.

### How to use it

Same pattern as `claude-context.md`:

```text
[paste of claude-gemini-cursor-protocol.md]

I need to add OAuth login (Google + GitHub) to my Next.js app.
Currently using email+password via Auth.js.
```

Claude responds with the Gemini audit prompt. You run it in Gemini
CLI, paste the report back, and the loop unfolds.

### Human role during the loop

You're watching the protocol stream by. There is no dedicated "human
checkpoint" step — you interject when something looks off, and
Claude folds your note into the artifact currently in flight without
restarting the loop. The protocol assumes you're ambient.

### What the conformance review covers

The post-execution review bundles three checks:

- **Plan conformance** — did the executor build what the plan said?
- **Executor purity in code** — did the executor stay dumb, or did
  smart logic creep in?
- **Security and correctness** — injection risks, auth bypasses,
  logic flaws, missing edge cases.

Plus **task-specific watchpoints** Claude generates from the audit
and accepted risks. 0-4 sharpened attention items per task.

Any finding worth flagging → REJECT. Gemini doesn't classify the size
of the remediation. That's Claude's job during triage.

### Triage on conformance REJECT

Claude reads the findings and decides between:

- **Path A — Small fix.** Bounded findings, same executor context can
  apply the fix. Claude writes two prompts in one response: the fix
  for Cursor and the re-review for Gemini. You run them in order.
- **Path B — Rollback.** Structural findings, fix would meaningfully
  exceed a small prompt, or this is the third REJECT round.
  `git` rollback, fresh Cursor context, new planner prompt.

Repeated small fixes = the protocol is leaking. Claude should be
recommending Path B by the third REJECT round.

### When this is the right tool

- A single feature that lands as one diff
- A bug fix where the root cause is understood
- Mechanical refactors with a clear endpoint
- Anything where the plan can be fully specified up front

### When it's the *wrong* tool

- Exploratory work where the right shape isn't clear yet
- Multi-PR stacks
- Work staged across environments with verification between stages

For those, use `claude-context.md`.

---

## Choosing between them

- If you can describe the end state in one sentence and the diff is
  bounded → `claude-gemini-cursor-protocol.md`.
- If you'd want to stop and look around after each piece lands →
  `claude-context.md`.
- If you're not sure → `claude-context.md`. The cost of pausing
  unnecessarily is lower than letting an under-specified plan run.

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

- Repo-specific conventions (branch naming, commit message scope
  tags, CI requirements)
- Stack-specific knowledge (TypeScript/Python conventions,
  typecheck/lint script names)
- Personal preferences (when to use the protocol vs `claude-context`,
  watchpoints that show up across most tasks)
- New protocol variants if you find a workflow shape neither file
  captures

Treat both files as living docs. Edit them after each session where
you notice "I wish Claude had defaulted to X."