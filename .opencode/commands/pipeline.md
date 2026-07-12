---
description: Multi-model feature/bugfix pipeline - context, Fable plan, review swarm, GLM build, verify loop
agent: pipeline-orchestrator
---
Run the engineering pipeline for the following request.

REQUEST: $ARGUMENTS

First parse any flags embedded in the request (`--tier=trivial|standard|complex`, `--review=standard|deep|critical`, `--no-checkpoint`) and treat the remaining text as the task description. Then execute the pipeline exactly as defined in your system prompt, starting at Stage 0 (context prep) — including the complexity router, the Stage 2.5 human checkpoint (unless --no-checkpoint), the escalation rule, and per-stage spend logging in .pipeline/status.md.
