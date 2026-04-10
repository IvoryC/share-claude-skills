---
name: snakemake
description: >
  This skill should be used when the user asks to "write a Snakemake rule",
  "create a Snakemake workflow", "set up a Snakemake pipeline", "run Snakemake
  on SLURM", "debug a Snakemake error", "use Snakemake wrappers", "use
  checkpoints in Snakemake", "modularize a Snakefile", or mentions working with
  a Snakefile or .smk files. This skill is also appropriate when the user
  says "I'm using Snakemake" and asks for help with bioinformatics workflow
  design.
version: 1.0.0
user-invocable: true
allowed-tools: Read, Bash, Glob, Grep, Write, Edit
---

# Snakemake Workflow Development

Snakemake is a Python-based workflow manager for reproducible bioinformatics pipelines. Rules define how to produce output files from input files; Snakemake infers the execution order from a DAG. Current stable release: v9.x (as of April 2026). v8 was a major breaking release — check version with `snakemake --version` before assuming v7 patterns apply.

## Critical Rule-Writing Practices

These are non-obvious practices drawn from Nextstrain's production style guide and community experience. Apply them by default.

### Always quote shell interpolations

Snakemake does not quote interpolated values by default. Use the `:q` modifier on every interpolation:

```python
rule trim:
    input: "reads/{sample}.fastq"
    output: "trimmed/{sample}.fastq"
    params:
        adapter = lambda _: config["adapter"]
    shell:
        r"""
        trimmomatic SE {input:q} {output:q} \
            ILLUMINACLIP:{params.adapter:q}:2:30:10
        """
```

Without `:q`, values containing spaces, parentheses, or other shell metacharacters will silently break or become a security risk.

### Always use raw triple-quoted shell blocks

Use `r"""..."""` for every `shell:` block. This enables one-flag-per-line formatting, avoids nested quoting issues, and preserves backslash/newline formatting in Snakemake's log output.

### Always capture logs with tee and exec

Use this pattern for every rule that produces stdout/stderr:

```python
rule align:
    input: "trimmed/{sample}.fastq"
    output: "aligned/{sample}.bam"
    log: "logs/align/{sample}.log"
    shell:
        r"""
        exec &> >(tee {log:q})
        bwa mem {input:q} | samtools sort -o {output:q}
        """
```

`exec &> >(tee {log:q})` redirects the entire shell block's output to both the log file and the terminal. Without `exec`, only the last command's output is captured.

### Always add benchmark

Add `benchmark:` to every rule:

```python
benchmark: "benchmarks/{sample}.tsv"
```

This writes runtime and max RSS to a TSV automatically. Essential for identifying bottlenecks and estimating HPC resources without parsing logs.

### Never use run: blocks

`run:` blocks do not execute in the rule's conda environment, are harder to debug, and cannot be reused. Move any Python logic to an external script called via `script:` or `shell:`.

### Use params: to buffer config into shell

Never interpolate `config` directly in `shell:` commands. Map config values through `params:` first:

```python
params:
    min_len = lambda _: config["min_length"],
    flags    = lambda _: config["extra_flags"]
shell:
    r"""
    tool --min-length {params.min_len:q} {params.flags:q} \
        -i {input:q} -o {output:q}
    """
```

Benefits: supports arbitrary Python expressions, enables `--list-params-changes` detection, avoids non-standard `{config[key]}` interpolation syntax. Use `lambda _:` on params that may contain `{` or `}` to prevent wildcard resolution.

### Config access conventions

- `config[key]` — required key (raises clearly if missing)
- `key in config` — check if optional key is present
- `config.get(key, default)` — optional key with fallback
- **Never** `config.get(key)` without a default — silently returns `None`, masking missing required keys

### Avoid the message: attribute

`message:` suppresses Snakemake's default job output (job id, rule name, input, output paths). Omit it.

---

## Project Structure

Standard layout for a distributable workflow (required for Snakemake workflow catalog "standardized" tier):

```
workflow/
    Snakefile           ← main entry point (required for catalog)
    rules/
        *.smk
    envs/
        *.yaml
    scripts/
        *.py / *.R
    notebooks/
        *.py.ipynb
config/
    config.yaml
    README.md           ← required for catalog standardized tier
results/
resources/
.test/
    config/             ← minimal test data for CI
.snakemake-workflow-catalog.yml   ← required for catalog standardized tier
.github/workflows/
README.md
CHANGELOG.md
```

Format and lint before sharing:

```bash
snakefmt Snakefile workflow/rules/*.smk     # format (built on Black)
snakemake --lint                            # built-in linter
```

See `references/project-structure.md` for catalog submission requirements, CI setup, and snakefmt configuration.

---

## Snakemake v8 Breaking Changes (Quick Reference)

v8 was a major breaking release. Code written for v7 or earlier will likely need changes:

| v7 pattern | v8+ replacement |
|---|---|
| `--use-conda` | `--sdm conda` |
| `--use-singularity` | `--sdm apptainer` |
| `--kubernetes` | `--executor kubernetes` (plugin) |
| S3RemoteProvider, etc. | storage plugins (`--default-storage-provider s3`) |
| `dynamic()` output | `checkpoint` |
| `subworkflow:` directive | `module:` + `use rule` |
| `snakemake()` Python API | dataclass-based API |

Migration guide: https://snakemake.readthedocs.io/en/stable/getting_started/migration.html

---

## Using Wrappers

Wrappers are versioned, tested rule snippets that bundle their own conda environment. Use them for common tools instead of writing rules from scratch:

```python
rule bwa_mem:
    input:
        reads = ["reads/{sample}.fastq"]
    output:
        "mapped/{sample}.bam"
    log:
        "logs/bwa_mem/{sample}.log"
    params:
        extra = r"-R '@RG\tID:{sample}\tSM:{sample}'"
    wrapper:
        "v9.4.2/bio/bwa/mem"
```

Browse available wrappers: https://snakemake-wrappers.readthedocs.io/
Always pin to a specific wrapper version. The wrapper's conda env is activated automatically.

---

## Running on SLURM (Snakemake ≥ 8)

Install the executor plugin:

```bash
pip install snakemake-executor-plugin-slurm
```

Basic submission:

```bash
snakemake --executor slurm \
    --sdm conda \
    --default-resources slurm_account=<account> slurm_partition=<partition> \
    --jobs 100
```

Annotate resource requirements per rule:

```python
rule assemble:
    resources:
        mem_mb   = 32000,
        runtime  = 240,     # minutes
        cpus_per_task = 8,
        slurm_partition = "highmem"
    threads: 8
```

For adaptive memory on retry:

```python
resources:
    mem_mb = lambda wildcards, attempt: attempt * 8000
```

This doubles memory on each retry without manual intervention.

See `references/cluster-execution.md` for profiles, job grouping, and latency handling.

---

## Modularization

Four levels, from lightweight to heavyweight:

1. **`include: "rules/align.smk"`** — merges rules into current scope; shares config
2. **Wrappers** — versioned single-rule reuse via the wrapper repository
3. **`module`** (v6+) — imports an external workflow as a named module; rules can be renamed with `use rule * from mymodule as prefix_{name}`
4. **`use rule`** — selectively import or override individual rules from a module

For large workflows, split rules into `workflow/rules/*.smk` and `include:` them from the main `Snakefile`.

---

## Debugging

Snakemake execution has three distinct phases — knowing which phase an error comes from narrows the fix:

1. **Parse phase** — Python syntax errors, invalid rule directives. Fix: read the traceback carefully; these occur before any jobs run.
2. **DAG phase** — `MissingInputException`, `AmbiguousRuleException`, wildcard mismatches. Fix: `snakemake --dry-run` to inspect the DAG without executing.
3. **Execute phase** — tool failures, missing output files, resource issues. Fix: check the rule's log file; run with `--show-failed-logs` to print failed logs on exit.

Useful flags:

```bash
snakemake --dry-run -p          # print shell commands without executing
snakemake --dag | dot -Tpng     # visualize the DAG
snakemake --show-failed-logs    # print failed job logs on exit (always use this)
snakemake --rerun-incomplete    # resume after interrupted run
snakemake --forcerun rulename   # force re-execution of a rule
```

---

## Reference Files

For detailed guidance, read these files as needed:

| File | Contents |
|---|---|
| `references/rule-patterns.md` | Full rule-writing patterns: wildcards, input functions, checkpoints, temp/protected, shadow |
| `references/project-structure.md` | Directory layout, catalog requirements, snakefmt, CI/CD setup |
| `references/config-and-samples.md` | Config file design, sample sheets (pandas/PEPs), schema validation |
| `references/cluster-execution.md` | SLURM v8+ plugin, profiles, resource estimation, adaptive retry |
| `references/snakemake-style-guide.rst.txt` | Full Nextstrain style guide (source RST) |

