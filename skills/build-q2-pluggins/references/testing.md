# Testing QIIME 2 Plugins

> Official references:
> - [How to test QIIME 2 plugins](https://develop.qiime2.org/en/latest/plugins/how-to-guides/test-plugins.html)
> - [Automate testing of your plugin](https://develop.qiime2.org/en/latest/plugins/how-to-guides/automate-testing.html)
> - [Testing API reference](https://develop.qiime2.org/en/latest/plugins/references/api/testing.html)

---

## TestPluginBase

The recommended base class for QIIME 2 plugin tests. Extends `unittest.TestCase` with QIIME 2-specific helpers.

```python
from qiime2.plugin.testing import TestPluginBase

class TestMyMethod(TestPluginBase):
    package = 'q2_pluginname.tests'  # resolves data paths to q2_pluginname/tests/data/

    def test_basic(self):
        fixture_path = self.get_data_path('fixture.tsv')
        # fixture_path is an absolute path to q2_pluginname/tests/data/fixture.tsv
        ...
```

`get_data_path(filename)` resolves relative to the `tests/data/` directory of the package named by `self.package`.

---

## Fixture Data

Keep test fixtures small — just enough to exercise the logic. Place them in `q2_pluginname/tests/data/`.

```
q2_pluginname/tests/data/
├── feature-table.biom
├── sequences.fasta
├── taxonomy.tsv
└── sample-metadata.tsv
```

---

## Loading Fixtures as QIIME 2 Artifacts

Use `qiime2.Artifact.import_data` to build an artifact from a fixture file in tests:

```python
import qiime2
import pandas as pd

class TestMyMethod(TestPluginBase):
    package = 'q2_pluginname.tests'

    def setUp(self):
        super().setUp()
        # Load fixture as QIIME 2 artifact
        table_path = self.get_data_path('feature-table.biom')
        self.table = qiime2.Artifact.import_data(
            'FeatureTable[Frequency]', table_path
        )

    def test_filter_table(self):
        from q2_pluginname._methods import filter_table
        result_df = filter_table(
            table=self.table.view(pd.DataFrame),
            min_frequency=0.01,
        )
        self.assertIsInstance(result_df, pd.DataFrame)
        self.assertGreater(len(result_df.columns), 0)
```

---

## Using Transformers in Tests

Convert fixture files to Python objects using `qiime2.plugin.util.transform`:

```python
from qiime2.plugin.util import transform
from q2_types.feature_data import DNAFASTAFormat, DNAIterator

class TestMyMethod(TestPluginBase):
    package = 'q2_pluginname.tests'

    def test_sequence_processing(self):
        fasta_path = self.get_data_path('sequences.fasta')
        fmt = DNAFASTAFormat(fasta_path, mode='r')
        seqs = transform(fmt, to_type=DNAIterator)
        ...
```

---

## Testing via the Artifact API

For end-to-end tests that exercise the full QIIME 2 action pipeline (including type checking and provenance):

```python
import qiime2

class TestMyMethod(TestPluginBase):
    package = 'q2_pluginname.tests'

    def setUp(self):
        super().setUp()
        self.plugin = qiime2.core.testing.TestPluginBase  # or load via PluginManager

    def test_method_via_api(self):
        table_artifact = qiime2.Artifact.import_data(
            'FeatureTable[Frequency]',
            self.get_data_path('feature-table.biom'),
        )
        # Call the action via artifact API
        result, = qiime2.ActionResults(...)
```

In practice it's often simpler to test the Python function directly (passing format objects or DataFrames) and rely on integration testing for the full pipeline.

---

## Skipping Tests That Require External Tools

```python
import unittest

def flash_available():
    import shutil
    return shutil.which('flash') is not None

@unittest.skipUnless(flash_available(), 'flash not installed')
class TestFlashIntegration(TestPluginBase):
    package = 'q2_pluginname.tests'

    def test_join_pairs(self):
        ...
```

---

## Common Assertions

```python
import numpy as np
import pandas as pd

# DataFrame shape
self.assertEqual(result.shape, (10, 5))

# No NaN values
self.assertFalse(result.isnull().any().any())

# Column presence
self.assertIn('Taxon', result.columns)

# Approximate float equality
self.assertAlmostEqual(result.loc['sample1', 'feature1'], 42.0, places=3)

# Index equality
pd.testing.assert_index_equal(result.index, expected.index)

# DataFrame equality
pd.testing.assert_frame_equal(result, expected)
```

---

## Automated CI Testing

See [Automate testing of your plugin](https://develop.qiime2.org/en/latest/plugins/how-to-guides/automate-testing.html) for a GitHub Actions workflow. The typical CI pipeline:

1. Install the conda environment from `environment-files/`
2. `pip install -e .`
3. `flake8 q2_pluginname` (lint)
4. `py.test q2_pluginname/tests/` (unit tests)
5. `qiime dev refresh-cache` and smoke-test the CLI

---

## Running Tests Locally

```bash
# From the repo root
make dev              # install in editable mode first
make test             # py.test q2_pluginname/tests/

# Or directly:
py.test q2_pluginname/tests/ -v
py.test q2_pluginname/tests/test_methods.py::TestMyMethod::test_filter -v
```
