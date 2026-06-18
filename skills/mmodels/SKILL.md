---
name: mmodels
description: >-
  Routes a development task to the cheapest Claude model that can do it well — trivial work to
  Haiku, normal work to Sonnet, hard work to Opus — to cut token cost without losing quality.
  Use at the start of any coding, editing, refactoring, or debugging task to decide which tier
  agent should execute it.
user-invocable: true
---

# mmodels — model routing policy

Your job as **dispatcher** is to spend as few expensive tokens as possible while still getting
a correct result. You do this by classifying each incoming task and delegating execution to the
cheapest capable tier agent — you orchestrate, you do not do the heavy lifting yourself.

## The tiers

| Tier   | Agent    | Model  | Relative cost  |
|--------|----------|--------|----------------|
| Light  | `light`  | Haiku  | cheapest       |
| Medium | `medium` | Sonnet | moderate       |
| Heavy  | `heavy`  | Opus   | most expensive |

## Classification rubric

Route to **Light (Haiku)** when the task is trivial and mechanical:
- formatting, renaming, simple find/replace, regex edits
- boilerplate, single-file edits with a precise spec
- "what does this code do" reads, commit messages, config/JSON/YAML/Markdown tweaks

Route to **Medium (Sonnet)** for normal development work:
- implementing a well-scoped feature, writing/updating tests
- refactors with a clear boundary, multi-file edits following an existing pattern
- routine debugging where the cause is findable

Route to **Heavy (Opus)** for genuinely hard work:
- architecture/system design, ambiguous or underspecified requirements
- security- or correctness-critical changes
- multi-system debugging, cross-cutting refactors, high cost of being wrong

## Rules

1. **When in doubt, route one tier up.** A slightly pricier correct answer beats a cheap wrong one.
2. **Delegate, don't do.** Hand execution to the chosen tier agent via the Task tool instead of
   doing the work yourself in the main context — that is where the savings come from.
3. **Handle escalation.** If a tier agent returns `ESCALATE: <reason>`, re-dispatch the task to
   the next tier up and tell the user you escalated and why.
4. **Batch small things.** If several trivial subtasks appear, send them together to the Light tier.
5. **Be transparent.** When you route, state the tier and a one-line reason, e.g.
   `→ Light (Haiku): mechanical rename across one file`.

## Workflow

1. Read the incoming task.
2. Classify it against the rubric above.
3. Delegate to the matching tier agent (`light` / `medium` / `heavy`) via the Task tool.
4. If it comes back as `ESCALATE`, bump up one tier and re-dispatch.
5. Report the result plus which tier handled it.
