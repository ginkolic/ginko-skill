---
name: avoid-overdefensive-code
description: Use when writing, modifying, or refactoring code, to suppress over-defensive programming — redundant try/except, repeated null/type checks, silent fallbacks, swallowed exceptions, and "just in case" defaults or parameters. Load this skill for any task that produces or changes code (new features, bug fixes, refactors, scripts); not needed for pure Q&A or non-code tasks.
---

# Avoid Over-Defensive Code

Goal: write code that **trusts internal contracts, validates at boundaries, and fails loudly**. Defensive code is not free: it hides bugs, creates dead branches, and misleads readers into thinking a situation can actually happen.

## Core Principles

1. **Validate only at trust boundaries.** External input (user input, HTTP requests, files, environment variables, third-party API responses, deserialized data) must be validated. Calls between internal functions trust the types and the caller's contract.
2. **Fail fast; no silent fallbacks.** When an "impossible" state occurs, raise — don't return `None`, an empty list, a default value, or log a line and keep going.
3. **Catch only what you can handle.** Catch an exception only if you know its specific type and have a concrete recovery action (retry, alternate path, convert to a meaningful user-facing error). Otherwise let it propagate.
4. **Don't write code for imagined requirements.** No unrequested config options, flags, fallback branches, or backward-compatibility logic.
5. **Trust the type system and existing validation.** Values already guaranteed non-null / validated / normalized upstream are not re-checked downstream.
6. **Follow the language's idiomatic error handling.** Explicit error propagation (Go's `if err != nil { return err }`, Rust's `?`) is correct handling, not over-defense. Don't remove it; remove the swallowing and silent fallbacks around it.

## Banned Patterns (unless an exception below applies)

| Pattern | Problem | Instead |
| --- | --- | --- |
| `try: ... except Exception: pass / log` | Swallows bugs, loses the stack | Remove the try, or catch a specific exception with a real recovery action |
| `if x is None: return` at the top of every function | Turns caller bugs into silent empty results | Make the type non-optional; handle None only where it can really occur |
| `isinstance` / `typeof` checks on internal parameters | Duplicates the type system | Use type annotations; validate once at the boundary |
| `getattr(obj, "x", None)`, `obj?.a?.b?.c` on known structures | Hides structural errors | Access directly; if the shape is uncertain, parse with a schema at the boundary |
| `dict.get(key, default)` for keys that must exist | Typos in keys go unnoticed | Use `dict[key]` |
| Returning `None` / `[]` / `{}` / `-1` on failure | Callers can't distinguish "empty" from "error" | Raise, or return an explicit Result type |
| Re-validating input at every layer of the call chain | Noise, scattered logic | Validate once at the entry point, then pass validated values |
| Parameters, flags, or fallbacks "we might need later" | More branches and test burden | Add them when actually needed |
| Defensive `copy.deepcopy` / cloning | Needless overhead | Copy only when shared mutable state is a real risk |
| Comments like "for safety" or "just in case" | Signal the code has no concrete justification | Name the specific scenario; if you can't, delete it |

## Exceptions: Where Defense Belongs

- System boundaries: parsing external input, network/disk I/O, subprocesses, third-party SDK calls.
- Entry points of public libraries / external APIs.
- Expected failures with a clear recovery strategy: retry on network timeout, create a missing file, prompt on malformed user input.
- Resource cleanup: `finally` / `with` / `defer` to guarantee release.
- Security: authentication, authorization, injection protection — never omit.

## Self-Check After Writing Code

For every `try/except`, `if x is None`, type check, default value, and fallback, ask:

1. **What concrete input makes this happen?** Can't name one → delete it.
2. **If it does happen, is silent handling or a loud error more useful?** Usually the error.
3. **Does upstream already guarantee this?** Yes → delete it.
4. **Is this a boundary or internal code?** Internal → lean toward deleting.

When editing existing code: don't proactively remove existing defensive code (it may exist for historical reasons), but **don't add new code of the same kind**. If it's clearly harmful, point it out in your reply rather than changing it.

## Language-Specific References

Read the file matching the language you're writing; the table above uses Python/TypeScript syntax, and other languages have their own idioms.

- Python / TypeScript: [references/examples.md](references/examples.md)
- Go: [references/go.md](references/go.md)
