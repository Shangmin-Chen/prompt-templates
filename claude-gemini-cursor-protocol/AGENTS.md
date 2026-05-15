# claude-gemini-cursor-protocol — one-shot implementation

A four-role handoff protocol for tasks that should land as a single
diff. Trades per-chunk human pause points for an **independent
reviewer (Gemini CLI)** that gates execution and reviews the diff
afterward.

## Roles

| Role | Tool | Job |
|---|---|---|
| Thinker | Claude (chat) | Writes prompts for the other three, interprets every artifact, triages reviewer rejections |
| Auditor / Reviewer | Gemini CLI | Grounds Claude with audit, reviews plan, reviews diff. Every verdict binary. |
| Planner | Cursor (single comprehensive plan) | Produces one complete `planner.md` |
| Writer | Cursor multi-agent | Executes `planner.md` concurrently, no pauses |

Every prompt Claude writes for an agent opens with a `## Your role`
block. Each agent only sees the one prompt — so the role is declared
every time.

## Shape

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

## Five load-bearing principles

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

## How to use it

Same pattern as multi-phase context protocol:

```text
[paste of claude-gemini-cursor-protocol.md]

I need to add OAuth login (Google + GitHub) to my Next.js app.
Currently using email+password via Auth.js.
```

The full protocol text to paste is in [`claude-gemini-cursor-protocol.md`](./claude-gemini-cursor-protocol.md) in this directory.

Claude responds with the Gemini audit prompt. You run it in Gemini
CLI, paste the report back, and the loop unfolds.

## Human role during the loop

You're watching the protocol stream by. There is no dedicated "human
checkpoint" step — you interject when something looks off, and
Claude folds your note into the artifact currently in flight without
restarting the loop. The protocol assumes you're ambient.

## What the conformance review covers

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

## Triage on conformance REJECT

Claude reads the findings and decides between:

- **Path A — Small fix.** Bounded findings, same executor context can
  apply the fix. Claude writes two prompts in one response: the fix
  for Cursor and the re-review for Gemini. You run them in order.
- **Path B — Rollback.** Structural findings, fix would meaningfully
  exceed a small prompt, or this is the third REJECT round.
  `git` rollback, fresh Cursor context, new planner prompt.

Repeated small fixes = the protocol is leaking. Claude should be
recommending Path B by the third REJECT round.

## When this is the right tool

- A single feature that lands as one diff
- A bug fix where the root cause is understood
- Mechanical refactors with a clear endpoint
- Anything where the plan can be fully specified up front

## When it's the *wrong* tool

- Exploratory work where the right shape isn't clear yet
- Multi-PR stacks
- Work staged across environments with verification between stages

For those, use [`../claude-context/AGENTS.md`](../claude-context/AGENTS.md).

## Choosing between protocols

- If you can describe the end state in one sentence and the diff is
  bounded → this directory.
- If you'd want to stop and look around after each piece lands →
  [`../claude-context/AGENTS.md`](../claude-context/AGENTS.md).
- If you're not sure → [`../claude-context/AGENTS.md`](../claude-context/AGENTS.md).
