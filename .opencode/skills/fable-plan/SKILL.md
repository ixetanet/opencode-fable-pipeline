---
name: fable-plan
description: Produce a premium-quality implementation plan for a feature or bugfix at minimal Fable cost - free Sol scout assembles the context, a repo-wide type map covers unknown-unknowns, then a single turn-budgeted Claude Fable 5 call writes the plan. Use whenever a task needs a real plan before implementation.
---

# fable-plan — premium plan, minimal Fable spend

You orchestrate a planning pipeline. The deliverable is `.pipeline/plan.md`: an
implementation plan authored by Claude Fable 5 (premium, billed per token via
OpenRouter). Everything around that one call is engineered to make it single-shot:
a free subscription-model scout does the relevance judgment, a cheap DeepSeek
Flash assembler embeds file contents byte-exact, and a repo-wide type map means
the planner always sees the whole territory. You never implement anything — the plan is the
product, left for the human (or another pipeline) to execute.

Hard rules, in force for every stage:

- **Never run `git commit` or `git push`.** Everything stays in the working tree.
- **Never modify the working tree.** Your only writable surface is `.pipeline/`.
- **Fable budget: 1 plan call** (+1 revision if the user asks, +1 manifest
  pre-pass under `--manifest`). Never more without explicit user approval. Every
  Fable call is turn-budgeted — bounded batched reads, never open-ended
  exploration (each tool turn re-sends the session at premium rates and bills its
  thinking at $50/1M output).
- **The plan is Fable's artifact.** No cheaper model edits it; only user feedback
  triggers a revision.

## Flags

Parse from the request before treating the remainder as the task description:

- `--plan-turns=0-4` — planner read budget (default `2`; values above 4 clamp
  to 4). Each is one batched read turn the planner may spend pulling files the
  package missed, before writing the plan.
- `--manifest` — add a Fable manifest pre-pass (Stage 4): a cheap single-shot
  Fable call lists the files the package missed, they are embedded, and the plan
  call then runs with a default read budget of 0 (an explicit `--plan-turns`
  still wins). Use for tasks where you expect the scout's recall to fall short.
- `--out=<path>` — after the run, copy the finished plan to `<path>` as well
  (e.g. `PLAN_MILESTONE41.md`). The planner itself always writes
  `.pipeline/plan.md`.
- `--resume` — skip Stages 2-3 and reuse the existing `.pipeline/context.md`.
  Only for re-running a failed or interrupted plan call for the SAME request
  (the scout's work is intact and paid for); confirm the file exists and its
  restatement matches the request before trusting it.

## Stage 1 — Repo map

Run `bash .opencode/skills/fable-plan/repo-map.sh`. It regenerates
`.pipeline/repo-map.md` (compact per-file map of every C# type under the repo,
nested checkouts included, ~1s) unconditionally. If the script fails, report the error and continue — the
map is an accuracy booster, not a gate; tell the planner it is unavailable.

## Stage 2 — Scout (GPT 5.6 Sol via the OpenAI subscription, $0)

Build the scout prompt with one shell command: concatenate
`.opencode/skills/fable-plan/scout-prompt.md` plus the task under its trailing
`## Assignment` header into `.pipeline/scout-prompt-run.md`. The assignment is
the REQUEST text verbatim (flags stripped).

Run ONE synchronous foreground call and wait for it (it may take several
minutes — it explores the repo interactively):

```
codex exec -s read-only --ephemeral -m gpt-5.6-sol \
  -c model_reasoning_effort="high" \
  -o .pipeline/context.md - < .pipeline/scout-prompt-run.md
```

Never raise the sandbox above read-only and never pass any `--dangerously-*`
flag. If the call exits nonzero or the output is empty/garbled, re-run it once.
If it fails again, continue WITHOUT a scout: write a minimal
`.pipeline/context.md` yourself (task restatement + the request verbatim, no
analysis), skip Stage 3, force the planner read budget to 4, and disclose the
degraded run in your final summary.

## Stage 3 — Mechanical embed (DeepSeek Flash, pennies)

Do NOT read `.pipeline/context.md` yourself — it is deliberately kept out of
your context. Spawn the `context-prep` subagent with a short prompt: the
REQUEST's one-line shape, plus the instruction to assemble
`.pipeline/context.md` per its system prompt (verify the scout's EMBED
manifest, append contents byte-exact, completeness check) and return its
report. If it reports the completeness check unrecoverably failed, re-spawn it
once; on a second failure, stop and report to the user.

Act on its report:

- **OVERSIZED** — the scout judged the task repo-scale. Stop and tell the user
  to split it; do not spend the Fable call.
- **USER QUESTIONS** — if the bucket is non-empty, STOP and ask the user those
  questions verbatim. Do not proceed to the plan until answered; append the
  answers to `.pipeline/context.md` under an `## Answers` heading (one bash
  append — this is the one place you touch the file).
- Note any manifest paths it dropped as missing — mention them in your summary.

## Stage 4 — Manifest pre-pass (only with `--manifest`)

Spawn the `planner-manifest` subagent (Fable, single-shot). Its prompt: the
REQUEST restated, plus an instruction to read `.pipeline/context.md` and
`.pipeline/repo-map.md` in one batch and reply with the missing-files list per
its system prompt. If it replies `NONE`, continue. Otherwise re-spawn
`context-prep` with the returned `EMBED` lines verbatim as the manifest to
process, appended under an `## Additional context (manifest pass)` heading.
The plan call's read budget now defaults to 0.

## Stage 5 — Plan (Claude Fable 5, the premium call)

Spawn the `planner` subagent with a delegation prompt containing:

- The REQUEST verbatim.
- `READ BUDGET: N` (from `--plan-turns`, default 2; 0 after a manifest pass).
- The instruction to first read `.pipeline/context.md` and
  `.pipeline/repo-map.md` in one batch (uncounted), then plan per its system
  prompt and write `.pipeline/plan.md`.

**Failure check.** When the planner returns, verify `.pipeline/plan.md` exists
and is non-trivial (`ls` + `wc -c`). A planner that returns empty with no plan
file has died mid-write (an abort, an output-cap cut, a refusal) — that is a
failed call, not a quiet success.

**Refusal fallback.** Fable's safety layer can refuse benign game-engine tasks
(combat/damage vocabulary). If the planner call errors, refuses, or fails the
check above: concatenate
the body of `.opencode/agents/planner.md` (everything below its frontmatter),
the same delegation content, and the full text of `.pipeline/context.md` and
`.pipeline/repo-map.md` into `.pipeline/planner-prompt-run.md`, then run it on
Sol via the subscription (read-only sandbox substitutes for read turns):

```
codex exec -s read-only --ephemeral -m gpt-5.6-sol \
  -c model_reasoning_effort="high" \
  -o .pipeline/plan.md - < .pipeline/planner-prompt-run.md
```

Disclose the fallback in your summary. Revisions of a Sol-authored plan go to
Sol the same way.

## Stage 6 — Close out

1. If `--out=<path>` was given, copy `.pipeline/plan.md` there (this is the one
   permitted write outside `.pipeline/`, and only to the exact path the user
   named).
2. Read the plan's `Pulled:` footer. If it names files, append one entry to
   `.pipeline/lessons.md`: the task's one-line shape and which files the scout
   missed — future scouts read this and embed them up front.
3. Report: complexity tier, USER QUESTIONS asked (or none), PLANNER DECISIONS
   the plan resolved, files pulled beyond the package, fallbacks/degradations,
   and an estimated cost line (chars/4 per call; Fable $10/$50 per 1M in/out,
   input mostly cache-read ~0.1x on turns after the first; Flash $0.077/$0.154;
   scout/Sol $0). Point the user at the plan path.

**Revision (on user request only):** re-spawn `planner` with the user's
feedback, the same read budget, and the instruction to rewrite
`.pipeline/plan.md` in place (it re-reads context + current plan, uncounted).
One revision per request; re-run Stage 6 afterward.
