# Registering Actions in QIIME 2 Plugins

> Official how-to guides:
> - [Create and register a Method](https://develop.qiime2.org/en/latest/plugins/how-to-guides/create-register-method.html)
> - [Create and register a Visualizer](https://develop.qiime2.org/en/latest/plugins/how-to-guides/create-register-visualizer.html)
> - [Create and register a Pipeline](https://develop.qiime2.org/en/latest/plugins/how-to-guides/create-register-pipeline.html)
> - [Plugin & Registration API](https://develop.qiime2.org/en/latest/plugins/references/api/plugin.html)

---

## Action Types

| Action type | Returns | Use when |
|---|---|---|
| **Method** | One or more QIIME 2 Artifacts | Producing data from data |
| **Visualizer** | A single Visualization (index.html) | Producing a human-readable summary |
| **Pipeline** | Mix of Artifacts and Visualizations | Chaining multiple Methods/Visualizers |

---

## Plugin Object

```python
from qiime2.plugin import Plugin, Citations
import q2_pluginname

citations = Citations.load('citations.bib', package='q2_pluginname')

plugin = Plugin(
    name='pluginname',            # used in CLI: `qiime pluginname <action>`
    version=q2_pluginname.__version__,
    website='https://github.com/yourorg/q2-pluginname',
    package='q2_pluginname',
    description='Full description shown in `qiime pluginname --help`.',
    short_description='One-line description.',
)
```

---

## Methods

### Implementation (`_methods.py`)

A Method is a plain Python function. QIIME 2 calls it with format objects / Python types; return values are the outputs.

```python
import pandas as pd

def filter_table(table: pd.DataFrame, min_frequency: float) -> pd.DataFrame:
    """Filter features below a relative frequency threshold."""
    mask = (table / table.sum()).max(axis=0) >= min_frequency
    return table.loc[:, mask]
```

- Inputs arrive as the mapped Python type (see `type-system.md`).
- Return a tuple if multiple outputs; a single value if one output.
- Return `pd.DataFrame` for `FeatureTable[*]` — QIIME 2 transformers handle BIOM.

### Registration (`plugin_setup.py`)

```python
from qiime2.plugin import Float, Range
from q2_types.feature_table import FeatureTable, Frequency
from q2_pluginname._methods import filter_table

plugin.methods.register_function(
    function=filter_table,
    inputs={
        'table': FeatureTable[Frequency],
    },
    parameters={
        'min_frequency': Float % Range(0.0, 1.0, inclusive_start=False),
    },
    outputs=[
        ('filtered_table', FeatureTable[Frequency]),
    ],
    input_descriptions={
        'table': 'The feature table to filter.',
    },
    parameter_descriptions={
        'min_frequency': (
            'Features present in at least this fraction of samples are retained.'
        ),
    },
    output_descriptions={
        'filtered_table': 'Filtered feature table.',
    },
    name='Filter Feature Table',
    description=(
        'Filter features from a FeatureTable[Frequency] based on minimum '
        'relative frequency across samples.'
    ),
    citations=[citations['Author2024']],
)
```

### Multiple Outputs

```python
outputs=[
    ('table', FeatureTable[Frequency]),
    ('representative_sequences', FeatureData[Sequence]),
]
```

The Python function must return a matching tuple:

```python
def my_method(...) -> (pd.DataFrame, DNASequencesDirectoryFormat):
    ...
    return filtered_df, sequences_dir_fmt
```

---

## Visualizers

### Implementation (`_visualizers.py`)

Visualizers receive an `output_dir: str` parameter — write your HTML there. They return `None`.

```python
import os
import pandas as pd
import q2templates

def summarize(output_dir: str, table: pd.DataFrame) -> None:
    stats = table.describe()
    stats.to_html(os.path.join(output_dir, 'index.html'))
```

Using `q2templates` (recommended for consistent QIIME 2 styling):

```python
import q2templates

def summarize(output_dir: str, table: pd.DataFrame) -> None:
    context = {'df': table.to_html()}
    templates = os.path.join(os.path.dirname(__file__), 'assets', 'templates')
    q2templates.render(templates, output_dir, context=context)
```

### Registration

Visualizers have **no** `outputs` or `output_descriptions` parameters:

```python
from q2_pluginname._visualizers import summarize

plugin.visualizers.register_function(
    function=summarize,
    inputs={'table': FeatureTable[Frequency]},
    parameters={},
    input_descriptions={'table': 'The feature table to summarize.'},
    parameter_descriptions={},
    name='Summarize Feature Table',
    description='Generate an interactive summary of a feature table.',
    citations=[],
)
```

---

## Pipelines

### Implementation (`_pipelines.py`)

Pipelines call QIIME 2 actions via the `ctx` (context) object. They do not call Python functions directly — they call registered actions so that provenance is captured for each step.

```python
def full_pipeline(ctx, sequences, trim_left, trunc_len):
    # Look up registered actions
    denoise = ctx.get_action('dada2', 'denoise_single')
    summarize = ctx.get_action('feature_table', 'summarize')

    table, rep_seqs, denoising_stats = denoise(
        demultiplexed_seqs=sequences,
        trim_left=trim_left,
        trunc_len=trunc_len,
    )
    visualization, = summarize(table=table)
    return table, rep_seqs, visualization
```

### Registration

```python
from qiime2.plugin import Int, Range
from q2_types.per_sample_sequences import SampleData, SequencesWithQuality
from q2_types.feature_table import FeatureTable, Frequency
from q2_types.feature_data import FeatureData, Sequence

plugin.pipelines.register_function(
    function=full_pipeline,
    inputs={'sequences': SampleData[SequencesWithQuality]},
    parameters={
        'trim_left': Int % Range(0, None),
        'trunc_len': Int % Range(1, None),
    },
    outputs=[
        ('table', FeatureTable[Frequency]),
        ('representative_sequences', FeatureData[Sequence]),
        ('visualization', Visualization),
    ],
    input_descriptions={'sequences': 'Demultiplexed sequences.'},
    parameter_descriptions={
        'trim_left': 'Bases to trim from 5\' end.',
        'trunc_len': 'Length to truncate reads to.',
    },
    output_descriptions={
        'table': 'Feature table.',
        'representative_sequences': 'Representative sequences.',
        'visualization': 'Denoising summary.',
    },
    name='Full Denoising Pipeline',
    description='Runs DADA2 denoising and summarizes results.',
    # No citations needed if underlying actions already cite them
)
```

---

## Citations

`citations.bib` uses standard BibTeX format. Keys must be unique:

```bibtex
@article{Author2024,
  author  = {Author, A. and Coauthor, B.},
  title   = {Title of the paper},
  journal = {Journal Name},
  year    = {2024},
  volume  = {10},
  pages   = {1--15},
  doi     = {10.1234/example},
}
```

In `plugin_setup.py`:

```python
citations = Citations.load('citations.bib', package='q2_pluginname')
# Then pass to register_function:
citations=[citations['Author2024']]
```

Each action call logs its citations in the provenance of every output artifact — your users can always discover what to cite. Pipelines automatically inherit citations from the underlying Methods and Visualizers they call.

---

## TypeMatch for Flexible Inputs

```python
from qiime2.plugin import TypeMatch
from q2_types.per_sample_sequences import SequencesWithQuality, JoinedSequencesWithQuality

T = TypeMatch([SequencesWithQuality, JoinedSequencesWithQuality])

plugin.methods.register_function(
    function=my_method,
    inputs={'sequences': SampleData[T]},
    outputs=[('result', SampleData[T])],
    ...
)
```

---

## Registering Custom Semantic Types and Formats

If your data doesn't fit existing types:

```python
from qiime2.plugin import SemanticType

# In _types.py:
MyNewType = SemanticType('MyNewType')

# In plugin_setup.py:
plugin.register_semantic_types(MyNewType)
plugin.register_formats(MyDirectoryFormat)
plugin.register_artifact_class(
    MyNewType,
    directory_format=MyDirectoryFormat,
    description='Description of what this artifact class represents.',
)
```

See [Add a new Artifact Class](https://develop.qiime2.org/en/latest/plugins/tutorials/add-artifact-class.html) for the full tutorial.
