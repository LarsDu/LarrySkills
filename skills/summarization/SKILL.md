---
name: summarization
description: Produce faithful compressed digests of files, logs, or tool outputs for the main session. Consult before summarizing — keeps signatures and path:line anchors, drops boilerplate, preserves exact identifiers, and flags ambiguity instead of silently paraphrasing.
---

# Summarization

Scope: how a low-cost summarization agent compresses a large input (a file, a set of files, a log, a tool result) into a digest the main model can act on. The goal is faithful compression — the main model reads your digest *instead of* the source, so anything you drop or distort is lost.

## Do / Don't

### Keep signatures and anchors

Preserve exact function/class signatures, route paths, error strings, and `path:line` anchors. The main model uses these to decide what to read next.

```
# Do
- src/api/orders.py:44 — POST /orders: creates order, validates body via OrderSchema, returns 201
- src/api/orders.py:88 — GET /orders/{id}: fetches from cache then DB, 404 on miss

# Don't — drop the signature and the line
- there are endpoints for creating and fetching orders
```

### Preserve exact identifiers

Never rename, normalize, or "fix" identifiers. `getUserById` and `get_user_by_id` are different symbols; report them as written.

### Drop boilerplate, not meaning

Cut imports, logging noise, repeated scaffolding, and cosmetic formatting. Keep the logic, the data flow, and anything that could be a bug or a decision point.

### Flag ambiguity rather than paraphrase

If you are not sure what a block does, say "unclear — appears to X but may Y" and point the main model at the `path:line`. Never silently guess and present the guess as fact.

```
# Do
- src/worker/retry.py:31 — unclear: looks like exponential backoff, but the base is read from an unset env var; main model should verify

# Don't — assert a confident wrong summary
- src/worker/retry.py:31 — exponential backoff with base 2
```

### Preserve numbers and types

Exact counts, thresholds, status codes, types, and config values must come through verbatim. "retries 3 times" and "retries 5 times" are different bugs.

### Structure the digest

Lead with what the main model asked for. Group by entity (file, endpoint, error class), not by order of discovery. End with a short "what to look at next" if the input suggests follow-ups.

## Pitfalls: symptom -> cause -> fix

- Main model acts on a detail that wasn't in the source -> you paraphrased or inferred it -> only report what's in the input; flag inference explicitly.
- Main model re-reads the file you summarized -> your digest dropped the signature/anchor it needed -> always keep `path:line` + signature for every entity.
- Digest is barely shorter than the source -> you kept boilerplate -> cut imports, logging, scaffolding; keep logic and data flow.
- Main model trusts a number that was wrong -> you rounded or "simplified" a value -> preserve numbers and types verbatim.

## Sources

- Practices for technical summarization under token constraints
