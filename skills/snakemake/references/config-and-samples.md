# Snakemake Configuration and Sample Sheets

## Config File Basics

Declare the config file at the top of the Snakefile. Snakemake merges it into the global `config` dict:

```python
configfile: "config/config.yaml"
```

Multiple config files are merged in order (later files override earlier ones):
```python
configfile: "config/defaults.yaml"
configfile: "config/project.yaml"
```

Override individual values at the command line:
```bash
snakemake --config min_length=50 reference=hg38.fa
snakemake --configfile custom.yaml    # merge an additional file
```

---

## Config Access Patterns

Four access patterns exist; two should be used, one is forbidden:

```python
# Required key — raises KeyError clearly if missing
value = config["key"]

# Check if optional key is present
if "key" in config:
    ...

# Optional key with explicit default
value = config.get("key", "default_value")

# NEVER use this — silently returns None, masking missing keys
value = config.get("key")   # ← forbidden
```

---

## Mapping Config into Shell via params:

Never interpolate `config` directly in `shell:` commands. Map through `params:` first:

```python
rule trim:
    params:
        adapter    = lambda _: config["trimming"]["adapter"],
        min_length = lambda _: config["trimming"]["min_length"],
        extra      = lambda _: config.get("trimming_extra", "")
    shell:
        r"""
        trimmomatic SE {input:q} {output:q} \
            ILLUMINACLIP:{params.adapter:q}:2:30:10 \
            MINLEN:{params.min_length} \
            {params.extra}
        """
```

Always use `lambda _:` when the param value may contain `{` or `}` — prevents Snakemake from attempting wildcard resolution on the value.

Benefits over direct config interpolation:
- Enables `--list-params-changes` (shows which output files are affected by config changes)
- Supports arbitrary Python expressions (list comprehensions, conditionals)
- Avoids the non-standard `{config[key]}` dict interpolation syntax

---

## Typical config.yaml Structure

```yaml
# Sample sheet
samples: config/samples.tsv

# Reference genome
reference: resources/genome.fa

# Trimming
trimming:
  activate: true
  adapter: "AGATCGGAAGAGC"
  min_length: 36
  min_quality: 20

# Alignment
aligner: bwa   # or bwa2, bowtie2

# Calling
calling:
  min_depth: 10
  min_qual: 30

# Resource hints (optional; can also be set per-rule)
default_mem_mb: 8000
default_runtime: 120
```

---

## Sample Sheets

### Minimal TSV approach

```python
# workflow/Snakefile
import pandas as pd

samples = pd.read_csv(config["samples"], sep="\t", dtype=str).set_index("sample", drop=False)

def get_reads(wildcards):
    row = samples.loc[wildcards.sample]
    return {"r1": row.fq1, "r2": row.fq2}
```

Sample sheet (`config/samples.tsv`):
```
sample  unit    fq1                     fq2
A       lane1   data/A_L1_R1.fastq.gz   data/A_L1_R2.fastq.gz
A       lane2   data/A_L2_R1.fastq.gz   data/A_L2_R2.fastq.gz
B       lane1   data/B_L1_R1.fastq.gz   data/B_L1_R2.fastq.gz
```

Validate the sample sheet against a schema at startup:
```python
from snakemake.utils import validate
validate(samples, schema="config/schemas/samples.schema.yaml")
```

### Glob-based sample discovery

For simpler cases where files follow a naming convention:

```python
from glob import glob
SAMPLES = [os.path.basename(f).replace("_R1.fastq.gz", "")
           for f in glob("reads/*_R1.fastq.gz")]
```

### PEP (Portable Encapsulated Projects)

For more complex sample metadata, use PEP (https://pep.databio.org/):

```python
import peppy
pep = peppy.Project("config/pep/project_config.yaml")
samples = pep.sample_table
```

PEPs support subsetting, amendments, and derived columns — useful when the same sample sheet is shared across multiple workflows.

---

## Conditional Workflow Steps

Use config flags to enable/disable steps:

```python
rule all:
    input:
        expand("results/{sample}.bam", sample=samples.index),
        expand("qc/{sample}_fastqc.html", sample=samples.index) if config["qc"]["activate"] else []
```

Or at the rule level:
```python
rule trim:
    input:  lambda wildcards: get_raw_reads(wildcards) if not config["trimming"]["activate"]
                              else get_trimmed_reads(wildcards)
```

---

## Paramspace (Grid Searches)

For running analyses across parameter combinations:

```python
from snakemake.utils import Paramspace
import pandas as pd

params = Paramspace(pd.read_csv("config/params.tsv", sep="\t"))

rule run_analysis:
    input: "data/{sample}.tsv"
    output: f"results/{params.wildcard_pattern}/{{sample}}.tsv"
    params: **params.instance
    shell: r"""
        tool --param1 {params.param1} --param2 {params.param2} \
            {input:q} {output:q}
        """
```

---

## Config Schema Validation

Full schema example for the config above:

```yaml
# config/schemas/config.schema.yaml
$schema: "http://json-schema.org/draft-07/schema#"
description: "Workflow configuration schema"

properties:
  samples:
    type: string
    description: "Path to samples TSV"
  reference:
    type: string
    description: "Path to reference FASTA"
  trimming:
    type: object
    properties:
      activate:
        type: boolean
      adapter:
        type: string
      min_length:
        type: integer
        minimum: 1
    required: [activate, adapter, min_length]
  aligner:
    type: string
    enum: [bwa, bwa2, bowtie2]

required: [samples, reference]
additionalProperties: false
```

Activate in the Snakefile:
```python
configfile: "config/config.yaml"
from snakemake.utils import validate
validate(config, schema="config/schemas/config.schema.yaml")
```

Schema validation runs during the DAG phase, before any jobs execute — config errors surface immediately.
