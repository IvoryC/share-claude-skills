# Snakemake Project Structure

## Standard Directory Layout

This layout is required for the Snakemake workflow catalog "standardized" tier and is enforced by the `snakemake-workflows` GitHub organization template:

```
workflow/
    Snakefile               ← main entry point (required for standardized catalog)
    rules/
        align.smk
        call.smk
    envs/
        bwa.yaml
        gatk.yaml
    scripts/
        plot.py
        filter.R
    notebooks/
        explore.py.ipynb
    report/
        template.rst        ← Snakemake report templates
config/
    config.yaml             ← default configuration
    README.md               ← documents all config keys (required for catalog)
    schemas/
        config.schema.yaml  ← JSON Schema validation for config
        samples.schema.yaml
results/                    ← generated outputs (gitignore this)
resources/                  ← downloaded reference files (gitignore this)
logs/                       ← rule log files (gitignore this)
benchmarks/                 ← benchmark TSVs (gitignore this)
.test/
    config/
        config.yaml         ← minimal test config
        samples.tsv         ← minimal test sample sheet
.snakemake-workflow-catalog.yml   ← required for standardized tier
.github/
    workflows/
        main.yml            ← CI (runs --dry-run + test on push)
README.md
CHANGELOG.md
LICENSE
```

The `workflow/` subdirectory is the key structural decision. Without it (`Snakefile` at repo root), the workflow can still run but will not qualify for the catalog's "standardized" tier.

---

## .snakemake-workflow-catalog.yml

Required for catalog standardized tier. Declares what the workflow supports:

```yaml
name: My Workflow
description: One-sentence description.
authors:
  - name: Your Name
    github: yourhandle

usage:
  mandatory:
    - --sdm conda
  optional:
    - --sdm apptainer

software-stack-deployment:
  conda: true
  apptainer: true
  conda+apptainer: true

report: true    # set true if workflow generates an HTML report
```

---

## Formatting and Linting

Run both before committing any Snakefile:

```bash
# Format (built on Black)
snakefmt workflow/Snakefile workflow/rules/*.smk

# Lint (built-in; checks best practice violations)
snakemake --lint -s workflow/Snakefile
```

Configure snakefmt in `pyproject.toml`:
```toml
[tool.snakefmt]
line_length = 99
include = "\.smk$|^Snakefile$"
```

snakefmt integrates with GitHub Actions via super-linter and pre-commit hooks.

---

## Config Documentation (config/README.md)

Document every config key. This file is parsed by the catalog to generate per-workflow deployment instructions. Minimum viable content:

```markdown
# Configuration

## samples

Path to the sample sheet TSV. Required columns: `sample`, `unit`, `fq1`, `fq2`.

## reference

Path to the reference genome FASTA. Must be indexed with `bwa index` before running.

## trimming

| Key | Description | Default |
|-----|-------------|---------|
| `activate` | Enable adapter trimming | `true` |
| `adapters` | Adapter sequence | `AGATCGGAAGAGC` |
| `min_length` | Discard reads shorter than this | `36` |
```

---

## Config Validation with JSON Schema

Validate config at workflow start to catch misconfiguration early:

```python
# workflow/Snakefile
configfile: "config/config.yaml"

from snakemake.utils import validate
validate(config, schema="config/schemas/config.schema.yaml")
```

Schema example (`config/schemas/config.schema.yaml`):
```yaml
$schema: "http://json-schema.org/draft-07/schema#"
properties:
  samples:
    type: string
  reference:
    type: string
  trimming:
    type: object
    properties:
      min_length:
        type: integer
        minimum: 1
required:
  - samples
  - reference
```

---

## CI with GitHub Actions

Minimal `.github/workflows/main.yml`:

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: conda-incubator/setup-miniconda@v3
        with:
          miniforge-version: latest
          activate-environment: snakemake
          environment-file: .test/config/test-env.yaml

      - name: Lint
        run: snakemake --lint -s workflow/Snakefile

      - name: Dry-run
        run: snakemake -n --configfile .test/config/config.yaml

      - name: Test
        run: |
          snakemake --sdm conda \
            --configfile .test/config/config.yaml \
            --cores 2
```

The `.test/` directory must contain minimal data that completes in a few minutes. This enables the catalog to display a tube map (rulegraph visualization).

---

## .gitignore

```gitignore
# Snakemake runtime
.snakemake/
__pycache__/

# Generated outputs
results/
logs/
benchmarks/

# Downloaded resources (re-downloadable)
resources/

# Large reference files
*.fa
*.fasta
*.fastq
*.fastq.gz
```

Commit `config/`, `workflow/`, `.github/`, `.test/config/`, `.snakemake-workflow-catalog.yml`, `README.md`, `CHANGELOG.md`, `LICENSE`. Do not commit `results/`, `logs/`, or `resources/`.

---

## HTML Reports

Generate an HTML report after a successful run:

```bash
snakemake --report report.html
```

Add report captions to rules:

```python
rule align:
    input: ...
    output:
        report("results/{sample}.bam", caption="report/align.rst", category="Alignment")
```

---

## Workflow Catalog Submission

**Generic tier** (auto-discovered): public GitHub repo + README mentioning "snakemake" and "workflow" + `Snakefile` or `workflow/Snakefile` present.

**Standardized tier** (displayed separately, higher visibility): additionally requires:
- `workflow/Snakefile` (not just `Snakefile` at root)
- `config/README.md`
- `.snakemake-workflow-catalog.yml`
- Passing snakefmt check
- Passing snakemake --lint check
- `.test/` data for tube map generation (not strictly required but enables key catalog features)

Catalog URL: https://snakemake.github.io/snakemake-workflow-catalog/
