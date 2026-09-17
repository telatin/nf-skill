# Lean Pipeline Patterns

## Recommended Layout

Start with only what the pipeline needs:

```text
main.nf
nextflow.config
conf/
  base.config
  modules.config
workflows/
  pipeline.nf
modules/
  tool/subcommand/main.nf
assets/
  samplesheet.csv
tests/
  data/
```

For a one- or two-process pipeline, orchestration may remain in `main.nf`. Add `workflows/` when it improves separation. Omit empty directories and unused config files.

Do not add nf-core-specific `modules.json`, `.nf-core.yml`, `meta.yml`, or schema machinery unless the user requests that ecosystem contract or the repository already uses it.

## Thin Entry Point

```groovy
#!/usr/bin/env nextflow

include { PIPELINE } from './workflows/pipeline'

params.input  = null
params.outdir = 'results'

workflow {
    if (!params.input) {
        error 'Missing required parameter: --input'
    }

    ch_input = channel.fromPath(params.input, checkIfExists: true)
    PIPELINE(ch_input)
}
```

Prefer validating parameters before constructing expensive channels. For rich samplesheets, use a dedicated parsing function or local validation process, but keep the resulting channel shape obvious.

## Samplesheet Boundary

Map rows into metadata plus paths immediately:

```groovy
ch_samples = channel
    .fromPath(params.input, checkIfExists: true)
    .splitCsv(header: true, sep: ',')
    .map { row ->
        if (!row.sample || !row.fastq_1) {
            error "Samplesheet rows require sample and fastq_1: ${row}"
        }

        def paired = row.fastq_2 as boolean
        def meta   = [id: row.sample, single_end: !paired]
        def reads  = paired
            ? [file(row.fastq_1, checkIfExists: true), file(row.fastq_2, checkIfExists: true)]
            : [file(row.fastq_1, checkIfExists: true)]

        tuple(meta, reads)
    }
```

Validate uniqueness of `meta.id` when one row is expected per sample. If lanes or replicates legitimately repeat IDs, represent lane/replicate identity explicitly and group at a defined step.

## Workflow Composition

```groovy
include { TOOL_A } from '../modules/tool/a/main'
include { TOOL_B } from '../modules/tool/b/main'

workflow PIPELINE {
    take:
    ch_samples

    main:
    TOOL_A(ch_samples)
    TOOL_B(TOOL_A.out.result)

    ch_versions = TOOL_A.out.versions.mix(TOOL_B.out.versions)

    emit:
    result   = TOOL_B.out.result
    versions = ch_versions
}
```

Aggregate versions at workflow boundaries. Publish final or user-relevant outputs, not every intermediate by default.

## Shared References

A shared reference should be reusable for all sample tasks. Construct it as a value channel when it is a single staged object:

```groovy
ch_reference = channel.value(file(params.reference, checkIfExists: true))
ALIGN(ch_samples, ch_reference)
```

For an indexed reference composed of multiple files, create one tuple or collected value with an explicit shape. Do not pass a queue that emits index files one at a time unless the process should run once per index file.

## Branching

Branch on metadata in the workflow, not by embedding pipeline policy in a generic module:

```groovy
ch_samples.branch { meta, reads ->
    paired: !meta.single_end
    single: meta.single_end
}
```

Use process `when:` for configurable process gating through `task.ext.when`. Use workflow branching when records take structurally different paths.

## Joining

Join channels only on stable explicit keys. Maps are poor join keys if unrelated metadata fields can differ. Reshape to make the key first when needed:

```groovy
ch_left_keyed  = ch_left.map  { meta, file -> tuple(meta.id, meta, file) }
ch_right_keyed = ch_right.map { meta, file -> tuple(meta.id, file) }
ch_joined      = ch_left_keyed.join(ch_right_keyed)
```

Check for missing and duplicate keys before a join if silent record loss would be scientifically dangerous.

## Resource Configuration

Modules declare intent with labels; config declares infrastructure values:

```groovy
process {
    errorStrategy = { task.exitStatus in [137, 140, 143] ? 'retry' : 'terminate' }
    maxRetries = 2

    withLabel: process_low {
        cpus   = { 2 * task.attempt }
        memory = { 4.GB * task.attempt }
        time   = { 2.h * task.attempt }
    }

    withLabel: process_medium {
        cpus   = { 4 * task.attempt }
        memory = { 8.GB * task.attempt }
        time   = { 4.h * task.attempt }
    }
}
```

Set project-appropriate caps if retries could exceed cluster limits.

## Publishing

Keep publishing policy out of reusable modules when practical. Configure it by process selector:

```groovy
process {
    withName: 'TOOL_B' {
        publishDir = [
            path: { "${params.outdir}/tool_b" },
            mode: 'copy',
            pattern: '*.result.txt'
        ]
    }
}
```

Never treat `publishDir` as process-to-process communication. Downstream tasks consume process output channels from work directories.

## Parameters Versus ext

- Use `params` at the pipeline boundary for user-facing choices and paths.
- Convert result-affecting parameters into explicit workflow/process inputs when they are part of a reusable component contract.
- Use `ext.args` for optional CLI rendering and `ext.prefix` for naming.
- Use `ext.when` for config-driven process enablement.
- Do not access `params` from a process module.
