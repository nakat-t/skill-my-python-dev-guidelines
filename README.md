# My Python Development Guidelines (Agent Skill)

An [Agent Skill](https://www.anthropic.com/news/skills) that teaches an AI coding agent to follow my personal Python standards — design, package structure, dependency management, and quality gates.

## What's inside

### Design principles
- **Strict YAGNI.** No speculative abstractions or "future-proofing."
- **Granularity follows the caller's responsibilities,** not implementation convenience.

### OO-first & Pythonic
- Prefer a **Rich Domain Model** (data + behavior) over anemic data containers.
- **Match the public boundary to caller responsibilities:** expose a few "entry points" and hide the internals — like `pathlib.Path`, `requests.Session`, or `httpx.Client`.
- Stay Pythonic: favor composition and duck typing over deep inheritance; avoid Java-style boilerplate.

### Module / package structure
- Prefix non-public files and folders with `_` (e.g. `_invoice.py`).
- Expose only the public entry points via `__init__.py` / `__all__`.

### Environment & toolchain
- Default to **Python 3.12**.
- Use **`uv`** for workspace, package, and dependency management.
- **Never hand-edit `pyproject.toml`** for dependencies — use `uv add` / `uv remove`.
- **Never invent version constraints**; let `uv` resolve to the latest at install time.

### Quality gates
- All code must pass `ruff` (lint + format), `ty` (type check), and `pytest`.
- New or changed behavior always ships with tests focused on the public boundary.
