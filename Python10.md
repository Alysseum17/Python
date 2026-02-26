# Day 10: Virtual Environments, Package Management & Project Structure

## 🎯 Goal

Learn how Python manages dependencies, isolates projects, and structures code. Coming from JS (npm/yarn/pnpm) and Java (Maven/Gradle), the concepts are familiar — but Python has its own ecosystem with more choices. Today you'll learn which tools to actually use and how to set up a professional project.

---

## 1. Why Virtual Environments?

### The problem

```bash
# Python packages install GLOBALLY by default
pip install requests==2.28.0  # project A needs this
pip install requests==2.31.0  # project B needs this — overwrites A's version!

# In JS you don't have this problem because npm installs to ./node_modules
# In Java, Maven/Gradle manage per-project dependencies in ~/.m2 or build dirs
# Python needs virtual environments to isolate projects
```

### What a virtual environment does

```
A virtual environment is a self-contained directory with:
  - Its own Python interpreter (or symlink)
  - Its own pip
  - Its own site-packages/ directory
  
When activated, all pip installs go INTO that directory,
not into the global Python installation.

Think of it as a project-local node_modules — but for Python.
```

---

## 2. `venv` — Built-in Virtual Environments

The standard tool, ships with Python. No installation needed.

### Creating and using venv

```bash
# Create virtual environment
python3 -m venv .venv
# Creates .venv/ directory with isolated Python
# Convention: name it .venv or venv (the dot hides it in file browsers)

# Activate it
# Linux/macOS:
source .venv/bin/activate
# Windows:
.venv\Scripts\activate

# Your prompt changes to show the active env:
# (.venv) user@machine:~/project$

# Now pip installs go to .venv/
pip install requests
pip install flask

# Check where packages are installed
pip show requests
# Location: /home/user/project/.venv/lib/python3.12/site-packages

# Deactivate when done
deactivate
```

### Managing dependencies

```bash
# Freeze current packages to a file (like npm's package-lock.json)
pip freeze > requirements.txt

# The file looks like:
# certifi==2024.2.2
# charset-normalizer==3.3.2
# requests==2.31.0
# urllib3==2.2.0

# Install from requirements file (like npm install)
pip install -r requirements.txt

# Upgrade a package
pip install --upgrade requests

# Uninstall
pip uninstall requests

# List installed packages
pip list

# Show package info
pip show requests
```

### The `requirements.txt` approach — simple but limited

```bash
# requirements.txt — your dependencies
requests>=2.28,<3.0
flask==3.0.0
pydantic>=2.0
sqlalchemy[asyncio]>=2.0

# requirements-dev.txt — development dependencies
-r requirements.txt     # include production deps
pytest>=7.0
mypy>=1.0
ruff>=0.1.0
black>=23.0

# Install production deps:
pip install -r requirements.txt

# Install everything (dev included):
pip install -r requirements-dev.txt
```

### Problems with plain pip + requirements.txt

```
1. No dependency resolution — pip doesn't always find compatible versions
2. No lock file — requirements.txt doesn't pin transitive dependencies
3. No separation of direct vs transitive deps
4. No dev/prod distinction built-in
5. Manual process — must remember to freeze after installs

This is like npm without package-lock.json — it works but is fragile.
That's why modern tools exist: Poetry, uv, pdm, etc.
```

---

## 3. Poetry — Modern Dependency Management

Poetry is the most popular modern Python package manager. It's like npm/yarn but for Python.

### Installation

```bash
# Official installer (recommended)
curl -sSL https://install.python-poetry.org | python3 -

# Or with pipx (isolated tool installation)
pipx install poetry
```

### Creating a project

```bash
# New project from scratch
poetry new myproject
# Creates:
# myproject/
# ├── pyproject.toml      ← project config (like package.json)
# ├── README.md
# ├── myproject/
# │   └── __init__.py
# └── tests/
#     └── __init__.py

# Or init in existing directory
cd existing-project
poetry init    # interactive setup
```

### `pyproject.toml` — the modern config file

```toml
[tool.poetry]
name = "myproject"
version = "0.1.0"
description = "A data pipeline project"
authors = ["Your Name <you@example.com>"]
readme = "README.md"
python = "^3.11"

[tool.poetry.dependencies]
python = "^3.11"
requests = "^2.31"
pydantic = "^2.0"
sqlalchemy = {version = "^2.0", extras = ["asyncio"]}

[tool.poetry.group.dev.dependencies]
pytest = "^7.0"
mypy = "^1.0"
ruff = "^0.1"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

# Compare with package.json:
# dependencies     → [tool.poetry.dependencies]
# devDependencies  → [tool.poetry.group.dev.dependencies]
# package-lock.json → poetry.lock
```

### Daily Poetry commands

```bash
# Add dependency (like npm install <package>)
poetry add requests
poetry add pydantic sqlalchemy

# Add dev dependency (like npm install -D <package>)
poetry add --group dev pytest mypy ruff

# Remove dependency
poetry remove requests

# Install all dependencies from lock file (like npm ci)
poetry install

# Update dependencies (like npm update)
poetry update                # update all
poetry update requests       # update specific package

# Show installed packages
poetry show
poetry show --tree           # dependency tree (very useful!)

# Run a command in the virtual environment
poetry run python main.py
poetry run pytest
poetry run mypy src/

# Activate the venv shell (like entering node_modules/.bin)
poetry shell

# Export to requirements.txt (for Docker, legacy systems)
poetry export -f requirements.txt -o requirements.txt --without-hashes
```

### Poetry lock file

```bash
# poetry.lock — auto-generated, pins EXACT versions of ALL dependencies
# This is like package-lock.json or yarn.lock
# ALWAYS commit poetry.lock to git!

# When you run 'poetry install', it uses the lock file
# This guarantees everyone gets identical dependencies

# To update lock file after manual pyproject.toml edits:
poetry lock
```

---

## 4. `uv` — The Fast New Kid

`uv` is a new tool from the creators of `ruff`. It's written in Rust and is **10-100x faster** than pip/Poetry. It's gaining adoption fast.

### Installation

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Or with pip
pip install uv
```

### Using uv

```bash
# Create a new project
uv init myproject
cd myproject

# Add dependencies (writes to pyproject.toml automatically)
uv add requests pydantic
uv add --dev pytest ruff mypy

# Remove dependencies
uv remove requests

# Install/sync all dependencies from lock file
uv sync

# Run commands
uv run python main.py
uv run pytest

# Create virtual environment explicitly
uv venv

# uv also replaces pip for installation:
uv pip install requests        # 10-100x faster than pip!
uv pip install -r requirements.txt
```

### uv's `pyproject.toml`

```toml
[project]
name = "myproject"
version = "0.1.0"
description = "My project"
requires-python = ">=3.11"
dependencies = [
    "requests>=2.31",
    "pydantic>=2.0",
]

[dependency-groups]
dev = [
    "pytest>=7.0",
    "ruff>=0.1",
    "mypy>=1.0",
]

# Note: uv uses standard [project] table (PEP 621)
# Poetry uses [tool.poetry] (custom)
# uv's approach is more future-proof and standard-compliant
```

---

## 5. pyenv — Manage Python Versions

Like `nvm` for Node.js — lets you install and switch between Python versions.

```bash
# Install pyenv
curl https://pyenv.run | bash

# List available Python versions
pyenv install --list

# Install specific version
pyenv install 3.12.1
pyenv install 3.11.7

# Set global default
pyenv global 3.12.1

# Set local version for a project (creates .python-version file)
cd myproject
pyenv local 3.11.7

# List installed versions
pyenv versions

# Which python is active?
pyenv which python
```

---

## 6. Tool Comparison — What to Use

```
┌─────────────────────────────────────────────────────────┐
│              Which tool should you use?                   │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Beginner / simple project:                              │
│    → venv + pip + requirements.txt                       │
│    → Simple, built-in, everyone understands              │
│                                                          │
│  Professional / team project:                            │
│    → Poetry or uv                                        │
│    → Lock files, dependency resolution, dev/prod split   │
│                                                          │
│  Fastest / cutting-edge:                                 │
│    → uv                                                  │
│    → Rust-powered, 10-100x faster, standard pyproject    │
│                                                          │
│  Multiple Python versions:                               │
│    → pyenv + any of the above                            │
│                                                          │
│  Data Science:                                           │
│    → conda (manages non-Python deps like CUDA, etc.)     │
│                                                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  My recommendation for you:                              │
│    → uv (fast, modern, standard-compliant)               │
│    → pyenv for Python version management                 │
│    → Know Poetry too (many projects use it)              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Comparison table

| Feature | pip + venv | Poetry | uv |
|---------|-----------|--------|-----|
| Speed | Slow | Medium | Very fast |
| Lock file | ❌ (manual freeze) | ✅ poetry.lock | ✅ uv.lock |
| Dependency resolution | Basic | Good | Good |
| Dev/prod groups | Manual | ✅ | ✅ |
| pyproject.toml | Partial | Custom format | Standard (PEP 621) |
| Virtual env management | Manual | Automatic | Automatic |
| Build & publish | Manual | ✅ | ✅ |
| Adoption | Universal | Very popular | Growing fast |

### Comparison with JS/Java

```
npm/yarn/pnpm  →  pip/Poetry/uv
package.json   →  pyproject.toml
node_modules   →  .venv/
package-lock   →  poetry.lock / uv.lock
npx            →  poetry run / uv run
nvm            →  pyenv
.nvmrc         →  .python-version

Maven/Gradle   →  pip/Poetry/uv
pom.xml        →  pyproject.toml
~/.m2          →  .venv/
mvn test       →  poetry run pytest / uv run pytest
```

---

## 7. `pyproject.toml` — The Universal Config File

`pyproject.toml` is Python's answer to `package.json`. It can configure everything.

```toml
# ============================================================
# Project metadata (PEP 621 standard)
# ============================================================
[project]
name = "my-data-pipeline"
version = "0.1.0"
description = "ETL pipeline for analytics data"
authors = [{name = "Your Name", email = "you@example.com"}]
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.11"
dependencies = [
    "pydantic>=2.0",
    "sqlalchemy>=2.0",
    "requests>=2.31",
]

[dependency-groups]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "mypy>=1.0",
    "ruff>=0.4",
]

# ============================================================
# Tool-specific configuration
# ============================================================

# Ruff — linter and formatter (replaces flake8, isort, black)
[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort (import sorting)
    "N",    # naming conventions
    "UP",   # pyupgrade
    "B",    # bugbear (common bugs)
    "SIM",  # simplify
]

# Mypy — type checker
[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
disallow_untyped_defs = true

# Pytest
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"

# ============================================================
# Build system
# ============================================================
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

---

## 8. Code Quality Tools

### Ruff — The All-in-One Linter & Formatter

Ruff replaces flake8, isort, black, pyflakes, and more. Written in Rust — incredibly fast.

```bash
# Install
uv add --dev ruff
# or
pip install ruff

# Lint (find issues)
ruff check .
ruff check src/

# Lint and auto-fix
ruff check --fix .

# Format (like black/prettier)
ruff format .

# Check formatting without changing files
ruff format --check .
```

```python
# Example: before ruff format
def   messy_function(  x,y,z   ):
    result=x+y+z
    return(result)
import os
import sys
from pathlib import Path
import json

# After ruff format + ruff check --fix
import json
import os
import sys
from pathlib import Path


def messy_function(x, y, z):
    result = x + y + z
    return result
```

### Black — The Opinionated Formatter

Black was the standard before Ruff. Still widely used.

```bash
pip install black

# Format files
black .
black src/ tests/

# Check without formatting
black --check .

# See what would change
black --diff .
```

### Mypy — Static Type Checker

```bash
pip install mypy

# Check types
mypy src/
mypy src/ --strict    # strict mode

# Example errors mypy catches:
def add(a: int, b: int) -> int:
    return a + b

add("hello", "world")  # mypy: error: Argument 1 has incompatible type "str"; expected "int"
```

### Pre-commit — Run Checks Automatically on Git Commit

```bash
pip install pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.9.0
    hooks:
      - id: mypy
        additional_dependencies: [pydantic>=2.0]
```

```bash
# Install hooks
pre-commit install

# Now every git commit automatically runs ruff + mypy
# If checks fail, commit is blocked until you fix issues

# Run manually on all files
pre-commit run --all-files
```

---

## 9. Project Structure — Professional Layout

### Simple project (script/CLI tool)

```
my-tool/
├── pyproject.toml
├── README.md
├── .gitignore
├── .python-version         # pyenv
├── src/
│   └── my_tool/
│       ├── __init__.py
│       ├── main.py
│       ├── config.py
│       └── utils.py
└── tests/
    ├── __init__.py
    ├── test_main.py
    └── test_utils.py
```

### Data Engineering project

```
data-pipeline/
├── pyproject.toml
├── README.md
├── .gitignore
├── .python-version
├── .env.example            # template for environment variables
├── docker-compose.yml      # local development (Postgres, Redis, etc.)
├── Dockerfile
│
├── src/
│   └── pipeline/
│       ├── __init__.py
│       ├── config.py           # Pydantic Settings
│       ├── exceptions.py       # custom exceptions
│       │
│       ├── models/             # Pydantic models / schemas
│       │   ├── __init__.py
│       │   ├── events.py
│       │   └── users.py
│       │
│       ├── extractors/         # data sources
│       │   ├── __init__.py
│       │   ├── base.py         # abstract extractor
│       │   ├── api.py          # REST API extractor
│       │   ├── database.py     # DB extractor
│       │   └── files.py        # file extractor (CSV, JSON, etc.)
│       │
│       ├── transformers/       # data transformations
│       │   ├── __init__.py
│       │   ├── base.py
│       │   ├── cleaning.py
│       │   └── enrichment.py
│       │
│       ├── loaders/            # data destinations
│       │   ├── __init__.py
│       │   ├── base.py
│       │   ├── postgres.py
│       │   └── s3.py
│       │
│       └── utils/
│           ├── __init__.py
│           ├── logging.py
│           └── retry.py
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py             # pytest fixtures
│   ├── test_extractors/
│   ├── test_transformers/
│   └── test_loaders/
│
├── scripts/                    # one-off scripts, migrations
│   ├── seed_db.py
│   └── run_pipeline.py
│
└── data/                       # local data (gitignored)
    ├── raw/
    ├── processed/
    └── samples/                # small sample data for tests
```

### Web application (FastAPI / Django)

```
web-app/
├── pyproject.toml
├── README.md
├── .gitignore
├── .env.example
├── Dockerfile
├── docker-compose.yml
│
├── src/
│   └── app/
│       ├── __init__.py
│       ├── main.py             # FastAPI app entry point
│       ├── config.py
│       ├── dependencies.py     # dependency injection
│       │
│       ├── models/             # SQLAlchemy models (DB schema)
│       │   ├── __init__.py
│       │   └── user.py
│       │
│       ├── schemas/            # Pydantic models (API request/response)
│       │   ├── __init__.py
│       │   └── user.py
│       │
│       ├── routers/            # API endpoints
│       │   ├── __init__.py
│       │   ├── auth.py
│       │   └── users.py
│       │
│       ├── services/           # business logic
│       │   ├── __init__.py
│       │   └── user_service.py
│       │
│       └── db/                 # database
│           ├── __init__.py
│           ├── session.py
│           └── migrations/     # Alembic migrations
│
├── tests/
│   ├── conftest.py
│   ├── test_routers/
│   └── test_services/
│
└── alembic.ini
```

---

## 10. `.gitignore` for Python

```gitignore
# Virtual environments
.venv/
venv/
env/

# Python
__pycache__/
*.py[cod]
*$py.class
*.egg-info/
dist/
build/
*.egg

# IDE
.idea/
.vscode/
*.swp
*.swo

# Environment variables
.env
.env.local
.env.*.local

# Data (large files)
data/raw/
data/processed/
*.csv
*.parquet
*.pkl

# OS
.DS_Store
Thumbs.db

# Testing
.coverage
htmlcov/
.pytest_cache/

# Type checking
.mypy_cache/

# Distribution
dist/
build/
```

---

## 11. Setting Up a New Project — Step by Step

### With uv (recommended)

```bash
# 1. Create project
uv init my-data-pipeline
cd my-data-pipeline

# 2. Set Python version (if using pyenv)
pyenv local 3.12.1

# 3. Add dependencies
uv add pydantic sqlalchemy requests
uv add --dev pytest ruff mypy

# 4. Create project structure
mkdir -p src/pipeline/{models,extractors,transformers,loaders,utils}
mkdir -p tests
touch src/pipeline/__init__.py
touch src/pipeline/models/__init__.py
touch src/pipeline/config.py
touch src/pipeline/exceptions.py

# 5. Configure tools in pyproject.toml (add ruff, mypy, pytest sections)

# 6. Initialize git
git init
# Create .gitignore (see above)

# 7. Create .env.example
echo "DATABASE_URL=postgresql://user:pass@localhost:5432/mydb" > .env.example

# 8. Run your code
uv run python -m pipeline.main
uv run pytest
uv run ruff check .
uv run mypy src/
```

### With Poetry

```bash
# 1. Create project
poetry new my-data-pipeline
cd my-data-pipeline

# 2. Set Python version
pyenv local 3.12.1

# 3. Add dependencies
poetry add pydantic sqlalchemy requests
poetry add --group dev pytest ruff mypy

# 4. Create project structure (same as above)

# 5-8. Same as above, but use 'poetry run' instead of 'uv run'
poetry run pytest
poetry run ruff check .
```

---

## 12. Practical Workflow

### Daily development workflow

```bash
# Start of day
cd my-project
source .venv/bin/activate  # or: poetry shell / uv shell

# Write code...

# Run tests
pytest                     # or: uv run pytest
pytest tests/test_specific.py -v    # specific test file
pytest -k "test_function_name"      # specific test by name

# Check code quality
ruff check .               # lint
ruff format .              # format
mypy src/                  # type check

# Before committing
ruff check . && ruff format --check . && mypy src/ && pytest

# Commit
git add .
git commit -m "feat: add user extraction pipeline"
```

### Adding a new dependency

```bash
# With uv
uv add pandas              # add to project deps
uv add --dev pytest-mock   # add to dev deps
# uv.lock is auto-updated

# With Poetry
poetry add pandas
poetry add --group dev pytest-mock
# poetry.lock is auto-updated

# With pip (manual approach)
pip install pandas
pip freeze > requirements.txt   # don't forget this step!
```

---

## 📝 Practice Tasks

### Task 1: Project Setup
Create a new Python project from scratch using uv (or Poetry). Set up:
- `pyproject.toml` with metadata, dependencies, and tool configs
- Project structure for a "URL shortener" service
- ruff, mypy, pytest configuration
- `.gitignore`, `.env.example`
- A basic `main.py` that prints "Hello from {project_name} v{version}"

### Task 2: Dependency Investigation
For your project, add these dependencies and explore them:
- `requests` — make a real HTTP GET request
- `pydantic` — create a config model
- `rich` — pretty-print a table of installed packages
Run `uv show --tree` (or `poetry show --tree`) and understand the dependency tree.

### Task 3: Code Quality Pipeline
Write a Python module with intentional issues (bad formatting, type errors, unused imports, naming violations). Then:
- Run `ruff check` and fix all issues
- Run `ruff format` and see the changes
- Add type hints and run `mypy --strict`
- Make everything pass with zero warnings

### Task 4: Multi-Environment Config
Create a Pydantic Settings class that reads from:
1. Default values in code
2. `.env` file (override defaults)
3. Environment variables (override .env)
Support three environments: dev, staging, prod — each with different defaults for database URL, debug mode, log level.

### Task 5: Package Your Project
Take any project from previous days and properly package it:
- Move code into `src/` layout
- Add `pyproject.toml` with all metadata
- Add entry point so it can be run as a CLI tool
- Install it in editable mode (`uv pip install -e .` or `pip install -e .`)
- Run it as a command: `my-tool --help`

### Task 6: Makefile / Task Runner
Create a `Makefile` (or `justfile`) with common tasks:
```makefile
.PHONY: install test lint format check all

install:
	uv sync

test:
	uv run pytest

lint:
	uv run ruff check .

format:
	uv run ruff format .

typecheck:
	uv run mypy src/

check: lint typecheck test  # run all checks

clean:
	find . -type d -name __pycache__ -exec rm -rf {} +
	rm -rf .pytest_cache .mypy_cache .ruff_cache
```

---

## 📚 Resources

- [Python Packaging User Guide](https://packaging.python.org/)
- [pyproject.toml specification (PEP 621)](https://peps.python.org/pep-0621/)
- [Poetry Documentation](https://python-poetry.org/docs/)
- [uv Documentation](https://docs.astral.sh/uv/)
- [Ruff Documentation](https://docs.astral.sh/ruff/)
- [Real Python — Virtual Environments](https://realpython.com/python-virtual-environments-a-primer/)
- [Real Python — pip](https://realpython.com/what-is-pip/)
- [Mypy Documentation](https://mypy.readthedocs.io/)
- [pyenv GitHub](https://github.com/pyenv/pyenv)
- [Hypermodern Python (excellent project setup guide)](https://cjolowicz.github.io/posts/hypermodern-python-01-setup/)

---

## 🔑 Key Takeaways

1. **Always use a virtual environment** — never install project deps globally.
2. **`uv` is the future** — fast, standard-compliant, handles everything. Learn it now.
3. **Know Poetry too** — many existing projects use it, you'll encounter it.
4. **`pyproject.toml` is the one config file** — project metadata, deps, tool configs, all in one place.
5. **Ruff replaces 5+ tools** — linter, formatter, import sorter, all in one. Use it.
6. **Always have a lock file** — `poetry.lock` or `uv.lock`. Commit it to git.
7. **`src/` layout** — put your code in `src/package_name/`, not at the root. Avoids import confusion.
8. **Code quality from day one** — ruff + mypy + pytest. Set up before writing code, not after.
9. **`.env` for secrets** — never commit secrets. Use `.env.example` as template.
10. **`Makefile`** — standardize commands across the team. `make test`, `make lint`, `make format`.

---

> **Tomorrow (Day 11):** Threading & Multiprocessing — the GIL, `threading`, `multiprocessing`, `concurrent.futures`, and when to use what. This is where Python gets tricky.
