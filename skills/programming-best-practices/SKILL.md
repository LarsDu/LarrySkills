---
name: programming-best-practices
description: Shared engineering-judgment skill for every LarrySkills agent. Consult before adding abstraction (interfaces, config systems, plugin/strategy layers, base classes) or before deduplicating similar-looking code. Applies a non-dogmatic reading of SOLID, DRY, and YAGNI that actively guards against the over-abstraction LLMs are known to produce.
---

## Scope

This is a judgment lens, not a style guide. It does not replace the domain
skills (pytorch, godot-shaders, blender-mesh-modeling, etc.) — it governs
whether and when to introduce structure while using them. LLM-generated code
measurably trends toward two opposite failure modes: premature abstraction
(interfaces/factories with one implementation) and copy-paste churn (real
duplication left unmerged). Both come from not identifying the actual reason
code would change. Everything below is aimed at that judgment call.

## The load-bearing rule: name the second thing

Before extracting an interface, base class, strategy, config flag, or shared
function, name a concrete second consumer, variant, or implementation that
exists *right now*. Not one you expect next quarter — one that exists today.

- Do: two call sites already need different bbox-normalization backends ->
  extract `normalize_bbox(verts, backend)`.
- Don't: one call site, but "we might support a second backend later" ->
  write the concrete function, not a `NormalizerStrategy` base class.

If you can't name the second thing, write the concrete version and stop.

## SOLID — apply at real boundaries, not by reflex

### Single Responsibility
Group code by what changes together, not by mechanical role-splitting.

```python
# Do
class Invoice:
    def total(self) -> float:
        return sum(i.amount for i in self.items)
# persistence/email live in separate functions only because they
# genuinely change on a different schedule (different stakeholders)

# Don't: one 20-line operation split into a "responsibility" per step
class Stage1Validate: ...
class Stage2ComputeTax: ...
class Stage3FormatRows: ...
```
Require two *named* reasons to change (two people/systems you can point to)
before splitting a cohesive unit. "What counts as a reason to change" is
famously fuzzy even in Robert Martin's own definition — don't let that fuzz
default to maximal splitting.

### Open/Closed
Add behavior by adding a case, when a second real variant exists.

```python
# Do (variants are real and known)
def discount_for(kind: str, amount: float) -> float:
    if kind == "loyalty": return amount * 0.9
    if kind == "clearance": return amount * 0.5
    return amount

# Don't: registry/factory built for a single known case
class DiscountStrategy(ABC): ...
class DiscountStrategyFactory: ...
```
If you already know every type and don't expect new *behavior* to be layered
on top of them uniformly, a plain conditional is the OCP-compliant answer —
dynamic dispatch adds an indirection that only pays off once real extension
happens.

### Liskov Substitution
Treat this as a contract check, not a hierarchy-design mandate. The canonical
failure is subclassing a mutable `Rectangle` with `Square` (setting width
silently breaks the height invariant) — fix by not forcing an is-a
relationship where the contract doesn't hold, not by adding `isinstance`
special-casing to paper over it. Don't build a deep hierarchy pre-emptively
to "be safe" for substitutability you don't use yet.

### Interface Segregation
Split an interface only when multiple clients already need disjoint subsets.

```python
# Don't: one interface, one implementation, one caller
class Readable(Protocol): ...
class Writable(Protocol): ...
class Seekable(Protocol): ...
# all three implemented by the same single class for the same single caller

# Do: split when a second, genuinely different client shows up
# (a read-only consumer that must not be able to write)
```

### Dependency Inversion
Invert across a boundary you actually have two implementations for, or a
boundary you actually need to fake in tests.

```python
# Do: two real implementations exist (card, wallet)
class Authorizer(Protocol):
    def authorize(self, amount: float) -> bool: ...

# Don't: interface + factory + DI container wrapping one concrete call
# "for testability" when nothing else will ever implement it and a
# plain monkeypatch of the one function would test it fine
```

## DRY — merge by shared change-reason, not by shared shape

Two blocks that *look* alike today but change for different reasons are
coincidental duplication, not knowledge duplication. Merging them is the
"wrong abstraction" (Sandi Metz) — worse than the duplication it removed.

```python
# Don't: collapsed because they looked similar today
def calc_fee(item, kind, cfg):
    if kind == "shipping": ...   # changes with carrier/insurance rules
    if kind == "bank": ...       # changes with bank transfer regulations

# Do: keep them separate; they will diverge for unrelated reasons
def shipping_fee(item): ...
def bank_fee(item): ...
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
def on_signup(user):
    send_welcome_email(user)
```
Generalize when the second requirement actually arrives, and generalize by
refactoring the concrete code you already have (with tests), not by guessing
the shape in advance.

## Balance rules for this agent's output

1. YAGNI is the tie-breaker: any abstraction with zero current consumers gets
   rejected or deferred, regardless of which SOLID letter justifies it.
2. The "who else?" test: every proposed split/interface/strategy must name a
   concrete second user. No name, no extract.
3. DRY by change-reason, not by shape: prefer duplication over merging code
   that will diverge for unrelated reasons.
4. Cost lens for hot paths: in performance-sensitive code (a Godot
   `_process`/`_physics_process` callback, a per-batch training step, a
   per-mesh bmesh operation), indirection (virtual dispatch, extra interface
   layers) has a real, measurable cost — don't pay it for structure that
   isn't earning its keep there.
5. Default to the simplest correct concrete implementation for the stated
   requirement. Ask "what is the smallest slice that satisfies what was
   actually asked?" before reaching for a pattern.
6. When abstraction is genuinely warranted, extract it from working, tested
   concrete code after the second use case appears — don't design it ahead
   of time from imagined future cases.

## Sources

- https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html
- https://blog.cleancoder.com/uncle-bob/2014/05/12/TheOpenClosedPrinciple.html
- https://en.wikipedia.org/wiki/Liskov_substitution_principle
- https://en.wikipedia.org/wiki/Interface_segregation_principle
- https://en.wikipedia.org/wiki/Dependency_inversion_principle
- https://stackoverflow.blog/2021/11/01/why-solid-principles-are-still-the-foundation-for-modern-software-architecture/
- https://www.computerenhance.com/p/clean-code-horrible-performance
- https://github.com/unclebob/cmuratori-discussion/blob/main/cleancodeqa.md
- https://vimeo.com/157708450
- https://www.sandimetz.com/blog/2016/1/20/the-wrong-abstraction
- https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
- https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming)
- https://infiniteundo.com/post/158826857988/software-as-narrative-11n
- https://martinfowler.com/bliki/Yagni.html
- https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it
- https://www.gitclear.com/coding_on_copilot_data_shows_ais_downward_pressure_on_code_quality
- https://dora.dev/dora-report-2024/
- https://arxiv.org/abs/2412.18989
- https://github.com/cloud-atlas-ai/superego
- https://news.ycombinator.com/item?id=23612415
