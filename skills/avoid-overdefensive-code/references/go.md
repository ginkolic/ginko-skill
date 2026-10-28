# Go

Go's explicit error handling is **not** over-defensive. `if err != nil { return ..., err }` is the idiom — keep it. Over-defense in Go means swallowing errors, guarding against nil values that can't occur, and recovering from panics that indicate bugs.

## Go-specific rules

- **Propagate errors; don't swallow them.** Every returned `error` is either handled with a concrete recovery action or returned up (optionally wrapped).
- **Wrap with context once per meaningful layer** using `fmt.Errorf("doing X: %w", err)`. Don't re-wrap the same message at every layer, and don't wrap when the caller already has the context.
- **Rely on nil-safe zero values.** `len(nil)`, `range` over a nil slice/map, and reading from a nil map are all safe.
- **Panic is fail-fast for programmer errors.** Invariant violations and impossible states may panic (`regexp.MustCompile`, unreachable `default` in an exhaustive switch). Don't convert them into silent zero values.
- **`recover()` only at process boundaries** — the top of a long-lived worker goroutine or a server's request entry point, and only to log + report, never to continue as if nothing happened. `net/http` already recovers per request.

## Banned patterns

| Pattern | Problem | Instead |
| --- | --- | --- |
| `if err != nil { log.Println(err); return nil }` | Caller sees success | `return fmt.Errorf("...: %w", err)` |
| `if err != nil { continue }` in a loop | Silently drops items | Return the error, or collect with `errors.Join` if partial success is a requirement |
| `_ = doSomething()` / ignoring a returned `error` | Hides failures | Handle or return it |
| `n, _ := strconv.Atoi(s)` | Coercion: bad input silently becomes `0` | Check the error; return it or reject the input |
| `if v, ok := m["user_id"]; ok { ... } else { v = -1 }` for required keys | Fabricates data | Return an error when a required key is missing |
| Returning zero value + `nil` error on failure | Caller can't tell "empty" from "failed" | Return a non-nil error |
| `if s == nil { return }` at the top of internal methods | Hides the bug that produced a nil receiver | Let it panic; fix the caller |
| `if len(xs) == 0 { return }` before `for range xs` | Redundant; range over empty/nil is a no-op | Remove |
| `if m != nil { v, ok := m[k] ... }` | Reading a nil map is safe | Remove the nil check |
| `if ctx == nil { ctx = context.Background() }` | Masks a caller bug | Require a context; let nil fail |
| `v, ok := x.(T); if !ok { return }` when `x` is always `T` | Hides a broken invariant | Single-value assertion `x.(T)` (panics on bug), or a type switch with a panicking default |
| `defer func() { recover() }()` in ordinary functions | Turns crashes into corrupt state | Remove; recover only at process boundaries |
| Adding a `sync.Mutex` to a type that's never shared across goroutines | Noise, false signal of concurrency | Add when concurrency is real; document ownership |
| `errors.Is(err, X)` branches for errors the callee can never return | Dead code | Remove |
| Wrapping at every layer: `"get user: get user from db: query user: ..."` | Repetitive messages | Wrap where you add new information |

## Where defense belongs in Go

- Every `error` from I/O, network, `os`, `exec`, `json.Unmarshal`, and third-party SDKs.
- Validating decoded external input (request bodies, config files, env vars) right after decoding.
- `defer f.Close()` for cleanup, and **checking the `Close` error on writable files** (it can report a failed flush).
- Honoring `ctx.Done()` / `ctx.Err()` in long-running loops and blocking calls.
- Exported functions of a library: validate arguments and return descriptive errors.

## Before / After

### Swallowed error

❌
```go
func LoadConfig(path string) *Config {
	data, err := os.ReadFile(path)
	if err != nil {
		log.Printf("read config: %v", err)
		return &Config{}
	}
	var cfg Config
	if err := json.Unmarshal(data, &cfg); err != nil {
		log.Printf("parse config: %v", err)
		return &Config{}
	}
	return &cfg
}
```

✅
```go
func LoadConfig(path string) (*Config, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("read config: %w", err)
	}
	var cfg Config
	if err := json.Unmarshal(data, &cfg); err != nil {
		return nil, fmt.Errorf("parse config %s: %w", path, err)
	}
	return &cfg, nil
}
```
If "missing file means defaults" is an explicit requirement, handle only that case:
```go
if errors.Is(err, fs.ErrNotExist) {
	return DefaultConfig(), nil
}
```

### Redundant nil and length guards

❌
```go
func (s *Service) Total(items []Item) int {
	if s == nil {
		return 0
	}
	if items == nil || len(items) == 0 {
		return 0
	}
	total := 0
	for _, it := range items {
		if it.Price < 0 {
			continue
		}
		total += it.Price
	}
	return total
}
```

✅
```go
func (s *Service) Total(items []Item) int {
	total := 0
	for _, it := range items {
		total += it.Price
	}
	return total
}
```
If a negative price is possible, it's a validation problem at the boundary where items enter the system — not something to silently skip here.

### Recover hiding a bug

❌
```go
func process(job Job) (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = nil // keep going
		}
	}()
	return job.Run()
}
```

✅
```go
func process(job Job) error {
	return job.Run()
}

// Only at the worker boundary:
func worker(jobs <-chan Job) {
	for job := range jobs {
		func() {
			defer func() {
				if r := recover(); r != nil {
					log.Printf("job %s panicked: %v\n%s", job.ID, r, debug.Stack())
					metrics.JobPanics.Inc()
				}
			}()
			if err := process(job); err != nil {
				log.Printf("job %s failed: %v", job.ID, err)
			}
		}()
	}
}
```

### Boundary validation (correct)

```go
func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
	var in OrderIn
	if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
		http.Error(w, "invalid JSON", http.StatusBadRequest)
		return
	}
	if err := in.Validate(); err != nil { // boundary: validate external input
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
	order, err := h.orders.Create(r.Context(), in) // internal: trust validated input
	if err != nil {
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	writeJSON(w, http.StatusCreated, order)
}
```
