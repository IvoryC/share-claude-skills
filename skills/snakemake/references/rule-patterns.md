# Snakemake Rule Patterns

## Wildcards

Wildcards are inferred from the target file requested by `rule all`. Define wildcards in `output:` and reference them in `input:`, `log:`, `benchmark:`, and `params:`.

```python
rule all:
    input:
        expand("results/{sample}.bam", sample=config["samples"])

rule align:
    input:
        r1 = "reads/{sample}_R1.fastq.gz",
        r2 = "reads/{sample}_R2.fastq.gz"
    output:
        "results/{sample}.bam"
    log:
        "logs/align/{sample}.log"
    benchmark:
        "benchmarks/align/{sample}.tsv"
    shell:
        r"""
        exec &> >(tee {log:q})
        bwa mem {input.r1:q} {input.r2:q} | samtools sort -o {output:q}
        """
```

**Filename-as-provenance convention (NBIS):** Name intermediate files to reflect their processing history. `{sample}.trimmed.filtered.sorted.bam` is self-documenting; `{sample}_processed.bam` is not.

**Downstream determines wildcards:** Design the DAG working backward from the final output. The wildcard values needed by a rule are constrained by what the downstream rule requests — not by what the upstream rule produces.

### Expand

```python
# Single dimension
expand("results/{sample}.bam", sample=samples)

# Cross product (all combinations)
expand("results/{sample}_{unit}.bam", sample=samples, unit=units)

# Zip (paired: sample[0]↔unit[0], sample[1]↔unit[1])
expand("results/{sample}_{unit}.bam", zip, sample=samples, unit=units)
```

---

## Input Functions

Use lambda functions for input that depends on wildcard values or config:

```python
rule trim:
    input:
        lambda wildcards: config["samples"][wildcards.sample]["reads"]
    output:
        "trimmed/{sample}.fastq.gz"
```

Use named functions (defined before the rule) only when logic is complex enough to warrant it:

```python
def get_reads(wildcards):
    row = samples.loc[wildcards.sample]
    return {"r1": row.r1, "r2": row.r2}

rule align:
    input:
        unpack(get_reads)
    output:
        "aligned/{sample}.bam"
```

`unpack()` expands a dict-returning function into named inputs.

---

## Checkpoints

Use checkpoints when the number of output files is not known until runtime (e.g., the result of splitting or clustering). Replace the removed `dynamic()` directive.

```python
checkpoint split_by_chromosome:
    input: "all.vcf"
    output:
        directory("split/")
    shell:
        r"""
        mkdir -p {output:q}
        bcftools view {input:q} | split-by-chrom.sh {output:q}
        """

def get_chromosomes(wildcards):
    # Called after checkpoint completes
    checkpoint_output = checkpoints.split_by_chromosome.get(**wildcards).output[0]
    return expand("split/{chrom}.vcf",
                  chrom=glob_wildcards(os.path.join(checkpoint_output, "{chrom}.vcf")).chrom)

rule process_chromosomes:
    input:
        get_chromosomes
    output:
        "merged.vcf"
    shell:
        r"""
        bcftools concat {input:q} -o {output:q}
        """
```

---

## Output Modifiers

```python
output:
    temp("intermediates/{sample}.bam")       # deleted after all downstream rules complete
    protected("final/{sample}.bam")          # write-protected after creation
    directory("results/{sample}/")           # explicitly marks a directory output
    touch("flags/{sample}.done")             # creates empty file (for flag-based rules)
    ancient("reference/genome.fa")           # input modifier: ignore mtime, never re-trigger
    multiext("prefix", ".bed", ".bai")       # multiple outputs sharing a prefix
```

Avoid `directory()` except when necessary — it makes Snakemake's dependency tracking less precise. Prefer explicit file outputs.

---

## Conda Environments

Annotate each rule with a versioned conda environment:

```python
rule trim:
    conda: "envs/trimmomatic.yaml"
    shell: ...
```

Environment file (`workflow/envs/trimmomatic.yaml`):
```yaml
channels:
  - bioconda
  - conda-forge
dependencies:
  - trimmomatic =0.39
```

Activate with `--sdm conda` (Snakemake ≥8) or `--use-conda` (v7, deprecated).

Pin versions in env files. Unpinned environments produce irreproducible results.

---

## Containers

```python
rule trim:
    container: "docker://biocontainers/trimmomatic:0.39"
    shell: ...
```

Activate with `--sdm apptainer`. Combine conda + container for maximum reproducibility:

```bash
snakemake --sdm conda apptainer
```

---

## Wrappers

Prefer wrappers over hand-written rules for common tools. Wrappers are versioned and bundle their own conda environment:

```python
rule fastqc:
    input:
        "reads/{sample}.fastq.gz"
    output:
        html = "qc/{sample}_fastqc.html",
        zip  = "qc/{sample}_fastqc.zip"
    log:
        "logs/fastqc/{sample}.log"
    wrapper:
        "v9.4.2/bio/fastqc"
```

Browse: https://snakemake-wrappers.readthedocs.io/
Always pin to a specific version. Do not use the `master` branch — it changes.

---

## Scripts and Notebooks

```python
rule plot:
    input:
        counts = "counts/{sample}.tsv"
    output:
        figure = "figures/{sample}.png"
    script:
        "scripts/plot_counts.py"    # or .R, .Rmd, .ipynb
```

Inside the script, Snakemake injects a `snakemake` object:
```python
# scripts/plot_counts.py
import pandas as pd
df = pd.read_csv(snakemake.input.counts)
df.plot().get_figure().savefig(snakemake.output.figure)
```

In R: `snakemake@input$counts`, `snakemake@output$figure`.
Scripts run in the rule's conda environment. This is why `run:` blocks are avoided — they do not.

---

## Resources and Threads

```python
rule assemble:
    threads: 16
    resources:
        mem_mb        = 64000,
        runtime       = 480,         # minutes; used by SLURM executor
        cpus_per_task = 16,
        slurm_partition = "highmem"
    shell:
        r"""
        assembler --threads {threads} --memory 64g {input:q} -o {output:q}
        """
```

`threads` is available as `{threads}` in `shell:`. Set `threads` and `resources.cpus_per_task` consistently.

---

## Shadow Rules

Shadow directories isolate rule execution, preventing interference between wildcard instances:

```python
rule noisy_tool:
    shadow: "shallow"    # or "full" for complete isolation
    shell:
        r"""
        tool {input:q}    # writes tempfiles relative to shadow dir
        mv result.txt {output:q}
        """
```

Use when a tool writes temp files to the working directory by name (not to a specified path).

---

## Rule Inheritance and Modularization

Include additional rule files:
```python
include: "workflow/rules/align.smk"
include: "workflow/rules/call.smk"
```

Use module imports to reuse rules from external workflows:
```python
module external:
    snakefile: "https://github.com/org/workflow/raw/main/workflow/Snakefile"
    config: config

use rule * from external as ext_{name}
```

Override specific rules with `with:` clauses:
```python
use rule align from external as ext_align with:
    resources:
        mem_mb = 16000
```
