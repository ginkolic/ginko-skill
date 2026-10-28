# Before / After Examples

## 1. Swallowed exceptions

❌ Over-defensive
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

✅ Better
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

## 2. Redundant validation in internal functions

❌ Over-defensive
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

✅ Better
```python
def calc_total(items: list[Item]) -> Decimal:
    return sum((item.price for item in items), Decimal(0))
```

## 3. Optional chaining hiding structural errors

❌ Over-defensive
```ts
function getUserCity(user: User): string {
  return user?.profile?.address?.city ?? "";
}
```

✅ Better (`profile` and `address` are required in the `User` type)
```ts
function getUserCity(user: User): string {
  return user.profile.address.city;
}
```
If the data comes from an external API, parse it once at the boundary with a schema (zod / pydantic), then trust the types internally.

## 4. Silent fallback return values

❌ Over-defensive
```python
def find_user(user_id: int) -> User | None:
    try:
        return db.query(User).filter_by(id=user_id).one()
    except Exception:
        return None
```
A dropped database connection and "user not found" become the same result.

✅ Better
```python
def find_user(user_id: int) -> User | None:
    return db.query(User).filter_by(id=user_id).one_or_none()
```

## 5. Defense where it belongs (correct example)

```python
@app.post("/orders")
def create_order(payload: dict):
    order = OrderIn.model_validate(payload)   # boundary: validate external input
    return order_service.create(order)        # internal: trust the validated order
```
