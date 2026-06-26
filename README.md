# Model Router

**Route every task to the cheapest Claude model that can actually do it — and stop paying Opus prices for Haiku work.**

Model Router is a [Claude Code](https://code.claude.com) plugin. It turns your main session into a **dispatcher** that classifies each task by difficulty and hands execution to the cheapest capable model:

- 🟢 **Trivial / mechanical** → **Haiku** (cheapest)
- 🟡 **Normal development** → **Sonnet**
- 🔴 **Hard / risky** → **Opus** (most capable)

Instead of running every file read, rename, and boilerplate edit on an expensive model, the dispatcher delegates them to a Haiku sub-agent and reserves Opus for the work that truly needs it.

> **This is not a "council".** Multi-model consensus tools (e.g. [karpathy/llm-council](https://github.com/karpathy/llm-council)) ask *all* models the *same* question and cost **more** for higher quality. Model Router uses *one* model per task, chosen by difficulty, to cost **less**.

## How it works

```
   Your task
       │
       ▼
  ┌────────────┐   classifies difficulty
  │ Dispatcher │   (your main Opus session)
  └─────┬──────┘
        ├── trivial ──▶ light  (Haiku)   format · rename · regex · boilerplate · config
        ├── normal ───▶ medium (Sonnet)  features · tests · scoped refactors · debugging
        └── hard ─────▶ heavy  (Opus)    architecture · ambiguous specs · security · hard bugs
```

- The plugin ships three **tier sub-agents** (`light`, `medium`, `heavy`), each pinned to a model via the agent's `model:` field.
- The **`mmodels` skill** gives the dispatcher the rubric and tells it to delegate rather than do the work itself — that delegation is where the savings come from.
- **Escalation:** a lower tier that finds a task harder than expected returns `ESCALATE: <reason>`, and the dispatcher re-dispatches one tier up. The guiding rule is *"when in doubt, route up"* — a slightly pricier correct answer beats a cheap wrong one.

## Routing rubric

| Tier | Model | Use for |
|------|-------|---------|
| **Light** | Haiku | formatting, renames, find/replace, regex, boilerplate, single-file edits with a precise spec, "what does this do" reads, commit messages, config/JSON/YAML/Markdown |
| **Medium** | Sonnet | well-scoped features, writing/updating tests, refactors with a clear boundary, pattern-following multi-file edits, routine debugging |
| **Heavy** | Opus | architecture/design, ambiguous requirements, security- or correctness-critical changes, multi-system debugging, cross-cutting refactors |

## Requirements

- [Claude Code](https://code.claude.com) (the CLI / IDE agent).
- Access to Haiku, Sonnet, and Opus on your plan (Pro/Max subscription **or** API billing). The routing logic is identical either way — on a subscription you save rate-limit usage; on API you save real money.

## Install

In Claude Code, add this repo as a plugin marketplace and install the plugin:

```
/plugin marketplace add mkrupkin/plng-mmodels
/plugin install model-router@plng-mmodels
```

The tier agents, the `mmodels` skill, and the `/mmodels` command become available immediately.

## Usage

**Explicit** — route a single task with `/mmodels`:

```
/mmodels rename every getUser() call to fetchUser() across the auth module
→ Light (Haiku): mechanical rename, single module
```

```
/mmodels design a plugin system with sandboxed third-party extensions
→ Heavy (Opus): architecture decision, high cost of being wrong
```

**Automatic** — the `mmodels` skill activates on coding tasks and routes them for you. You'll see a one-line note such as `→ Medium (Sonnet): add tests for the parser` before each delegation.

### Force routing on every prompt (`UserPromptSubmit` hook)

Skill auto-activation is best-effort — on a busy turn the dispatcher can forget to classify before diving in. To make routing fire **on every single prompt**, add a `UserPromptSubmit` hook. It runs before Claude responds and injects a one-line reminder into the context, so the dispatcher always classifies first.

Add to your `~/.claude/settings.json` (or a project-level `.claude/settings.json`):

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Before responding, invoke the mmodels skill to classify this task and route it to the cheapest capable model tier - Haiku for trivial, Sonnet for normal, Opus for hard.\""
          }
        ]
      }
    ]
  }
}
```

How it works: a `UserPromptSubmit` hook's stdout is added to the model's context for that turn. The `echo` above simply restates the routing instruction, which nudges the dispatcher to run the `mmodels` skill before doing anything else. It costs nothing per turn beyond a few tokens and makes routing deterministic instead of best-effort.

> The reminder is just a string — tweak the wording to taste, or scope the hook to a single project by putting it in that project's `.claude/settings.json` instead of the global one.

## Recommended setup

Run your **main session on Opus** so the dispatcher makes the best classification calls, then let it push execution down to Sonnet/Haiku:

```
claude --model opus
```

Most of your token spend then shifts to cheaper models, while Opus is spent only on routing decisions and genuinely hard sub-tasks.

> Prefer maximum savings over routing accuracy? Run the main session on Sonnet instead. Classification gets slightly less reliable, but the dispatcher itself costs less. The rubric is deliberately rule-based so a cheaper dispatcher can still follow it.

## Token-saving compaction (proactive `/compact`)

Long sessions pile up context — old tool output, file reads, exploration — and every new turn pays for carrying it. Model Router ships two small helpers so compaction happens **sooner and smarter** instead of right before the limit.

### What the plugin provides

1. **PreCompact hook** — fires automatically on every compaction (auto or manual) and injects a focus instruction so Claude preserves what `mmodels` needs: your active task, routing decisions, ESCALATE events, recently touched files, and tier-agent outcomes. The conversation summary stays useful instead of generic.
2. **`/mmodels-compact`** — a one-step slash command that prints a ready-to-paste `/compact …` line with the same focus instruction baked in. Use it whenever the session feels heavy.

### Make auto-compact fire earlier (the real "auto" lever)

Claude Code already runs auto-compact when context **approaches the window limit**. Setting **`autoCompactWindow`** lower makes it fire **sooner**, so you spend fewer tokens dragging stale context around. The plugin cannot set this for you (it lives in your global settings), but it's one line:

Add to `~/.claude/settings.json`:

```json
{
  "autoCompactEnabled": true,
  "autoCompactWindow": 400000
}
```

- `400000` (~400k chars) is a balanced default — compaction kicks in well before the ceiling, the PreCompact hook makes the summary mmodels-aware, and you carry roughly half the context you would otherwise.
- Drop to `200000`–`300000` for aggressive savings (more frequent compaction, slightly more risk of dropping useful detail).
- Range allowed by the schema: `100000`–`1000000`.

### Honest limits

- **No plugin can call `/compact` on its own.** Only the user or the built-in auto-compact (under the limit) can trigger it. The hook only runs *during* a compaction; it cannot start one.
- The PreCompact hook ships **only via the plugin install** (`/plugin install model-router@plng-mmodels`). A manual install of just the skill/agents to `~/.claude/` does not register the hook.

## Configuration

Want different models per tier? Edit the `model:` field in the agent files:

| File | Field |
|------|-------|
| `agents/light.md`  | `model: haiku`  |
| `agents/medium.md` | `model: sonnet` |
| `agents/heavy.md`  | `model: opus`   |

Accepted values for a plugin agent's `model`: `haiku`, `sonnet`, `opus`. Adjust the rubric in `skills/mmodels/SKILL.md` to change what counts as trivial vs. hard.

## Why it saves tokens (and the honest caveats)

**Savings:** the bulk of agentic work — reading files, search, boilerplate, mechanical edits — is offloaded to Haiku/Sonnet sub-agents instead of running on Opus.

**Caveats — read these:**

1. **The dispatcher itself costs tokens.** Classifying tasks and reading each sub-agent's result runs on your main model. The rubric is kept rule-based so this stays cheap and predictable.
2. **A cheaper model can misjudge difficulty.** Mitigated by *"route up when in doubt"* plus the `ESCALATE` protocol.
3. **No built-in token accounting yet.** Savings aren't measured for you — that's on the roadmap.

## Roadmap

- [x] **v0.2:** PreCompact hook with mmodels-aware focus + `/mmodels-compact` + `autoCompactWindow` recipe
- [ ] Token/cost accounting + a per-session savings report
- [ ] A cheap Haiku pre-classifier (deterministic first hop) instead of dispatcher-judged routing
- [ ] Optional cheapest tier via local models (Ollama) or OpenRouter
- [ ] Auto-tuning of the rubric from observed escalations
- [ ] PostCompact hook with a compaction log

## Repository layout

```
plng-mmodels/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # makes this repo installable as a marketplace
├── agents/
│   ├── light.md            # Haiku tier
│   ├── medium.md           # Sonnet tier
│   └── heavy.md            # Opus tier
├── skills/
│   └── mmodels/
│       └── SKILL.md        # classification rubric + dispatch workflow
├── commands/
│   ├── mmodels.md          # /mmodels slash command
│   └── mmodels-compact.md  # /mmodels-compact (smart-focus /compact)
├── hooks/
│   └── precompact-focus.json  # PreCompact hook payload (mmodels-aware focus)
├── README.md
└── LICENSE
```

## License

[MIT](LICENSE).
