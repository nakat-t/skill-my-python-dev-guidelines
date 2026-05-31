---
name: my-python-dev-guidelines
description: "My Python development standards. Always consult this skill for any work that involves writing, designing, refactoring, or reviewing Python code, deciding package structure, or adding dependencies. Specifically: creating or editing Python files / packages / modules, designing classes or public APIs, managing workspaces / packages / dependencies with uv, editing pyproject.toml, running ruff / ty / pytest, and any design decision involving Rich Domain Models or public boundaries (public symbols). Consult this skill for any Python implementation or design task, even when the user does not explicitly say 'guidelines'."
---

# Python Development Guidelines

The standards I follow when writing and designing Python. Design according to these principles before you start writing code, and once you're done, back it up with tool checks and tests.

## Design principles to always keep in mind

- **Strictly follow YAGNI.** Don't create abstractions, parameters, or extension points justified by "we might use it in the future." Speculative Generality is forbidden. Implement only the responsibilities needed right now.
- **Decide granularity by "responsibility as seen from the caller's side."** Don't decide how to split classes or modules based on implementation convenience. Start from what responsibilities the caller expects, and split along those lines. When implementation convenience conflicts with the caller's expectations, prioritize the caller's expectations.

These two are the judgment criteria behind all the concrete rules below. When in doubt, return to them.

## Environment and toolchain

- **The default Python version is 3.12.** Follow a different version only when the user explicitly specifies one.
- **Use `uv` for workspace management and package management.** Do everything through uv: creating virtual environments, resolving dependencies, and running commands (`uv run ...`).
- **Do not directly edit `pyproject.toml` for the purpose of adding, changing, or updating dependencies.** This is an absolute rule. Editing it by hand leads to outdated or vague versions, so always do it via commands as shown below, so that the latest version at the time of execution is recorded in pyproject.toml:
  - Add: `uv add <package>` (**package name only; do not write a version specifier**)
  - Remove: `uv remove <package>`
  - (Editing `pyproject.toml` is allowed only for metadata unrelated to dependencies — project name, script entry points, tool settings, and the like.)
- **The AI must not decide and write version constraints for dependency packages.** Unless explicitly specified by the user, do not, on the AI's own judgment, add minimum versions, upper bounds, ranges, or other version constraints. This is as important as the previous rule.
  - Writing version specifiers (`>=`, `==`, `~=`, `<`, etc.) inside the arguments of a `uv add` command is **forbidden**, just as hand-writing `pyproject.toml` is. Specifying versions via arguments like `uv add "pkg>=1.2.3"` to avoid editing `pyproject.toml` is a loophole and is not allowed.
  - Correct: `uv add fastapi pydantic "uvicorn[standard]" pyyaml` (square brackets denoting extras, like `[standard]`, are fine to include — these are not version constraints)
  - Wrong: `uv add "fastapi>=0.136.3" "pydantic>=2.13.4" "uvicorn[standard]>=0.48.0" "pyyaml>=6.0"`
  - **Why the AI must not decide versions:** There is a Python convention that "you should set an appropriate range of supported versions." But when the AI decides that "appropriate range," it ends up just guessing plausible numbers commonly seen in the many examples on GitHub and elsewhere — writing versions with no clear basis. That can be harmful. Leave version resolution to `uv add` and **assume the latest version at the time of execution**; that is the correct approach. If well-grounded constraints are needed, the user — who can judge that — should specify them explicitly.
  - Therefore, even in situations where a specific version seems necessary, the AI must not specify a version on its own; first run `uv add` with no version constraint. If there seems to be a reason a version constraint is needed, confirm with the user before running the command.
- **Always pass the `ruff` and `ty` checks.** After writing code, run lint / format / type checks, eliminate the findings, and only then consider it done. Don't skip the checks.
  - Example: `uv run ruff check .` / `uv run ruff format .` / `uv run ty check`

## Design: OO-first and Pythonic

- **Design in a Pythonic, OO-first way.** Rather than a data container with no behavior (an Anemic Domain Model), use a **Rich Domain Model** that keeps data and the responsibilities for that data in the same object.
- **However, the Rich Domain Model is a means, not an end.** It is a tool for organizing the **internals** of a module / package. Don't make "creating lots of rich objects" itself the goal.

### Match the granularity of the public boundary to the caller's responsibilities

This is the heart of these guidelines.

- **Match the granularity of the public boundary (the surface visible to the caller) to the granularity of the responsibilities the caller expects.** If the caller expects a certain responsibility as "one thing," expose a **single Rich Domain Object** that carries that responsibility, and hide the multiple objects that compose it and their interrelationships behind it.
- **Splitting things richly on the inside is good. Just don't let it leak to the public surface.** It's fine to split the internals into six objects for organization. The problem is a state where, to use a single responsibility, the caller has to understand those six objects and their interrelationships. That is over-design, so avoid it.
- **Modern OO APIs to use as models:** `pathlib.Path`, `requests.Session`, `httpx.Client`, SQLAlchemy's Session / Engine, and the like. However complex their internals, they show the caller only a small number of entry-point objects, one per responsibility. Emulate this stance of "narrowing the entry points."

**Bad example (internal structure leaks to the public surface):**
```python
# Caller side — forced to understand 6 concepts just to get a total
from billing import Invoice, LineItem, TaxPolicy, Discount, Currency, RoundingRule

invoice = Invoice(
    items=[LineItem(name="A", unit_price=1000, qty=2)],
    tax_policy=TaxPolicy(rate=0.10),
    discounts=[Discount(kind="coupon", amount=200)],
    currency=Currency("JPY"),
    rounding=RoundingRule.floor(),
)
total = invoice.total()
```

**Good example (narrow the responsibility's entry point to one, hide the internal richness):**
```python
# Caller side — sees only the entry point for the single responsibility "handle billing amounts"
from billing import Billing

billing = Billing.for_currency("JPY")
billing.add_line("A", unit_price=1000, qty=2)
billing.apply_coupon(amount=200)
total = billing.total()   # tax, discounts, and rounding are all handled internally
```
`LineItem` / `TaxPolicy` / `RoundingRule` and the like may exist internally as Rich Domain Objects. Just place them in `_`-prefixed modules and do not make them public.

### Stay Pythonic

- Avoid unnecessary inheritance. In most cases composition or protocol / duck typing is enough.
- Avoid over-design and formalistic, Java-style code (meaningless getters/setters, Factories for everything, overuse of abstract base classes, etc.).
- Lean on the conventions of the standard library (`dataclass`, `@property`, context managers, the iterator protocol, etc.).

## Module / package structure

- **Name non-public files and folders used for implementation splitting with a leading `_`.** Examples: `_internal.py`, `_billing/`. These signal "don't touch from outside."
- **Expose public symbols via `__init__.py`.** Use something like `__all__` in `__init__.py` to list only the responsibility entry points the caller needs. Don't put internal composing objects on the public list.
- Non-public symbols do not need names that start with `_`. For example, the `LineItem` class can be defined normally as `class LineItem:` inside `_invoice.py`. However, it must not be visible to the caller under the name `billing.LineItem`. Keep public symbols strictly to "responsibility entry points."
- The result you're aiming for: when a caller imports the package, all they see are the "responsibility entry points."

```
billing/
├── __init__.py        # exposes only Billing
├── _invoice.py        # internal Rich Objects like LineItem, Invoice
├── _tax.py            # TaxPolicy
└── _rounding.py       # RoundingRule
```

## Tests

- **When you add or change functionality, always implement pytest tests that verify that behavior alongside it.** Don't call a feature addition "done" without tests.
- Center tests on the behavior of the public boundary (the entry points the caller uses). Avoid tests that cling too closely to the details of internal objects, since they get in the way of internal refactoring.
- **Confirm that all tests pass before considering the work done.** Example: `uv run pytest`

## Finishing checklist

Before considering a Python implementation or change "done," confirm that all of the following are satisfied:

1. Does it follow YAGNI, with no speculative abstractions introduced?
2. Are the exposed symbols narrowed to the caller's responsibility entry points, with internal composing objects hidden behind a leading `_`?
3. Were dependency additions / changes done via `uv add` / `uv remove`, without hand-editing `pyproject.toml`? And, except where explicitly specified by the user, are there no version constraints (like `>=`) written on the AI's own judgment — neither in `uv add` arguments nor in `pyproject.toml`?
4. Do `ruff check` / `ruff format` / `ty check` pass?
5. Are there pytest tests for the added / changed behavior, and does `pytest` pass entirely?
