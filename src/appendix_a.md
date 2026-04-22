# Appendix A: Recommended Tools and Environments

This appendix lists the tools, libraries, and environment configurations used throughout the book, along with installation instructions and recommended alternatives.

---

## A.1 Development Environment

### Python

All code examples in this book use Python 3.11 or later. Python 3.11 introduced significant performance improvements and better error messages; Python 3.12 and 3.13 continue that trajectory. Check your version:

```bash
python --version
```

Install via [python.org](https://www.python.org/downloads/), [pyenv](https://github.com/pyenv/pyenv) (recommended for managing multiple versions), or your system package manager.

**pyenv (recommended)**:
```bash
# Install pyenv (macOS/Linux)
curl https://pyenv.run | bash

# Install Python 3.11
pyenv install 3.11.9
pyenv global 3.11.9
```

### Virtual Environments

Always use a virtual environment for each project:

```bash
# Create
python -m venv .venv

# Activate (macOS/Linux)
source .venv/bin/activate

# Activate (Windows)
.venv\Scripts\activate

# Deactivate
deactivate
```

**Alternative**: [uv](https://github.com/astral-sh/uv) — a fast Python package manager and virtual environment tool written in Rust:
```bash
pip install uv
uv venv
uv pip install anthropic pytest
```

---

## A.2 AI Model APIs

### Anthropic (Claude)

Used for all AI API examples in this book. Sign up at [console.anthropic.com](https://console.anthropic.com/).

```bash
pip install anthropic
```

```bash
# Set API key (add to your shell profile for persistence)
export ANTHROPIC_API_KEY=sk-ant-...
```

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from environment

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.content[0].text)
```

**Model identifiers** (verify current list at [docs.anthropic.com/en/docs/about-claude/models](https://docs.anthropic.com/en/docs/about-claude/models)):

| Model | ID | Best for |
|---|---|---|
| Claude Haiku 4.5 | `claude-haiku-4-5-20251001` | High-volume, simple tasks |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | Feature implementation |
| Claude Opus 4.7 | `claude-opus-4-7` | Architecture review, complex reasoning |

### OpenAI (Equivalent Setup)

The OpenAI SDK follows the same pattern:

```bash
pip install openai
export OPENAI_API_KEY=sk-...
```

```python
from openai import OpenAI

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello"}],
)
print(response.choices[0].message.content)
```

---

## A.3 Testing Tools

### pytest

The standard Python testing framework. Used throughout Chapters 4, 7, and the course project:

```bash
pip install pytest pytest-cov
```

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=src --cov-report=term-missing

# Run specific test file
pytest tests/test_task_service.py

# Run tests matching a keyword
pytest -k "overdue"

# Verbose output
pytest -v
```

### pytest-cov

Coverage reporting for pytest. Generates HTML, XML, and terminal reports:

```bash
# HTML report (open htmlcov/index.html in browser)
pytest --cov=src --cov-report=html
```

### Hypothesis

Property-based testing library. Automatically generates test inputs from specifications (referenced in Chapter 12):

```bash
pip install hypothesis
```

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_idempotent(xs):
    assert sorted(sorted(xs)) == sorted(xs)
```

---

## A.4 Code Quality Tools

### ruff

A fast Python linter and formatter (replaces flake8, isort, and partially black):

```bash
pip install ruff

# Lint
ruff check .

# Format
ruff format .

# Fix automatically
ruff check --fix .
```

Configure in `pyproject.toml`:
```toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "W", "I"]  # pycodestyle, pyflakes, isort
```

### mypy

Static type checker for Python. Catches type errors before runtime:

```bash
pip install mypy

mypy src/
```

Configure in `pyproject.toml`:
```toml
[tool.mypy]
python_version = "3.11"
strict = true
ignore_missing_imports = true
```

### bandit

Security-focused static analyser. Identifies common Python security issues (Chapter 9):

```bash
pip install bandit

# Scan all Python files
bandit -r src/

# With specific severity threshold
bandit -r src/ -l  # low and above
bandit -r src/ -ll # medium and above
bandit -r src/ -lll # high only
```

---

## A.5 Security Tools

### GitLeaks

Scans git history for accidentally committed secrets (API keys, passwords, tokens):

```bash
# macOS
brew install gitleaks

# Linux
# Download from https://github.com/gitleaks/gitleaks/releases

# Scan current repository
gitleaks detect

# Scan with verbose output
gitleaks detect -v
```

### Presidio

Microsoft's PII detection library. Used in Chapter 9 for detecting personally identifiable information in code and text:

```bash
pip install presidio-analyzer presidio-anonymizer
python -m spacy download en_core_web_lg
```

```python
from presidio_analyzer import AnalyzerEngine

analyzer = AnalyzerEngine()
results = analyzer.analyze(
    text="Call me at 555-123-4567 or email me at alice@example.com",
    language="en",
)
for result in results:
    print(f"{result.entity_type}: score={result.score:.2f}")
```

---

## A.6 CI/CD Tools

### GitHub Actions

All CI/CD examples in Chapter 4 use GitHub Actions. No installation required — workflows are defined in `.github/workflows/*.yml` files in your repository.

Key GitHub Actions documentation:
- Workflow syntax: [docs.github.com/en/actions/writing-workflows](https://docs.github.com/en/actions/writing-workflows)
- Starter workflows: [github.com/actions/starter-workflows](https://github.com/actions/starter-workflows)

**Minimal Python CI workflow** (save as `.github/workflows/ci.yml`):
```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install -r requirements.txt
      - run: ruff check .
      - run: mypy src/
      - run: pytest --cov=src
      - run: bandit -r src/ -ll
```

---

## A.7 License Auditing

### pip-licenses

Audits installed package licences (Chapter 10):

```bash
pip install pip-licenses

# Table view
pip-licenses --format=table

# CSV export
pip-licenses --format=csv --output-file=licenses.csv

# Fail if copyleft licences found
pip-licenses --fail-on="GPL;AGPL"
```

---

## A.8 mdBook (This Book)

This book is built with [mdBook](https://rust-lang.github.io/mdBook/), a Rust-based static site generator for documentation and books.

```bash
# Install Rust (required)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Install mdBook
cargo install mdbook

# Serve locally with hot reload
mdbook serve

# Build static site
mdbook build
```

The built book is in `book/` and can be deployed to GitHub Pages, Netlify, or any static hosting service.

---

## A.9 Dependency Management

### requirements.txt (simple projects)

```
anthropic>=0.28.0
pytest>=8.0.0
pytest-cov>=5.0.0
ruff>=0.4.0
mypy>=1.10.0
bandit>=1.7.0
python-dotenv>=1.0.0
```

```bash
pip install -r requirements.txt
```

### pyproject.toml (modern projects)

For projects that need both package metadata and tool configuration:

```toml
[project]
name = "task-management-api"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "anthropic>=0.28.0",
    "fastapi>=0.111.0",
    "sqlalchemy>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-cov>=5.0.0",
    "ruff>=0.4.0",
    "mypy>=1.10.0",
    "bandit>=1.7.0",
]
```

---

## A.10 Environment Variables

All API keys and secrets should be stored in environment variables, never in code. Use a `.env` file for local development (add `.env` to `.gitignore`):

```bash
# .env (never commit this file)
ANTHROPIC_API_KEY=sk-ant-...
DATABASE_URL=postgresql://localhost:5432/tasks
SECRET_KEY=your-secret-key-here
```

```python
# Load in Python
from dotenv import load_dotenv
import os

load_dotenv()  # loads from .env file

api_key = os.environ["ANTHROPIC_API_KEY"]
```

```bash
pip install python-dotenv
```

Verify your `.gitignore` includes:
```
.env
.env.local
*.env
```
