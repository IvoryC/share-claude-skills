---
name: build-q2-pliggins
description: Create a qiime2 pluggin for a new tool or as a wrapper to an existing tool to operate in the qiime2 ecosystem and track provenance.
version: 1.0.0
user-invocable: false
---

# How to Build QIIME 2 Plugins

Notes for Claude Code on building QIIME 2 plugins in this lab environment.

## Template

Start from the official template:
https://develop.qiime2.org/en/stable/plugins/tutorials/create-from-template.html

Reference plugin in this lab: `/projects/afodor_research2/ieclabau/privateGit/q2-bridges`

## Repo Structure

```
q2_PLUGINNAME/
├── __init__.py          # just version import, don't touch
├── _methods.py          # Python implementations of all actions
├── plugin_setup.py      # registers actions with QIIME 2 type system
├── citations.bib        # BibTeX for citations
└── tests/
    ├── __init__.py
    ├── test_methods.py
    └── data/            # small fixtures for unit tests
pyproject.toml           # entry point: q2_PLUGINNAME.plugin_setup:plugin
Makefile                 # make test / make dev / make lint
environment-files/       # conda env YAMLs
```

## Key Conventions

### Function signatures in `_methods.py`

- QIIME 2 calls your Python function with **format objects**, not semantic types.
- Return `pd.DataFrame` for `FeatureTable[Frequency]` — QIIME 2 transformers handle BIOM conversion.
- Return `DNASequencesDirectoryFormat` for `FeatureData[Sequence]`.
- QIIME 2 semantic type → Python format class mappings:

| Semantic type | Python format class |
|---|---|
| `SampleData[JoinedSequencesWithQuality]` | `SingleLanePerSampleSingleEndFastqDirFmt` |
| `SampleData[SequencesWithQuality]` | `SingleLanePerSampleSingleEndFastqDirFmt` |
| `FeatureTable[Frequency]` | `pd.DataFrame` (samples × features) |
| `FeatureData[Sequence]` | `DNASequencesDirectoryFormat` |
| `FeatureData[Taxonomy]` | `pd.DataFrame` (Feature ID index, Taxon + Confidence columns) |

### Writing `DNASequencesDirectoryFormat`

```python
from q2_types.feature_data import DNASequencesDirectoryFormat, DNAFASTAFormat

result = DNASequencesDirectoryFormat()
ff = DNAFASTAFormat()
shutil.copy(src_fasta_path, str(ff))          # or write content via ff.open()
result.file.write_data(ff, DNAFASTAFormat)
return result
```

### Reading from `SingleLanePerSampleSingleEndFastqDirFmt`

```python
manifest = sequences.manifest.view(pd.DataFrame)
for sample_id in manifest.index:
    src = manifest.loc[sample_id, 'forward']  # path to .fastq.gz
```

### `plugin_setup.py` imports

```python
from qiime2.plugin import Citations, Float, Int, Plugin, Range, TypeMatch
from q2_types.sample_data import SampleData
from q2_types.feature_table import FeatureTable, Frequency
from q2_types.feature_data import FeatureData, Sequence, Taxonomy
from q2_types.per_sample_sequences import (
    JoinedSequencesWithQuality, SequencesWithQuality
)
```

### Registering a method

```python
plugin.methods.register_function(
    function=my_func,
    inputs={'sequences': SampleData[JoinedSequencesWithQuality]},
    parameters={
        'threshold': Int % Range(1, None),
        'fdr': Float % Range(0.0, 1.0, inclusive_start=False, inclusive_end=False),
    },
    outputs=[
        ('table', FeatureTable[Frequency]),
        ('representative_sequences', FeatureData[Sequence]),
    ],
    input_descriptions={...},
    parameter_descriptions={...},
    output_descriptions={...},
    name='Human-readable name',
    description='Long description...',
    citations=[citations['MyKey']]
)
```

Use `TypeMatch` when a function accepts multiple related types:
```python
T = TypeMatch([SequencesWithQuality, JoinedSequencesWithQuality])
inputs={'sequences': SampleData[T]}
```

## Testing

- Use `TestPluginBase` from `qiime2.plugin.testing`.
- Set `package = 'q2_PLUGINNAME.tests'` so `self.get_data_path('file.txt')` resolves to `tests/data/`.
- Use `qiime2.plugin.util.transform` to convert fixture files to Python objects.
- Skip integration tests that need external tools:
  ```python
  @unittest.skipUnless(tool_available(), 'tool not installed')
  class MyIntegrationTests(TestPluginBase): ...
  ```

## Calling External Tools (R, Java, etc.)

```python
proc = subprocess.run(
    ['Rscript', '--vanilla', '-e', r_script],
    capture_output=True, text=True,
)
if proc.returncode != 0:
    raise RuntimeError(
        f'Tool failed (exit {proc.returncode}).\n'
        f'STDOUT:\n{proc.stdout}\nSTDERR:\n{proc.stderr}'
    )
```

- Name temp files `{sample_id}.fastq.gz` so output tables use the same sample IDs as QIIME 2.
- Construct all QIIME 2 format objects (e.g., `DNASequencesDirectoryFormat`) inside the `with tempfile.TemporaryDirectory()` block, before the cleanup.
- Return values can be used after the `with` block exits because `pd.DataFrame` is in-memory and QIIME 2 format objects manage their own storage.

## Development Workflow

```bash
conda activate q2-hashseq-dev   # or the relevant env
make dev                         # pip install -e .
make test                        # py.test
qiime dev refresh-cache          # make QIIME 2 CLI aware after changes
qiime hashseq --help
```

## Plugins in This Lab

| Plugin | Path | Purpose |
|--------|------|---------|
| q2-bridges | `/projects/afodor_research2/ieclabau/privateGit/q2-bridges` | Wraps Kraken2 → QIIME 2; shuffle sequences for negative controls |
| q2-hashseq | `/projects/afodor_research2/ieclabau/privateGit/q2-hashseq` | Wraps HashSeq R package for 16S ASV inference |

## Gotchas

- `FeatureTable[Frequency]` returned as `pd.DataFrame` must have **samples as rows**, features as columns. If your tool outputs features as rows, transpose with `df.T`.
- The QIIME 2 amplicon distribution (`qiime2-amplicon`) includes all the relevant types for 16S analysis. Use that as the base conda env.
- `citations.bib` key is the string used in `citations['Key']`. Keep keys unique.
- `JAVA_TOOL_OPTIONS=-Xmx4g` in the environment is the cleanest way to increase JVM heap when calling R packages that use rJava.
