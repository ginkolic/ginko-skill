# TypeScript / JavaScript

With `strict` mode on, the compiler already guarantees most of what defensive runtime checks try to enforce. Over-defense in TypeScript means optional chaining on required fields, `?? ""` fallbacks that hide missing data, `typeof` checks on typed parameters, and `try/catch` blocks that turn rejections into `null`.

## TypeScript-specific rules

- **Trust the types inside the program.** If a field is declared required, access it directly. Make a field optional (`?:` / `| undefined`) only when absence is a real, handled state.
- **Parse `unknown` at the boundary** with a schema (zod, valibot, io-ts) or a hand-written type guard. Data from `fetch`, `JSON.parse`, `localStorage`, `postMessage`, and env vars is `unknown` until parsed — never cast it with `as`.
- **Let promise rejections propagate.** Don't wrap every `await` in `try/catch`; catch where you can recover or where you translate to a user-facing error.
- **`catch (e)` gives `unknown`.** Narrow it (`e instanceof SomeError`) and rethrow anything you don't handle.
- **Use exhaustive `switch` with a `never` check** instead of a silent `default` branch.

## Banned patterns

| Pattern | Problem | Instead |
| --- | --- | --- |
| `a?.b?.c` on required fields | Hides structural bugs, returns `undefined` silently | `a.b.c` |
| `value ?? ""` / `?? 0` / `?? []` on required values | Missing data becomes plausible-looking data | Access directly; fix the type if it really can be missing |
| `typeof x === "string"` on a parameter typed `string` | Duplicates the compiler | Remove |
| `try { ... } catch { return null }` around `await` | Network failure and "not found" look the same | Let it reject, or catch a specific error and rethrow others |
| `catch (e) { console.error(e) }` and continue | Swallows the failure | Rethrow, or handle with a real recovery |
| `if (!arr \|\| arr.length === 0) return` before `.map` / `for...of` | Empty iteration is already a no-op | Remove |
| `Array.isArray(x)` on a value typed as an array | Duplicates the compiler | Remove |
| `const data = (await res.json()) as User` | Unchecked cast at a boundary — the one place you *should* validate | `UserSchema.parse(await res.json())` |
| Silent `default: return null` in a `switch` over a union | New union members get ignored | `default: { const _: never = x; throw new Error(...) }` |
| `structuredClone` / spreading args "to be safe" | Needless cost | Copy only when shared mutation is real |

## Where defense belongs in TypeScript

- `fetch` responses: check `res.ok`, then parse the body with a schema.
- `JSON.parse`, `localStorage`, URL params, form data, `process.env` — parse into typed values once.
- Public library APIs consumed by plain JavaScript callers — validate arguments.
- Top-level boundaries: framework error handlers, React error boundaries, the CLI entry point.
- `finally` / `AbortController` for cleanup and cancellation.

## Before / After

### Optional chaining hiding structural errors

❌
```ts
function getUserCity(user: User): string {
  return user?.profile?.address?.city ?? "";
}
```

✅ (`profile` and `address` are required in the `User` type)
```ts
function getUserCity(user: User): string {
  return user.profile.address.city;
}
```

### Swallowed rejection

❌
```ts
async function getUser(id: string): Promise<User | null> {
  try {
    const res = await fetch(`/api/users/${id}`);
    return (await res.json()) as User;
  } catch (e) {
    console.error(e);
    return null;
  }
}
```
Network errors, 500s, and malformed bodies all become `null`; a 404 body is cast to `User` unchecked.

✅
```ts
async function getUser(id: string): Promise<User | null> {
  const res = await fetch(`/api/users/${id}`);
  if (res.status === 404) return null;
  if (!res.ok) throw new Error(`GET /api/users/${id} failed: ${res.status}`);
  return UserSchema.parse(await res.json());
}
```

### Redundant runtime type checks

❌
```ts
function formatPrice(amount: number, currency: string): string {
  if (typeof amount !== "number" || isNaN(amount)) return "";
  if (!currency) currency = "USD";
  return new Intl.NumberFormat("en-US", { style: "currency", currency }).format(amount);
}
```

✅
```ts
function formatPrice(amount: number, currency: string): string {
  return new Intl.NumberFormat("en-US", { style: "currency", currency }).format(amount);
}
```

### Exhaustive switch instead of a silent default

❌
```ts
function label(status: Status): string {
  switch (status) {
    case "active": return "Active";
    case "paused": return "Paused";
    default: return "";
  }
}
```

✅
```ts
function label(status: Status): string {
  switch (status) {
    case "active": return "Active";
    case "paused": return "Paused";
    default: {
      const unreachable: never = status;
      throw new Error(`Unhandled status: ${unreachable}`);
    }
  }
}
```

### Boundary validation (correct)

```ts
const OrderIn = z.object({ productId: z.string(), quantity: z.number().int().positive() });

app.post("/orders", async (req, res) => {
  const order = OrderIn.parse(req.body);          // boundary: validate external input
  res.status(201).json(await orderService.create(order)); // internal: trust the parsed order
});
```
