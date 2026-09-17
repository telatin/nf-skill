# nf-skill

Agent skill for building lean Nextflow DSL2 pipelines. Creates custom pipelines with pipeline-local modules, avoiding nf-core scaffolding unless explicitly requested.

## Quick Start

See [AGENTS.md](AGENTS.md) for instructions.

## Key Files

- [`SKILL.md`](SKILL.md) — skill definition
- [`AGENTS.md`](AGENTS.md) — agent instructions
- [`references/module-contract.md`](references/module-contract.md) — process module interface
- [`references/pipeline-patterns.md`](references/pipeline-patterns.md) — channel and graph patterns
- [`references/verification.md`](references/verification.md) — verification commands

## Verification Commands

```bash
# Config and static checks
nextflow config -profile test
nextflow config -profile docker
nextflow config -profile singularity

# Stub run (fastest verification)
nextflow run main.nf -profile test -stub-run

# Docker smoke test
nextflow run main.nf -profile test,docker
```
