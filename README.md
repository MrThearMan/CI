# Reusable CI pipelines for poetry projects

> This is meant for personal use for my own projects.

## Requirements

There are 3 different pipeline templates, one for testing,
one for building docs, and one for releasing the library to [PyPI].

All templates require the project to use [poetry].
The testing pipeline requires [tox] and [coverage], as well as 
[coveralls-python] for sending coverage results to [coveralls].
The docs building pipeline requires [mkdocs].

## How to use

Here is a minimal `pyproject.toml` setup to get started:

```toml
# ...

[tool.poetry.group.test.dependencies]
pytest = "..."  # use latest version
coverage = "6.5.0"  # <7.x needed for coveralls-python
tox = "..."  # use latest version
tox-gh-actions = "..."  # use latest version
coveralls = "..."  # use latest version

# This is only needed for the docs CI
[tool.poetry.group.docs.dependencies]
mkdocs = "..."  # use latest version

[tool.coverage.run]
relative_files = true

[tool.coverage.report]
omit = [
    "tests/*",
    ".tox/*",
]

[tool.tox]
legacy_tox_ini = """
[tox]
envlist = py{39, 310, 311}
isolated_build = true

[gh-actions]
python =
    3.9: py39
    3.10: py310
    3.11: py311

[testenv]
allowlist_externals =
    poetry
setenv =
    PYTHONPATH = {toxinidir}
commands =
    poetry install
    poetry run coverage run -m pytest
"""

[build-system]
requires = ["poetry-core>=1.6.0"]
build-backend = "poetry.core.masonry.api"
```

---

### Testing pipeline

This pipeline uses a [job stategy matrix] to run tests in a number of
python environments and operating systems in parallel. All dependencies
are cached for each os and environment resulting from the strategy
to ensure the CI runs fast when dependencies are not updated.

To set up the pipeline, add a `yml` file to `./.github/workflows/` 
with the following job configuration.

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.3.3
```

This job can take a number of inputs via the [with]-keyword.

---

#### `python-version`

Configure the pyhton versions the test will be run with.
The tox [environments] used are configured with the
`[gh-actions]` setting in `pyproject.toml`.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.3.3
    with:
      python-version: '["3.9", "3.10", "3.11"]'
```

---

#### `os`

Configure the operating systems the tests will be run with.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.3.3
    with:
      os: '["ubuntu-latest", "macos-latest", "windows-latest"]'
```

---

#### `poetry-version`

Configure the poetry version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.3.3
    with:
      poetry-version: "1.5.1"
```

---

#### `submit-python-version`

Configure the python version used for the job that executes 
the coveralls [parallel builds webhook].

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.3.3
    with:
      submit-python-version: "3.11"
```

---

#### `submit-os`

Configure the operating system used for the job that executes 
the coveralls [parallel builds webhook].

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.3.3
    with:
      submit-os: "ubuntu-latest"
```

---

#### `exclude`

GitHub [job stategy matrix] exclusion pattern, in JSON form.
Using [yaml flow style], a list of dicts can be conveted into
multiple exclusions if necessary.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.3.3
    with:
      exclude: '[{"os": "none", "python-version": "none"}]'  # this ignores nothing
```

---

### Docs building pipeline

This pipeline can be used to build and push the docs used for 
[mkdocs] from the `docs/` directory into [GitHub pages].

> Note that [mkdocs] also requires a separate configuration 
> file where the pages are set up. Here's a minimal example configuration.
> 
> ```yaml
> site_name: {{ Site name here }}
> 
> nav:
>   - Home: index.md  # ./docs/index.md
> ```

To set up the pipeline, add a `yml` file to `./.github/workflows/` 
with the following job configuration.

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.3.3
```

This job can take a number of inputs via the [with]-keyword.

---

#### `poetry-version`

Configure the poetry version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.3.3
    with:
      poetry-version: "1.5.1"
```

---

#### `python-version`

Configure the python version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.3.3
    with:
      python-version: "3.11"
```

---

#### `os`

Configure the operating system used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.3.3
    with:
      os: "ubuntu-latest"
```

---

### PyPI release pipeline

This pipeline can be used to build and release the library to [PyPI] with 
poetry using a [PyPI token] stored in the repository's [actions secrets].

> Note that the poetry [version] configuration needs to be updated or this
> job will fail.

To set up the pipeline, add a `yml` file to `./.github/workflows/`
with the following job configuration. The `pypi-token` input is required.

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.3.3
    secrets:
      pypi-token: ${{ secrets.PYPI_API_TOKEN }}
```

> Replace `PYPI_API_TOKEN` with the secret name of your choice

This job can take a number of inputs via the [with]-keyword.

---

#### `poetry-version`

Configure the poetry version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.3.3
    with:
      poetry-version: "1.5.1"
```

---

#### `python-version`

Configure the python version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.3.3
    with:
      python-version: "3.11"
```

---

#### `os`

Configure the operating system used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.3.3
    with:
      os: "ubuntu-latest"
```

---

## Pipeline hooks

Some of the pipeline templates have hooks that can be defined
to run additional setup and postprocessing steps if necessary.
To enable them, simply add a `yaml` file to the appropriate directory
in your project and write a [composite action]. Here is a template for one.

```yaml
runs:
  using: composite
  
  steps:
    - name: "Setup"
      shell: bash
      run: ...
```

> Here is a little trick you can do to pass the inputs from 
> the testing job to the composite action when needed
> 
> ```yaml
> inputs:
>   python-version:
>     default: ${{ matrix.python-version }}
> ```

For the testing pipeline, the hooks are:

- `Pre-test`: Add the file `.github/actions/pre-test/action.yml`
- `Post-test`: Add the file `.github/actions/post-test/action.yml`

For the release pipeline, the hooks are:

- `Pre-release`: Add the file `.github/actions/pre-release/action.yml`
- `Post-release`: Add the file `.github/actions/post-release/action.yml`


[poetry]: https://python-poetry.org/
[tox]: https://tox.wiki/en/latest/
[coverage]: https://coverage.readthedocs.io/en/latest/
[coveralls]: https://docs.coveralls.io/
[coveralls-python]: https://github.com/TheKevJames/coveralls-python
[mkdocs]: https://www.mkdocs.org/
[PyPI]: https://pypi.org/
[with]: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepswith
[environments]: https://tox.wiki/en/latest/config.html#envlist
[parallel builds webhook]: https://docs.coveralls.io/parallel-build-webhook
[job stategy matrix]: https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs
[yaml flow style]: https://yaml.org/spec/1.2.2/#chapter-7-flow-style-productions
[GitHub pages]: https://pages.github.com/
[pypi token]: https://pypi.org/help/#apitoken
[actions secrets]: https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository
[version]: https://python-poetry.org/docs/pyproject#version
[composite action]: https://docs.github.com/en/actions/creating-actions/creating-a-composite-action
