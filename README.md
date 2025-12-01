# Python Project & Packaging Guide (for Python Devs)

> **Context:** This guide is for client X projects. Names and repos here are generic; adjust to your actual repos when you copy this into a real project.

---

## 1. Why Environment & Dependency Management Matters

Never develop directly in your **global Python** environment.

- You risk:
  - Conflicting versions between projects.
  - “Works on my machine” bugs.
  - Breaking system tools that rely on the global Python.

Always:
- Use **isolated environments / python package managers ** (`venv` or `uv`).
- Track dependencies in a **single source of truth** (e.g. `pyproject.toml` + `uv.lock`), not just ad‑hoc `pip install`.

---

## 2. Classic `venv` + `pip` Workflow (Legacy / Fallback)

This is the traditional approach. You should understand it, but for new projects we’ll prefer **uv**.

```bash
python -m venv .venv
source .venv/bin/activate          # Linux / macOS

# Windows (PowerShell)
py -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Install packages:

```bash
pip install httpx
```

Freeze dependencies:

```bash
pip freeze > requirements.txt
```

Reinstall from `requirements.txt`:

```bash
pip install -r requirements.txt
```

### Problems with this approach

- **Version drift:** A dev installing `httpx` today may get a different version than someone installing it in 3 months (if you don't go with pip freeze, just creating requirements.txt by hand).
- **Manual syncing:** You must remember to:
  - Update `requirements.txt` after every install
  - Re‑run `pip install -r requirements.txt` after every change
- **Human error:** Easy to:
  - Forget to add a new dependency
  - Edit `requirements.txt` by hand and break things

This approach **works**, but is **fragile and not ideal** for team work.

---

## 3. Modern Tooling: Why We Use `uv` (vs Poetry / plain pip)

Tools like **Poetry** improved the situation by adding:

- `pyproject.toml` as a single source of truth
- Dependency resolution + lockfiles
- Packaging support

However, **[uv](https://github.com/astral-sh/uv)** has become the preferred choice for us because:

- Feels familiar to `pip`/`venv` users - it's possible to keep using the pip interface.
- Very fast.
- Handles:
  - Dependency resolution
  - Lockfiles
  - Python version management
  - Project creation & packaging
- Easy to adopt in enterprise/VDI environments.

> In client X’s environment:
> - UV basic functionality has been tested and works.
> - For **build backend** we recommend `hatchling` over the `uv` backend due to issues observed with the latter (see pyproject example below).

---

## 4. Installing `uv`

### 4.1 Using Homebrew (macOS)

```bash
brew install uv
```

### 4.2 Using `pipx` (works well in VDI / constrained environments)

First, install `pipx`:

```bash
# Linux / macOS
python3 -m pip install --user pipx
python3 -m pipx ensurepath

# Windows
py -m pip install --user pipx
py -m pipx ensurepath
```

Then install `uv`:

```bash
pipx install uv
```

> After `ensurepath`, you may need to open a **new terminal** for the changes to take effect.

Check installation:

```bash
uv --version
```

---

## 5. Basic `uv` Workflow

### 5.1 Create a New Project

```bash
uv init my-app
cd my-app
```

If you already have a project:
```bash
cd current-project
uv init
```

`uv init` creates:

- A `pyproject.toml` (project metadata & deps)
- A `.venv` (by default, local virtual environment)
- A basic package structure (under `src/` if configured)
- Git setup (optional, depending on `uv` version/flags)

This gives you **reproducible** and **package-ready** structure from day 1.

---

### 5.2 Add Dependencies

```bash
uv add pandas
```

Effects:

- `pandas` is added to `[project.dependencies]` in `pyproject.toml`.
- `uv` resolves compatible versions and installs dependencies.
- `uv.lock` is updated with an **exact** dependency snapshot.

If you add another library that conflicts with `pandas`, `uv` will attempt to find a compatible resolution or fail fast with a clear error.

---

### 5.3 Remove Dependencies

```bash
uv remove pandas
```

- Removes `pandas` from `pyproject.toml`.
- Updates `uv.lock` accordingly.

---

### 5.4 The `uv.lock` File

`uv.lock` is critical for team environments:

- Contains the **exact resolved versions** of all dependencies.
- Is **not** updated if dependency resolution fails.
- Must be **committed to Git**.

This ensures:

- Any teammate running `uv sync` or `uv run` gets the **same environment**.
- CI/CD and dev environments are consistent.

> **Do not edit `uv.lock` manually.** It’s managed by `uv`, but is human-readable TOML.

---

### 5.5 Running Code with `uv run`

```bash
uv run ./main.py
```

Or any command:

```bash
uv run flask run -p 3000
uv run python scripts/example.py
```

Before running, `uv run` will:

1. Ensure the lockfile is up-to-date with `pyproject.toml`.
2. Ensure the environment matches `uv.lock`.

This removes the need for manual “did you re‑install dependencies?” checks. It **guarantees** your command runs in a consistent environment (there is no need to run `uv sync` then).

---

### 5.6 Syncing the Environment Explicitly

```bash
uv sync
```

This:

- Creates/updates `.venv` to match `uv.lock`.

Then you can manually activate the environment:

```bash
# Linux / macOS
source .venv/bin/activate

# Windows PowerShell
.\.venv\Scripts\Activate.ps1

# Example
flask run -p 3000
python example.py
```

---

## 6. Useful `uv` Commands Cheat Sheet

> For projects with a `pyproject.toml` (i.e., proper Python projects)

```bash
uv init              # Create a new Python project
uv add <pkg>         # Add dependency
uv remove <pkg>      # Remove dependency
uv sync              # Sync environment with uv.lock
uv lock              # (Re)create lockfile
uv run <cmd>         # Run cmd in project environment
uv tree              # Show dependency tree
uv build             # Build distributions (wheel/sdist)
uv publish           # Publish to a package index
```

---

## 7. Important Files & Folders

### 7.1 `.venv/`

- Local virtual environment for the project.
- All project-specific packages go here.
- This is the environment `uv` uses by default.

You *can* use a different active venv:

```bash
export VIRTUAL_ENV=/migration_project/venvs/.my-venv
source /migration_project/venvs/.my-venv/bin/activate
```

Then use the `--active` flag:

```bash
uv run --active python script.py
```

- `--active` tells `uv` to use the currently activated environment instead of `.venv`.
- Useful if `.venv` cannot be created/used (e.g. permission issues).

---

### 7.2 `pyproject.toml` (Core Metadata)

It's in this files where pip, uv, poetry, etc will understand that there is a package (that can be installed afterwards)

- Modern, PEP 621-compliant project configuration.
- Single source of truth for:
  - Project metadata (name, version, description, authors, license, etc.)
  - Dependencies
  - Entry points (scripts/CLIs)
  - Build system (backend, config)

We’ll show full examples in [Section 9](#9-pyprojecttoml-examples).

---

### 7.3 `uv.lock`

- Exact dependency snapshot.
- Checked into Git.
- Used by `uv` to reproduce environments across machines and in CI.

---

### 7.4 `dist/`

- Created by `uv build` / build backend (`hatchling` here).
- Contains built artifacts:
  - `.whl` (wheel files)
  - `.tar.gz` (source distributions, if configured)
- Usually added to `.gitignore`.

---

## 8. Integrating External Sources

### 8.1 Git Dependencies

Add a package directly from Git:

```bash
uv add git+https://github.com/psf/requests
```

Or a specific branch:

```bash
uv add git+https://github.com/encode/httpx --branch main
```

This will be reflected in `pyproject.toml` (see example below).

---

### 8.2 Migrating from `requirements.txt`

If you have a legacy project:

```bash
uv add -r requirements.txt           # Add all dependencies from requirements.txt
uv add -r requirements.txt -c constraints.txt  # With constraints
```

`uv` will:

- Parse `requirements.txt`
- Resolve dependencies
- Save them in `pyproject.toml` and `uv.lock`

---

## 9. `pyproject.toml` Examples

### 9.1 Minimal Project

```toml
[project]
name = "example-pkg"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = []
```

When you run `uv add <package>`, `dependencies` is populated:

```toml
[project]
name = "example-pkg"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
  "pandas==2.2.0"
]
```

---

### 9.2 Recommended Metadata

```toml
[project]
name = "example-pkg"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
license = "MIT"
authors = [{ name = "your-name", email = "your@email.com" }]
requires-python = ">=3.11"
dependencies = [
  "pandas==2.2.0"
]
```

> **Tip:** For real projects, keep `version` and `requires-python` up‑to‑date.

---

### 9.3 Adding a Git Source

```toml
[project]
name = "example-pkg"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
license = "MIT"
authors = [{ name = "your-name", email = "your@email.com" }]
requires-python = ">=3.11"
dependencies = [
  "my-package==0.1.0",
  "httpx",
]

[tool.uv.sources]
httpx = { git = "https://github.com/encode/httpx", branch = "main" }
```

- `httpx` is declared in `dependencies`.
- Its **source** is specified under `[tool.uv.sources]`.

---

### 9.4 Adding Script Entry Points (CLI)

If you want `uv run my-package` or `my-package` as a CLI command:

```toml
[project]
name = "example-pkg"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
license = "MIT"
authors = [{ name = "your-name", email = "your@email.com" }]
requires-python = ">=3.11"
dependencies = [
  "my-package-core==0.1.0",
  "httpx",
]

[project.scripts]
my-package = "my_package:main"

[tool.uv.sources]
httpx = { git = "https://github.com/encode/httpx", branch = "main" }
```

This assumes a structure like:

```text
repo/
  src/
    my_package/
      __init__.py
      __main__.py
      main.py     # contains a function `main()`
```

Then you can run:

```bash
uv run my-package        # via uv
# or, after installing the package:
my-package               # system CLI
```

---

### 9.5 Build Backend with `hatchling`

Due to issues with the `uv` backend in client X, we recommend **hatchling**:

```toml
[project]
name = "example-pkg"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
license = "MIT"
authors = [{ name = "your-name", email = "your@email.com" }]
requires-python = ">=3.11"
dependencies = [
  "my-package-core==0.1.0",
  "httpx",
]

[project.scripts]
my-package = "my_package:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_package"]

[tool.uv.sources]
httpx = { git = "https://github.com/encode/httpx", branch = "main" }
```

- `build-system` tells tools (pip, uv, build, etc.) **how** to build the package.
- `[tool.hatch.build.targets.wheel]` tells `hatchling` where your package lives (`src/my_package`).

---

## 10. Building & Installing Your Package

With a `pyproject.toml` like above, you can build and install your package locally.

### 10.1 Using `uv`

```bash
uv build
```

This will create artifacts under `dist/`:

- e.g. `dist/example_pkg-0.1.0-py3-none-any.whl`

You can then install the built wheel with `uv` or `pip`:

```bash
uv pip install dist/example_pkg-0.1.0-py3-none-any.whl
```

---

### 10.2 Using `pip` Directly (Local Dev)

From the repo root:

```bash
# Activate venv
source .venv/bin/activate

cd my-package-repo
pip install .
```

Now you should be able to run:

```bash
my-package
```

or run modules:

```bash
python -m my_package
```

---

## 11. `uv` vs `pip` in Client X Repos

For **client X**, a reasonable strategy:

- **Repo A (consumer)**:
  - Support both **uv** and **pip**.
  - In `README.md`:
    - Recommend `uv` as the primary workflow.
    - Provide instructions for `pip` fallback (e.g. `requirements.txt`).
  - If the consumer depends on a git repo (e.g. repo B), document:
    - `uv add git+<url>` usage.
    - How to reflect that dependency in `requirements.txt` for pip users (including the git URL).

- **Repo B (producer / library)**:
  - Use **uv only**.
  - Provide:
    - `pyproject.toml` with proper metadata and build-system.
    - Instructions for installing via `uv` or `pip` (from git or built artifact).

There are example repos like `vd-hkkd-test-consumer` / `vd-hkkd-test-producer`. Keep those as reference templates for engineers to consult.

---

## 12. Recommended Practices

1. **Always** work inside a venv:
   - If the project has `uv`, prefer:
     ```bash
     uv sync
     uv run <command>
     ```
2. **Never** install project dependencies globally.
3. Don’t edit `uv.lock` or `pyproject.toml` by hand unless you know what you’re doing.
   - Use `uv add` / `uv remove` instead.
4. Commit:
   - `pyproject.toml`
   - `uv.lock`
   - `README.md`
   - Source code under `src/…`
5. Don’t commit:
   - `.venv/`
   - `dist/`
   - `__pycache__/`
6. For bug reports:
   - Capture:
     - Exact `uv` command
     - Output
     - `python --version`
   - This makes debugging much easier.

---

## 13. Quick Start Summary

For a **new project**:

```bash
uv init my-app
cd my-app
uv add httpx pandas
uv run python -c "import httpx, pandas; print('OK')"
```

For an **existing project** (with `pyproject.toml` and `uv.lock`):

```bash
uv sync
uv run pytest          # or any command defined in scripts
```

For **pip‑only users** (fallback):

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

> **Team rule:** Prefer `uv` for all new work. Use pip only if there is a strong reason (e.g. tooling limitation) and follow the documented fallback path.

---
