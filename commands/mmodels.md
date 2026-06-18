---
description: Classify a task by difficulty and run it on the cheapest capable model tier (Haiku/Sonnet/Opus).
argument-hint: "[task description]"
---

Apply the `mmodels` routing policy to the following task and execute it on the cheapest
capable tier.

Task: $ARGUMENTS

Steps:
1. Classify the task as **Light** (Haiku), **Medium** (Sonnet), or **Heavy** (Opus) using the
   routing rubric. When in doubt, route one tier up.
2. State the chosen tier and a one-line reason.
3. Delegate execution to the matching tier agent (`light`, `medium`, or `heavy`) via the Task tool.
4. If the agent returns `ESCALATE: <reason>`, re-dispatch to the next tier up.
5. Report the result and which tier handled it.
