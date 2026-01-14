# Quick Reference Guide - Programming Best Practices

## Go Quick Reference

### Error Handling
```go
// ✅ Wrap errors with context
return fmt.Errorf("fetch user %d: %w", id, err)

// ✅ Check errors, never ignore
if err != nil {
    return err
}
```

### Concurrency
```go
// ✅ WaitGroup pattern
var wg sync.WaitGroup
wg.Add(1)  // Before goroutine
go func() {
    defer wg.Done()
    // work
}()
wg.Wait()

// ✅ Context for cancellation
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

### Resource Management
```go
// ✅ Always defer Close
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close()
```

### HTTP Production Settings
```go
// ✅ Configure timeouts
client := &http.Client{
    Timeout: 10 * time.Second,
}

server := &http.Server{
    Addr:         ":8080",
    ReadTimeout:  15 * time.Second,
    WriteTimeout: 15 * time.Second,
}
```

### Common Mistakes to Avoid
- ❌ Using `http.DefaultClient` (no timeout)
- ❌ Not checking `defer Close()` errors on writes
- ❌ Calling `wg.Add()` inside goroutine
- ❌ Variable shadowing with `:=`
- ❌ Not propagating context

---

## Rust Quick Reference

### Error Handling
```rust
// ✅ Use ? operator
let data = std::fs::read_to_string(path)?;

// ✅ Add context with anyhow
use anyhow::Context;
let data = std::fs::read_to_string(path)
    .context("Failed to read config")?;
```

### Ownership
```rust
// ✅ Borrow when you don't need ownership
fn process(data: &[u8]) { }

// ✅ Take ownership when consuming
fn process(data: Vec<u8>) { }

// ✅ Use Arc for shared ownership
let data = Arc::new(vec![1, 2, 3]);
```

### Async/Await
```rust
// ✅ Async function
async fn fetch_user(id: UserId) -> Result<User> {
    let response = client.get(url).await?;
    response.json().await
}

// ✅ Spawn tasks
tokio::spawn(async move {
    process_data().await
});
```

### Concurrency
```rust
// ✅ Protect shared state
let cache = Arc::new(Mutex::new(HashMap::new()));

// ✅ Use channels
let (tx, rx) = mpsc::channel(100);
```

### Common Mistakes to Avoid
- ❌ Using `unwrap()` in production
- ❌ Cloning unnecessarily
- ❌ Not handling async task errors
- ❌ Concurrent map access without mutex
- ❌ Copying sync types (Mutex, RwLock)

---

## Testing Commands

### Go
```bash
# Run tests with race detector
go test -race ./...

# With coverage
go test -race -coverprofile=coverage.out ./...

# Run linter
golangci-lint run
```

### Rust
```bash
# Run tests
cargo test

# With all features
cargo test --all-features

# Run clippy
cargo clippy -- -D warnings

# Check formatting
cargo fmt -- --check

# Security audit
cargo audit
```

---

## CI/CD Essentials

### Go GitHub Actions
```yaml
- name: Run tests with race detector
  run: go test -v -race ./...

- name: Run linter
  uses: golangci/golangci-lint-action@v4
```

### Rust GitHub Actions
```yaml
- name: Check formatting
  run: cargo fmt -- --check

- name: Run clippy
  run: cargo clippy -- -D warnings

- name: Run tests
  run: cargo test --all-features
```

---

## Performance Tips

### Go
- Pre-allocate slices: `make([]T, 0, capacity)`
- Use `strings.Builder` for concatenation
- Avoid `defer` in tight loops
- Profile with `pprof`

### Rust
- Use iterators instead of loops
- Pre-allocate with `Vec::with_capacity()`
- Use `Cow` for conditional cloning
- Profile with `cargo flamegraph`

---

## Production Checklist

### Both Languages
- [ ] All errors handled
- [ ] Resources properly closed
- [ ] Tests passing (including race/concurrency tests)
- [ ] Linter warnings addressed
- [ ] Documentation complete
- [ ] Logging configured
- [ ] Metrics instrumented
- [ ] Graceful shutdown implemented
- [ ] Configuration externalized
- [ ] Security audit passed
