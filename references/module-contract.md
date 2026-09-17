# Pipeline-Local Module Contract

Use this contract for every external-tool process. Adapt names and I/O, not the reproducibility rules.

## Canonical Shape

```groovy
process TOOL_SUBCOMMAND {
    tag "$meta.id"
    label 'process_medium'

    conda "${moduleDir}/environment.yml"
    container "${ workflow.containerEngine in ['singularity', 'apptainer'] && !task.ext.singularity_pull_docker_container ?
        'https://depot.galaxyproject.org/singularity/tool:1.2.3--h1234567_0' :
        'quay.io/biocontainers/tool:1.2.3--h1234567_0' }"

    input:
    tuple val(meta), path(input_file)

    output:
    tuple val(meta), path("*.result.txt"), emit: result
    path "versions.yml",                    emit: versions

    when:
    task.ext.when == null || task.ext.when

    script:
    def args   = task.ext.args ?: ''
    def prefix = task.ext.prefix ?: "${meta.id}"
    """
    tool subcommand \
        $args \
        --threads $task.cpus \
        --input $input_file \
        --output ${prefix}.result.txt

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        tool: \$(tool --version 2>&1 | sed 's/^tool //')
    END_VERSIONS
    """

    stub:
    def prefix = task.ext.prefix ?: "${meta.id}"
    """
    touch ${prefix}.result.txt

    cat <<-END_VERSIONS > versions.yml
    "${task.process}":
        tool: "0.0.0-stub"
    END_VERSIONS
    """
}
```

Remove the `conda` directive and `environment.yml` together if Conda is unsupported. Never leave an invalid fallback.

## Interface Rules

- Use `tuple val(meta), path(...)` for sample-bearing inputs and outputs.
- Give independent tuples independent metadata names: `meta`, `meta2`, and so on.
- Use plain `path` for shared databases and plain `val` for scalar controls.
- If changing a scalar changes scientific results or task caching, prefer a declared `val` input. Reserve `task.ext.args` for optional command customization controlled by config.
- Name every output with `emit:`.
- Mark an output `optional: true` only if absence is a valid result, not to hide failures.
- Avoid broad globs that can capture staged inputs. Prefer `${prefix}`-based patterns.
- A process with no sample identity should omit `tag` rather than inventing metadata.

## Image Rules

The Docker and Singularity paths must resolve to equivalent builds. A valid pair normally shares the complete version/build suffix. Check both endpoints; matching-looking strings are not proof that they exist.

Use this conditional:

```groovy
container "${ workflow.containerEngine in ['singularity', 'apptainer'] && !task.ext.singularity_pull_docker_container ?
    '<verified-prebuilt-sif-or-url>' :
    '<verified-docker-image>' }"
```

The override supports sites that deliberately let Singularity convert a Docker image. The default remains a prebuilt image to avoid conversion races on shared filesystems.

## Script Rules

- Put optional CLI flags in `task.ext.args`, `args2`, and so on for multiple tools.
- Use `task.cpus` and other resolved resources instead of fixed thread counts.
- Use Groovy for decisions that determine output names or declared output presence.
- Use shell logic only for runtime facts unavailable before task execution.
- Escape Bash variables and command substitutions as `\$VAR`, `\${VAR}`, and `\$(command)` so Nextflow does not interpolate them.
- Quote paths where the target CLI permits it. Treat lists deliberately; do not stringify nested lists accidentally.
- Fail early with `error "message"` for impossible combinations such as CRAM output without a reference.
- Obtain versions from the actual executable. Do not infer them from the image tag.

## Stub Rules

A stub tests the dataflow contract without provisioning software.

- Recompute every filename-related `def` because `stub:` does not inherit `script:` locals.
- Create every output expected for the selected branch.
- Emit valid empty formats where necessary: use `printf '' | gzip -c > file.gz`, not `touch file.gz`.
- Use a minimal valid header when a downstream stub or validation reads structure.
- Write a static fake version such as `0.0.0-stub`.
- Never invoke the real tool, Conda, a container command, or the network.

## Multiple Output Modes

Compute the extension once and duplicate the same logic in the stub:

```groovy
script:
def args      = task.ext.args ?: ''
def prefix    = task.ext.prefix ?: "${meta.id}"
def extension = args.contains('--cram') ? 'cram' : args.contains('--sam') ? 'sam' : 'bam'
if (extension == 'cram' && !fasta) {
    error 'A FASTA reference is required for CRAM output'
}
"""
tool $args -o ${prefix}.${extension}
...
"""

stub:
def args      = task.ext.args ?: ''
def prefix    = task.ext.prefix ?: "${meta.id}"
def extension = args.contains('--cram') ? 'cram' : args.contains('--sam') ? 'sam' : 'bam'
"""
touch ${prefix}.${extension}
...
"""
```

Declare mutually exclusive patterns as optional outputs only when downstream consumers intentionally handle separate channels.

## Conda Environment

```yaml
channels:
  - conda-forge
  - bioconda
dependencies:
  - bioconda::tool=1.2.3
```

Pin the package version. Keep the environment process-specific and small.
