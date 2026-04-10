# Snakemake Cluster Execution

## Snakemake v8+: Executor Plugin Architecture

In Snakemake ≥8, all cluster and cloud backends are executor plugins installed separately. The old `--cluster` flag and `Snakemake-Profiles/slurm` profile are deprecated/unsupported for v8+.

Install the SLURM executor:
```bash
pip install snakemake-executor-plugin-slurm
# or via conda:
conda install -c conda-forge snakemake-executor-plugin-slurm
```

---

## Basic SLURM Submission (v8+)

```bash
snakemake \
    --executor slurm \
    --sdm conda \
    --default-resources \
        slurm_account="myaccount" \
        slurm_partition="general" \
        mem_mb=8000 \
        runtime=120 \
    --jobs 50 \
    --latency-wait 60 \
    --show-failed-logs
```

- `--jobs 50` — maximum concurrent jobs
- `--latency-wait 60` — seconds to wait for output files to appear after a job finishes (important on shared filesystems with metadata lag)
- `--show-failed-logs` — print failed job logs to terminal on exit (always use this)

---

## Per-Rule Resource Annotations

Annotate resources in every rule. These map directly to SLURM `#SBATCH` directives:

```python
rule assemble:
    input: "reads/{sample}.fastq.gz"
    output: "assembly/{sample}/contigs.fa"
    log: "logs/assemble/{sample}.log"
    benchmark: "benchmarks/assemble/{sample}.tsv"
    threads: 16
    resources:
        mem_mb          = 64000,
        runtime         = 480,          # minutes
        cpus_per_task   = 16,
        slurm_partition = "highmem",
        slurm_account   = "myaccount"
    conda: "envs/megahit.yaml"
    shell:
        r"""
        exec &> >(tee {log:q})
        megahit -1 {input:q} -t {threads} -o assembly/{wildcards.sample}
        cp assembly/{wildcards.sample}/final.contigs.fa {output:q}
        """
```

`threads` and `resources.cpus_per_task` should match. `runtime` is in minutes.

---

## Adaptive Memory Scaling on Retry

Automatically increase memory allocation on each retry without manual intervention:

```python
rule map_reads:
    resources:
        mem_mb  = lambda wildcards, attempt: attempt * 8000,
        runtime = lambda wildcards, attempt: attempt * 120
```

On first attempt: 8 GB / 120 min. On second: 16 GB / 240 min. On third: 24 GB / 360 min.

Configure retries globally:
```bash
snakemake --retries 3 ...
```

Or per rule:
```python
rule heavy_job:
    retries: 3
```

---

## Profiles

A profile collects command-line defaults into a YAML file so they don't need to be specified on every run.

Create `profiles/slurm/config.yaml`:
```yaml
executor: slurm
sdm:
  - conda
jobs: 100
latency-wait: 60
show-failed-logs: true
retries: 2
default-resources:
  - slurm_account=myaccount
  - slurm_partition=general
  - mem_mb=8000
  - runtime=120
  - cpus_per_task=1
```

Run using the profile:
```bash
snakemake --profile profiles/slurm
```

Store profiles in the workflow repo at `profiles/<name>/config.yaml`. The profile directory name is the profile identifier.

---

## Cluster Resource Estimation

Use `seff <JOBID>` after jobs complete to get CPU and memory efficiency:

```
Job ID: 12345678
State: COMPLETED
Cores: 8
CPU Utilized: 07:23:15
CPU Efficiency: 84.5%
Memory Utilized: 14.2 GB
Memory Efficiency: 88.7% of 16 GB
```

Use the benchmark TSVs to estimate resources for future runs:

```python
import pandas as pd
df = pd.read_csv("benchmarks/assemble/sample_A.tsv", sep="\t")
# Columns: s, h:m:s, max_rss, max_vms, max_uss, max_pss, io_in, io_out, mean_load, cpu_time
max_rss_gb = df["max_rss"].max() / 1000
```

For resource estimation, use `max_rss * 1.2` for memory and `elapsed * 1.25` for walltime as conservative buffers.

---

## Job Grouping

Group short-running rules into a single SLURM job to avoid scheduler overhead:

```bash
snakemake --group-components group1=5 ...
```

Or annotate rules:
```python
rule short_rule:
    group: "preprocessing"
    resources:
        slurm_partition = "short"
```

---

## Handling Shared Filesystem Latency

On HPC systems with distributed filesystems (Lustre, GPFS), output files may not be immediately visible to the scheduler node after a job writes them. Symptoms: `MissingOutputException` on jobs that succeeded.

Fix:
```bash
snakemake --latency-wait 120   # increase from default 5 seconds
```

For workflows with many small files, also consider:
```bash
snakemake --wait-for-files    # explicit file presence check
```

---

## Keeping Jobs Running After Disconnect

Submit the Snakemake master process itself as a SLURM job:

```bash
# submit_workflow.sh
#!/bin/bash
#SBATCH --job-name=snakemake_master
#SBATCH --output=logs/snakemake_master_%j.out
#SBATCH --error=logs/snakemake_master_%j.err
#SBATCH --time=48:00:00
#SBATCH --mem=4G
#SBATCH --cpus-per-task=1
#SBATCH --partition=general

snakemake --profile profiles/slurm --jobs 100
```

```bash
sbatch submit_workflow.sh
```

The master process manages the DAG and submits child jobs; it needs only modest resources (4 GB RAM, 1 CPU) but must run for the full workflow duration.

---

## Useful Execution Flags

```bash
# Preview without running
snakemake --dry-run -p --profile profiles/slurm

# Visualize the DAG
snakemake --dag | dot -Tpng > dag.png
snakemake --rulegraph | dot -Tpng > rulegraph.png

# Resume after interruption
snakemake --rerun-incomplete --profile profiles/slurm

# Force re-run specific rule
snakemake --forcerun rulename --profile profiles/slurm

# Run only up to a certain rule
snakemake --until rulename --profile profiles/slurm

# Print shell commands that will be run
snakemake -p --dry-run --profile profiles/slurm

# Show which files are out of date
snakemake --list-input-changes
snakemake --list-params-changes
snakemake --list-code-changes
```

---

## Snakemake v7 and Earlier (Legacy)

For workflows still on v7, the community SLURM profile is:
https://github.com/Snakemake-Profiles/slurm

This profile is not maintained for v8+. Migrate to `--executor slurm` when upgrading.

Key v7 → v8 cluster-related changes:
- `--cluster "sbatch ..."` → `--executor slurm`
- `--use-conda` → `--sdm conda`
- `--use-singularity` → `--sdm apptainer`
- Remote providers (S3RemoteProvider etc.) → storage plugins
