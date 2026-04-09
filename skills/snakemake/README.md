# snakemake skill — source notes

This file documents where the information in this skill came from, how authoritative each source is, and example pipelines worth referencing. It is a companion to `SKILL.md` and is intended for humans maintaining or improving this skill.

Research performed April 2026 using live web fetches.

---

## Section 1: Official Documentation Sources

All official Snakemake documentation is versioned and hosted at **https://snakemake.readthedocs.io/en/stable/**. As of April 2026 this documents v9.19.0 (released March 28, 2026). This is the canonical source; everything else is supplementary.

| Page | URL | Notes |
|---|---|---|
| Main docs | https://snakemake.readthedocs.io/en/stable/ | Canonical. Versioned back to v6. |
| Tutorial (basics → advanced → reporting) | https://snakemake.readthedocs.io/en/stable/tutorial/tutorial.html | Developer-endorsed onboarding. RNA-seq alignment example. |
| **Best practices** (official) | https://snakemake.readthedocs.io/en/stable/snakefiles/best_practices.html | Only page officially labeled best practices. Covers folder structure, snakefmt, config conventions, wrappers, conda/container. |
| Rules reference | https://snakemake.readthedocs.io/en/stable/snakefiles/rules.html | All directives: input/output/params/log/benchmark/resources/threads/conda/container/shadow/cache/wrapper/checkpoint/etc. |
| Modularization | https://snakemake.readthedocs.io/en/stable/snakefiles/modularization.html | Four levels: wrappers, `include:`, `module:` (v6+), `use rule`. |
| Deployment / reproducibility | https://snakemake.readthedocs.io/en/stable/snakefiles/deployment.html | SDM (`--sdm conda/apptainer`), cache, archiving. |
| Migration guide | https://snakemake.readthedocs.io/en/stable/getting_started/migration.html | **Critical for v7→v8 and v8→v9 breaking changes.** |
| Changelog | https://snakemake.readthedocs.io/en/stable/project_info/history.html | Per-version change detail. |
| Plugin catalog | https://snakemake.github.io/snakemake-plugin-catalog/index.html | 21 executor plugins, 21 storage plugins, 7 logger plugins (as of April 2026). Needed for Snakemake ≥8. |
| Workflow catalog | https://snakemake.github.io/snakemake-workflow-catalog/ | Auto-discovered catalog of all public Snakemake workflows on GitHub. Two tiers: generic and standardized. |
| GitHub repo | https://github.com/snakemake/snakemake | 2,700+ stars, 633 forks, MIT license, NumFOCUS affiliated. |
| Citations | https://snakemake.readthedocs.io/en/stable/project_info/citations.html | Primary: Mölder et al. F1000Res 2021 (~3,000 citations). Secondary: Köster & Rahmann Bioinformatics 2012. |

### Key breaking changes (v8 and v9)

**v8.0** was a major breaking release:
- `snakemake()` Python API removed; replaced with dataclass-based API
- All cluster/cloud backends moved to executor plugins (`--kubernetes` → `--executor kubernetes`)
- Remote providers (S3RemoteProvider etc.) removed; replaced by storage plugins
- `dynamic()` output removed → use `checkpoint`
- `subworkflow:` directive removed
- `--use-conda` / `--use-singularity` deprecated → `--sdm conda` / `--sdm apptainer`

**v9.0** had only one breaking change: custom loggers must now be specified as logger plugins via `--logger`.

---

## Section 2: Best Practices and Conventions

This section is split into two subsections:

- **2a — Core team guidance:** Best practices, tooling, and standards published or maintained by the Snakemake developers themselves. These are authoritative for the ecosystem.
- **2b — Third-party conventions:** Style guides and conventions published by other groups who use Snakemake heavily in production. These reflect hard-won experience but may be shaped by domain-specific or organizational concerns. Read them with that context in mind.

---

## Section 2a: Core Team Guidance

### snakemake-workflows GitHub organization
**https://github.com/snakemake-workflows**

Semi-official. Maintained by Johannes Köster (Snakemake's creator) and collaborators. 30 repositories. Directly linked from the official docs as example workflows. Enforces:
- One workflow per repository
- Must use the [snakemake-workflow-template](https://github.com/snakemake-workflows/snakemake-workflow-template) (126 stars) layout
- Well-documented YAML config with `config/README.md`
- Mandatory GitHub Actions CI with `.test/` data
- Wrappers wherever feasible
- MIT licensing
- Strict review before a workflow is accepted

The template enforces this layout (required for Snakemake ≥8):
```
.github/workflows/
.test/config/
config/
profiles/
workflow/
    Snakefile
    rules/
    envs/
    scripts/
.snakemake-workflow-catalog.yml
CHANGELOG.md
README.md
LICENSE
```

### Snakemake workflow catalog — standardized tier
**https://snakemake.github.io/snakemake-workflow-catalog/docs/catalog.html**

Official infrastructure. Maintained by the Snakemake team. Two inclusion tiers:

**Generic** (auto-discovered): public repo + README mentioning "snakemake" and "workflow" + `Snakefile` or `workflow/Snakefile` present.

**Standardized** (higher tier): must have `workflow/Snakefile`, `config/README.md`, and `.snakemake-workflow-catalog.yml` declaring supported deployment methods (conda, apptainer, or both). Catalog displays snakefmt status, linter status, tube map (if `.test/` data provided), and config schema.

### snakemake-wrappers repository
**https://github.com/snakemake/snakemake-wrappers** | **https://snakemake-wrappers.readthedocs.io/**

241 stars, 209 forks, 272 releases, latest v9.4.2 (April 2026). Official. Versioned wrappers for common bioinformatics tools — each wrapper bundles its conda environment, wrapper script, and test data. Referenced in rules as:
```python
wrapper: "v9.4.2/bio/bwa/mem"
```
Using wrappers pins tool versions and makes reproducibility automatic. Official best practices say to use them wherever possible and to contribute new ones.

### snakefmt (official formatter)
**https://github.com/snakemake/snakefmt** — 188 stars, 31 forks.

Built on Black. Required by best practices before publishing any workflow. Reads config from `pyproject.toml`. Integrates with GitHub Actions via super-linter. Directive sorting is default behavior.

### Mölder et al. 2021 — "Sustainable data analysis with Snakemake"
**https://f1000research.com/articles/10-33** | PMC: https://pmc.ncbi.nlm.nih.gov/articles/PMC8114187/

~3,000 citations, 12–14 new per week. A "rolling paper" updated as Snakemake adds features (latest update September 2025). This is the recommended citation for all publications using Snakemake and also the most comprehensive articulation of the design philosophy and best practices. Primary reference for the skill.

### Snakemake SLURM executor (v8+)
For Snakemake ≥8, cluster submission is via the official plugin: `--executor slurm`. Documented at the plugin catalog. The old community profile at https://github.com/Snakemake-Profiles/slurm is on hold for v8+ support. For v8+, also see: https://github.com/jdblischak/smk-simple-slurm for a simpler profile approach.

### Carpentries Incubator tutorial
**https://carpentries-incubator.github.io/snakemake-novice-bioinformatics/**

Community training materials. Not officially endorsed by The Carpentries but widely used. Covers basics through cluster execution using an RNA-seq example. Useful for beginners; not a reference for production best practices.

---

## Section 2b: Third-Party Conventions

Style guides and conventions from groups using Snakemake heavily in production. These are distinct from official docs and distinct from example pipelines — they represent deliberate, written-down opinions about how to write Snakemake workflows, born from real experience. Apply with awareness of each group's domain and architectural context.

### Nextstrain Snakemake Style Guide
**https://docs.nextstrain.org/en/latest/reference/snakemake-style-guide.html**

Source file on disk: `references/snakemake-style-guide.rst.txt`

**Published/maintained:** Actively maintained by the Nextstrain team. Nextstrain upgraded their own pipelines from Snakemake v7 to v9 in July 2025, so the guide reflects current practice. Not endorsed by the Snakemake developers, but Nextstrain is a high-profile pathogen surveillance platform with thousands of pipeline users.

**Use with judgment:** Some advice is universally applicable; some is motivated by Nextstrain's specific ncov architecture (e.g., the "literal input paths" rule exists partly to support their custom rule injection system). Apply the shell-level coding practices universally; apply the structural/architectural advice with awareness of context.

**What it covers (from the actual source file):**

- **Avoid `run:` blocks** — use external scripts called from `shell:` instead. `run:` blocks don't execute in the rule's conda environment, are harder to debug, and can't be reused.
- **Literal input paths** — define `input:` with literal strings rather than referencing rule output variables; enables workflow rewiring via custom rule injection. (Nextstrain-specific motivation — use with judgment.)
- **Relative paths only** — never use absolute paths; see Snakemake docs for how relative paths are interpreted in different contexts.
- **Avoid the `message:` attribute** — it suppresses default Snakemake output (job id, rule name, input/output paths).
- **Config in YAML files** — use `configfile: "defaults.yaml"` for defaults; allow overrides via `--configfile` / `--config`.
- **Access `config` correctly** — use `config[key]` for required keys, `key in config` to check optional keys, `config.get(key, default)` for optional keys with defaults. **Never use `config.get(key)` without a default** — it silently masks missing required keys.
- **Use `params:` to map config into shell** — don't interpolate `config` directly in shell commands. `params:` supports arbitrary Python expressions and enables `--list-params-changes` detection.
- **Use lambda on `params` with curly braces** — if a config value may contain `{` or `}`, use `lambda w: config["key"]` to prevent Snakemake from treating it as a wildcard.
- **Always use `:q` quoting** — `{params.file:q}` quotes shell interpolations; without it, values with spaces or special characters will break. Also a security consideration. For multi-value params, use a list — Snakemake interpolates it correctly with `:q`.
- **Raw triple-quoted shell blocks** — use `r"""..."""` for all shell blocks; enables one-option-per-line formatting, avoids nested quoting issues, and preserves backslash/newline formatting in logs.
- **Log with `tee` and `exec`** — canonical pattern:
  ```python
  log: "logs/rulename.txt"
  shell:
      r"""
      exec &> >(tee {log:q})
      tool --input {input:q} --output {output:q}
      """
  ```
  `exec` redirects the entire shell block; `tee` writes to both log and terminal.
- **Always use `benchmark:`** — for every rule; tracks runtime and memory without parsing logs.
- **Run with `--show-failed-logs`** — prints failed job logs to terminal on exit; saves hunting for the log file.

**Note:** The Nextstrain ncov pipeline (`https://github.com/nextstrain/ncov`) is a real-world example of these practices at scale.

---

### NBIS / SciLifeLab Reproducible Research Workshop — Snakemake section
**https://nbisweden.github.io/workshop-reproducible-research/pages/snakemake.html**

**Publisher:** NBIS (National Bioinformatics Infrastructure Sweden) / SciLifeLab — a nationally funded infrastructure organization serving all Swedish academic bioinformatics. Course runs twice yearly and is actively updated.

**Classification:** Workshop tutorial with deliberate opinionated content. Not a standalone style guide, but more prescriptive than average tutorials. Probably the best complement to Nextstrain found in this search.

**Specific practices — what it adds beyond Nextstrain and official docs:**
- **Filename-as-provenance convention** — output filenames should reflect their processing history (e.g., `sample_a.trimmed.deduplicated.sorted.bam`), so the path itself documents what was done
- **Downstream rule determines wildcards** — explicit design principle: work backward from what you want, letting the final target drive wildcard inference up the DAG
- **Docstrings for rules** — document what each rule does in a comment block
- **`snakemake --lint` and snakefmt as mandatory pre-publish steps** (reinforces official guidance)

---

### SIB Swiss Institute of Bioinformatics — Containers and Snakemake Training
**https://sib-swiss.github.io/containers-snakemake-training/latest/**

**Publisher:** SIB (Swiss Institute of Bioinformatics). Course runs annually; actively maintained.

**Classification:** Training course materials. The debugging and design chapters contain explicit opinionated guidance.

**Specific practices — what it adds:**
- **Avoid `directory()` except as last resort** — more specific anti-pattern than official docs, which describe `directory()` neutrally
- **`temp()` and `protected()` usage guidance** — flag intermediaries as `temp()` to save disk; use `protected()` for expensive results that should not be accidentally deleted
- **Three-phase execution mental model for debugging** — parse phase → DAG phase → execute phase; knowing which phase an error comes from narrows the fix

---

### C. Titus Brown — 2023 Snakemake Book Draft
**https://ngs-docs.github.io/2023-snakemake-book-draft/**

**Publisher:** C. Titus Brown and contributors, UC Davis DIB Lab. Draft as of 2023; not a finished reference. Linked from Snakemake's own "more resources" page.

**Classification:** Comprehensive tutorial book. Not a style guide — no dedicated conventions section — but Section 4 ("Snakemake Patterns and Recipes") is the closest freely available resource to an advanced pattern catalog.

**Use for:** Looking up how to implement a specific pattern (checkpoints, dynamic inputs, parameter spaces) rather than as a source of conventions.

---

### Tessa Pierce Ward (UC Davis DIB Lab) — HPC Snakemake Tips
**https://bluegenes.github.io/hpc-snakemake-tips/**

**Publisher:** Tessa Pierce Ward, UC Davis. Published August 2020; not recently updated.

**Classification:** Short blog post, HPC-focused. Narrow scope but covers two practices not found elsewhere:
- **Adaptive memory scaling by retry** — `resources: mem_mb = lambda wildcards, attempt: attempt * 4000` automatically increases memory on retry without manual intervention
- **Cluster resource etiquette** — "use no more than 1/3 of shared partition resources" as an explicit convention for shared HPC environments

---

### "Ten Simple Rules for Creating Workflow-as-Applications"
**https://pmc.ncbi.nlm.nih.gov/articles/PMC9754251/**

Roach et al., *PLOS Computational Biology*, December 2022. Peer-reviewed. From Flinders University / UC Davis / CSIRO.

**Classification:** Paper with Cookiecutter templates. Not a Snakemake style guide per se — addresses how to wrap Snakemake workflows into user-friendly applications. Worth noting because it fills a gap Nextstrain doesn't touch: the workflow-as-product pattern (config stacking, staged execution, real-time feedback, Bioconda distribution, HPC profiles, power-user escape hatches).

---

### Notable null results

Searched and confirmed absent — no published Snakemake style guides from:
- **Broad Institute** — uses WDL/Terra as primary workflow system; minimal Snakemake footprint
- **EMBL/EBI** — has internal Snakemake templates (git.embl.de) but not publicly documented
- **Wellcome Sanger** — workflows in WorkflowHub but no conventions guide
- **Max Planck institutes** — snakePipes has user documentation but no separate style guide
- **DKFZ** — key authors of the F1000Research paper, but no separate style guide
- No nf-core equivalent exists for Snakemake; the `snakemake-workflows` org is the closest structural analog but enforces standards through templates and review gates rather than prose

---

### Adjacent: empirical study of what users actually do
**https://dl.acm.org/doi/10.1145/3676288.3676290**

"How Do Users Design Scientific Workflows? The Case of Snakemake and Nextflow." ACM SSDBM 2024. Empirical analysis of 1,602 Snakemake repositories from the workflow catalog. Useful as a counterpoint to conventions documents — shows the gap between stated best practices and real-world adoption patterns.

---

## Section 3: Example Pipelines

### Highlighted pipelines

These are worth reading carefully — officially endorsed, highly cited, or excellent structural examples.

---

**rna-seq-star-deseq2** — https://github.com/snakemake-workflows/rna-seq-star-deseq2

354 stars, 209 forks. Latest release v3.1.1 (December 2025). Highest-starred standardized workflow in the catalog. Differential expression via STAR + DESeq2. Part of the official `snakemake-workflows` organization. **The canonical example of the snakemake-workflow-template applied correctly.** Wrappers throughout, full CI, `.snakemake-workflow-catalog.yml`, conda per rule. Zenodo DOI: 10.5281/zenodo.4737358.

---

**dna-seq-gatk-variant-calling** — https://github.com/snakemake-workflows/dna-seq-gatk-variant-calling

262 stars, 148 forks. GATK best-practices germline variant calling. Official `snakemake-workflows` member. **Excellent structural example of 100% wrapper-driven rules** — every rule calls a wrapper; the Snakefile itself contains almost no tool invocations. Zenodo DOI: 10.5281/zenodo.4675661. Warning: latest release May 2021 — check whether updated for GATK v4.4+ and Snakemake v8+.

---

**ATLAS (metagenome-atlas)** — https://github.com/metagenome-atlas/atlas

404 stars, 98 forks, 2,824 commits, v2.19.0 (July 2024). Complete metagenomics: QC, assembly, binning, taxonomic + functional annotation. Published: Kieser et al. *BMC Bioinformatics* 21:257 (2020), DOI: 10.1186/s12859-020-03585-4. **Production-grade large-scale workflow with ReadTheDocs documentation.** Conda per rule, wildcard-based multi-sample handling, resource specs for HPC, Bioconda installable. Good model for complex multi-step pipelines.

---

**dna-seq-varlociraptor** — https://github.com/snakemake-workflows/dna-seq-varlociraptor

89 stars, 47 forks, 130 releases (very active), latest v6.6.1 (March 2026). Unified variant calling for any scenario (tumor/normal, germline, pedigree, population) via Varlociraptor's statistical model. **Best example of handling multiple analysis scenarios in a single configurable workflow.** Highly modular. The 130-release cadence signals serious production use.

---

**metaGEM** — https://github.com/franciscozorrilla/metaGEM

257 stars, 46 forks. Reconstructs genome-scale metabolic models from metagenomes; predicts microbial community metabolic interactions. Published: Zorrilla et al. *Nucleic Acids Research* (2021), DOI: 10.1093/nar/gkab815. **Good example of wrapping multiple complex external tools cohesively**, with wiki documentation and tutorials. Bioconda installable.

---

### General pipeline list

Use this list when asking "how do most people handle X in Snakemake pipelines?" — a starting point for further investigation, not vetted recommendations.

| Pipeline | URL | What it does | Annotation |
|---|---|---|---|
| rna-seq-kallisto-sleuth | https://github.com/snakemake-workflows/rna-seq-kallisto-sleuth | RNA-seq DE via Kallisto + Sleuth | Good. Official snakemake-workflows member, standard template, actively maintained (v4.0.0 Dec 2025). |
| single-cell-rna-seq | https://github.com/snakemake-workflows/single-cell-rna-seq | scRNA-seq analysis | **DATED** — last release 2019. Has official org imprimatur but does not reflect current scRNA-seq tools or Snakemake v8+. Reference only. |
| V-pipe | https://github.com/cbg-ethz/V-pipe | Viral NGS: consensus, SNV calls, haplotypes | Good. Active maintenance, SARS-CoV-2 surveillance use, Docker + conda. Published bioRxiv 2023. |
| chipseq | https://github.com/snakemake-workflows/chipseq | ChIP-seq peak calling, QC, differential analysis | Moderate. Standard template layout. Port of nf-core/chipseq — useful comparison point. No publication. |
| SnakePipes | https://github.com/maxplanck-ie/snakepipes | Suite of 9 NGS workflows (ChIP, ATAC, RNA, scRNA, HiC, WGBS) | Good institutional pipeline. Published Bioinformatics 2019. Uses `rules/` at top level (pre-2021 layout) — does not meet standardized catalog criteria but widely functional. |
| ATAC-seq pipeline (epigen) | https://github.com/epigen/atacseq_pipeline | ATAC-seq processing through peak annotation | Good. Snakemake 8 compliant. Part of the MrBiomics modular ecosystem. Used in multiple 2025 publications. |
| Tourmaline | https://github.com/aomlomics/tourmaline | Amplicon (16S/18S/ITS) via QIIME 2 + Snakemake | Moderate. Peer-reviewed (GigaScience 2022). Requires QIIME 2 infrastructure. |
| ont-assembly-snake | https://github.com/pmenzel/ont-assembly-snake | Bacterial genome assembly from ONT ± Illumina reads | Good. Unusual pattern: users specify pipeline by naming folders (e.g., `filtlong+flye+racon+medaka`). Published GigaByte 2024. |
| Nanopype | https://github.com/giesselmann/nanopype | ONT basecalling, alignment, downstream | **ARCHIVED** March 2023. Historical reference for ONT Snakemake patterns only. |
| sv-callers | https://github.com/GooglingTheCancerGenome/sv-callers | Structural variant calling in cancer genomes | Good. Catalog-registered standardized workflow. |
| grenepipe | https://github.com/moiexpositoalonsolab/grenepipe | Flexible variant calling, configurable aligner+caller | Good. Catalog-registered, 102 stars. |
| enrichment_analysis (epigen) | https://github.com/epigen/enrichment_analysis | Genomic region + gene set enrichment (LOLA, GREAT, GSEApy) | Good. Part of the coordinated epigen/MrBiomics modular workflow suite. |

---

## Deprecated patterns to watch for

When reading older community pipelines:
- `dynamic()` output → removed in v8, use `checkpoint`
- `subworkflow:` directive → removed in v8
- `--use-conda` / `--use-singularity` → deprecated, now `--sdm conda/apptainer`
- Remote providers (`S3RemoteProvider` etc.) → removed in v8, now storage plugins
- Flat `Snakefile` + `rules/` at repo root → pre-2021 convention, valid but won't achieve standardized catalog status
