# /pipeline — Multi-Model Feature/Bugfix Pipeline (OpenCode)

A cost-optimized engineering pipeline for OpenCode, routed through OpenRouter. A premium model (Claude Fable 5) supplies the judgment — the plan, the per-task review strategy, and deadlock-breaking — while cheap open-weight models do the construction and checking. Fable's plan is never edited or overruled by a cheaper model; only the human (at the checkpoint) can send it back for revision.

## Invocation

```
/pipeline <feature description or bug report>
```

Flags (embed anywhere in the request):

| Flag | Values | Default | Effect |
|---|---|---|---|
| `--tier=` | `trivial` \| `standard` \| `complex` \| `architectural` | auto-classified | Force the complexity tier (trivial skips Fable entirely; architectural runs the multi-step migration path) |
| `--review=` | `standard` \| `deep` \| `critical` \| `exhaustive` | `deep` | Review envelope. Default (`deep`): the planner's 3–8 angle selection from the fixed catalog + a falsification pass per finding. `standard` opts down to angles 1–2 (no falsification); `exhaustive` forces all 8 angles; `critical` runs angle 1 on Fable (extra cost, warned first) |
| `--no-checkpoint` | — | off | Skip the Stage 2.5 human plan-approval pause (fully autonomous run; ignored on architectural runs — their checkpoint is mandatory) |
| `--step-checkpoint` | — | off | Architectural runs only: pause after every completed migration step instead of continuing automatically |
| `--mode=` | `investigate` \| `review` | auto-classified | Force the run mode: diagnosis-only (answer + evidence, no diff) / audit existing work with the review swarm |
| `--base=` | git ref | working tree | Review mode: review `git diff <ref>...HEAD` instead of the uncommitted working tree |
| `--fix` | — | off | Review mode: after confirmation, run the fix loop on BLOCKING/SHOULD-FIX findings |

**Run modes.** Beyond the default change-delivering run: **investigate** — the deliverable is an evidence-backed answer in `.pipeline/investigation.md` (reproduce, instrument, bisect; tree left clean; an INCONCLUSIVE verdict offers one opt-in Fable synthesis call); **review** — the review swarm + empirical confirmation applied to code the pipeline didn't write (no plan; the orchestrator selects angles under the same fleet rules). Performance-shaped *change* runs additionally require a numeric contract: measurement command + baseline captured on the unchanged tree + target, verified at Stage 6.

Setup: run `/connect` in OpenCode once and add your OpenRouter API key. Nothing else — no environment flags or system changes.

While a run is active, the orchestrator maintains a live stage checklist via OpenCode's todo tool — the TUI shows every stage of the resolved path (plus migration steps on architectural runs) ticking from pending → in-progress → done, with mid-run additions (fix loops, escalation) appended as they arise. The checklist is session-scoped UI; durable state lives in `.pipeline/status.md` and `migration-state.md`.

## Stages and models

| Stage | Agent | Model (OpenRouter slug) | Effort |
|---|---|---|---|
| Orchestration | `pipeline-orchestrator` | `deepseek/deepseek-v4-pro` | high |
| 0 Context prep | `context-prep` | `deepseek/deepseek-v4-flash` | high (base; cost controlled by output cap) |
| 1 Plan + review strategy | `planner` | `anthropic/claude-fable-5` | **high, fixed — never any other level** |
| 1 fallback (Fable guardrail refusal) | `planner-sol` → `planner-sol-openrouter` | `openai/gpt-5.6-sol` (subscription) → `openrouter/openai/gpt-5.6-sol` (billed) | high |
| 2 Plan annotation (advisory) | `plan-annotator` | `deepseek/deepseek-v4-flash` | high |
| 2.5 Checkpoint | human | — | — |
| 3 Implement | `implementer` | `z-ai/glm-5.2:exacto` (Exacto = tool-calling-accuracy routing) | high |
| 3.5 Gate / 6 Verify / 7 Post-mortem | `verifier` | `deepseek/deepseek-v4-flash` | high |
| 4 Code review (fixed 8-angle catalog; planner selects 3–8 + focus notes) | `reviewer-gpt-5.6-sol` (all angles; **OpenAI subscription, not OpenRouter**) | `openai/gpt-5.6-sol` | high |
| 4 fallback fleet (per-angle, on Sol rate-limit/error) | `reviewer-deepseek-v4-pro` / `reviewer-kimi-k2.7-code` / `reviewer-minimax-m3` | `deepseek/deepseek-v4-pro` / `moonshotai/kimi-k2.7-code` / `minimax/minimax-m3` | high / model default / none |
| 4 falsification fan-out (cross-family, deliberate) | `reviewer-deepseek-v4-pro` | `deepseek/deepseek-v4-pro` | high |
| 4 (critical only) | `reviewer-fable` | `anthropic/claude-fable-5` | high, fixed |
| 4 Aggregation | `review-aggregator` | `deepseek/deepseek-v4-flash` | high |
| Escalation / investigation synthesis | `escalation-debugger` | `anthropic/claude-fable-5` | high, fixed |
| Investigate mode | `investigator` | `deepseek/deepseek-v4-pro` | high |

Fable is called at most: 1x plan + 1x plan revision (human-requested at the checkpoint; on `--no-checkpoint` runs, auto-triggered only by an objective COVERAGE-GAP) + 1x escalation (only after the same failure survives 2 fix loops) + optionally 1x critical review pass. Effort ceiling everywhere is `high` — `xhigh` is banned pipeline-wide (multiplies thinking-token spend for marginal gains).

## How judgment vs. construction is split

- **The plan is decision-level.** Fable writes approach + rationale, per-file intent, public interfaces, edge cases as required behavior, and the **acceptance tests** (the binding contract). Test format is tier-conditional: COMPLEX runs get full test code; STANDARD runs get a test contract table (exact arrange values + exact expected assertions per test) plus one exemplar test to pin fixture idioms — full code is reserved for vacuous-pass-prone tests (asserting nothing changed / an exception thrown), which fail invisibly if scaffolded wrong. Implementation bodies belong to the implementer; the only other exception is a sparingly-used, explicitly flagged `JUDGMENT-CRITICAL` snippet where the algorithm itself is the judgment call.
- **The planner sizes the review; the catalog fixes its shape.** Plan section 7 selects 3–8 angles from a fixed 8-angle catalog (correctness, state-lifecycle/blast-radius, 2× adversarial, persistence/serialization, cross-file parity, test fidelity, simplification), scaled to the change's blast radius, and attaches task-specific focus notes that sharpen the selected angles' rubrics. The orchestrator enforces minimums it cannot waive: correctness always, state-lifecycle whenever the diff touches persisted state/events/reservations, never GLM, ceiling 8. All finder angles run on GPT 5.6 Sol via the OpenAI subscription (zero marginal cost; a third family, distinct from the GLM implementer and the Fable planner) with per-angle fallback to the OpenRouter fleet on rate limits; cross-family independence lives in the DeepSeek falsification pass — same-model multi-angle agreement is tagged MULTI-ANGLE and only becomes HIGH-CONFIDENCE by surviving that cross-family check. TRIVIAL keeps its single correctness pass; architectural NORMAL steps run a 3-pass core (correctness, state-lifecycle, adversarial) while RISKY steps use their step-plan's selection.
- **Reviewers have full repo access, not just the patch.** Every pass reads the complete current contents of the touched files and traces beyond the diff with read-only `grep` (does that published event have a subscriber? is that reservation released on death?) — a patch-only reviewer structurally cannot find defects in pre-existing code the diff makes load-bearing. The assigned rubric is a focus, not a blinder: off-rubric defects are reported tagged `off_angle` and deduped by the aggregator. At `deep` (the default) and above, review is followed by a falsification fan-out — one refuter call per finding, with refuted items recorded in a `REFUTED` section, never silently dropped.
- **Architectural changes run as a step sequence, not one giant run.** A repo-scale refactor/reorg doesn't fit one context package, plan, or reviewable diff, so the ARCHITECTURAL tier splits the shape: context-prep produces a **map** (module inventory, dependency edges, public seams with call-site counts — not file contents); Fable writes a **migration plan** (target architecture, cross-step invariants, and an ordered list of pipeline-sized steps, each with scope, a suite-green contract, a NORMAL/RISKY flag, and a review note); then each step runs through the normal machinery with a bounded context. NORMAL steps need no further Fable call — the step contract is the plan; RISKY steps get one single-shot Fable step-plan each (count disclosed at the mandatory checkpoint). `.pipeline/migration-state.md` tracks progress so the migration survives across sessions, and the full suite must be green after every step — the tree is always landable.
- **Stage 0 is engineered for wall-clock, not just cost.** An agent's real time cost is sequential tool turns (each re-sends the conversation and re-reasons), so context-prep works in as few turns as possible: discovery probes are batched several-per-call, all prose is one write, and every file embed happens in one shell command (echo headers/fences + cat contents) instead of a per-file ritual, with each file embedded exactly once. The single-command embed also makes the summarize-instead-of-embed failure mode structurally hard — it either embeds everything byte-exact or fails loudly. (A parallel `context-scout` fan-out was tried 2026-07 and removed: it needed an experimental OpenCode flag and produced bloated, duplicated context packages.)
- **Context captures project intent from the repo's own docs.** The repo brief records a per-repo **documentation map** (discovered at runtime — an index file is used if present, never required; "none beyond README" is a valid outcome), and each run's context embeds the pertinent docs whole (small) or by section with **every omitted section named** (large) — silent partial capture of intent bounces at the Stage 0 check the same way a summarized code file does. Doc-vs-code disagreements are recorded as paired facts for the planner to judge, and the annotator flags mapped docs the context neither captured nor ruled out.
- **Plan review is advisory, not authoritative.** A cheap annotator runs mechanical checks (acceptance-criteria coverage, referenced files exist, tests in the right framework, strategy well-formed) plus evidence-cited observations — delivered to *you* at the checkpoint. It cannot edit the plan and cannot trigger a Fable revision; only you can (that consumes the one revision slot). On `--no-checkpoint` runs, only an objective COVERAGE-GAP auto-triggers the revision.

## Rough cost per run

Prices (per 1M in/out): flash $0.077/$0.154 - v4-pro $0.435/$0.87 - glm-5.2 $0.392/$1.232 - kimi-k2.7-code $0.72/$3.50 - minimax-m3 $0.30/$1.20 - **fable-5 $10/$50**.

**The dominant cost is Fable session shape, not plan length.** An agent session re-sends its entire conversation on every tool round-trip, and each turn's thinking bills as output at $50/1M — so a 10-15-turn exploratory Fable session costs 5-10x a single-shot one (observed before this rule: ~$5.50/task in Fable spend). The pipeline therefore runs every Fable call **single-shot**: the orchestrator inlines all inputs in the delegation prompt, and the agent makes at most one tool call (its artifact write). OpenCode applies OpenRouter cache breakpoints for Claude models automatically (cache reads bill at 0.1x), which softens the turns that remain; DeepSeek caching is automatic.

- **TRIVIAL**: ~$0.05–0.20 (no Fable at all)
- **STANDARD**: ~$1–2 — the single-shot Fable plan call dominates; review finder passes cost $0 in dollars (OpenAI subscription) and the falsification fan-out cents. The review's real budget is subscription rate-limit headroom — a large angle selection spends quota, not money (and fallback passes bill on OpenRouter normally). A checkpoint-requested revision adds a second Fable call of similar size.
- **COMPLEX / --review=critical**: add ~$0.5–2 per extra Fable call (critical review pass, escalation).
- **ARCHITECTURAL**: one larger migration-plan call (~$2–4: map input plus a bigger decomposition output) + ~$0.30–0.80 per NORMAL step + ~$1–2 per RISKY step (its own Fable step-plan). A 10-step migration with 2 risky steps ≈ $8–14 total, spread across sessions.
- **INVESTIGATE**: ~$0.10–0.50 (v4-pro session; scales with how much reproducing/bisecting is needed) + ~$0.50–1.50 for the optional opt-in Fable synthesis.
- **REVIEW-ONLY**: ~$0.05–0.30 depending on the `--review` envelope and diff size; a critical-tier Fable pass adds ~$0.50–1.50.

Each run logs an estimated per-stage spend table in `.pipeline/status.md`. Estimates are chars/4 with input multiplied by the agent's tool-turn count and reasoning output doubled for thinking tokens — still rough; exact usage/cost is in your OpenRouter activity dashboard, filterable by model. The final summary includes the per-stage breakdown — use it to tune models with data.

## Pipeline state (`.pipeline/`, gitignored)

| File | Lifetime | Contents |
|---|---|---|
| `repo-brief.md` | **persistent** | Repo map, conventions, build/test/lint commands, generated-at HEAD. Reused across runs; regenerated only when stale (>500 changed lines or build-config changes since its HEAD). |
| `lessons.md` | **persistent** | Post-mortem institutional memory (incl. which review angles paid off), capped at 50 entries. Fed into every future context package. |
| `context.md` | per run | Task restatement, tier, touched-file contents, relevant lessons, acceptance criteria (map form on architectural runs; step-scoped in the step loop) |
| `migration-state.md` | **per migration** | Architectural runs: approved step list + status, cross-step invariants, per-step HEAD/diff-hash/spend records. The resume point — a later `/pipeline` invocation picks up the next pending step. |
| `plan.md` | per run | The approved plan: decisions, contracts, acceptance tests, definition of done, review strategy |
| `plan-annotations.md` | per run | Advisory annotator output: COVERAGE checks + OBSERVATIONS for the checkpoint |
| `code-reviews.md` | per run | Aggregated + confirmed review findings |
| `verify-log.md` | per run | Verification attempts, diagnoses, escalation diagnosis |
| `investigation.md` | per run | Investigate mode: questions, evidence-backed answer + verdict, ruled-out hypotheses, recommended next action (+ optional Fable synthesis) |
| `status.md` | per run | Current stage, flags, loop counters, deviations, spend table |
| `diff.patch` | per run | The diff under review |

Files are structured stable-content-first and append-only, so OpenRouter prompt caching stays effective when the same package is fed to the planner, reviewers, and implementer.

## Porting this pipeline to another repo

The pipeline is repo-agnostic by design: everything repo-specific is discovered at runtime and lives in `.pipeline/` state, not in the config. To port it, copy into the target repo:

- `.opencode/` (all agents + the command)
- `opencode.jsonc` (provider/model config — needs your OpenRouter key connected once via `/connect` in OpenCode on that machine)
- the `.pipeline/` line in `.gitignore`
- this `PIPELINE.md`

**Never copy `.pipeline/`** — it is per-repo state. `repo-brief.md` regenerates on the first run (build/test/lint commands, conventions, whether a determinism/consistency harness exists), and `lessons.md` must start empty: this repo's lessons and its cross-cutting invariants block are specific to this engine. The new repo accretes its own invariants block through post-mortems. Rubric mentions of hashing/determinism are conditional ("where the repo has them") — repos without such harnesses simply skip those checks.

## Adjusting models later

- **Restart OpenCode after ANY change to `.opencode/agents/*.md` or `opencode.jsonc`.** Agent definitions are loaded once per app process and never hot-reloaded (memoized instance state, no file watcher — verified in v1.17.18). A run started in an already-open OpenCode window silently uses the definitions from when that window launched.
- **Swap a model**: change the `model:` line in the agent's file under `.opencode/agents/`, and add/adjust the matching entry under `provider.openrouter.models` in `opencode.jsonc` (effort + output cap). Verify the slug exists on openrouter.ai first, and check its `supported_parameters` before sending `reasoning`/`reasoning_effort` (MiniMax M3 and Kimi K2.7-code do not accept effort).
- **Add/remove a reviewer**: create/delete a `reviewer-<model>.md` agent (copy an existing one — they're generic; the rubric arrives per-task), then update the catalog's primary/fallback mapping in the orchestrator's Stage 4 and the catalog note in `planner.md`. The primary reviewer (`reviewer-gpt-5.6-sol`) runs on the OpenAI **subscription** login, not OpenRouter. The orchestrator classifies Sol failures: an auth/provider error (login missing or lapsed) switches ALL remaining angles to the OpenRouter fallback fleet after a single failed call, flagged in `status.md` and the final summary since those passes bill per token; a rate-limit/transient error falls back per-angle only.
- **Effort policy**: send only OpenRouter-normalized values (`low`/`medium`/`high`) — never DeepSeek's native `max` (400 error). `high` is the pipeline-wide ceiling; Fable agents are always exactly `high`.
- **Planner fallback chain**: Fable's safety layer can refuse benign game-engine tasks (combat/damage vocabulary). On a planner error or refusal the orchestrator sends the identical prompt to `planner-sol` (Sol on the OpenAI subscription), then `planner-sol-openrouter` (billed per token) — same budget slot, disclosed at the checkpoint; revisions go to whichever model authored the plan. The two fallback agents' prompt bodies are **byte mirrors of `planner.md`** — after editing `planner.md`, regenerate them: `awk 'c==2{print} /^---$/{c++}' planner.md` gives the body; keep each fallback file's own frontmatter and replace everything below it.
- **Implementer routing**: it runs on `z-ai/glm-5.2:exacto`. If you see truncated diffs or flaky tool calls, the cheapest GLM routes (32K output caps, undisclosed quantization) are the usual suspect — pin a provider via the commented-out `provider` block in `opencode.jsonc`.
- **Model-diversity rules to preserve**: no diff reviewer may share a model family with the implementer (Z.ai/GLM) — which is why no GLM reviewer agent exists — and every diff must be reviewed by at least two distinct model families.

## Guarantees

- The pipeline **never runs `git commit` or `git push`** — all output stays in the working tree for human review (repo rule).
- **Only the implementer modifies code, within plan scope.** The orchestrator and verifier never mutate the working tree (no formatters, auto-fixers, or codegen — quality checks come exclusively from the commands the repo brief records, so a tool the repo doesn't configure is never run), and the Stage 3.5/6 gate mechanically compares `git status` against the plan's file list, failing loudly on out-of-scope changes instead of ever cleaning them up itself.
- **Fable's plan is never edited by a cheaper model**, and cheaper models' critiques never force a revision — revision is human-triggered (or objective-coverage-triggered on autonomous runs).
- Reviewers never edit code; the implementer never reviews its own diff; reviewers work blind to each other, only the aggregator sees everything.
- Reviewed code always passed tests/lint/type-check first (Stage 3.5 gate); BLOCKING findings are empirically confirmed with failing repro tests before anything gets "fixed".
