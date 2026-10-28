# Python

Python's EAFP style (`try: d[k] except KeyError:`) is idiomatic when the exception is **specific** and the except branch is a **real recovery**. Over-defense in Python means broad excepts, `None` guards everywhere, `hasattr`/`getattr` probing on known objects, and runtime type checks that duplicate annotations.

## Python-specific rules

- **Catch specific exceptions only.** Never bare `except:` or `except Exception:` unless at a process boundary (CLI main, worker loop, request handler) that logs and reports.
- **Keep `try` blocks narrow** — wrap only the call that can raise the exception you handle, not the whole function.
- **Use `raise ... from err`** when translating exceptions, so the original cause is preserved.
- **Annotate types and trust them.** Use `Optional[T]` / `T | None` only where `None` is a real value; otherwise the parameter is non-optional and unchecked.
- **Validate external data once with a schema** (pydantic, dataclasses + explicit parsing, attrs) at the boundary; internal code receives typed objects.
- **`assert` is fine for internal invariants** — it's fail-fast, not defensive noise.

## Banned patterns

| Pattern | Problem | Instead |
| --- | --- | --- |
| `except:` / `except Exception: pass` | Swallows bugs, including `KeyboardInterrupt` for bare except | Remove, or catch the specific exception with a recovery action |
| `except Exception as e: logger.error(e); return None` | Caller sees "no result" instead of a failure | Let it propagate, or raise a domain error `from e` |
| `try` wrapping an entire function body | Catches exceptions from unrelated lines | Narrow the `try` to the one call that can fail |
| `if x is None: return` on non-optional params | Hides caller bugs | Remove; type the param as non-optional |
| `isinstance(x, list)` on annotated internal params | Duplicates type hints | Remove; run a type checker |
| `hasattr(obj, "x")` / `getattr(obj, "x", None)` on known classes | Hides attribute typos and refactor breakage | `obj.x` |
| `d.get("key", default)` for required keys | Misspelled keys silently return defaults | `d["key"]` |
| `value or default` where `0` / `""` / `[]` are valid | Silently replaces valid falsy values | Handle `None` explicitly, only if `None` is possible |
| `copy.deepcopy(arg)` "to be safe" | Needless cost | Copy only when mutation of shared state is real |
| Returning `None` / `[]` / `{}` on failure | "Empty" and "failed" are indistinguishable | Raise |

## Where defense belongs in Python

- File, network, subprocess, and database calls — catch the specific exceptions you can recover from (`FileNotFoundError`, `TimeoutError`, `requests.HTTPError`).
- Parsing external input: `json.loads`, request bodies, env vars, CLI args → validate into typed objects.
- `with` statements for resources (files, locks, connections).
- Top-level boundaries (CLI `main`, worker loops, web framework error handlers) — log, report, exit or return an error response.

## Before / After

### Swallowed exceptions

❌
```python
def load_config(path):
    try:
        with open(path) as f:
            return json.load(f)
    except Exception as e:
        logger.warning(f"failed to load config: {e}")
        return {}
```
A corrupted config file lets the program keep running with an empty config, and the problem surfaces far away in a confusing form.

✅
```python
def load_config(path: Path) -> dict:
    with path.open() as f:
        return json.load(f)
```
If "use defaults when the file is missing" is an **explicit requirement**, catch only that case:
```python
def load_config(path: Path) -> dict:
    try:
        with path.open() as f:
            return json.load(f)
    except FileNotFoundError:
        return DEFAULT_CONFIG
```

### Redundant validation in internal functions

❌
```python
def calc_total(items: list[Item]) -> Decimal:
    if items is None:
        return Decimal(0)
    if not isinstance(items, list):
        raise TypeError("items must be a list")
    total = Decimal(0)
    for item in items:
        if item is None or not hasattr(item, "price"):
            continue
        total += item.price or Decimal(0)
    return total
```

✅
```python
def calc_total(items: list[Item]) -> Decimal:
    return sum((item.price for item in items), Decimal(0))
```

### Silent fallback return values

❌
```python
def find_user(user_id: int) -> User | None:
    try:
        return db.query(User).filter_by(id=user_id).one()
    except Exception:
        return None
```
A dropped database connection and "user not found" become the same result.

✅
```python
def find_user(user_id: int) -> User | None:
    return db.query(User).filter_by(id=user_id).one_or_none()
```

### Translating exceptions without losing the cause

❌
```python
try:
    resp = client.get(url, timeout=5)
    resp.raise_for_status()
except Exception:
    raise PaymentError("payment service failed")
```

✅
```python
try:
    resp = client.get(url, timeout=5)
    resp.raise_for_status()
except httpx.HTTPError as e:
    raise PaymentError("payment service failed") from e
```

### Boundary validation (correct)

```python
@app.post("/orders")
def create_order(payload: dict):
    order = OrderIn.model_validate(payload)   # boundary: validate external input
    return order_service.create(order)        # internal: trust the validated order
```
