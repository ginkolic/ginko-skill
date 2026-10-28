---
name: avoid-overengineering
description: Use when writing, modifying, refactoring, or reviewing code, to suppress over-engineering — premature abstractions, single-use helpers and wrappers, needless layers and files, unrequested config options, feature flags, retries, async, and code bloat such as excessive logging, trivial comments, and docstrings. Load for any task that produces, changes, or audits code (new features, bug fixes, refactors, scripts, code review); not needed for pure Q&A or non-code tasks.
---

# Avoid Over-Engineering

Goal: write the **simplest code that solves the stated problem**, in the shape the surrounding codebase already uses. Every abstraction, layer, option, and line of infrastructure must pay for itself today, not in a hypothetical future.

## Core Principles

1. **Solve the task as stated.** Don't add options, modes, extension points, or generality the user didn't ask for. Requirements that "might come later" are added later.
2. **Abstractions follow repetition.** Introduce an interface, base class, factory, or strategy only when there are at least two real implementations now.
3. **Inline single-use code.** Don't extract a helper used once unless it hides a genuinely complex boundary or its name makes the caller clearly easier to read.
4. **Keep related logic together.** Prefer one function or file over a spread of tiny ones when the code is small and only used in one place.
5. **Match the codebase.** Use the existing structure, patterns, and libraries. Don't introduce a new architectural style, dependency, or layer for one change.
6. **Infrastructure needs a reason.** Retries, caching, async, concurrency, config, and feature flags each need a concrete problem they solve here.

## Banned Patterns (unless an exception below applies)

### Structure

| Pattern | Problem | Instead |
| --- | --- | --- |
| Interface / abstract base class / factory / strategy with one implementation | Indirection with no payoff; readers chase layers to find the logic | Use the concrete type; extract the abstraction when a second implementation exists |
| Helper or wrapper called from exactly one place (`get_config()` that just reads an env var) | Extra hop, extra name to learn | Inline it |
| New file or module for a few lines of logic | Navigation cost; deep nesting in a small codebase | Put it next to its only caller |
| Event emitters, hooks, pub/sub, or callbacks for a simple linear flow | Control flow becomes hard to follow | Call the functions directly in order |
| Wrapper classes or data classes created for one-off use | Boilerplate | Use the existing type, a dict/object literal, or a tuple |
| "Manager", "Handler", "Service", "Util" layers wrapping a single call | Names without responsibility | Call the underlying function |

### Infrastructure

| Pattern | Problem | Instead |
| --- | --- | --- |
| Retries / backoff on local calls or non-transient operations | Delays real failures, hides bugs | Retry only network or other genuinely transient operations |
| Config or env var for every constant | Configuration surface nobody uses; values that should never change become changeable | Hard-code it; make it configurable when someone needs to change it |
| Feature flags or boolean parameters for behavior that has no reason to vary | Doubles code paths and tests | One code path |
| async / threads / worker pools for fast or CPU-bound work | Complexity and new failure modes with no speedup | Synchronous code until there's a measured need |
| Caching without a measured performance problem | Invalidation bugs, stale data | No cache until profiling shows a need |
| New dependency for something the standard library or an existing dependency already does | Supply-chain and maintenance cost | Use what's already there |

### Bloat

| Pattern | Problem | Instead |
| --- | --- | --- |
| Logging on every function entry/exit, or logging internal params and return values | Noise drowns out useful logs | Log meaningful events and failures |
| Comments restating the code (`# increment counter`) or narrating the change (`# added for issue X`) | Noise, goes stale | Comment only non-obvious *why*; match the comment density of the file |
| Docstrings that repeat the function name (`def add(a, b): """Add a and b."""`) | Noise | Omit, or document what isn't obvious from the signature |
| Named constants for values obvious in context (`ONE = 1`, `EMPTY_STRING = ""`) | Indirection with no clarity gain | Use the literal; name constants that carry meaning |
| Rewriting or reformatting code outside the task's scope | Bigger diff, harder review | Change only what the task requires |

## Exceptions: Where Structure Belongs

- A second real implementation exists now (or the task explicitly requires pluggability).
- Code is genuinely reused across several call sites or packages.
- A public API or library boundary, where a stable interface protects callers.
- Testability that can't be achieved otherwise (e.g. a seam around a real external service).
- The codebase already uses the pattern consistently — follow it.
- Measured performance or reliability problems justify caching, concurrency, or retries.

### Speculative needs: ask, don't decide silently

If you believe an extension point, option, or abstraction will clearly be needed but the task didn't ask for it, **don't build it**. Mention it in one line in your reply and let the user decide.

## Self-Check After Writing Code

For every new abstraction, file, helper, option, and piece of infrastructure, ask:

1. **Does the task require it today?** No → remove it.
2. **How many real call sites or implementations does it have?** One → inline it.
3. **Would a reader find the logic faster without it?** Yes → remove it.
4. **Does the surrounding codebase do this?** No → reconsider.
5. **Could the diff be smaller and still solve the task?** Yes → make it smaller.

When editing existing code: don't refactor existing structure that isn't part of the task. If it's clearly over-engineered, point it out in your reply rather than changing it.

## Cleanup Mode

When asked to review or clean up code (e.g. "simplify this branch", "remove over-engineering"):

1. Diff against the base branch (or the commit the user names). Only consider code **introduced in that diff**.
2. Apply the banned patterns and self-check above to each addition.
3. If the user asked for a review, report findings (file, line, pattern, suggested fix) without editing. If they asked for a cleanup, make the edits.
4. Finish with a 1–3 sentence summary of what changed or was found.

## Examples

### Premature abstraction

❌
```python
class NotificationSender(ABC):
    @abstractmethod
    def send(self, user: User, message: str) -> None: ...

class EmailSender(NotificationSender):
    def send(self, user: User, message: str) -> None:
        smtp.send(user.email, message)

class NotificationFactory:
    @staticmethod
    def create(kind: str = "email") -> NotificationSender:
        return EmailSender()

NotificationFactory.create().send(user, "Welcome!")
```

✅
```python
def send_email(user: User, message: str) -> None:
    smtp.send(user.email, message)

send_email(user, "Welcome!")
```

### Single-use wrapper and unrequested options

❌
```ts
const DEFAULT_TIMEOUT_MS = Number(process.env.FETCH_TIMEOUT_MS ?? 5000);

interface FetchUserOptions { retries?: number; useCache?: boolean; timeoutMs?: number }

function buildUserUrl(id: string): string {
  return `/api/users/${id}`;
}

async function fetchUser(id: string, opts: FetchUserOptions = {}): Promise<User> {
  logger.debug("fetchUser called", { id, opts });
  // ...retry loop, cache lookup, timeout handling...
}
```

✅ (no retry, cache, or timeout requirement in the task)
```ts
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`GET /api/users/${id} failed: ${res.status}`);
  return UserSchema.parse(await res.json());
}
```

### Logging and comment bloat

❌
```python
def apply_discount(order: Order, rate: Decimal) -> Decimal:
    """Apply discount to order."""
    logger.info("apply_discount start: order=%s rate=%s", order.id, rate)
    # Calculate the discounted total
    total = order.total * (1 - rate)
    logger.info("apply_discount end: total=%s", total)
    return total
```

✅
```python
def apply_discount(order: Order, rate: Decimal) -> Decimal:
    return order.total * (1 - rate)
```

## Related

Over-defensive error handling (swallowed exceptions, redundant checks, fabricated defaults) is covered by the `avoid-overdefensive-code` skill.
