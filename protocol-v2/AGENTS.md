# Protocol v2 — How to run it

Three tools, three roles, JSON handoffs.

- **Chat Claude** thinks and writes prompts. Can't see your code.
- **Gemini CLI** audits before, reviews after. Filesystem access, fresh context each call.
- **Claude Code** plans and implements in one session.

## Setup

1. Open a new chat Claude conversation. Paste `system-prompt.md` as the first message.
2. Have Gemini CLI installed and runnable from your repo root.
3. Have Claude Code ready in the same repo.

## The loop

````
You → chat Claude:   describe the task
chat Claude → you:   audit prompt
you → Gemini CLI:    paste audit prompt, run it
Gemini → you:        audit JSON
you → chat Claude:   paste audit JSON
chat Claude → you:   Claude Code prompt + review prompt (both in one message)
you → Claude Code:   paste Claude Code prompt, let it run
Claude Code → you:   summary of what changed
you → Gemini CLI:    paste review prompt, run it
Gemini → you:        verdict JSON
  ├─ APPROVE → done
  └─ REJECT → paste verdict back to chat Claude, follow its triage
````

## What each step looks like

**1. Describe the task.** One paragraph in chat Claude. Mention "high-stakes" if rolling back would be expensive (migrations, auth, deployed state).

**2. Run the audit.** Chat Claude hands you a prompt. Paste into Gemini CLI from the repo root. Gemini returns a JSON block.

**3. Paste audit back.** Chat Claude hands you two prompts in one response: one for Claude Code (with audit baked in), one for Gemini's later review.

**4. Run Claude Code.** Paste the implementation prompt. Claude Code reads the files, plans internally, implements, runs tests, stages the diff. It does not commit.

**5. Run the review.** Paste the review prompt into Gemini CLI. Gemini diffs against base, runs tests, returns verdict JSON.

**6a. APPROVE.** Done. Commit, push, whatever you normally do.

**6b. REJECT.** Paste the verdict back to chat Claude. It triages:
- *Small fix*: chat Claude writes a short fix prompt. Paste into Claude Code (same session). Re-run the review.
- *Rollback*: chat Claude tells you to `git reset --hard <ref>` and opens a fresh approach.

## Interjecting

If something looks wrong mid-stream, jump into chat Claude with the concern. Don't restart. It folds your note into the current prompt.

## When to skip the audit

Chat Claude will skip it on trivially small tasks (one-file, no ambiguity). For anything touching multiple files or unfamiliar code, audit always runs.

## When to add a pre-execution gate

Say "high-stakes" in your task description. Chat Claude will tell Claude Code to output its plan as JSON first and stop. You paste the plan into Gemini for a pre-execution APPROVE / REJECT. Use this for migrations, auth, schema changes, anything where rolling back hurts.

## Files

- `system-prompt.md` — paste into a new chat Claude conversation
- `README.md` — this file