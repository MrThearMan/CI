# Reusable CI pipelines for poetry projects

> This is meant for personal use for my own projects.

## Requirements

There are 4 different pipeline templates, one for testing,
one for building docs, one for releasing the library to [PyPI],
and one for approving pull requests by certain authors automatically.

All templates require the project to use [poetry].
The testing pipeline requires [tox] and [coverage], as well as 
Coverage results are sent to [coveralls].
The docs building pipeline requires [mkdocs].

## How to use

Here is a minimal `pyproject.toml` setup to get started:

```toml
# ...

[tool.poetry.group.test.dependencies]
pytest = "..."  # use latest version
coverage = "..."  # use latest version
tox = "..."  # use latest version
tox-gh-actions = "..."  # use latest version
# 'distutils' is not included in virtual environments from Python 3.12 onwards,
# so you might need to explcitly include it via setuptools.
setuptools = {version = "...", python = ">=3.12"}  # use latest version

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
envlist = py{39, 310, 311, 312}
isolated_build = true

[gh-actions]
python =
    3.9: py39
    3.10: py310
    3.11: py311
    3.12: py312

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
requires = ["poetry-core>=1.8.1"]
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
name: Tests

on:
  push:
    branches:
      - main
    paths:
      - "**.py"
      - "pyproject.toml"
      - "poetry.lock"
  pull_request:
  workflow_dispatch:

jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.4.9
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
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.4.9
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
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.4.9
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
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.4.9
    with:
      poetry-version: "1.7.1"
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
    uses: MrThearMan/CI/.github/workflows/test.yml@v0.4.9
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
name: Docs

on:
  push:
    branches:
      - main
    paths:
      - "docs/**"
      - "mkdocs.yml"
  workflow_dispatch:

jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.4.9
```

This job can take a number of inputs via the [with]-keyword.

---

#### `poetry-version`

Configure the poetry version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.4.9
    with:
      poetry-version: "1.7.1"
```

---

#### `python-version`

Configure the python version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.4.9
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
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.4.9
    with:
      os: "ubuntu-latest"
```

---

### PyPI release pipeline

This pipeline can be used to build and release the library to [PyPI] with 
poetry using a [PyPI token] stored in the repository's [actions secrets].

> Note that the poetry [version] configuration needs to be updated and match
> the tag created for the release or this job will fail (can include v-prefix,
> e.g., `v0.0.1`).

To set up the pipeline, add a `yml` file to `./.github/workflows/`
with the following job configuration. The `pypi-token` input is required.

```yaml
name: Release

on:
  release:
    types:
      - released

jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.4.9
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
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.4.9
    with:
      poetry-version: "1.7.1"
```

---

#### `python-version`

Configure the python version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.4.9
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
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.4.9
    with:
      os: "ubuntu-latest"
```

---

### Pull request approval pipeline

This pipeline can be used to automatically approve pull request by some users.
By default, it is set to approve pull requests by the [dependabot] and
[pre-commit.ci] bots.

To set up the pipeline, add a `yml` file to `./.github/workflows/`
with the following job configuration.

```yaml
name: Auto approve PRs

on: 
  pull_request_target:

jobs:
  approve:
    permissions:
      pull-requests: write
      contents: write
    uses: MrThearMan/CI/.github/workflows/approve.yml@v0.4.9
```

This job can take a number of inputs via the [with]-keyword.

---

#### `users`

Configure the users whose pull requests can be automatically approved.

Default configuration:

```yaml
jobs:
  approve:
    uses: MrThearMan/CI/.github/workflows/approve.yml@v0.4.9
    with:
      users: '["dependabot[bot]", "pre-commit-ci[bot]"]'
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

---

## Extra actions

### Poetry install action

Installs poetry to the current python version, e.g., if used after 
`actions/setup-python`, poetry is installed with that python version.

```yaml
jobs:
  <foo>:
    steps:
      - ...
      - uses: MrThearMan/CI/.github/actions/poetry@v0.4.9
        with:
          os: "ubuntu-latest"
          poetry-version: "1.7.1"
```

However, `actions/setup-python` [poetry caching] cannot be used if poetry is not installed.
In this case, a custom cache must be created:

```yaml
jobs:
  <foo>:
    steps:
      - ...
      - name: "Load cached poetry environment"
        uses: actions/cache@v4
        with:
          path: .venv
          key: <unique-key-per-env>
```

### Git changed filetypes

> Not tested yet.

Can be used to check if certain filetypes were changed in a pull request.

```yaml
jobs:
  <foo>:
    steps:
      - uses: MrThearMan/CI/.github/actions/get-changed-filetypes@v0.4.9
        id: changed
        with:
          filetypes: "py|yaml"
      - if: ${{ changed.changed-filetypes }}
```


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
[poetry caching]: https://github.com/actions/setup-python/blob/main/docs/advanced-usage.md#caching-packages
[dependabot]: https://github.com/dependabot
[pre-commit.ci]: https://github.com/apps/pre-commit-ci
