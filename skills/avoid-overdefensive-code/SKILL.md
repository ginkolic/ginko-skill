---
name: avoid-overdefensive-code
description: Use when writing, modifying, refactoring, or reviewing code, to suppress over-defensive programming — swallowed exceptions, redundant null/type checks, fabricated default values, silent fallbacks, type coercion that hides bad data, and compatibility shims. Load for any task that produces, changes, or audits code (new features, bug fixes, refactors, scripts, code review, "clean up this diff"); not needed for pure Q&A or non-code tasks.
---

# Avoid Over-Defensive Code

Goal: write code that **trusts internal contracts, validates at boundaries, and fails loudly**. Defensive code is not free: it hides bugs, creates dead branches, and misleads readers into thinking a situation can actually happen.

## Why Models Over-Produce Defensive Code

- **Training data bias.** Most public code is web/app code, where graceful degradation is often correct. That habit gets applied everywhere, including internal logic where it's wrong.
- **Feedback bias.** A visible crash is penalized more than a silently wrong result, so "never crash" looks safer than "fail fast". It isn't: silent corruption is harder to detect and more expensive to fix.

Counter this bias deliberately. When unsure, prefer the version that fails loudly.

## Core Principles

1. **Validate only at trust boundaries.** External input (user input, HTTP requests, files, environment variables, third-party API responses, deserialized data) must be validated. Calls between internal functions trust the types and the caller's contract.
2. **Fail fast; no silent fallbacks.** When an "impossible" state occurs, raise — don't return `None`, an empty list, a default value, or log a line and keep going.
3. **Catch only what you can handle.** Catch an exception only if you know its specific type and have a concrete recovery action (retry, alternate path, convert to a meaningful user-facing error). Otherwise let it propagate.
4. **Never fabricate data.** A missing required value is an error, not an occasion to invent `-1`, `""`, `0`, `"unknown"`, or the current time.
5. **Trust the type system and existing validation.** Values already guaranteed non-null / validated / normalized upstream are not re-checked downstream.
6. **Follow the language's idiomatic error handling.** Explicit error propagation (Go's `if err != nil { return err }`, Rust's `?`) is correct handling, not over-defense. Don't remove it; remove the swallowing and silent fallbacks around it.
7. **Match the surrounding code.** If nearby code in the same area doesn't guard against something, neither should yours.

## Banned Patterns (unless an exception below applies)

Language-specific syntax for each pattern is in the reference files below.

| Pattern | Concrete harm | Instead |
| --- | --- | --- |
| **Swallowing errors**: catch, then pass / log / return a placeholder | Downstream code gets `None` or partial data; monitoring shows no errors while results are wrong | Let it propagate, or catch a specific error with a real recovery action |
| **Catch-all handlers**: catching every exception type | Also catches bugs, interrupts, and out-of-memory errors no code should handle | Catch specific exception types |
| **Logging instead of raising** on an unexpected state | Warnings are ignored; the problem surfaces later as a bigger failure | Raise an error |
| **Fabricated defaults** for required values | A `-1` user ID joins to nothing and analytics silently show zero; a `now()` default stamps historical records with today's date | Access the value directly so absence errors out |
| **Type coercion instead of validation** (`int(x or 0)`, `Number(x) \|\| 0`) | A wrong type is a sign of an upstream bug; coercing turns it into plausible-looking wrong data | Validate at the boundary; let mismatches surface internally |
| **Compatibility shims**: try the new API, fall back to the old one | Breaking changes go undetected and temporary debt becomes permanent | Call the new API; migrate callers |
| **Guarding guaranteed values**: null / type / existence checks on values the types or upstream already guarantee | Readers assume the value can be absent; checks multiply across the codebase | Access directly |
| **Optional access on known structures** (`getattr(o, "x", None)`, `a?.b?.c`) | Hides typos and structural changes after refactors | Access directly; if the shape is uncertain, parse with a schema at the boundary |
| **Failure disguised as empty**: returning `None` / `[]` / `{}` / `-1` on error | Callers can't distinguish "nothing found" from "something broke" | Raise, or return an explicit result/error type |
| **Re-validating at every layer** | Validation logic is duplicated and scattered | Validate once at the entry point, pass validated values inward |
| **Bypassing the type checker** (`as any`, `# type: ignore`, unchecked casts) | Removes the guarantees the rest of these rules rely on | Fix the type, or parse the value properly |
| **"Just in case" comments** ("for safety", "just in case") | Signal the code has no concrete justification | Name the specific scenario; if you can't, delete the code |

## Exceptions: Where Defense Belongs

- System boundaries: parsing external input, network/disk I/O, subprocesses, third-party SDK calls.
- Entry points of public libraries / external APIs.
- Expected failures with a clear recovery strategy: retry on network timeout, create a missing file, prompt on malformed user input.
- Resource cleanup: `finally` / `with` / `defer` to guarantee release.
- Security: authentication, authorization, injection protection — never omit.
- Top-level process boundaries (CLI entry point, worker loop, request handler) may catch broadly to log and report, never to continue as if nothing happened.

### Speculative edge cases: ask, don't decide silently

If you believe a rare race condition, security edge case, or failure mode deserves extra handling but the task didn't ask for it, **don't add it silently and don't drop it silently**. Briefly tell the user: the concrete failure, how likely it is, and the complexity cost. Let them decide.

## Self-Check After Writing Code

For every error handler, null/type check, default value, and fallback, ask:

1. **What concrete input makes this happen?** Can't name one → delete it.
2. **If it does happen, is silent handling or a loud error more useful?** Usually the error.
3. **Does upstream already guarantee this?** Yes → delete it.
4. **Is this a boundary or internal code?** Internal → lean toward deleting.
5. **Does the surrounding code do this?** No → delete it.

When editing existing code: don't proactively remove existing defensive code (it may exist for historical reasons), but **don't add new code of the same kind**. If it's clearly harmful, point it out in your reply rather than changing it.

## Cleanup Mode

When asked to review or clean up code (e.g. "clean up this branch", "remove defensive slop"):

1. Diff against the base branch (or the commit the user names). Only consider code **introduced in that diff**.
2. Apply the banned patterns and self-check above to each addition.
3. If the user asked for a review, report findings (file, line, pattern, suggested fix) without editing. If they asked for a cleanup, make the edits.
4. Finish with a 1–3 sentence summary of what changed or was found.

## Language-Specific References

Read the file matching the language you're writing:

- Python: [references/python.md](references/python.md)
- TypeScript / JavaScript: [references/typescript.md](references/typescript.md)
- Go: [references/go.md](references/go.md)
