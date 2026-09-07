---
name: scouting
description: Read-only codebase recon discipline for low-cost scout agents. Consult before searching a codebase — minimizes tokens by grepping before reading, reporting path:line anchors instead of pasting code, and capping exploration depth.
---

# Scouting

Scope: how a cheap read-only agent searches a codebase efficiently and reports back without burning tokens. The goal is a compressed location report the main model can act on — not a code dump, not a review, not an analysis.

## Do / Don't

### Grep before read

Don't `read` whole files to find a symbol. `grep` to locate it first, then `read` only the relevant excerpt with `offset`/`limit`.

```bash
# Don't — read the whole file to find one function
read src/server.py

# Do — locate first, then read a tight window
grep -n "def handle_auth" src/server.py    # -> src/server.py:142
# then read src/server.py offset=138 limit=20
```

### Report anchors, not code

Every finding is a `path:line` anchor plus a one-line summary. Never paste code blocks — the main model can `read` the anchor itself.

```
# Do
- src/auth/session.py:88 — validates session token, raises on expiry
- src/auth/session.py:120 — refreshes token, calls rotate_key()

# Don't — paste the function body
- src/auth/session.py
  def validate(token):
      if expired(token): raise ...
```

### Read excerpts, not whole files

Use `offset`/`limit` on `read` to pull only the region around a hit. Whole-file reads are for small files only.

```
# Do
read src/big_module.py offset=400 limit=60

# Don't
read src/big_module.py
```

### Parallel independent lookups

Fire independent `grep`/`find`/`read` calls in one batch when none depends on another's result. Don't serialize searches that could run together.

### Cap depth and bail early

Stop the moment the answer is found. Don't "also check" adjacent files out of curiosity. If the search breadth was "quick", do one targeted lookup and return.

### State what you did NOT read

Always report the boundary of what you searched: which directories, which file types, which symbols. If you grepped `src/` but not `tests/` or `scripts/`, say so. The main model needs to know the blind spots.

```
# Do
Searched: src/**.py for "handle_auth". Found 3 hits (above).
Did not search: tests/, scripts/, any *.ts files.

# Don't — leave the main model assuming you covered everything
```

## Pitfalls: symptom -> cause -> fix

- Report is huge and full of pasted code -> you `read` whole files and copied them -> report anchors and one-line summaries only.
- Main model re-does your search -> your report didn't state what you did not read -> always state the search boundary.
- You spent many turns on a "quick" lookup -> you kept exploring past the first hit -> bail as soon as the answer is found.
- You missed the definition -> you grepped only one naming convention -> grep both `camelCase` and `snake_case`, and `find` by file pattern when the name is ambiguous.

## Sources

- pi `read` tool offset/limit semantics
- ripgrep / grep `-n` line-number output
