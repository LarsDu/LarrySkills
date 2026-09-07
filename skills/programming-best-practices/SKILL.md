---
name: programming-best-practices
description: Shared engineering-judgment skill for every LarrySkills agent. Consult before adding abstraction (interfaces, config systems, plugin/strategy layers, base classes) or before deduplicating similar-looking code. Applies a non-dogmatic reading of SOLID, DRY, and YAGNI that actively guards against the over-abstraction LLMs are known to produce.
---

## Scope

This is a judgment lens, not a style guide. It does not replace the domain skills — it governs whether and when to introduce structure while using them. LLM-generated code measurably trends toward two opposite failure modes: premature abstraction (interfaces/factories with one implementation) and copy-paste churn (real duplication left unmerged). Both come from not identifying the actual reason code would change. Everything below is aimed at that judgment call.

## SOLID — apply at real boundaries, not by reflex

### Single Responsibility
Group code by what changes together, not by mechanical role-splitting. Require two named reasons to change (two people or systems you can point to) before splitting a cohesive unit.

```python
# Do — total and persistence change on different schedules
class Invoice:
    def total(self) -> float:
        return sum(i.amount for i in self.items)

def save(invoice: Invoice, db: DB) -> None:
    db.write(invoice)

# Don't — one operation split into a "responsibility" per step
class Stage1Validate: ...
class Stage2ComputeTax: ...
class Stage3FormatRows: ...
```

### Open/Closed
A class, method, or function should be open for extension but closed for modification: add new behavior by adding new code, not by editing existing code. A plain conditional is fine when every case is already known; reach for dispatch only when new variants actually arrive.

```python
# Do — a new RPG class is added by adding a case; existing ones are untouched
def damage_for(role: str, base: float) -> float:
    if role == "warrior": return base * 1.2
    if role == "mage":    return base * 0.8
    return base

# Don't — registry/factory built for a single known case
class DamageStrategy(ABC): ...
class DamageStrategyFactory: ...
```
A plain conditional satisfies OCP here because adding a new role adds a new line without editing existing branches — the existing code is closed for modification. Reach for dynamic dispatch only when new variants actually arrive and a conditional stops paying its keep.

### Liskov Substitution
Subtypes must be substitutable for their base types without breaking the program. If a subclass violates the parent's contract, the is-a relationship is wrong — don't paper over it with `isinstance` special-casing; avoid `isinstance` checks where possible.

```python
# Don't — Square breaks Rectangle's width/height invariant
class Rectangle:
    def __init__(self, w: float, h: float) -> None:
        self.w, self.h = w, h
    def set_width(self, w: float) -> None:
        self.w = w
    def set_height(self, h: float) -> None:
        self.h = h

class Square(Rectangle):
    def set_width(self, w: float) -> None:
        self.w = self.h = w   # silently breaks the height invariant

# Do — model them as peers, not parent/child
class Shape: ...
class Rectangle(Shape): ...
class Square(Shape): ...
```

```python
# Don't — isinstance special-casing to make a broken hierarchy work
def area(shape: Shape) -> float:
    if isinstance(shape, Square):
        return shape.w * shape.w
    return shape.w * shape.h

# Do — each shape owns its own area(); no branching on type
def area(shape: Shape) -> float:
    return shape.area()
```

### Interface Segregation
Don't force clients to depend on methods they don't use. Split an interface only when multiple clients need disjoint subsets. Use `ABC` for explicit inheritance-checked contracts and `Protocol` for structural ones; when using `Protocol`, enforce its typing with `mypy`/`pyright` wired into pre-commit and `pyproject.toml` so the structural contract is actually checked.

```python
# ABC — explicit; subclasses must implement
from abc import ABC, abstractmethod

class Reader(ABC):
    @abstractmethod
    def read(self, n: int) -> bytes: ...

class FileSink(Reader):
    def read(self, n: int) -> bytes: ...
```

```python
# Protocol — structural; no inheritance required
from typing import Protocol

class Readable(Protocol):
    def read(self, n: int) -> bytes: ...

def consume(r: Readable) -> bytes:
    return r.read(1024)   # any object with a matching read() satisfies it
```

Enforce Protocol typing in `pyproject.toml` and pre-commit:
```toml
[tool.mypy]
strict = true
```
```yaml
# .pre-commit-config.yaml
- repo: https://github.com/pre-commit/mirrors-mypy
  rev: v1.10.0
  hooks:
    - id: mypy
```

```python
# Don't — one fat interface, one implementation, one caller
class ReadWriteSeek(Protocol):
    def read(self, n: int) -> bytes: ...
    def write(self, b: bytes) -> int: ...
    def seek(self, off: int) -> int: ...
# all three implemented by one class for one caller that only reads
```

### Dependency Inversion
Depend on abstractions, not concretions. High-level code depends on an interface (Protocol/ABC) and the concrete implementation is injected, so the high-level code is testable and swappable. Invert only across a boundary you actually have two implementations for, or one you need to fake in tests.

```python
# Do — high-level checkout depends on an abstraction, injected
class PaymentGateway(Protocol):
    def charge(self, amount: float) -> bool: ...

class Checkout:
    def __init__(self, gateway: PaymentGateway) -> None:
        self.gateway = gateway
    def pay(self, amount: float) -> bool:
        return self.gateway.charge(amount)

# Don't — interface + factory + DI container wrapping one concrete call
# "for testability" when a plain monkeypatch would test it fine
```

## DRY — merge by shared change-reason, not by shared shape

Two blocks that *look* alike today but change for different reasons are
coincidental duplication, not knowledge duplication. Merging them is the
"wrong abstraction" (Sandi Metz) — worse than the duplication it removed.

```python
# Don't: collapsed because they looked similar today
def calc_fee(item: Item, kind: str, cfg: Config) -> float:
    if kind == "shipping": ...   # changes with carrier/insurance rules
    if kind == "bank": ...       # changes with bank transfer regulations

# Do: keep them separate; they will diverge for unrelated reasons
def shipping_fee(item: Item) -> float: ...
def bank_fee(item: Item) -> float: ...
```

Test before extracting: "if I edit this shared function, is there a caller
that did NOT want that change?" If yes, don't share it. The "rule of three"
(extract on the third occurrence) is a floor, not a trigger — occurrence
count alone doesn't establish shared meaning.

## YAGNI — the tie-breaker

SOLID pushes toward extensibility; taken on its own that becomes speculative
generality, which is exactly what YAGNI forbids. When SOLID and YAGNI
conflict, YAGNI wins until a second real requirement lands.

```python
# Don't: nobody asked for a second provider yet
class EmailProvider(ABC): ...
class SmtpEmail(EmailProvider): ...
class SesEmail(EmailProvider): ...
config = load("provider", "retries", "queue", "webhook_url")

# Do: implement the one requested path
def on_signup(user: User) -> None:
    send_welcome_email(user)
```
Generalize when the second requirement actually arrives, and generalize by
refactoring the concrete code you already have (with tests), not by guessing
the shape in advance.

## Composition over inheritance

Favor composition over inheritance. Avoid large inheritance DAGs in favor of shallow (one level) or no inheritance, and compose small independent objects instead. Inheritance locks in a rigid hierarchy and makes changes ripple; composition lets you swap pieces independently.

```python
# Don't — deep inheritance chain
class Vehicle: ...
class Car(Vehicle): ...
class RaceCar(Car): ...
class F1Car(RaceCar): ...

# Do — compose capabilities
class Engine: ...
class Aerodynamics: ...
class F1Car:
    def __init__(self, engine: Engine, aero: Aerodynamics) -> None:
        self.engine = engine
        self.aero = aero
```

## Balance

SOLID is only good up to the point where it negatively impacts performance — too much breakdown of responsibilities into high-overhead objects beats performant code. Apply SOLID where it earns its keep, default to the simplest correct concrete implementation, and let YAGNI be the tie-breaker.

## Sources

- https://en.wikipedia.org/wiki/SOLID
- https://en.wikipedia.org/wiki/Liskov_substitution_principle
- https://en.wikipedia.org/wiki/Interface_segregation_principle
- https://en.wikipedia.org/wiki/Dependency_inversion_principle
