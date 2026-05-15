# generator — meta-prompt for building agent prompts

Agent whose job is to **generate other agent prompts** from a user
description: purpose, runtime/tools, inputs, outputs, and constraints.
It emits design reasoning (or uses native thinking) then a finished
prompt inside `<prompt>` tags.

The full meta-prompt is in [`generator.md`](./generator.md).
