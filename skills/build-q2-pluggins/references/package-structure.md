# QIIME 2 Plugin Package Structure

> Official reference: [The structure of QIIME 2 plugin packages](https://develop.qiime2.org/en/latest/plugins/explanations/package-structure.html)

## Conventional Layout

```
q2-pluginname/                   ← distribution package (repo root)
├── q2_pluginname/               ← Python package (underscores, not hyphens)
│   ├── __init__.py              ← version string only; don't add logic here
│   ├── _methods.py              ← implementations registered as Methods
│   ├── _visualizers.py          ← implementations registered as Visualizers
│   ├── _pipelines.py            ← implementations registered as Pipelines (optional)
│   ├── _formats.py              ← custom DirectoryFormat / FileFormat classes (if any)
│   ├── _types.py                ← custom semantic type definitions (if any)
│   ├── _transformers.py         ← transformers between format objects (if any)
│   ├── plugin_setup.py          ← Plugin object; all registrations
│   ├── citations.bib            ← BibTeX citations referenced in registration calls
│   └── tests/
│       ├── __init__.py
│       ├── test_methods.py
│       ├── test_visualizers.py  ← (if applicable)
│       └── data/                ← small fixture files for unit tests
├── pyproject.toml               ← build config, dependencies, entry point
├── Makefile                     ← make test / make dev / make lint
└── environment-files/           ← conda YAML files for reproducible envs
    ├── qiime2-amplicon-environment.yml
    └── qiime2-tiny-environment.yml
```

The leading underscore on `_methods.py`, `_visualizers.py`, etc. is a Python convention signalling that external code should not import from these files directly. This gives you freedom to refactor internals without breaking downstream callers.

## pyproject.toml

The entry point tells the QIIME 2 PluginManager where to find your plugin:

```toml
[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "q2-pluginname"
version = "0.0.1"
description = "Short description."
license = {text = "BSD-3-Clause"}
dependencies = [
    "qiime2 >= 2024.10",
    "q2-types",
    # add your other dependencies here
]

[project.entry-points."qiime2.plugins"]
"q2-pluginname" = "q2_pluginname.plugin_setup:plugin"
```

Only one entry point per package is typical, but a single package can register multiple plugins.

## plugin_setup.py

This is where the QIIME 2 `Plugin` object is instantiated and all actions, semantic types, formats, and transformers are registered. By convention it lives at `q2_pluginname/plugin_setup.py`, though QIIME 2 itself does not require this name.

Minimal skeleton:

```python
from qiime2.plugin import Plugin, Citations
import q2_pluginname

citations = Citations.load('citations.bib', package='q2_pluginname')

plugin = Plugin(
    name='pluginname',                      # used in CLI: qiime pluginname ...
    version=q2_pluginname.__version__,
    website='https://github.com/yourorg/q2-pluginname',
    package='q2_pluginname',
    description='Longer description.',
    short_description='One-line description.',
)

# --- import and register actions, types, formats below ---
from q2_pluginname._methods import my_method
from q2_types.feature_table import FeatureTable, Frequency

plugin.methods.register_function(
    function=my_method,
    ...
)
```

## __init__.py

Keep this minimal — just expose the version:

```python
__version__ = '0.0.1'
```

## Makefile Conventions

```makefile
dev:
	pip install -e .

test:
	py.test q2_pluginname/tests/

lint:
	flake8 q2_pluginname

all: dev test
```

## environment-files/

Provide conda YAML files so users can reproduce your development environment. Typically one for each QIIME 2 distribution you support (e.g., `qiime2-amplicon`, `qiime2-tiny`). See the [QIIME 2 install docs](https://docs.qiime2.org/2024.10/install/) for current distribution YAML sources.

## As a Plugin Grows

When `_methods.py` gets large, refactor into a directory:

```
q2_pluginname/
├── _methods/
│   ├── __init__.py
│   ├── _diversity.py
│   └── _filtering.py
```

Keep `plugin_setup.py` importing from the public API of your package (`from q2_pluginname import my_method`) rather than from private submodules directly.
