# QIIME 2 Type System

> Official references:
> - [Semantic types, data types, file formats, and artifact classes](https://develop.qiime2.org/en/latest/plugins/explanations/types-of-types.html)
> - [Types API reference](https://develop.qiime2.org/en/stable/plugins/references/api/types.html)
> - [Semantic types (user-facing list)](https://docs.qiime2.org/2024.10/semantic-types/)

QIIME 2 distinguishes two fundamental kinds of types:

| Kind | Used for | Examples |
|---|---|---|
| **Semantic types** | Artifact inputs/outputs | `FeatureTable[Frequency]`, `SampleData[SequencesWithQuality]` |
| **Primitive types** | Parameters (non-artifact values) | `Int`, `Float`, `Str`, `Bool`, `Metadata` |

---

## Semantic Types

Semantic types describe *what* data represents, independently of how it is stored. They appear in `inputs` and `outputs` of `register_function` calls.

### Common Semantic Types → Python Format Classes

When QIIME 2 calls your function it passes **format objects** (or Python objects via registered transformers), not semantic types directly. The mapping:

| Semantic Type | Typical Python type / format class | Notes |
|---|---|---|
| `FeatureTable[Frequency]` | `pd.DataFrame` | samples × features, integer counts; return `df` directly |
| `FeatureTable[RelativeFrequency]` | `pd.DataFrame` | samples × features, float; values sum to 1.0 per sample |
| `FeatureTable[PresenceAbsence]` | `pd.DataFrame` | boolean/integer 0/1 |
| `FeatureData[Sequence]` | `DNASequencesDirectoryFormat` | unaligned representative sequences |
| `FeatureData[AlignedSequence]` | `DNASequencesDirectoryFormat` (aligned) | multiple sequence alignment |
| `FeatureData[Taxonomy]` | `pd.DataFrame` | index=Feature ID; cols: Taxon, Confidence |
| `SampleData[SequencesWithQuality]` | `SingleLanePerSampleSingleEndFastqDirFmt` | demuxed single-end FASTQ |
| `SampleData[JoinedSequencesWithQuality]` | `SingleLanePerSampleSingleEndFastqDirFmt` | joined paired-end FASTQ |
| `SampleData[PairedEndSequencesWithQuality]` | `SingleLanePerSamplePairedEndFastqDirFmt` | paired-end FASTQ |
| `SampleData[AlphaDiversity]` | `pd.Series` | alpha diversity per sample |
| `Phylogeny[Unrooted]` | `NewickFormat` | phylogenetic tree |
| `Phylogeny[Rooted]` | `NewickFormat` | rooted phylogenetic tree |
| `DistanceMatrix` | `skbio.DistanceMatrix` | symmetric distance matrix |
| `PCoAResults` | `skbio.OrdinationResults` | PCoA scores |
| `TaxonomicClassifier` | classifier-specific format | trained classifier object |

> **Note:** The full list evolves with each QIIME 2 release. Always check the [semantic types page](https://docs.qiime2.org/2024.10/semantic-types/) and the type definitions in the relevant plugin packages (e.g., `q2-types`, `q2-types-genomics`).

### Constructing Format Objects

**`DNASequencesDirectoryFormat`** (for `FeatureData[Sequence]`):

```python
from q2_types.feature_data import DNASequencesDirectoryFormat, DNAFASTAFormat
import shutil

def my_method(...) -> DNASequencesDirectoryFormat:
    result = DNASequencesDirectoryFormat()
    ff = DNAFASTAFormat()
    shutil.copy(src_fasta_path, str(ff))
    result.file.write_data(ff, DNAFASTAFormat)
    return result
```

**Reading from `SingleLanePerSampleSingleEndFastqDirFmt`** (for `SampleData[...]`):

```python
def my_method(sequences, ...):
    manifest = sequences.manifest.view(pd.DataFrame)
    for sample_id in manifest.index:
        fastq_path = manifest.loc[sample_id, 'forward']  # path to .fastq.gz
        # process fastq_path ...
```

**`FeatureTable[Frequency]` as `pd.DataFrame`:**

```python
def my_method(table: pd.DataFrame, ...) -> pd.DataFrame:
    # table: rows=samples, cols=features
    # Must return same orientation
    result = table.copy()
    ...
    return result  # QIIME 2 handles BIOM conversion
```

If your tool returns features as rows, transpose: `return result.T`

---

## Primitive Types

Used in `parameters` dict of `register_function`. These map to Python scalars.

| QIIME 2 type | Python type | Notes |
|---|---|---|
| `Int` | `int` | |
| `Float` | `float` | 64-bit |
| `Str` | `str` | unicode |
| `Bool` | `bool` | |
| `Metadata` | `qiime2.Metadata` | tabular metadata (samples or features) |
| `MetadataColumn[Categorical]` | `qiime2.CategoricalMetadataColumn` | single column, categorical |
| `MetadataColumn[Numeric]` | `qiime2.NumericMetadataColumn` | single column, numeric |

### Predicates (constraints on primitive types)

```python
from qiime2.plugin import Int, Float, Str, Bool, Range, Choices

# Integer, must be >= 1
Int % Range(1, None)

# Float in [0.0, 1.0], both ends inclusive
Float % Range(0.0, 1.0)

# Float in (0.0, 1.0), ends exclusive
Float % Range(0.0, 1.0, inclusive_start=False, inclusive_end=False)

# Str restricted to a fixed set of values
Str % Choices(['option_a', 'option_b', 'option_c'])
```

### Metadata in Function Signatures

```python
import qiime2

def my_method(table, metadata: qiime2.Metadata, group_column: str):
    df = metadata.to_dataframe()
    # or get a single column:
    col = metadata.get_column('my_column')  # returns MetadataColumn
    series = col.to_series()
```

In `plugin_setup.py`:
```python
from qiime2.plugin import Metadata, MetadataColumn, Categorical, Numeric

parameters={
    'metadata': Metadata,
    'group_column': Str,
}
```

---

## TypeMatch (Multiple Accepted Types)

When a function should accept several related semantic types:

```python
from qiime2.plugin import TypeMatch
from q2_types.per_sample_sequences import SequencesWithQuality, JoinedSequencesWithQuality
from q2_types.sample_data import SampleData

T = TypeMatch([SequencesWithQuality, JoinedSequencesWithQuality])

plugin.methods.register_function(
    function=my_method,
    inputs={'sequences': SampleData[T]},
    outputs=[('result', SampleData[T])],
    ...
)
```

---

## Artifact Collections

Inputs or outputs can be collections (lists or dicts) of artifacts:

```python
from qiime2.plugin import List, Collection

inputs={'tables': List[FeatureTable[Frequency]]}
# or keyed:
inputs={'tables': Collection[FeatureTable[Frequency]]}
```

See [Use Artifact Collections as Action inputs or outputs](https://develop.qiime2.org/en/latest/plugins/how-to-guides/artifact-collections-as-io.html).

---

## Defining Custom Artifact Classes

If you need to store a new kind of data, you define a format and register it as a new artifact class. This is an advanced topic — start from the tutorial:

- [Add a new Artifact Class](https://develop.qiime2.org/en/latest/plugins/tutorials/add-artifact-class.html)
- [Creating and registering a Transformer](https://develop.qiime2.org/en/latest/plugins/how-to-guides/create-register-transformer.html)

High-level pattern in `plugin_setup.py`:

```python
plugin.register_semantic_types(MyNewType)
plugin.register_formats(MyDirectoryFormat)
plugin.register_artifact_class(
    MyNewType,
    directory_format=MyDirectoryFormat,
)
```

And in `_transformers.py`:

```python
from qiime2.plugin import TransformerSetup

@plugin.register_transformer
def _1(data: MyDirectoryFormat) -> pd.DataFrame:
    # read format, return python object
    ...

@plugin.register_transformer
def _2(data: pd.DataFrame) -> MyDirectoryFormat:
    # write python object to format
    ...
```
