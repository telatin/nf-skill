# Nextflow Pipeline Agent Instructions

Read these in order before writing or modifying pipelines:

- [SKILL.md](SKILL.md) — overall operating mode
- [references/module-contract.md](references/module-contract.md) — process module interface
- [references/pipeline-patterns.md](references/pipeline-patterns.md) — channel and graph design
- [references/verification.md](references/verification.md) — verification commands and acceptance

## What to preserve from existing pipelines

When modifying:

- Existing channel shapes, `meta` fields, and process I/O contracts
- Profile names and container engine choices (docker/singularity/apptainer)
- Resource label names and their assigned values in config
- Stub behavior and placeholder formats for each output
- Project-local config structure vs. nf-core conventions (only add nf-core scaffolding if requested)

## Key conventions that agents often miss

### Container images

- Every external-tool process MUST have both Docker and Singularity/Apptainer images verified and pinned to the same version/build.
- Use this conditional (never hard-code one path):
  ```groovy
  container "${ workflow.containerEngine in ['singularity', 'apptainer'] && !task.ext.singularity_pull_docker_container ?
      'https://depot.galaxyproject.org/singularity/tool:1.2.3--h1234567_0' :
      'quay.io/biocontainers/tool:1.2.3--h1234567_0' }"
  ```
- Do not fabricate Biocontainers build hashes. If no matching artifacts exist, use a versioned project-owned image plus a versioned SIF from a controlled registry.

### Process modules

- Follow [references/module-contract.md](references/module-contract.md) exactly:
  - Input: `tuple val(meta), path(input_file)` for sample-bearing channels
  - Outputs: `tuple val(meta), path("*.result.txt"), emit: result` and `path "versions.yml", emit: versions`
  - Every process has: named outputs, a `when:` guard, `versions.yml`, and a `stub:` block
  - Tool options: `task.ext.args`; output prefixes: `task.ext.prefix`
- Detect both `singularity` and `apptainer` in the image conditional.
- Escape shell `$` as `\$` and `$(...)` as `\$(...)` in script strings.
- Derive output-affecting decisions (filenames, extensions, conditionals) in Groovy before the script.
- Stub rules:
  - Recompute all filename-related `def` values; stub does not inherit script locals.
  - Create all outputs for all branches, including valid empty formats (e.g., `printf '' | gzip -c > file.gz`, not `touch file.gz`).
  - Never invoke tools, package managers, containers, or the network from stubs.

### Channel design

- Convert samplesheet rows to `tuple(meta, reads)` immediately at the boundary.
- Pass files and result-affecting values as process inputs, not `params` or `ext`.
- Use `join` for key-based pairing, `combine` for products, `groupTuple` only with a known key.
- Do not consume the same queue channel independently in ways that race or split records.
- Do not read `params.*` inside reusable modules.
- Shared references: use `channel.value(file(params.reference))` for single objects; for multi-file indexes, use a collected tuple with explicit shape.
- Branch on metadata in the workflow using `ch_samples.branch { meta, reads -> ... }`, not process `when:`.

### Configuration

- Profiles are mutually exclusive per engine: `docker`, `singularity`, `apptainer`.
- Conda is optional; add only when every required process has a valid pinned environment.
- Default resources and profiles: `nextflow.config`; resource labels: `conf/base.config`; per-process overrides: `conf/modules.config`.
- Configure `publishDir` by process selector in config, not inside modules.

## Verification commands (run in order)

```bash
# 1. Static and config
nextflow config -profile test
nextflow config -profile docker
nextflow config -profile singularity
# Optional if supported: NXF_SYNTAX_PARSER=v2 nextflow lint .

# 2. End-to-end stub run
nextflow run main.nf -profile test -stub-run \
    -with-trace .verification/stub-trace.txt \
    -with-report .verification/stub-report.html

# 3. Real Docker smoke test (if Docker is available)
nextflow run main.nf -profile test,docker \
    -with-trace .verification/docker-trace.txt \
    -with-report .verification/docker-report.html

# 4. Singularity or Apptainer smoke test (if available)
nextflow run main.nf -profile test,singularity \
    -with-trace .verification/singularity-trace.txt
```

Accept stub runs only when:
- The workflow reports success.
- Every intended process appears in the trace for the selected fixture.
- Published filenames match real-script filenames.
- Placeholder compressed files are valid (can be decompressed).
- Every process emits `versions.yml` with stub values.
- Optional and branched outputs behave as designed.

## Failure diagnosis

From `.nextflow.log` or trace, find the first failed task. Inspect its work directory:

```text
.command.sh       rendered command
.command.run      Nextflow wrapper and container invocation
.command.err      stderr
.command.out      stdout
.exitcode         task exit status
```

Classify and respond:

| Class | Evidence | Fix |
|---|---|---|
| Graph/interface | missing tuple fields, no task launched | fix producer/consumer shapes |
| Staging | missing path, filename collision | fix declarations or runtime mounts |
| Command | usage error, bad quoting | fix script rendering or inputs |
| Resource | exit 137/140/143 | adjust labels/retry policy |
| Provisioning | image not found, executable absent | correct and re-verify artifact |
| Output contract | command succeeds but declared output missing | align command, glob, and branching logic |

## Completion standard

A pipeline is complete only when:

- Every declared input has a producer with the correct channel shape.
- Every process output exists under all legitimate branches or is explicitly optional.
- Docker and Singularity/Apptainer artifacts are pinned and verified.
- Real runs emit actual discovered tool versions.
- Stub runs need no installed tools and emit valid `versions.yml` placeholders.
- The end-to-end `-stub-run` passes.
- At least one real container path passes when its runtime is available.
- The final report states exactly what was run and any unexecuted runtime checks.

## Scope boundary

This skill makes custom, maintainable pipelines. If the user explicitly asks for an nf-core community pipeline or distributable nf-core module, request clarification and add the required template layout, `meta.yml`, schema, nf-test snapshots, linting, and community conventions.
