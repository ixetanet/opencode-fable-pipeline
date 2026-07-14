# The Automated Development Process

This repo is built with a standardized, document-driven process. The idea is
simple: first you write the documents (requirements, design, implementation
checklist) together with Claude Code, then you let an automated pipeline build
the checklist for you, one milestone at a time.

This page explains the whole flow. All commands are Claude Code slash commands
that live in `.claude/commands/`.

## The big picture

Every product in the repo has a **document prefix** (all caps, e.g. `RTSENGINE`
or `STUDIO`). All documents that share a prefix belong together:

| Document | What it is |
|---|---|
| `<PREFIX>_REQUIREMENTS.md` | WHAT the product must do and WHY. The foundation. |
| `<PREFIX>_CORE_DESIGN.md` | The central design doc: vision, big picture, primary features, HOW. |
| `<PREFIX>_CORE_IMPLEMENTATION.md` | The build checklist: ordered milestones with checkboxes, worked top to bottom. |
| `<PREFIX>_<FEATURE>_DESIGN.md` | Design for one big feature that is too large for the core design doc. |
| `<PREFIX>_<FEATURE>_IMPLEMENTATION.md` | A separate checklist for that feature (only when it needs one). |

`DOCS.md` is the index of all documents — every command that creates or changes
a document also updates its row there.

The flow, start to finish:

```
requirements  ->  core design  ->  core implementation checklist  ->  automated build
   (create/expand)   (create/expand)      (create)                    (implement_full_*)
```

## Step 1: Write the requirements

```
/create_requirements_doc <PREFIX> <describe the project>
```

Claude explores the repo, then asks you a batch of questions (purpose, users,
constraints, non-goals, success criteria). It also proposes features you did
not mention but that fit the project's intent — accept, defer, or decline
each. Every question comes with 2-4 suggested answers and Claude's pick is
marked "(Recommended)". Your answers become `<PREFIX>_REQUIREMENTS.md`.

Then deepen it as many times as you like:

```
/expand_requirements_doc <PREFIX>
```

This re-reads the doc, finds gaps and open questions, and also **proposes new
feature-worthy additions** that fit the project's intent — things the doc does
not mention yet but that a project like this would want. Everything comes back
as another round of questions; what you accept goes in the doc, what you
decline is recorded as a non-goal so it is not proposed again. Run it until
the coverage feels good.

## Step 2: Write the core design

```
/create_core_design_doc <PREFIX>
```

Reads the requirements doc and asks about the major design decisions
(architecture, components, data model, contracts). Like the requirements
step, it also proposes features that fit the project's vision for you to
accept or decline. Writes `<PREFIX>_CORE_DESIGN.md` — the "central" doc with
the vision and big picture. Deep per-feature detail is deliberately left out;
it goes in feature design docs later.

Deepen it the same way, as often as needed:

```
/expand_core_design_doc <PREFIX>
```

Like the requirements expander, this finds gaps AND proposes new features that
fit the project's vision. Accepted proposals enter the design marked
`(planned)`; a proposal too big for the core doc is recommended as its own
feature design doc instead (see below). The same applies to the feature-level
commands (`/create_feature_design_doc`, `/expand_feature_design_doc`) within a
single feature's scope. Only the two implementation-doc commands never propose
features — they strictly turn an approved design into a checklist.

## Step 3: Write the implementation checklist

```
/create_core_implementation_doc <PREFIX>
```

Turns the core design into `<PREFIX>_CORE_IMPLEMENTATION.md`: milestones in
dependency order, each one small enough for a single automated build run.
Milestone ids are prefixed with `C` (Core): `C1`, `C2a`, `C2b`, ... Each
milestone (or lettered sub-unit) is one **unit of work**.

Implementation docs live for the whole project and get very large, so they
stay terse by rule: a one-line intro per milestone and one line per checkbox —
no design prose (that lives in the design doc; the checklist points at it).
Terse does not mean lossy: each checkbox still names the deliverable, its
non-obvious dependencies, and what "done" means. The same rule applies to
feature implementation docs.

## Step 4: Build it

Two ways to build:

**One unit at a time, interactively:**

```
/milestone-swarm-fable <UNIT>     (or -opus, or -sol)
```

This runs the milestone pipeline for one unit: Plan -> Implement -> Swarm
Review (many parallel reviewers) -> Falsify (each finding independently
verified) -> Fix -> Verify. It asks you clarifying questions and waits for
plan approval. Planning and aggregation always run on whatever model and
effort your console session is using — launch the console with the model you
want planning done at. The three variants differ in who does the reviewing:

- `-fable` — reviewers inherit your session's model too (launch on Fable for
  Fable reviews)
- `-opus` — Opus reviewers
- `-sol` — GPT 5.6 Sol reviewers via the codex CLI (runs on the codex
  subscription, so review cost is effectively free)

**The whole checklist, unattended:**

```
/implement_full_fable <document.md> [extra guidance]     (or _opus, or _sol)
```

This loops over every unchecked unit in the document, runs the matching
milestone-swarm pipeline for each with no human input, commits and pushes
after each passing unit, and halts on the first failure (leaving the working
tree intact for you to inspect). Because nobody approves the plans in this
mode, each plan gets a one-round critique from GPT 5.6 Sol (via the codex
CLI) before implementation starts: the planner folds in correctness and
clarity points, ignores scope or style suggestions, and records what it
accepted and rejected in the plan. Interactive `/milestone-swarm-*` runs skip
the critique — your approval gate covers it. This is the only part of the process that
commits on its own — everything else follows the normal rule: never commit
unless you ask.

## Big features get their own docs

When a feature is too large to fold into the core design doc:

```
/create_feature_design_doc <PREFIX>_<FEATURE> <describe the feature>
/expand_feature_design_doc <PREFIX>_<FEATURE>          (repeat as needed)
/create_feature_implementation_doc <PREFIX>_<FEATURE>
```

For example, `RTSENGINE_MULTIPLAYER` produces
`RTSENGINE_MULTIPLAYER_DESIGN.md` and `RTSENGINE_MULTIPLAYER_IMPLEMENTATION.md`.
The feature checklist uses feature-named milestone ids (`MULTIPLAYER1a`,
`MULTIPLAYER1b`, ...), and the core checklist gets a **single checkbox** that
points at it — the feature's steps are never inlined into the core list.

When `implement_full_*` reaches that single checkbox, it automatically descends
into the feature checklist, builds its units one by one, and only checks the
core box once the whole feature document is complete. So you can build a
feature directly (`/implement_full_sol RTSENGINE_MULTIPLAYER_IMPLEMENTATION.md`)
or just run the core checklist and let it get there on its own.

## Example: a brand-new project

1. `/create_requirements_doc PROJ I'm building a headless RTS engine to wire into Godot later`
2. Answer the questions; `/expand_requirements_doc PROJ` a few times.
3. `/create_core_design_doc PROJ`, then `/expand_core_design_doc PROJ` a few times.
4. `/create_core_implementation_doc PROJ`
5. `/implement_full_sol PROJ_CORE_IMPLEMENTATION.md` and let it run.
6. Months later, a big feature: `/create_feature_design_doc PROJ_MULTIPLAYER ...`,
   expand it, `/create_feature_implementation_doc PROJ_MULTIPLAYER`, then
   `/implement_full_sol PROJ_MULTIPLAYER_IMPLEMENTATION.md`.

## How it stays portable: `.claude/autodev.md`

The commands and agents themselves are **project-agnostic** — they never
hard-code anything about this repo. Every project-specific fact lives in one
file, `.claude/autodev.md`:

- the products and their document sets (prefix -> docs),
- how a milestone unit code maps to a domain, its repo(s), and required reading,
- the correctness contracts reviewers enforce (determinism, layering, boundaries),
- the build/test commands,
- the git policy and commit convention.

To port the whole system to another repository: copy `.claude/` (and `.codex/`
if you use the sol variant), rewrite `.claude/autodev.md` for the new project,
and write the new project's docs. Nothing else needs to change.
`bash .claude/check-agnostic.sh` enforces this — it fails if any generic file
under `.claude/` or `.codex/` mentions a project-specific name.

## Notes for this repo specifically

- The two products here are `RTSENGINE` (the engine, this directory) and
  `STUDIO` (the Godot editor, in the nested `rtsstudio/` repo).
- Their checklists predate the `C` convention and keep their historical
  milestone ids (bare numbers like `8`/`40b` for the engine, `S`-prefixed like
  `S1a` for Studio). Never renumber them — the `C` prefix is only for
  checklists created fresh by `/create_core_implementation_doc`.
- Before working here, read `CLAUDE.md` (the rules) and `DOCS.md` (the index).
