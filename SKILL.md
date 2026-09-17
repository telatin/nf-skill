---
name: nf-skill
description: Build and modify lean Nextflow DSL2 pipelines and pipeline-local modules. Use when the user asks to make a Nextflow pipeline, workflow, process, module, samplesheet-driven analysis, nextflow.config, container profile, or stub-run test; default to lightweight project code rather than nf-core community scaffolding.
metadata:
  version: "1.0.0"
---

# Nextflow Pipeline Maker

Build complete, runnable Nextflow DSL2 pipelines with small, explicit interfaces. Optimize for a pipeline owned by one project or lab. Do not impose nf-core's community-module scaffolding unless the user explicitly requests nf-core compliance.

## Non-Negotiable Defaults

- Use DSL2 and current syntax that is compatible with Nextflow's strict parser.
- Keep processes in pipeline-local modules and orchestration in workflows.
- Give every sample-bearing channel a `meta` map with at least `meta.id`.
- Pin every software version. Never use `latest`, floating tags, or guessed image tags.
- Every external-tool process supports Docker and Singularity/Apptainer with equivalent pinned builds.
- Conda is optional. Add it only when a valid pinned environment can be supplied.
- Every process has named outputs, a `when:` guard, `versions.yml`, and a tool-free `stub:` block.
- Tool options come from `task.ext.args`; output names come from `task.ext.prefix`.
- Verify channel wiring with `-stub-run`, then run the smallest feasible real container test.
- Make the smallest complete pipeline. Do not add `meta.yml`, nf-test, schema files, CI, MultiQC, or docs merely to resemble nf-core.

Read [references/module-contract.md](references/module-contract.md) before writing or changing any process module. Read [references/pipeline-patterns.md](references/pipeline-patterns.md) before designing channels or the project layout. Read [references/verification.md](references/verification.md) before claiming completion.

## Working Method

### 1. Establish the Contract

Inspect the repository before designing anything. Determine:

- Existing layout, style, config profiles, resource labels, tests, and Nextflow version constraints.
- Input records and cardinality: one item per sample, one shared reference, grouped replicates, or optional files.
- Exact tools, commands, expected outputs, and which outputs are optional.
- Target executors and container engines.
- Whether this is a new pipeline, an extension, or a repair.

Infer ordinary details from the code and user-provided files. Ask one concise question only when an unresolved scientific or interface decision would change results, such as reference assembly, paired-end interpretation, or mutually exclusive algorithms. Do not ask about choices that can be implemented safely as parameters.

For a new pipeline, turn the request into a short internal contract before editing:

```text
inputs -> channel shape -> PROCESS(inputs) -> named outputs -> next process
```

File existence is not enough. Define the semantic identity carried by `meta`, how records are paired or grouped, and whether shared inputs are reusable value channels.

### 2. Verify Software Artifacts

Before writing a `container` directive:

1. Identify an exact tool version compatible with the intended command.
2. Verify the Docker image and immutable version/build tag in its registry.
3. Verify the matching prebuilt Singularity image or SIF endpoint.
4. Ensure both refer to the same package version and build.
5. Record the tool's real version command and normalize it to a plain version string.

Use registry APIs, package metadata, or vendor documentation when network tools are available. Never fabricate a Biocontainers build hash. If no matching artifacts exist, use a versioned project-owned Docker image plus a versioned SIF/ORAS artifact from a controlled registry. If neither can be verified, stop and report the concrete provisioning gap rather than creating a misleading module.

### 3. Design the Graph

Prefer a linear graph until the science requires branching. Use named workflow outputs and named process emissions. Keep channel transformations near the workflow that owns them.

- Parse and validate input once at the boundary.
- Convert rows immediately to tuples such as `tuple(meta, reads)`.
- Pass files and result-affecting values as process inputs, not hidden globals or `ext` values.
- Use value channels for a reference or database reused by many sample tasks.
- Use `join` for key-based pairing, `combine` for deliberate products, and `groupTuple` only with a known grouping key and expected cardinality.
- Do not consume the same queue channel independently in ways that race or split records.
- Do not read `params.*` inside reusable modules.

Use `main.nf` as a thin entry point. For more than two or three processes, put orchestration in `workflows/<name>.nf`. See [references/pipeline-patterns.md](references/pipeline-patterns.md).

### 4. Implement Modules

Create one process per tool command or coherent piped operation. A module path follows the command:

```text
modules/<tool>/<subcommand>/main.nf
modules/<tool>/<subcommand>/environment.yml  # only if Conda is supported
```

A single-command tool may use `modules/<tool>/main.nf`. Process names are upper snake case and match the path, for example `BWA_MEM` or `SEQKIT_STATS`.

Follow [references/module-contract.md](references/module-contract.md) exactly unless existing project conventions are stricter. In particular:

- Detect both `singularity` and `apptainer` in the image conditional.
- Escape shell `$` expressions inside interpolated script strings.
- Derive output-affecting decisions in Groovy before the script.
- Reject invalid input/argument combinations before launching the command.
- Reproduce filename logic in the stub.
- Emit syntactically valid placeholder files when downstream tools inspect formats; do not merely `touch` a gzip file.
- Never execute a tool, package manager, or network command from a stub.

### 5. Add Configuration

Put defaults and profiles in `nextflow.config`, resource policies in `conf/base.config` when the pipeline is nontrivial, and per-process arguments/publishing in `conf/modules.config` when needed.

Provide mutually exclusive profiles for the engines the project supports:

```groovy
profiles {
    docker {
        docker.enabled = true
    }
    singularity {
        singularity.enabled = true
        singularity.autoMounts = true
    }
    apptainer {
        apptainer.enabled = true
        apptainer.autoMounts = true
    }
    conda {
        conda.enabled = true
        conda.useMamba = true
    }
}
```

Only expose profiles that all included modules can satisfy. Because Docker and Singularity/Apptainer support are mandatory for external tools, those profiles normally exist. Add Conda only if every required external-tool process has a valid environment.

Choose honest resource labels from actual tool behavior. Keep resource values in config so infrastructure can override them. Prefer bounded retry escalation with `task.attempt` for memory/time failures.

### 6. Build a Tiny Test Path

Every new pipeline needs an executable tiny path, not just syntactically plausible code.

- Add a `test` profile whose inputs are tiny local fixtures or stable pinned remote fixtures.
- Keep fixtures small enough for fast `-stub-run` and practical container smoke tests.
- Ensure stubs create exactly the names and basic formats expected downstream.
- Exercise single-end/paired-end, optional output, or branching cases when they alter the graph.
- Add nf-test only when the project already uses it or the user asks for component-level tests.

### 7. Verify and Repair

Run checks in escalating cost order and repair failures before moving on:

1. Inspect resolved config.
2. Run Nextflow lint or syntax validation when supported by the installed version.
3. Run the complete pipeline with the test profile and `-stub-run` without relying on tool availability.
4. Run the smallest feasible real test with Docker.
5. Run Singularity/Apptainer when the runtime is available; otherwise verify the artifact URL and say runtime execution was unavailable.
6. Inspect outputs, `versions.yml`, trace status, and `.nextflow.log`, not only the exit code.

Use the commands and acceptance checks in [references/verification.md](references/verification.md). Do not weaken assertions or remove outputs to make a test pass.

## Change Strategy

When modifying an existing pipeline, preserve established interfaces unless the requested behavior requires a break. Trace each changed channel from producer through every consumer. Update stubs whenever output names or shapes change. Do not replace working local conventions with nf-core conventions solely for uniformity.

When debugging, start from the first failed task and inspect `.command.sh`, `.command.err`, `.command.out`, and staged inputs in its work directory. Separate failures into graph/interface, staging, command, resource, and provisioning classes before editing.

## Completion Standard

A pipeline is complete only when:

- Every declared input has a producer with the correct channel shape.
- Every process output exists under all legitimate branches or is explicitly optional.
- Docker and Singularity/Apptainer artifacts are pinned and verified.
- Real runs emit actual discovered tool versions.
- Stub runs need no installed tools and emit valid `versions.yml` placeholders.
- The end-to-end `-stub-run` passes.
- At least one real container path passes when its runtime is available.
- The final report states exactly what was run and any unexecuted runtime checks.

## Scope Boundary

This skill makes custom, maintainable pipelines. If the user explicitly asks for an nf-core community pipeline or distributable nf-core module, switch modes and add the required template layout, `meta.yml`, schema, nf-test snapshots, linting, and community conventions. Do not silently mix that larger contract into a lab-local pipeline.
