---
name: python
description: General Python correctness and idiomatic pitfalls not specific to any ML library. Consult before writing or reviewing any Python code, especially things like defaults, closures, imports, packaging, asyncio, or exception handling.
---

# Python

Scope: stable CPython behavior that is easy to get subtly wrong. Does not cover ML libraries (see `pytorch`, `torchvision`, `huggingface`, `ray`, `lightning`) or testing (see `python-test-review-write`). Targets Python 3.11+.

## Do / Don't

### Mutable default arguments

Don't. The default binds once at def time and persists across calls.

```python
# Don't
def append_item(item, items=[]):
    items.append(item)
    return items

# Do
def append_item(item, items=None):
    items = [] if items is None else items
    items.append(item)
    return items
```

### Late-binding closures in loops

Don't capture a loop variable and call the closure later. It reads the final value of the binding.

```python
# Don't — all handlers print 3
handlers = [lambda: i for i in range(3)]

# Do
handlers = [lambda i=i: i for i in range(3)]
```

### Circular imports via package `__init__.py`

Don't import submodules inside `__init__.py` in a way that references each other during module setup. Prefer lazy imports inside functions, `from __future__ import annotations`, or type-only imports inside `if TYPE_CHECKING:` blocks when the import is only needed for annotations.

```python
# Don't — a/b import each other at module load
# a.py:  import b    / b.py:  import a

# Do — import inside the function, or import only for types
from __future__ import annotations
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from .b import B

def make_b() -> "B":
    from .b import B  # deferred
    return B()
```

### Packaging

Prefer `pyproject.toml` (`[build-system]`, `[project]`, `[tool.uv]`, `[project.optional-dependencies]`) over legacy `setup.py`. Use `uv` or `pip install -e .` against pyproject. Do not hand-maintain `requirements.txt` plus a `setup.py` that disagree.

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "pkg"
requires-python = ">=3.11"
dependencies = ["requests>=2.31"]
```

### Paths

Prefer `pathlib.Path` over `os.path` string surgery. `Path` gives composable operators and `.read_text()`/`.write_text()` with explicit encoding.

```python
# Do
from pathlib import Path
p = Path("data") / "raw" / "file.txt"
p.parent.mkdir(parents=True, exist_ok=True)
text = p.read_text(encoding="utf-8")
```

### Bare except

Don't catch everything silently. Catch the exception that can actually occur, and never swallow and continue without logging.

```python
# Don't
try:
    resp.raise_for_status()
except:
    pass

# Do
try:
    resp.raise_for_status()
except requests.HTTPError as exc:
    logger.warning("request failed: %s", exc)
    raise
```

### `ExceptionGroup` and chaining

Use `except*` for `ExceptionGroup`, `from` to preserve context in `raise`.

```python
try:
    run_all()
except (ValueError, TypeError) as exc:
    raise RuntimeError("pipeline failed") from exc
```

### dataclasses

Prefer `@dataclasses.dataclass(frozen=True, slots=True)` for immutable value objects; a frozen+slots dataclass gives you `__eq__`, `__repr__`, and hashability for free and prevents accidental mutation. Use plain `NamedTuple` for one-off lightweight records that need ordering.

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float
```

### asyncio: blocking calls inside async

Don't call blocking sync I/O (files, `time.sleep`, `requests`, big CPU work) directly inside `async def` bodies — it stalls the whole event loop. Use `await asyncio.to_thread(...)` / `loop.run_in_executor(...)` for blocking I/O, and `asyncio.sleep` not `time.sleep`.

```python
# Don't — sleeps the loop for everyone
async def slow():
    time.sleep(1)

# Do
async def slow():
    await asyncio.sleep(1)
```

### asyncio: missing await

Don't forget `await` on coroutines or you get a "coroutine was never awaited" warning and silent non-execution. Every call to an `async def` must be awaited or scheduled. Use `asyncio.gather` for concurrent tasks and collect the results — don't just fire tasks without awaiting them.

```python
# Don't
async def main():
    task = fetch(url)  # never runs

# Do
async def main():
    results = await asyncio.gather(fetch(a), fetch(b))
```

## Pitfalls: symptom -> cause -> fix

- `ValueError: generator already executing` -> a generator calling another generator recursively instead of `yield from`/`await` -> restructure with `yield from`.
- `RuntimeError: asyncio.run() cannot be called from a running event loop` -> calling `asyncio.run` inside a coroutine / from a worker thread -> use `await` instead, or `asyncio.get_event_loop().run_until_complete`.
- Names shadowing builtins (`list`, `dict`, `type`) silently changing behavior -> a parameter or variable named like a builtin -> rename.

## Sources

- https://docs.python.org/3/faq/programming.html
- https://docs.python.org/3/reference/compound_stmts.html
- https://docs.python.org/3/tutorial/asyncio.html
- https://docs.python.org/3/library/dataclasses.html
