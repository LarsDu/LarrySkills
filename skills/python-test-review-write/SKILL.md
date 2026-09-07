---
name: python-test-review-write
description: Writing and reviewing Python tests (pytest). Consult before writing or reviewing test suites, mocking, fixtures, parametrization, or evaluating whether a test suite is actually good.
---

# Python Test Review / Write

Scope: pytest idioms, fixtures, mocking-at-boundaries, what separates a good test from a bad one, a review checklist for AI-generated test suites, and when Hypothesis earns its keep. Does not cover the ML libraries themselves (see the per-library skills).

## Do / Don't

### Fixtures

Define shared setup in `conftest.py` as `@pytest.fixture` and request it by argument. Scope up (`module`, `session`) only when setup is genuinely expensive, and let the fixture own teardown via `yield`. Prefer `tmp_path` (unique per test, auto-cleaned) or `tempfile.TemporaryDirectory()` so tests never write to the real project filesystem; `monkeypatch` auto-reverts env and attribute patches.

`conftest.py`:
```python
from pathlib import Path
import pytest

@pytest.fixture
def upload_dir(tmp_path: Path) -> Path:
    d = tmp_path / "uploads"
    d.mkdir()
    return d
```

`test_upload.py`:
```python
from pathlib import Path
import tempfile

def test_files_landed_in_upload_dir(upload_dir: Path) -> None:
    target = upload_dir / "file.bin"
    write_bytes(target, b"...")
    assert target.exists()

def test_uses_tempdir_no_real_fs_writes() -> None:
    with tempfile.TemporaryDirectory() as d:
        out = Path(d) / "out.log"
        write_text(out, "...")
        assert out.read_text() == "..."
```

Avoid fixture overuse: a fixture that merely constructs one object is a smell — use a plain helper. Distinct resource variants belong in `@pytest.mark.parametrize`, not near-identical fixtures. Never share mutable session-scoped state across tests (breaks isolation).

### Parametrize kills copy-paste tests

Parametrized cases get free per-case failure names and `ids=` for readability.

```python
# Don't
def test_discount_0() -> None:    assert discount(0) == 0
def test_discount_50() -> None:   assert discount(50) == 25
def test_discount_100() -> None:  assert discount(100) == 60

# Do
@pytest.mark.parametrize("qty,expected", [(0, 0), (50, 25), (100, 60)],
                         ids=("zero", "half", "cap"))
def test_discount(qty: int, expected: int) -> None:
    assert discount(qty) == expected
```

### Mock at boundaries, not internals

Patch only slow, unavailable, or external boundaries (HTTP calls, DB connections, the clock). Mocking the logic under test produces tests that can't fail. Prefer `monkeypatch` for module attributes/env; use `unittest.mock.patch` (or `Mock`/`AsyncMock`) when you need to assert on call interactions at a real boundary.

```python
from unittest.mock import MagicMock

# Don't — patches the thing being tested; the test always passes
def test_calc(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setattr(FastMath, "add", lambda a, b: 4)
    assert FastMath().add(2, 2) == 4

# Do — patch only the outside boundary; real logic executes
def test_submit(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setattr(requests, "post", fake_post)
    assert submit(api, {"x": 1})

# Do — patch an external client at the seam and assert on the boundary call
def test_checkout_charges_gateway() -> None:
    gateway = MagicMock()
    gateway.charge.return_value = True
    assert Checkout(gateway).pay(9.99)
    gateway.charge.assert_called_once_with(9.99)
```

### Good tests: behavioral and structure-insensitive

Kent Beck's Test Desiderata: a test is Behavioral ("if the behavior changes, the test result changes") and Structure-insensitive ("the result should not change if the code structure changes"). Assert on public return values/effects, not internal call counts or private names. One logical behavior per test with clear Arrange-Act-Assert, named `test_<behavior>_<condition>`.

### No flakiness

Flakiness comes from un-isolated system state: re-testy, overly strict assertions, ordering dependence, global mutation, parallel interference. Use `monkeypatch` for time/env, and see [Reproducibility / seeding](#reproducibility--seeding) for randomness.

### No tautological tests

A test that always passes — because it asserts the mocked behavior it injected, or re-implements the code under test — has near-zero value. Screwdriver check: if you delete the production logic and the test still passes, it's a test that can't fail.

### Coverage is a signal, not a target

Line coverage measures only execution, not verification. Upgrade the signal with mutation testing (mutmut): it injects small faults (e.g. `==` -> `!=`) and flags surviving mutants as tests that failed to observe a behavior change. Treat survived mutants, not raw coverage, as the target.

### Reproducibility / seeding

Seed every source of randomness so a failing test reproduces. Seed Python, NumPy, and PyTorch explicitly:

```python
import random
import numpy as np
import torch

def seed_everything(seed: int) -> None:
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
```

For pytest, seed via a session fixture or `pytest-randomly` (it seeds by default and logs the seed on failure). Prefer fixed inputs or parametrized values over `random.random()` in tests; when randomness is unavoidable, seed it.

## Review checklist for an AI-generated test suite

1. Tests that mock the thing under test (can't fail).
2. Assertion-free tests (no `assert`, no `pytest.raises`) — smoke only.
3. Missing edge cases: `None`, empty input, all-empty strings, zero/negative/sentinel, boundary-adjacent values, Unicode, timezone/DST edges.
4. Over-reliance on snapshots — entrench bugs as "golden" and fail on intentional whitespace changes. Prefer explicit assertions on the fields that matter.
5. Implementation-coupled asserts: `assert_called_once_with` on internal collaborators, `_private` attribute pokes, call-order assertions.
6. Fixture hygiene: un-parametrized duplication, session-scoped mutating fixtures, autouse-where-explicit-would-read-better.
7. Missing negative paths: every branch that can raise (invalid input, HTTP errors, malformed data) needs a `pytest.raises`/`pytest.warns` counterpart.
8. Flakiness seeds: `time.sleep`, `random`, unseeded hashing, network calls, absolute paths, test-ordering dependence.

Each flagged item must name the concrete missing case or the line that can't fail.

## When Hypothesis earns its keep

Property-based testing shines for open-ended input spaces where a human can't enumerate cases: round-trips (`encode(decode(x)) == x`), invariants (`len(sorted(xs)) == len(xs)`), idempotence, "no exception for any valid input".

```python
@given(st.lists(st.integers()))
def test_sort_length_and_membership(lst: list[int]) -> None:
    out = sorted(lst)
    assert len(out) == len(lst)
    assert set(out) == set(lst)
```

Keep plain `parametrize` when inputs are fixed domain values, the table is the spec (price tiers), or the corpus is small enough to enumerate completely. Use `st.integers()` (unbounded) to catch 0/negative/overflow; `.filter()`/`assume()` for preconditions; `@composite` for dependent generation. Keep one concrete example-based test per behavior for intent, then let Hypothesis search.

## Sources

- https://docs.pytest.org/en/stable/how-to/fixtures.html
- https://docs.pytest.org/en/stable/how-to/parametrize.html
- https://docs.pytest.org/en/stable/how-to/monkeypatch.html
- https://docs.pytest.org/en/stable/flaky.html
- https://docs.pytest.org/en/stable/explanation/anatomy.html
- https://kentbeck.github.io/TestDesiderata/
- https://martinfowler.com/articles/mocksArentStubs.html
- https://hypothesis.readthedocs.io/en/latest/
- https://mutmut.readthedocs.io/en/latest/
