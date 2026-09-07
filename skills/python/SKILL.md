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
def append_item(item: str, items: list[str] = []) -> list[str]:
    items.append(item)
    return items

# Do
def append_item(item: str, items: list[str] | None = None) -> list[str]:
    items = [] if items is None else items
    items.append(item)
    return items
```

The same trap bites `dict` defaults:

```python
# Don't — shared across calls
def register(name: str, opts: dict[str, str] = {}) -> dict[str, str]:
    opts[name] = "on"
    return opts

# Do
def register(name: str, opts: dict[str, str] | None = None) -> dict[str, str]:
    opts = {} if opts is None else opts
    opts[name] = "on"
    return opts
```

### Late-binding closures in loops

Don't capture a loop variable and call the closure later — it reads the final value of the binding. Bind the current value as a default arg, or use `functools.partial`.

```python
# Don't — all handlers print 3
handlers = [lambda: i for i in range(3)]

# Do
handlers = [lambda i=i: i for i in range(3)]
# or
from functools import partial
handlers = [partial(print, i) for i in range(3)]
```

### Circular imports

Avoid circular imports by structuring modules so dependencies run one way: keep a clear layering (e.g. `models` -> `schemas` -> `utils`), put shared types in a leaf module that nothing imports back, and import only what you need at module top level. Reach for deferred (in-function) imports or `if TYPE_CHECKING:` only as a last resort when a genuine cycle can't be designed away.

```python
# Don't — a.py and b.py import each other at module load
# a.py: import b    /  b.py: import a

# Do — shared types live in a leaf module both depend on; no cycle
# schemas.py
from __future__ import annotations
from dataclasses import dataclass

@dataclass(frozen=True)
class User:
    name: str

# a.py / b.py
from .schemas import User   # one-way dependency through a leaf
```

If a cycle is truly unavoidable, defer the import inside the function that needs it, and use `from __future__ import annotations` + `if TYPE_CHECKING:` for type-only imports.

### Packaging

Use `pyproject.toml` as the single source of project metadata and dependencies, and manage the environment with `uv`. Don't hand-maintain `requirements.txt` or `setup.py` — they drift from `pyproject.toml` and from each other.

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "pkg"
requires-python = ">=3.11"
dependencies = ["requests>=2.31"]

[tool.uv]
dev-dependencies = ["pytest>=8", "mypy>=1.10"]
```

Add dependencies with `uv add <pkg>` (updates `pyproject.toml` and the lockfile) and sync with `uv sync`. Avoid ceiling pins (`<X`) unless a known incompatibility forces it — prefer floors (`>=X`) so transitive updates can resolve.

If you use `pixi` instead of `uv` (conda-forge environments), declare the pixi project in the same `pyproject.toml`:

```toml
[tool.pixi.project]
channels = ["conda-forge"]
platforms = ["linux-64", "osx-arm64", "win-64"]

[tool.pixi.dependencies]
python = ">=3.11"
numpy = "*"

[tool.pixi.pypi-dependencies]
pkg = { path = ".", editable = true }
```

### Paths

Prefer `pathlib.Path` over `os.path` string surgery. `Path` gives composable operators and `.read_text()`/`.write_text()` with explicit encoding.

```python
# Do
from pathlib import Path

def load_config() -> str:
    p = Path("data") / "raw" / "file.txt"
    p.parent.mkdir(parents=True, exist_ok=True)
    return p.read_text(encoding="utf-8")
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
def run_all() -> None: ...

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

### Typed containers

Don't pass around untyped `dict`s when the shape is fixed and known — use `TypedDict` (or a frozen dataclass) so callers get type-checked fields.

```python
# Don't — untyped dict; callers must remember key names and value types
def make_user(raw: dict) -> dict:
    return {"name": raw["name"], "age": int(raw["age"])}

# Do
from typing import TypedDict

class User(TypedDict):
    name: str
    age: int

def make_user(raw: dict[str, object]) -> User:
    return {"name": str(raw["name"]), "age": int(raw["age"])}
```

### asyncio: blocking calls inside async

Don't call blocking sync I/O (files, `time.sleep`, `requests`, big CPU work) directly inside `async def` bodies — it stalls the whole event loop. Use `await asyncio.to_thread(...)` / `loop.run_in_executor(...)` for blocking I/O, and `asyncio.sleep` not `time.sleep`.

```python
# Don't — sleeps the loop for everyone
async def slow() -> None:
    time.sleep(1)

# Do
async def slow() -> None:
    await asyncio.sleep(1)
```

### asyncio: missing await

Don't forget `await` on coroutines or you get a "coroutine was never awaited" warning and silent non-execution. Every call to an `async def` must be awaited or scheduled. Use `asyncio.gather` for concurrent tasks and collect the results — don't just fire tasks without awaiting them.

```python
async def fetch(url: str) -> bytes: ...

# Don't
async def main() -> None:
    task = fetch(url)  # never runs

# Do
async def main() -> None:
    results: list[bytes] = await asyncio.gather(fetch(a), fetch(b))
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
