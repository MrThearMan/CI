# Reusable CI pipelines for uv projects

> This is meant for personal use for my own projects.

## Requirements

There are 4 different pipeline templates, one for testing,
one for building docs, one for releasing the library to [PyPI],
and one for approving pull requests by certain authors automatically.

All templates require the project to use [uv].
The testing pipeline requires [nox] and [coverage].
Coverage results are sent to [coveralls].
The docs building pipeline requires [mkdocs].

## How to use

Here is the required configuration for your `pyproject.toml`:

```toml
# ...

[dependency-groups]
test = [
    "pytest==...",  # use latest version
    "coverage==...",  # use latest version
    "nox==...",  # use latest version
]
# This is only needed for the docs CI
docs = [
    "mkdocs==...",  # use latest version
]

[tool.coverage.run]
relative_files = true
branch = true

[build-system]
requires = ["uv_build>=0.12.5,<0.13.0"]
build-backend = "uv_build"
```

---

### Testing pipeline

This pipeline uses a [job strategy matrix] to run tests in a number of
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
      - "uv.lock"
  pull_request:
  workflow_dispatch:

jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
```

Test pipelines use [nox](https://nox.thea.codes/en/stable/) to run the tests.
You'll need to add a `noxfile.py` to the root of your project with the following
content:

```python
import nox


@nox.session(python=["3.11", "3.12", "3.13", "3.14"], reuse_venv=True)
def tests(session: nox.Session) -> None:
    venv = session.virtualenv.location
    env = {"UV_PROJECT_ENVIRONMENT": venv}
    session.run_install("uv", "sync", "--all-extras", "--all-groups", "--python", venv, external=True, env=env)
    session.run("coverage", "run", "--parallel-mode", "-m", "pytest", external="error")
    session.run("coverage", "combine", "--append")
```

> `UV_PROJECT_ENVIRONMENT` tells `uv sync` where the environment goes, not which interpreter
> fills it. Without `--python`, `uv sync` picks its own interpreter, and a `.python-version`
> file makes it the first entry there. It then replaces the virtualenv nox just made, so every
> session runs on the same Python and the matrix tests one version four times.

> `uv sync` removes every package the lockfile does not name, and that includes `pip`.
> A session that pins a dependency version on top of the sync must use
> `session.run_install("uv", "pip", "install", "--python", session.virtualenv.location, ...)`
> instead of `session.install(...)`.

This job can take a number of inputs via the [with]-keyword.

---

#### `python-version`

Configure the python versions the tests will be run with.
Each version becomes one nox session.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      python-version: '["3.11", "3.12", "3.13", "3.14"]'
```

---

#### `os`

Configure the operating systems the tests will be run with.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      os: '["ubuntu-latest", "macos-latest", "windows-latest"]'
```

---

#### `env`

Environment variables to set in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      env: '{"FOO": "bar"}'
```

---

#### `uv-version`

Configure the uv version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      uv-version: "0.12.10"
```

---

#### `exclude`

GitHub [job strategy matrix] exclusion pattern, in JSON form.
Using [yaml flow style], a list of dicts can be converted into
multiple exclusions if necessary.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      exclude: '[{"os": "none", "python-version": "none"}]'  # this ignores nothing
```

---

#### `submodules`

Should submodules be checked out with the repository?

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      submodules: false
```

---

#### `fetch-depth`

How many commit to fetch from the branch history?
If set to 0, every commit is fetched.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      fetch-depth: 1
```

---

#### `coveralls`

Should coverage results be submitted to [coveralls]?
Requires coveralls setup to already exist.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      coveralls: true
```

---

#### `coveralls-os`

Which operating system should be covered using [coveralls]?
Should be one of the operating systems configured with `os`.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      coveralls-os: "ubuntu-latest"
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
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.6.0
```

This job can take a number of inputs via the [with]-keyword.

---

#### `uv-version`

Configure the uv version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.6.0
    with:
      uv-version: "0.12.10"
```

---

#### `python-version`

Configure the python version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.6.0
    with:
      python-version: "3.14"
```

---

#### `os`

Configure the operating system used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.6.0
    with:
      os: "ubuntu-latest"
```

---

#### `env`

Environment variables to set in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/docs.yml@v0.6.0
    with:
      env: '{"FOO": "bar"}'
```

---

#### `submodules`

Should submodules be checked out with the repository?

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      submodules: false
```

---

#### `fetch-depth`

How many commit to fetch from the branch history?
If set to 0, every commit is fetched.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      fetch-depth: 1
```

---

### PyPI release pipeline

This pipeline can be used to build and release the library to [PyPI] with 
uv using a [PyPI token] stored in the repository's [actions secrets].

> Note that the `project.version` in `pyproject.toml` needs to be updated and match
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
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.6.0
    secrets:
      pypi-token: ${{ secrets.PYPI_API_TOKEN }}
```

> Replace `PYPI_API_TOKEN` with the secret name of your choice

This job can take a number of inputs via the [with]-keyword.

---

#### `uv-version`

Configure the uv version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.6.0
    with:
      uv-version: "0.12.10"
```

---

#### `python-version`

Configure the python version used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.6.0
    with:
      python-version: "3.14"
```

---

#### `os`

Configure the operating system used in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.6.0
    with:
      os: "ubuntu-latest"
```

---

#### `env`

Environment variables to set in the pipeline.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/release.yml@v0.6.0
    with:
      env: '{"FOO": "bar"}'
```

---

#### `submodules`

Should submodules be checked out with the repository?

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      submodules: false
```

---

#### `fetch-depth`

How many commit to fetch from the branch history?
If set to 0, every commit is fetched.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      fetch-depth: 1
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
    uses: MrThearMan/CI/.github/workflows/approve.yml@v0.6.0
```

This job can take a number of inputs via the [with]-keyword.

---

#### `users`

Configure the users whose pull requests can be automatically approved.

Default configuration:

```yaml
jobs:
  approve:
    uses: MrThearMan/CI/.github/workflows/approve.yml@v0.6.0
    with:
      users: '["dependabot[bot]", "pre-commit-ci[bot]"]'
```

---

#### `submodules`

Should submodules be checked out with the repository?

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      submodules: false
```

---

#### `fetch-depth`

How many commit to fetch from the branch history?
If set to 0, every commit is fetched.

Default configuration:

```yaml
jobs:
  test:
    uses: MrThearMan/CI/.github/workflows/test-nox.yml@v0.6.0
    with:
      fetch-depth: 1
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

### uv install action

uv is installed with [setup-uv], which also restores the download cache.

```yaml
jobs:
  <foo>:
    steps:
      - ...
      - uses: astral-sh/setup-uv@v7
        with:
          version: "0.12.10"
          python-version: "3.14"
          enable-cache: true
```

### Git changed filetypes

> Not tested yet.

Can be used to check if certain filetypes were changed in a pull request.

```yaml
jobs:
  <foo>:
    steps:
      - uses: MrThearMan/CI/.github/actions/get-changed-filetypes@v0.6.0
        id: changed
        with:
          filetypes: "py|yaml"
      - if: ${{ changed.changed-filetypes }}
```


[uv]: https://docs.astral.sh/uv/
[nox]: https://nox.thea.codes/en/stable/
[setup-uv]: https://github.com/astral-sh/setup-uv
[coverage]: https://coverage.readthedocs.io/en/latest/
[coveralls]: https://docs.coveralls.io/
[coveralls-python]: https://github.com/TheKevJames/coveralls-python
[mkdocs]: https://www.mkdocs.org/
[PyPI]: https://pypi.org/
[with]: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idstepswith
[parallel builds webhook]: https://docs.coveralls.io/parallel-build-webhook
[job stategy matrix]: https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs
[yaml flow style]: https://yaml.org/spec/1.2.2/#chapter-7-flow-style-productions
[GitHub pages]: https://pages.github.com/
[pypi token]: https://pypi.org/help/#apitoken
[actions secrets]: https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository
[composite action]: https://docs.github.com/en/actions/creating-actions/creating-a-composite-action
[dependabot]: https://github.com/dependabot
[pre-commit.ci]: https://github.com/apps/pre-commit-ci
