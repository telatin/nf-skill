# Verification Ladder

Run the applicable checks from cheapest to most expensive. Preserve logs long enough to diagnose failures.

## 1. Environment

```bash
nextflow -version
java -version
docker version
singularity --version
apptainer --version
```

Do not require every runtime to be installed. Record which are available and verify unavailable runtime artifacts remotely when possible.

## 2. Static and Config Checks

```bash
nextflow config -profile test
nextflow config -profile docker
nextflow config -profile singularity
nextflow lint .
```

Use `NXF_SYNTAX_PARSER=v2 nextflow lint .` when supported and when compatibility with the project's pinned Nextflow version has been confirmed. A missing `lint` command on an older Nextflow release is not a pipeline failure; continue with execution-based validation.

Inspect resolved config for:

- Exactly one enabled container engine per profile.
- Existing included config paths.
- Valid process selectors and resource labels.
- Test fixture paths that resolve from the launch directory.
- No secret tokens or machine-specific absolute paths committed as defaults.

## 3. End-to-End Stub Run

```bash
nextflow run main.nf -profile test -stub-run \
    -with-trace .verification/stub-trace.txt \
    -with-report .verification/stub-report.html
```

If the test profile enables a container engine, add a separate tool-free test profile or disable the engine for the plumbing test. The point of this run is to prove the graph without pulls or installed tools.

Accept only when:

- The workflow reports success.
- Every intended process appears in the trace for the selected fixture.
- Published filenames match real-script filenames.
- Placeholder compressed files can be decompressed.
- Every process emits parseable `versions.yml` with a stub value.
- Optional and branched outputs behave as designed.

## 4. Real Docker Smoke Test

```bash
nextflow run main.nf -profile test,docker \
    -with-trace .verification/docker-trace.txt \
    -with-report .verification/docker-report.html
```

If `test` and `docker` are not composable profiles, use the project's equivalent command. Do not assume profile composition; inspect resolved config.

Check:

- Every task used the expected pinned image.
- Outputs are structurally valid, not merely present.
- `versions.yml` reports the executable's discovered version.
- Output ownership and permissions are usable by the invoking user.
- Rerunning with `-resume` reuses unchanged tasks.

## 5. Singularity or Apptainer Smoke Test

```bash
nextflow run main.nf -profile test,singularity \
    -with-trace .verification/singularity-trace.txt
```

or:

```bash
nextflow run main.nf -profile test,apptainer \
    -with-trace .verification/apptainer-trace.txt
```

Confirm the prebuilt SIF path is selected rather than silently converting the Docker image, unless the project intentionally enables `ext.singularity_pull_docker_container`.

## 6. Failure Diagnosis

Locate the first failed task from `.nextflow.log` or trace, then inspect its work directory:

```text
.command.sh       rendered command
.command.run      Nextflow wrapper and container invocation
.command.err      stderr
.command.out      stdout
.exitcode         task exit status
```

Classify before editing:

| Class | Typical evidence | Correct response |
|---|---|---|
| Graph/interface | missing tuple fields, no task launched, channel mismatch | fix producer/consumer shapes |
| Staging | missing path, filename collision, inaccessible mount | fix declarations or runtime mounts |
| Command | usage error, malformed arguments, bad quoting | fix script rendering or inputs |
| Resource | exit 137/140/143, scheduler limit | adjust labels/retry policy |
| Provisioning | image not found, executable absent | correct and re-verify artifact |
| Output contract | command succeeds but declared output missing | align command, glob, and branching logic |

Do not use `optional: true`, `errorStrategy 'ignore'`, or weakened tests to conceal a contract violation.

## Completion Report

State:

- Files and pipeline behavior created or changed.
- Exact validation commands that passed.
- Which real runtimes were unavailable and therefore not executed.
- Any unverified image, scientific assumption, or external data dependency.

Do not claim Docker or Singularity support was tested if only a stub run passed.
