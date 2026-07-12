# opencode-fable-pipeline

A cost-optimized, multi-model engineering pipeline for [OpenCode](https://opencode.ai), built around a single premium planning call to **Claude Fable 5**.

The idea is simple: Fable is expensive ($10/$50 per 1M in/out) but its **judgment** — the plan, the review strategy, the deadlock call — is what you're actually paying for. So the pipeline uses Fable *only* for judgment, exactly once per task, in a single-shot call with all context pre-assembled on a silver platter. Cheap open-weight models do the reading, building, and checking around it.

**Result: ~$1–2 of Fable spend on a typical task**, versus ~$5.50/task observed when letting Fable drive an interactive agent loop.

> **Why now:** with Fable being removed from Anthropic subscriptions, per-token cost suddenly matters. This is one way to keep using it where it's strongest without the bill that comes from a 10–15-turn exploratory session.

---

## The core ideas

1. **Fable is used only for the plan.** Its strongest capability — judgment — is captured once, up front. It writes the approach, the per-file intent, the edge cases, and the binding acceptance tests. It never touches implementation loops.
2. **Everything Fable needs arrives in one file.** A single `context.md` package containing the task, the relevant file contents, conventions, and prior lessons. This eliminates the tool-use loop and the repeated conversation re-sends that make interactive Fable sessions so costly.
3. **That context is built by a cheap, large-context model.** DeepSeek V4 Flash assembles the package for roughly $0.10–0.30 per task.
4. **The review swarm runs on OpenAI's GPT 5.6 Sol via subscription** — multiple review angles at zero marginal token cost. (There are OpenRouter fallback reviewers if you don't have an OpenAI subscription; results are better with Sol.)
5. **Implementation runs on GLM 5.2** — a strong implementor, and deliberately a *different* model family from both the planner and the reviewers. It's mostly output tokens, averaging ~$1–2 per task. Swap it for any model you prefer.

Fable's plan is never edited or overruled by a cheaper model. Only you — at the checkpoint — can send it back for revision.

---

## Requirements

| | Required? | Why |
|---|---|---|
| **[OpenCode](https://opencode.ai)** | Yes | The runtime this pipeline is written for |
| **[OpenRouter](https://openrouter.ai) account + API key** | Yes | Routes Fable, DeepSeek, and GLM |
| **OpenAI account (Plus/Pro subscription)** | Optional but recommended | Runs the GPT 5.6 Sol review swarm at zero marginal cost; falls back to OpenRouter reviewers without it |

---

## Setup

### 1. Install OpenCode

Follow the [OpenCode install guide](https://opencode.ai/docs). Verify with:

```bash
opencode --version
```

### 2. Add the pipeline to your repo

Clone this repo and copy the pipeline files into the project you want to work on:

```bash
git clone https://github.com/ixetanet/opencode-fable-pipeline.git

# from inside your target repo:
cp -r /path/to/opencode-fable-pipeline/.opencode   .
cp    /path/to/opencode-fable-pipeline/opencode.jsonc .
cp    /path/to/opencode-fable-pipeline/PIPELINE.md   .
```

Then add the pipeline's per-repo state directory to your `.gitignore`:

```bash
echo ".pipeline/" >> .gitignore
```

`.pipeline/` holds runtime state (the context package, plan, review findings, spend log, and post-mortem "lessons"). It's per-repo and regenerates itself — **never commit it, and never copy it between repos.** See [Porting](#porting-to-another-repo) below.

> The pipeline is repo-agnostic: everything project-specific (build/test/lint commands, conventions, file layout) is discovered at runtime on the first run, not hard-coded in config.

### 3. Connect OpenRouter

In OpenCode, run:

```
/connect
```

and add your OpenRouter API key. Make sure your OpenRouter account has credit, and that the model slugs used here are available to it (see the table below).

### 4. (Optional but recommended) Log in to OpenAI for the review swarm

The primary reviewer, `reviewer-gpt-5.6-sol`, runs on your **OpenAI subscription login — not OpenRouter** — so every review angle costs $0 in tokens (you're spending subscription rate-limit headroom, not money). Authenticate OpenAI through OpenCode's auth (`opencode auth login` → OpenAI).

Without an OpenAI subscription the pipeline automatically falls back to a fleet of OpenRouter reviewers (DeepSeek V4 Pro, Kimi K2.7-Code, MiniMax M3). Those bill per token and are slightly less effective, but the pipeline runs fine.

### 5. Restart OpenCode

**This matters:** OpenCode loads agent definitions once per process and does *not* hot-reload them. After adding these files (or editing any `.opencode/agents/*.md` or `opencode.jsonc` later), fully restart OpenCode — otherwise a running window silently uses the old definitions.

---

## Usage

From an OpenCode session in your repo:

```
/pipeline <feature description or bug report>
```

The orchestrator classifies the task's complexity, has DeepSeek assemble the context package, sends Fable a single-shot planning call, pauses at a **human checkpoint** for you to approve or revise the plan, then has GLM implement it and the review swarm check it — looping through verification until tests, lint, and type-check pass.

### Flags

Embed anywhere in the request:

| Flag | Values | Default | Effect |
|---|---|---|---|
| `--tier=` | `trivial` \| `standard` \| `complex` \| `architectural` | auto | Force the complexity tier (`trivial` skips Fable entirely; `architectural` runs a multi-step migration path) |
| `--review=` | `standard` \| `deep` \| `critical` \| `exhaustive` | `deep` | How hard to review. `deep` selects 3–8 angles + a falsification pass per finding; `exhaustive` forces all 8 angles; `critical` adds a Fable review pass (extra cost) |
| `--no-checkpoint` | — | off | Skip the human plan-approval pause (fully autonomous) |
| `--mode=` | `investigate` \| `review` | auto | `investigate` = diagnosis only, no diff; `review` = audit existing code with the review swarm |
| `--base=` | git ref | working tree | Review mode: review `git diff <ref>...HEAD` instead of the working tree |
| `--fix` | — | off | Review mode: run the fix loop on blocking findings |

Examples:

```
/pipeline add a --dry-run flag to the export command
/pipeline fix the race where two workers grab the same job --review=critical
/pipeline why does the cache miss rate spike after a deploy? --mode=investigate
/pipeline --mode=review --base=main
```

---

## How it works

A premium model supplies judgment; cheap models do the construction and checking.

| Stage | Agent | Model | Role |
|---|---|---|---|
| **Orchestration** | `pipeline-orchestrator` | DeepSeek V4 Pro | Drives the whole run, classifies complexity, enforces the rules |
| **0 · Context prep** | `context-prep` | DeepSeek V4 Flash | Assembles the one-file context package (~$0.10–0.30) |
| **1 · Plan** | `planner` | **Claude Fable 5** | Single-shot: approach, per-file intent, acceptance tests, review strategy |
| **2 · Plan annotation** | `plan-annotator` | DeepSeek V4 Flash | Advisory checks delivered to you at the checkpoint |
| **2.5 · Checkpoint** | *you* | — | Approve the plan, or spend the one revision slot |
| **3 · Implement** | `implementer` | GLM 5.2 (Exacto routing) | Writes the code within plan scope (~$1–2, mostly output) |
| **3.5 · Gate** | `verifier` | DeepSeek V4 Flash | Tests / lint / type-check must pass before review |
| **4 · Review swarm** | `reviewer-gpt-5.6-sol` | **GPT 5.6 Sol** (subscription) | 3–8 review angles at $0 marginal cost |
| **4 · Fallback fleet** | DeepSeek / Kimi / MiniMax | OpenRouter | Per-angle fallback when Sol is unavailable |
| **4 · Falsification** | `reviewer-deepseek-v4-pro` | DeepSeek V4 Pro | Cross-family refutation pass — one refuter per finding |
| **6 · Verify** | `verifier` | DeepSeek V4 Flash | Confirms the fix; loops on failure |
| **Escalation** | `escalation-debugger` | **Claude Fable 5** | Only if the same failure survives 2 fix loops |

**Fable is called at most:** 1× plan + 1× optional revision (you trigger it) + 1× escalation (only after 2 failed fix loops) + optionally 1× critical review pass. Effort is pinned to `high` everywhere; `xhigh` is banned pipeline-wide because it multiplies thinking-token spend for marginal gains.

Full design rationale — the judgment-vs-construction split, the review catalog, the architectural migration path, and every guarantee — is in **[PIPELINE.md](PIPELINE.md)**.

---

## Cost per run

Model prices (per 1M in/out): Flash `$0.077/$0.154` · V4-Pro `$0.435/$0.87` · GLM 5.2 `$0.392/$1.232` · **Fable 5 `$10/$50`**.

The dominant cost is Fable *session shape*, not plan length — which is why every Fable call is single-shot.

| Tier | Typical total | Notes |
|---|---|---|
| **Trivial** | ~$0.05–0.20 | No Fable at all |
| **Standard** | **~$1–2** | The single-shot Fable plan dominates; review swarm is $0 on the OpenAI subscription |
| **Complex / `--review=critical`** | +~$0.5–2 | Per extra Fable call (critical review, escalation) |
| **Architectural** | ~$8–14 | A 10-step migration with 2 risky steps, spread across sessions |
| **Investigate** | ~$0.10–0.50 | +$0.50–1.50 for optional Fable synthesis |
| **Review-only** | ~$0.05–0.30 | Depends on `--review` envelope and diff size |

Every run logs an estimated per-stage spend table to `.pipeline/status.md`. Estimates are rough; exact usage is in your OpenRouter activity dashboard, filterable by model.

---

## Swapping models

GLM 5.2 is the default implementor purely because it's a strong builder in a different model family from the planner and reviewers — swap it for anything you like:

1. Change the `model:` line in the agent's file under `.opencode/agents/`.
2. Add/adjust the matching entry under `provider.openrouter.models` in `opencode.jsonc` (effort + output cap). Verify the slug exists on openrouter.ai and check its `supported_parameters` before sending a `reasoning` effort — some models (MiniMax M3, Kimi K2.7-Code) don't accept it.
3. **Restart OpenCode.**

Two model-diversity rules to preserve when swapping: no diff reviewer may share a model family with the implementer, and every diff must be reviewed by at least two distinct model families. Details and the full swap procedure (including the planner fallback chain and reviewer fleet) are in [PIPELINE.md → Adjusting models later](PIPELINE.md).

---

## Porting to another repo

Copy into the target repo:

- `.opencode/` (all agents + the `/pipeline` command)
- `opencode.jsonc` (connect your OpenRouter key once via `/connect` on that machine)
- the `.pipeline/` line in `.gitignore`
- `PIPELINE.md`

**Never copy `.pipeline/`** — it's per-repo state. On the first run in a new repo, the repo brief (build/test/lint commands, conventions) regenerates and the lessons log starts empty; the new repo accretes its own institutional memory over time.

---

## Guarantees

- The pipeline **never runs `git commit` or `git push`** — all output stays in the working tree for your review.
- **Only the implementer modifies code**, within plan scope. The orchestrator and verifier never mutate the tree; a mechanical gate fails loudly on any out-of-scope change instead of cleaning it up.
- **Fable's plan is never edited by a cheaper model.** Cheaper models' critiques are advisory; only you can trigger a revision.
- Reviewers never edit code, never review their own diff, and work blind to each other — only the aggregator sees everything.
- Reviewed code always passed tests/lint/type-check first; blocking findings are confirmed with a failing repro test before anything is "fixed".

---

## Repo layout

```
.opencode/
  agents/          # 16 agent definitions (planner, implementer, reviewers, verifier, …)
  commands/
    pipeline.md    # the /pipeline slash command
opencode.jsonc     # provider + per-model config (effort, output caps)
PIPELINE.md        # full design doc and rationale
README.md          # this file
```

---

## License

[MIT](LICENSE)
