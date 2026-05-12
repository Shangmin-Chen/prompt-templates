# prompt-templates

A collection of prompt templates by Simon.

## Files

### `claude-context.md`

A file used for chat interfaces.

### How to use it

Paste as message 1, then immediately follow with your actual task in message 2:

```text
[paste of claude-context.md]

I have a migration that needs to roll out to 3 environments and I'm worried about ordering. Here's the situation: ...
```

The context prompt establishes the working style. Your task description supplies the situation. From there, the new chat should naturally drop into diagnostic-first mode.

### What I tried to capture

- The two-phase shape (Phase 1 with you, Phase 2 with Cursor)
- Diagnostic-first instinct — ask for command output before proposing fixes
- Specific patterns — “run these, paste output, then we’ll know X”
- Agent prompt template with all the structural pieces (pre-flight, hard rules, recovery, pause points)
- Multi-agent specifics — main thread mutates, subagents verify in parallel
- How to handle agent reports — read carefully, distinguish real failures from harmless ones
- Tone matching, no sycophancy, willingness to retreat

### Things you might want to add over time

As you do more sessions, you'll probably notice patterns I missed. Consider adding:

- Repo-specific conventions (branch naming, commit message scope tags your team uses, CI requirements)
- Stack-specific knowledge (your TypeScript/Python conventions, what your typecheck/lint scripts are called)
- Personal preferences that evolved (preferred chunk size, how aggressive about subagent parallelism, etc.)

Treat the file as a living doc. Edit it after each session where you notice “I wish Claude had defaulted to X.”

## Other templates

Other files in this repo are agent prompts.
