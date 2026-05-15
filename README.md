# prompt-templates

A collection of prompt templates by Simon.

This repo holds two distinct working protocols. They are not
interchangeable — pick based on what kind of work you're doing.

| Directory | Style | When to use |
|---|---|---|
| [`claude-context/`](./claude-context/) | **Multi-phase, iterative** | Risky, exploratory, or staged work where each step's output feeds the next decision. Stacked PRs, migrations across environments, refactors where you want a human in the loop after every chunk. |
| [`claude-gemini-cursor-protocol/`](./claude-gemini-cursor-protocol/) | **One-shot implementation** | Tasks that should land as a single, complete diff. Features, bug fixes, mechanical changes where the planning loop produces a fully specified plan and the executor follows it without judgment. |

Start with each folder’s `AGENTS.md` for how to use it, then open the sibling `.md` file for the text to paste into chat.

Both protocols are pasted into a fresh chat as the first message, before
you describe your actual task.

---

## Protocol guides

- **[`claude-context/AGENTS.md`](./claude-context/AGENTS.md)** — multi-phase pair programming with Cursor (shape, how to use, when to choose it). Full paste: [`claude-context/claude-context.md`](./claude-context/claude-context.md).
- **[`claude-gemini-cursor-protocol/AGENTS.md`](./claude-gemini-cursor-protocol/AGENTS.md)** — one-shot four-role handoff with Gemini CLI + Cursor (roles, loop, conformance, triage). Full paste: [`claude-gemini-cursor-protocol/claude-gemini-cursor-protocol.md`](./claude-gemini-cursor-protocol/claude-gemini-cursor-protocol.md).

---

## Other templates

Reusable agent prompts live one directory each, with a short
`AGENTS.md` at the folder root and the pasteable template alongside.

| Directory | Purpose |
|---|---|
| [`summary/`](./summary/) | Read-only crawl → structured `summary.md` for codebase comprehension |
| [`generator/`](./generator/) | Meta-prompt that emits new agent prompts from a spec |
| [`refresh/`](./refresh/) | Legacy audit → prioritized refactoring roadmap |
| [`planner/`](./planner/) | High-level goal → self-sufficient executable plan |
| [`protocol-v2/`](./protocol-v2/) | Placeholder for a future protocol revision |

---

## Things to add over time

As you do more sessions, you'll probably notice patterns. Consider
adding:

- Repo-specific conventions (branch naming, commit message scope
  tags, CI requirements)
- Stack-specific knowledge (TypeScript/Python conventions,
  typecheck/lint script names)
- Personal preferences (when to use each protocol,
  watchpoints that show up across most tasks)
- New protocol variants if you find a workflow shape neither file
  captures

Treat both protocols as living docs. Edit them after each session where
you notice "I wish Claude had defaulted to X."
