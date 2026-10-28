# 正反例

## 1. 吞掉异常

❌ 过度防御
```python
def load_config(path):
    try:
        with open(path) as f:
            return json.load(f)
    except Exception as e:
        logger.warning(f"failed to load config: {e}")
        return {}
```
配置文件损坏时程序带着空配置继续跑，问题会在很远的地方以奇怪的形式出现。

✅ 改进
```python
def load_config(path: Path) -> dict:
    with path.open() as f:
        return json.load(f)
```
如果"文件不存在时使用默认配置"是**明确需求**，只捕获那一种：
```python
def load_config(path: Path) -> dict:
    try:
        with path.open() as f:
            return json.load(f)
    except FileNotFoundError:
        return DEFAULT_CONFIG
```

## 2. 内部函数重复校验

❌ 过度防御
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

✅ 改进
```python
def calc_total(items: list[Item]) -> Decimal:
    return sum((item.price for item in items), Decimal(0))
```

## 3. 可选链掩盖结构错误

❌ 过度防御
```ts
function getUserCity(user: User): string {
  return user?.profile?.address?.city ?? "";
}
```

✅ 改进（`User` 类型里 profile/address 是必填的）
```ts
function getUserCity(user: User): string {
  return user.profile.address.city;
}
```
如果数据来自外部 API，在边界用 schema（zod / pydantic）解析一次，之后内部全部信任类型。

## 4. 静默兜底返回值

❌ 过度防御
```python
def find_user(user_id: int) -> User | None:
    try:
        return db.query(User).filter_by(id=user_id).one()
    except Exception:
        return None
```
数据库连接断了和"用户不存在"变成了同一个结果。

✅ 改进
```python
def find_user(user_id: int) -> User | None:
    return db.query(User).filter_by(id=user_id).one_or_none()
```

## 5. 边界处该防御的地方（正确示例）

```python
@app.post("/orders")
def create_order(payload: dict):
    order = OrderIn.model_validate(payload)   # 边界：校验外部输入
    return order_service.create(order)        # 内部：信任已校验的 order
```
