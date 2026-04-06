---
name: build-q2-pluggins
description: Create a qiime2 plugin for a new tool or as a wrapper to an existing tool to operate in the qiime2 ecosystem and track provenance.
version: 2.0.0
user-invocable: true
---

# Building QIIME 2 Plugins

> **Documentation note:** QIIME 2 is actively developed and some older docs are flagged as outdated. Always refer to [develop.qiime2.org](https://develop.qiime2.org/en/latest/) for current guidance. The canonical book is *Developing with QIIME 2* by Caporaso and Bolyen.

## Quick Start: Create from Template

The fastest path to a new plugin is the official Copier template maintained by the Caporaso Lab:

```bash
# Install pipx and copier if needed
pip install pipx
pipx install copier

# Generate your plugin scaffold
copier copy https://github.com/caporaso-lab/plugin-template.git .
```

Copier will prompt for your plugin name, target distribution, and other metadata. Follow the generated `README.md` for installation steps.

- Official template: [caporaso-lab/plugin-template](https://github.com/caporaso-lab/plugin-template)
- Step-by-step tutorial: [develop.qiime2.org — Create from template](https://develop.qiime2.org/en/stable/plugins/tutorials/create-from-template.html)

---

## Repository Structure

See [→ references/package-structure.md](references/package-structure.md) for a detailed breakdown.

Conventional layout:

```
q2-pluginname/
├── q2_pluginname/
│   ├── __init__.py          # version import only
│   ├── _methods.py          # Python implementations of all actions
│   ├── _visualizers.py      # Visualizer implementations (produce index.html)
│   ├── plugin_setup.py      # instantiates Plugin; registers actions, types, formats
│   ├── citations.bib        # BibTeX entries referenced in registration
│   └── tests/
│       ├── __init__.py
│       ├── test_methods.py
│       └── data/            # small fixture files for unit tests
├── pyproject.toml           # entry point: q2_pluginname.plugin_setup:plugin
├── Makefile                 # make test / make dev / make lint
└── environment-files/       # conda YAML files for reproducible envs
```

The `pyproject.toml` entry point that makes QIIME 2 discover your plugin:

```toml
[project.entry-points."qiime2.plugins"]
"q2-pluginname" = "q2_pluginname.plugin_setup:plugin"
```

---

## The Type System

See [→ references/type-system.md](references/type-system.md) for a full reference.

QIIME 2 has two kinds of types:
- **Semantic types** — what an artifact *represents* (e.g., `FeatureTable[Frequency]`)
- **Primitive types** — plain Python values passed as parameters (e.g., `Int`, `Float`, `Str`)

Key semantic type → Python format class mappings for common amplicon work:

| Semantic type | Python format / return type |
|---|---|
| `FeatureTable[Frequency]` | `pd.DataFrame` (samples × features, raw counts) |
| `FeatureData[Sequence]` | `DNASequencesDirectoryFormat` |
| `FeatureData[Taxonomy]` | `pd.DataFrame` (Feature ID index; Taxon + Confidence cols) |
| `SampleData[SequencesWithQuality]` | `SingleLanePerSampleSingleEndFastqDirFmt` |
| `SampleData[JoinedSequencesWithQuality]` | `SingleLanePerSampleSingleEndFastqDirFmt` |
| `SampleData[PairedEndSequencesWithQuality]` | `SingleLanePerSamplePairedEndFastqDirFmt` |
| `Phylogeny[Unrooted]` | `NewickFormat` |
| `DistanceMatrix` | `skbio.DistanceMatrix` |

QIIME 2 **transformers** handle the conversion between format objects and Python types — you do not call them directly.

---

## Registering Actions

See [→ references/action-registration.md](references/action-registration.md) for complete examples.

### Plugin object (`plugin_setup.py`)

```python
from qiime2.plugin import Plugin, Citations
import q2_pluginname

citations = Citations.load('citations.bib', package='q2_pluginname')

plugin = Plugin(
    name='pluginname',
    version=q2_pluginname.__version__,
    website='https://github.com/yourorg/q2-pluginname',
    package='q2_pluginname',
    description='One-sentence description.',
    short_description='Short description.',
)
```

### Registering a Method

```python
from qiime2.plugin import Int, Float, Range
from q2_types.feature_table import FeatureTable, Frequency
from q2_types.feature_data import FeatureData, Sequence

plugin.methods.register_function(
    function=my_func,
    inputs={
        'table': FeatureTable[Frequency],
    },
    parameters={
        'n_threads': Int % Range(1, None),
        'min_frequency': Float % Range(0.0, 1.0),
    },
    outputs=[
        ('filtered_table', FeatureTable[Frequency]),
        ('representative_sequences', FeatureData[Sequence]),
    ],
    input_descriptions={'table': 'The feature table to process.'},
    parameter_descriptions={
        'n_threads': 'Number of threads to use.',
        'min_frequency': 'Minimum relative frequency cutoff.',
    },
    output_descriptions={
        'filtered_table': 'Filtered feature table.',
        'representative_sequences': 'Representative sequences.',
    },
    name='My Method',
    description='Longer description of what this method does.',
    citations=[citations['AuthorYYYY']],
)
```

Use `TypeMatch` to accept multiple related types:

```python
from qiime2.plugin import TypeMatch
T = TypeMatch([SequencesWithQuality, JoinedSequencesWithQuality])
inputs = {'sequences': SampleData[T]}
```

### Registering a Visualizer

Visualizers produce a `Visualization` (an `index.html` + assets). They have **no** `outputs` or `output_descriptions` parameters:

```python
plugin.visualizers.register_function(
    function=my_visualizer,
    inputs={'table': FeatureTable[Frequency]},
    parameters={},
    input_descriptions={},
    parameter_descriptions={},
    name='My Visualizer',
    description='Creates an interactive summary.',
)
```

### Registering a Pipeline

Pipelines chain Methods and Visualizers. Citations from underlying actions are inherited automatically:

```python
plugin.pipelines.register_function(
    function=my_pipeline,
    inputs={...},
    parameters={...},
    outputs=[...],
    input_descriptions={...},
    parameter_descriptions={...},
    output_descriptions={...},
    name='My Pipeline',
    description='End-to-end workflow.',
)
```

Official how-to guides:
- [Create and register a Method](https://develop.qiime2.org/en/latest/plugins/how-to-guides/create-register-method.html)
- [Create and register a Visualizer](https://develop.qiime2.org/en/latest/plugins/how-to-guides/create-register-visualizer.html)
- [Create and register a Pipeline](https://develop.qiime2.org/en/latest/plugins/how-to-guides/create-register-pipeline.html)

---

## Testing

See [→ references/testing.md](references/testing.md) for patterns and examples.

```python
from qiime2.plugin.testing import TestPluginBase

class MyTests(TestPluginBase):
    package = 'q2_pluginname.tests'

    def test_my_func(self):
        fixture = self.get_data_path('fixture.tsv')  # resolves to tests/data/
        # ...
```

Run tests:

```bash
make test          # or: py.test q2_pluginname/tests/
```

---

## Calling External Tools

See [→ references/external-tools.md](references/external-tools.md) for patterns when wrapping R, Java, or other CLI tools.

---

## Development Workflow

```bash
# One-time: install in editable mode
make dev            # equivalent to: pip install -e .

# After code changes
make test           # run unit tests
qiime dev refresh-cache    # update QIIME 2 CLI cache after changes to plugin_setup.py

# Verify registration
qiime pluginname --help
```

Set up your environment:
- [Set up your development environment](https://develop.qiime2.org/en/latest/plugins/how-to-guides/set-up-development-environment.html)
- Use the **QIIME 2 amplicon distribution** (`qiime2-amplicon`) as your base conda env for 16S/amplicon work — it includes all standard types.
- Use the **QIIME 2 Tiny Distribution** if you only need the framework (minimal env for new plugin development).

---

## Common Gotchas

- `FeatureTable[Frequency]` returned as `pd.DataFrame` must have **samples as rows, features as columns**. Transpose with `df.T` if your tool outputs it the other way.
- `citations.bib` keys must be unique and match exactly what you pass to `citations['Key']`.
- Construct all QIIME 2 format objects *before* the `with tempfile.TemporaryDirectory()` block exits, or inside it and return them before cleanup — format objects manage their own storage.
- After modifying `plugin_setup.py` or adding new actions, always run `qiime dev refresh-cache` before testing via the CLI.
- `JAVA_TOOL_OPTIONS=-Xmx4g` is the cleanest way to cap JVM heap when calling R packages that depend on rJava.
- The `_methods.py` convention signals that outside code should not import from it directly — you are free to refactor it without breaking downstream users.

---

## Reference Pages

| Topic | File |
|---|---|
| Package / repo structure | [references/package-structure.md](references/package-structure.md) |
| Type system (semantic, primitive, formats) | [references/type-system.md](references/type-system.md) |
| Registering methods, visualizers, pipelines | [references/action-registration.md](references/action-registration.md) |
| Testing | [references/testing.md](references/testing.md) |
| Calling external tools (R, Java, CLI) | [references/external-tools.md](references/external-tools.md) |

---

## Key Official Links

- Developer book: [develop.qiime2.org](https://develop.qiime2.org/en/latest/)
- Plugin template: [github.com/caporaso-lab/plugin-template](https://github.com/caporaso-lab/plugin-template)
- Forum (developer): [forum.qiime2.org](https://forum.qiime2.org/c/dev/16)
- Semantic types reference: [docs.qiime2.org/2024.10/semantic-types](https://docs.qiime2.org/2024.10/semantic-types/)
