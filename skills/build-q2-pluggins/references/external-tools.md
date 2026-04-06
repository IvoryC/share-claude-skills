# Calling External Tools from QIIME 2 Plugins

This covers patterns for wrapping command-line tools, R scripts, and Java-based programs as QIIME 2 Methods.

---

## General subprocess Pattern

```python
import subprocess

def run_tool(input_path: str, output_path: str, threads: int) -> None:
    cmd = [
        'my-tool',
        '--input', input_path,
        '--output', output_path,
        '--threads', str(threads),
    ]
    proc = subprocess.run(cmd, capture_output=True, text=True)
    if proc.returncode != 0:
        raise RuntimeError(
            f'my-tool failed (exit {proc.returncode}).\n'
            f'STDOUT:\n{proc.stdout}\n'
            f'STDERR:\n{proc.stderr}'
        )
```

Always check `returncode` and surface stdout/stderr in the error message — this is essential for users to debug failures.

---

## Using Temporary Directories

QIIME 2 methods often need to write intermediate files. Use `tempfile.TemporaryDirectory` as a context manager:

```python
import tempfile
import os
import shutil
import subprocess
import pandas as pd
from q2_types.feature_data import DNASequencesDirectoryFormat, DNAFASTAFormat

def my_wrapper(sequences: DNASequencesDirectoryFormat, threshold: float) -> pd.DataFrame:
    with tempfile.TemporaryDirectory() as tmp_dir:
        input_fasta = os.path.join(tmp_dir, 'input.fasta')
        output_tsv  = os.path.join(tmp_dir, 'output.tsv')

        # Extract FASTA from format object
        shutil.copy(str(sequences.file.view(DNAFASTAFormat)), input_fasta)

        proc = subprocess.run(
            ['my-tool', '--in', input_fasta, '--out', output_tsv,
             '--threshold', str(threshold)],
            capture_output=True, text=True,
        )
        if proc.returncode != 0:
            raise RuntimeError(
                f'my-tool failed:\nSTDOUT:\n{proc.stdout}\nSTDERR:\n{proc.stderr}'
            )

        result = pd.read_csv(output_tsv, sep='\t', index_col=0)

    # pd.DataFrame is in-memory — safe to return after tempdir cleanup
    return result
```

**Important:** QIIME 2 format objects manage their own storage outside `tmp_dir`, so it is safe to return format objects after the `with` block exits. But construct format objects *inside* the `with` block (before cleanup) and return them — do not construct them after.

---

## Naming Temp Files with Sample IDs

When processing per-sample files, use the sample ID as the filename so output tables automatically carry the correct sample IDs:

```python
from q2_types.per_sample_sequences import SingleLanePerSampleSingleEndFastqDirFmt
import gzip

def process_samples(sequences: SingleLanePerSampleSingleEndFastqDirFmt, ...) -> pd.DataFrame:
    manifest = sequences.manifest.view(pd.DataFrame)
    results = {}

    with tempfile.TemporaryDirectory() as tmp_dir:
        for sample_id in manifest.index:
            src = manifest.loc[sample_id, 'forward']  # path to .fastq.gz
            out = os.path.join(tmp_dir, f'{sample_id}.out')

            proc = subprocess.run(
                ['my-tool', '--input', src, '--output', out],
                capture_output=True, text=True,
            )
            if proc.returncode != 0:
                raise RuntimeError(
                    f'Failed on sample {sample_id}:\n{proc.stderr}'
                )

            results[sample_id] = parse_output(out)

    return pd.DataFrame(results).T  # samples × features

def parse_output(path: str) -> pd.Series:
    ...
```

---

## Calling R Scripts

```python
import subprocess
import tempfile
import os

def run_r_analysis(input_path: str, output_path: str) -> None:
    r_script = f"""
    library(myRpackage)
    data <- read.csv("{input_path}", row.names=1)
    result <- myFunction(data)
    write.csv(result, "{output_path}")
    """
    proc = subprocess.run(
        ['Rscript', '--vanilla', '-e', r_script],
        capture_output=True, text=True,
    )
    if proc.returncode != 0:
        raise RuntimeError(
            f'R script failed (exit {proc.returncode}).\n'
            f'STDOUT:\n{proc.stdout}\nSTDERR:\n{proc.stderr}'
        )
```

Alternative — call an R script file:

```python
proc = subprocess.run(
    ['Rscript', '--vanilla', '/path/to/script.R', input_path, output_path],
    capture_output=True, text=True,
)
```

### rJava / JVM Heap

R packages that use rJava (e.g., `rJava`, some bioinformatics packages) may need more JVM heap. Set via environment variable before the subprocess:

```python
import os

env = os.environ.copy()
env['JAVA_TOOL_OPTIONS'] = '-Xmx8g'

proc = subprocess.run(
    ['Rscript', '--vanilla', '-e', r_script],
    capture_output=True, text=True,
    env=env,
)
```

---

## Calling Java Tools Directly

```python
def run_java_tool(jar_path: str, input_path: str, output_path: str) -> None:
    cmd = [
        'java', '-Xmx4g', '-jar', jar_path,
        '--input', input_path,
        '--output', output_path,
    ]
    proc = subprocess.run(cmd, capture_output=True, text=True)
    if proc.returncode != 0:
        raise RuntimeError(
            f'Java tool failed:\nSTDOUT:\n{proc.stdout}\nSTDERR:\n{proc.stderr}'
        )
```

---

## Checking Tool Availability

Use this in tests to skip integration tests when the tool is not installed:

```python
import shutil
import unittest

def my_tool_available() -> bool:
    return shutil.which('my-tool') is not None

@unittest.skipUnless(my_tool_available(), 'my-tool not installed')
class TestMyToolIntegration(TestPluginBase):
    package = 'q2_pluginname.tests'

    def test_run(self):
        ...
```

And raise a clear error inside the Method itself:

```python
def my_wrapper(...):
    if shutil.which('my-tool') is None:
        raise RuntimeError(
            'my-tool is not installed or not on PATH. '
            'Install it with: conda install -c bioconda my-tool'
        )
    ...
```

---

## Including Tool Binaries in the Conda Package

For tools distributed via Bioconda or conda-forge, declare them in your `pyproject.toml` dependencies or your conda `environment-files/` YAML:

```yaml
# environment-files/qiime2-amplicon-environment.yml
dependencies:
  - qiime2-amplicon=2024.10
  - bioconda::my-tool=1.2.3
  - r-base=4.3
  - bioconductor-myrpackage
```

This ensures users install the correct tool version alongside your plugin.
