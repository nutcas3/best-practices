# Go Best Practices - Production-Ready Development Guide

## Skill Metadata
- **Domain**: Go Programming Language
- **Skill Level**: Intermediate to Advanced
- **Last Updated**: 2024
- **Sources**: 100 Go Mistakes and How to Avoid Them, Effective Go, Go Project Layout Standards

## Overview

This skill document provides comprehensive best practices for building production-ready Go applications. It combines lessons from industry-standard resources to help AI agents and developers write idiomatic, efficient, and maintainable Go code.

---

## 1. Project Organization & Structure

### Standard Project Layout

Use the following directory structure for maintainable, scalable Go projects:

```
/project-root
├── cmd/                    # Application entry points (main packages)
│   ├── api/               # API server binary
│   │   └── main.go
│   └── worker/            # Background worker binary
│       └── main.go
├── internal/              # Private application code (cannot be imported externally)
│   ├── domain/           # Business logic and domain models
│   ├── handler/          # HTTP handlers
│   ├── repository/       # Data access layer
│   └── service/          # Business service layer
├── pkg/                   # Public library code (can be imported by external projects)
│   └── logger/           # Reusable logging utilities
├── api/                   # API contracts and definitions
│   ├── openapi/          # OpenAPI/Swagger specs
│   └── proto/            # Protocol Buffer definitions
├── configs/               # Configuration files
│   ├── dev.yaml
│   └── prod.yaml
├── deployments/           # Deployment configurations
│   ├── docker/           # Dockerfiles
│   └── kubernetes/       # K8s manifests
├── scripts/               # Build and deployment scripts
├── test/                  # Additional test data and integration tests
├── .golangci.yml         # Linter configuration
├── go.mod                # Go module definition
├── go.sum                # Dependency checksums
├── Makefile              # Build automation
└── README.md             # Project documentation
```

### Key Principles

**✅ DO:**
- Place all application entry points in `/cmd` with descriptive subdirectories
- Use `/internal` for code that should remain private to your project
- Keep `/pkg` minimal and only for truly reusable libraries
- Organize code by domain/feature, not by technical layer

**❌ DON'T:**
- Create flat structures with all files in the root
- Use generic names like `utils`, `common`, or `helpers` as package names
- Mix business logic with infrastructure concerns
- Export everything by default

### Package Naming Best Practices

```go
// ❌ BAD: Generic, meaningless names
package utils
package helpers
package common

// ✅ GOOD: Descriptive, purpose-driven names
package encoding
package http
package postgres
```

**Rules:**
1. Name packages after what they **provide**, not what they contain
2. Use singular nouns (e.g., `user`, not `users`)
3. Keep names short and lowercase
4. Avoid stuttering: `http.Server`, not `http.HTTPServer`

### Minimize Exports

```go
// ❌ BAD: Everything exported
type UserRepository struct {
    DB *sql.DB
}

func (r *UserRepository) FindByID(id int) (*User, error) { }
func (r *UserRepository) BuildQuery(filters map[string]string) string { }
func (r *UserRepository) ValidateInput(input string) bool { }

// ✅ GOOD: Only essential parts exported
type UserRepository struct {
    db *sql.DB  // unexported
}

func (r *UserRepository) FindByID(id int) (*User, error) { }

// Helper methods unexported
func (r *UserRepository) buildQuery(filters map[string]string) string { }
func (r *UserRepository) validateInput(input string) bool { }
```

---

## 2. Idiomatic Code & Control Structures

### The Happy Path Left-Aligned

Keep successful execution flow aligned to the left by handling errors early:

```go
// ❌ BAD: Deep nesting
func ProcessUser(id int) error {
    user, err := fetchUser(id)
    if err == nil {
        if user.IsActive {
            profile, err := fetchProfile(user.ID)
            if err == nil {
                if profile.IsComplete {
                    return updateUser(user, profile)
                } else {
                    return errors.New("incomplete profile")
                }
            } else {
                return err
            }
        } else {
            return errors.New("inactive user")
        }
    }
    return err
}

// ✅ GOOD: Early returns, happy path left-aligned
func ProcessUser(id int) error {
    user, err := fetchUser(id)
    if err != nil {
        return fmt.Errorf("fetch user: %w", err)
    }
    
    if !user.IsActive {
        return errors.New("inactive user")
    }
    
    profile, err := fetchProfile(user.ID)
    if err != nil {
        return fmt.Errorf("fetch profile: %w", err)
    }
    
    if !profile.IsComplete {
        return errors.New("incomplete profile")
    }
    
    return updateUser(user, profile)
}
```

### Avoid Variable Shadowing

```go
// ❌ BAD: Variable shadowing with :=
func GetConfig() error {
    config, err := loadConfig()
    if err != nil {
        return err
    }
    
    if config.UseCache {
        // This creates a NEW 'config' variable in this scope!
        config, err := loadCacheConfig()
        if err != nil {
            return err
        }
        // Original config is unchanged
        fmt.Println(config.CacheSize)
    }
    
    // Still using the original config here
    return nil
}

// ✅ GOOD: Explicit variable names or reuse
func GetConfig() error {
    config, err := loadConfig()
    if err != nil {
        return err
    }
    
    if config.UseCache {
        cacheConfig, err := loadCacheConfig()
        if err != nil {
            return err
        }
        config.CacheSize = cacheConfig.CacheSize
    }
    
    return nil
}
```

### Functional Options Pattern

For constructors with many optional parameters:

```go
// ❌ BAD: Too many parameters
func NewServer(addr string, timeout int, maxConns int, enableTLS bool, 
               certFile string, keyFile string, logger *log.Logger) *Server {
    // ...
}

// ✅ GOOD: Functional options pattern
type Server struct {
    addr      string
    timeout   time.Duration
    maxConns  int
    enableTLS bool
    certFile  string
    keyFile   string
    logger    *log.Logger
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = d
    }
}

func WithTLS(certFile, keyFile string) Option {
    return func(s *Server) {
        s.enableTLS = true
        s.certFile = certFile
        s.keyFile = keyFile
    }
}

func WithLogger(logger *log.Logger) Option {
    return func(s *Server) {
        s.logger = logger
    }
}

func NewServer(addr string, opts ...Option) *Server {
    s := &Server{
        addr:     addr,
        timeout:  30 * time.Second, // defaults
        maxConns: 100,
        logger:   log.Default(),
    }
    
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}

// Usage
server := NewServer(":8080",
    WithTimeout(60*time.Second),
    WithTLS("cert.pem", "key.pem"),
)
```

### Avoid init() Functions

```go
// ❌ BAD: Using init() for state
var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", "connection-string")
    if err != nil {
        // Can't return error from init()!
        panic(err)
    }
}

// ✅ GOOD: Explicit initialization
type Database struct {
    conn *sql.DB
}

func NewDatabase(connStr string) (*Database, error) {
    conn, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, fmt.Errorf("open database: %w", err)
    }
    
    if err := conn.Ping(); err != nil {
        return nil, fmt.Errorf("ping database: %w", err)
    }
    
    return &Database{conn: conn}, nil
}
```

---

## 3. Concurrency Best Practices

### Goroutine Lifecycle Management

**Golden Rule:** Never start a goroutine without knowing how it will stop.

```go
// ❌ BAD: Goroutine leak - no way to stop
func StartProcessor() {
    go func() {
        for {
            process()
            time.Sleep(time.Second)
        }
    }()
}

// ✅ GOOD: Controlled lifecycle with context
func StartProcessor(ctx context.Context) {
    go func() {
        ticker := time.NewTicker(time.Second)
        defer ticker.Stop()
        
        for {
            select {
            case <-ticker.C:
                process()
            case <-ctx.Done():
                return
            }
        }
    }()
}
```

### WaitGroup Best Practices

```go
// ❌ BAD: Race condition - Add() called in goroutine
func ProcessItems(items []Item) {
    var wg sync.WaitGroup
    
    for _, item := range items {
        go func(i Item) {
            wg.Add(1)  // WRONG: May not execute before wg.Wait()
            defer wg.Done()
            process(i)
        }(item)
    }
    
    wg.Wait()
}

// ✅ GOOD: Add() called before starting goroutine
func ProcessItems(items []Item) {
    var wg sync.WaitGroup
    
    for _, item := range items {
        wg.Add(1)  // Correct: Called in parent goroutine
        go func(i Item) {
            defer wg.Done()
            process(i)
        }(item)
    }
    
    wg.Wait()
}
```

### Using errgroup for Error Handling

```go
import "golang.org/x/sync/errgroup"

// ✅ GOOD: Coordinated error handling and cancellation
func FetchMultipleResources(ctx context.Context, ids []string) error {
    g, ctx := errgroup.WithContext(ctx)
    
    results := make([]Resource, len(ids))
    
    for i, id := range ids {
        i, id := i, id // Capture loop variables
        g.Go(func() error {
            resource, err := fetchResource(ctx, id)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", id, err)
            }
            results[i] = resource
            return nil
        })
    }
    
    // Wait for all goroutines, returns first error if any
    if err := g.Wait(); err != nil {
        return err
    }
    
    return processResults(results)
}
```

### Context Propagation

```go
// ✅ GOOD: Context passed through call chain
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Add timeout for downstream operations
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    result, err := fetchData(ctx)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(result)
}

func fetchData(ctx context.Context) (*Data, error) {
    // Context propagated to database call
    return db.QueryContext(ctx, "SELECT ...")
}
```

### Safe Map Access

```go
// ❌ BAD: Concurrent map access causes panic
type Cache struct {
    data map[string]string
}

func (c *Cache) Set(key, value string) {
    c.data[key] = value  // PANIC if called concurrently!
}

// ✅ GOOD: Protected with mutex
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

// ✅ BETTER: Use sync.Map for high-concurrency scenarios
type Cache struct {
    data sync.Map
}

func (c *Cache) Set(key, value string) {
    c.data.Store(key, value)
}

func (c *Cache) Get(key string) (string, bool) {
    val, ok := c.data.Load(key)
    if !ok {
        return "", false
    }
    return val.(string), true
}
```

### Never Copy Sync Types

```go
// ❌ BAD: Copying mutex breaks synchronization
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c Counter) Increment() {  // Receiver by value - copies mutex!
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// ✅ GOOD: Pointer receiver
func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}
```

---

## 4. Data Types & Memory Management

### Slice Pre-allocation

```go
// ❌ BAD: Repeated allocations
func ProcessItems(n int) []Result {
    var results []Result
    for i := 0; i < n; i++ {
        results = append(results, process(i))  // May reallocate multiple times
    }
    return results
}

// ✅ GOOD: Pre-allocated capacity
func ProcessItems(n int) []Result {
    results := make([]Result, 0, n)  // Allocate once
    for i := 0; i < n; i++ {
        results = append(results, process(i))
    }
    return results
}

// ✅ ALSO GOOD: Pre-allocated length if you know exact size
func ProcessItems(n int) []Result {
    results := make([]Result, n)
    for i := 0; i < n; i++ {
        results[i] = process(i)
    }
    return results
}
```

### Slice Memory Leaks

```go
// ❌ BAD: Sub-slice keeps entire backing array in memory
func GetFirstTen(data []byte) []byte {
    // If data is 1GB, the entire 1GB stays in memory!
    return data[:10]
}

// ✅ GOOD: Copy to new slice
func GetFirstTen(data []byte) []byte {
    result := make([]byte, 10)
    copy(result, data[:10])
    return result
}
```

### Map Memory Management

```go
// Maps never shrink in Go - they only grow
type Cache struct {
    data map[string][]byte
}

// ❌ BAD: Map grows indefinitely
func (c *Cache) Set(key string, value []byte) {
    c.data[key] = value
}

func (c *Cache) Delete(key string) {
    delete(c.data, key)  // Memory not freed!
}

// ✅ GOOD: Recreate map periodically if size fluctuates
func (c *Cache) Cleanup() {
    if len(c.data) < cap(c.data)/2 {
        // Map is less than half full, recreate it
        newData := make(map[string][]byte, len(c.data))
        for k, v := range c.data {
            newData[k] = v
        }
        c.data = newData
    }
}
```

### Nil vs Empty Slices

```go
// ✅ GOOD: Prefer nil slices
var items []string  // nil slice, no allocation

// Only use empty slice if you need to distinguish in JSON
emptyItems := []string{}  // Empty slice, allocates
```

---

## 5. Error Management & Reliability

### Error Wrapping

```go
// ❌ BAD: Lost error context
func LoadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err  // Lost information about what we were doing
    }
    // ...
}

// ✅ GOOD: Wrapped errors with context
func LoadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("load config from %s: %w", path, err)
    }
    
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }
    
    return &cfg, nil
}

// Usage: Can inspect wrapped errors
cfg, err := LoadConfig("config.json")
if err != nil {
    if errors.Is(err, os.ErrNotExist) {
        // Handle missing file specifically
    }
    return err
}
```

### Custom Error Types

```go
// ✅ GOOD: Structured errors
type ValidationError struct {
    Field string
    Value interface{}
    Msg   string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed for %s: %s", e.Field, e.Msg)
}

func ValidateUser(u *User) error {
    if u.Email == "" {
        return &ValidationError{
            Field: "email",
            Value: u.Email,
            Msg:   "email is required",
        }
    }
    return nil
}

// Usage
if err := ValidateUser(user); err != nil {
    var valErr *ValidationError
    if errors.As(err, &valErr) {
        // Handle validation error specifically
        log.Printf("Invalid field: %s", valErr.Field)
    }
}
```

### Don't Handle Errors Twice

```go
// ❌ BAD: Logging and returning
func ProcessOrder(id string) error {
    order, err := fetchOrder(id)
    if err != nil {
        log.Printf("Failed to fetch order: %v", err)  // Logged here
        return err  // AND returned - caller might log again!
    }
    return nil
}

// ✅ GOOD: Return error, let caller decide
func ProcessOrder(id string) error {
    order, err := fetchOrder(id)
    if err != nil {
        return fmt.Errorf("fetch order %s: %w", id, err)
    }
    return nil
}

// Caller logs once at appropriate level
if err := ProcessOrder(orderID); err != nil {
    log.Printf("Process order failed: %v", err)
}
```

### Resource Management with defer

```go
// ✅ GOOD: Always defer Close() and check errors
func ReadConfig(path string) (*Config, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()  // Guaranteed to run
    
    var cfg Config
    if err := json.NewDecoder(f).Decode(&cfg); err != nil {
        return nil, err
    }
    
    return &cfg, nil
}

// ✅ GOOD: Check Close() error for writes
func WriteConfig(path string, cfg *Config) error {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = cerr  // Return close error if no other error
        }
    }()
    
    return json.NewEncoder(f).Encode(cfg)
}

// ✅ GOOD: HTTP response body must be closed
func FetchData(url string) ([]byte, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()  // Critical: prevents connection leak
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }
    
    return io.ReadAll(resp.Body)
}
```

### Use time.Duration, Not Integers

```go
// ❌ BAD: Ambiguous units
func SetTimeout(timeout int) {
    // Is this seconds? Milliseconds? Minutes?
}

// ✅ GOOD: Explicit duration type
func SetTimeout(timeout time.Duration) {
    // Clear and type-safe
}

// Usage
SetTimeout(30 * time.Second)
SetTimeout(500 * time.Millisecond)
```

---

## 6. Testing & Production Readiness

### Table-Driven Tests

```go
// ✅ GOOD: Standard Go testing pattern
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        {
            name:    "valid email",
            email:   "user@example.com",
            wantErr: false,
        },
        {
            name:    "missing @",
            email:   "userexample.com",
            wantErr: true,
        },
        {
            name:    "empty email",
            email:   "",
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateEmail(tt.email)
            if (err != nil) != tt.wantErr {
                t.Errorf("ValidateEmail() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

### Race Detection

```bash
# Always run tests with race detector
go test -race ./...

# Run specific package
go test -race ./internal/service

# With coverage
go test -race -coverprofile=coverage.out ./...
```

### Production HTTP Clients

```go
// ❌ BAD: Default client has no timeouts
resp, err := http.Get("https://api.example.com/data")

// ✅ GOOD: Custom client with timeouts
var httpClient = &http.Client{
    Timeout: 10 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 10 * time.Second,
    },
}

func FetchData(url string) ([]byte, error) {
    resp, err := httpClient.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    return io.ReadAll(resp.Body)
}
```

### Production HTTP Servers

```go
// ❌ BAD: Default server has no timeouts
http.ListenAndServe(":8080", handler)

// ✅ GOOD: Configured server with timeouts
server := &http.Server{
    Addr:         ":8080",
    Handler:      handler,
    ReadTimeout:  15 * time.Second,
    WriteTimeout: 15 * time.Second,
    IdleTimeout:  60 * time.Second,
}

if err := server.ListenAndServe(); err != nil {
    log.Fatal(err)
}
```

### Graceful Shutdown

```go
func main() {
    server := &http.Server{
        Addr:    ":8080",
        Handler: setupRoutes(),
    }
    
    // Start server in goroutine
    go func() {
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server failed: %v", err)
        }
    }()
    
    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit
    
    log.Println("Shutting down server...")
    
    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Server forced to shutdown: %v", err)
    }
    
    log.Println("Server exited")
}
```

---

## 7. Interface Design

### Accept Interfaces, Return Structs

```go
// ✅ GOOD: Accept interface (flexible)
func SaveUser(w io.Writer, user *User) error {
    return json.NewEncoder(w).Encode(user)
}

// ✅ GOOD: Return concrete type (clear)
func NewUserService(db *sql.DB) *UserService {
    return &UserService{db: db}
}
```

### Define Interfaces at Usage Site

```go
// ❌ BAD: Interface defined with implementation
package database

type UserRepository interface {
    FindByID(id int) (*User, error)
    Save(user *User) error
}

type PostgresUserRepository struct{}

func (r *PostgresUserRepository) FindByID(id int) (*User, error) { }
func (r *PostgresUserRepository) Save(user *User) error { }

// ✅ GOOD: Interface defined where it's used
package service

// Only define what you need
type userRepository interface {
    FindByID(id int) (*User, error)
}

type UserService struct {
    repo userRepository
}

// Implementation in separate package
package database

type UserRepository struct{}

func (r *UserRepository) FindByID(id int) (*User, error) { }
func (r *UserRepository) Save(user *User) error { }
```

---

## 8. Performance Optimization

### String Concatenation

```go
// ❌ BAD: String concatenation in loop
func BuildQuery(fields []string) string {
    query := "SELECT "
    for i, field := range fields {
        query += field  // Creates new string each iteration
        if i < len(fields)-1 {
            query += ", "
        }
    }
    return query
}

// ✅ GOOD: Use strings.Builder
func BuildQuery(fields []string) string {
    var b strings.Builder
    b.WriteString("SELECT ")
    for i, field := range fields {
        b.WriteString(field)
        if i < len(fields)-1 {
            b.WriteString(", ")
        }
    }
    return b.String()
}
```

### Avoid Unnecessary Allocations

```go
// ❌ BAD: Allocates on every call
func GetDefaultConfig() Config {
    return Config{
        Timeout:  30 * time.Second,
        MaxRetry: 3,
    }
}

// ✅ GOOD: Reuse constant
var defaultConfig = Config{
    Timeout:  30 * time.Second,
    MaxRetry: 3,
}

func GetDefaultConfig() Config {
    return defaultConfig
}
```

---

## 9. CI/CD Configuration

### GitHub Actions Workflow

```yaml
name: Go CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.22'
        cache: true
    
    - name: Download dependencies
      run: go mod download
    
    - name: Verify dependencies
      run: go mod verify
    
    - name: Run go vet
      run: go vet ./...
    
    - name: Run golangci-lint
      uses: golangci/golangci-lint-action@v4
      with:
        version: latest
        args: --timeout=5m
    
    - name: Run tests with race detector
      run: go test -v -race -coverprofile=coverage.out ./...
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.out
    
    - name: Run integration tests
      run: go test -v -tags=integration ./test/integration/...
      env:
        DATABASE_URL: postgres://test:test@localhost:5432/testdb

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.22'
    
    - name: Build binaries
      run: |
        go build -v -o bin/api ./cmd/api
        go build -v -o bin/worker ./cmd/worker
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: binaries
        path: bin/
```

---

## 10. Linter Configuration

### .golangci.yml

```yaml
run:
  timeout: 5m
  tests: true
  modules-download-mode: readonly

linters:
  enable:
    - errcheck      # Check for unchecked errors
    - gosimple      # Simplify code
    - govet         # Standard Go analyzer
    - ineffassign   # Detect ineffectual assignments
    - staticcheck   # Advanced static analysis
    - unused        # Find unused code
    - gofmt         # Check formatting
    - goimports     # Check imports
    - misspell      # Fix spelling mistakes
    - revive        # Fast, configurable linter
    - gosec         # Security issues
    - bodyclose     # Check HTTP body closed
    - noctx         # Find HTTP requests without context
    - errname       # Check error naming
    - errorlint     # Find error wrapping issues

linters-settings:
  errcheck:
    check-type-assertions: true
    check-blank: true
  
  govet:
    check-shadowing: true
  
  revive:
    rules:
      - name: exported
        severity: warning
      - name: unexported-return
        severity: warning
      - name: var-naming
        severity: warning

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - errcheck
        - gosec
```

---

## Summary Checklist

### Before Committing Code

- [ ] All errors are handled or explicitly ignored with `_ = err`
- [ ] Resources (files, HTTP bodies, DB rows) are closed with `defer`
- [ ] No goroutines without clear lifecycle management
- [ ] `wg.Add()` called before starting goroutines
- [ ] Context passed through call chains
- [ ] No variable shadowing with `:=`
- [ ] Slices pre-allocated when size is known
- [ ] HTTP clients and servers have timeouts configured
- [ ] Tests pass with `-race` flag
- [ ] Linter passes with no warnings

### Production Deployment

- [ ] Graceful shutdown implemented
- [ ] Structured logging in place
- [ ] Metrics and monitoring configured
- [ ] Health check endpoints exposed
- [ ] Configuration externalized (12-factor app)
- [ ] Secrets managed securely (not in code)
- [ ] Database migrations automated
- [ ] CI/CD pipeline configured

---

## References

1. **100 Go Mistakes and How to Avoid Them** - https://100go.co/
2. **Effective Go** - https://go.dev/doc/effective_go
3. **Go Project Layout** - https://github.com/golang-standards/project-layout
4. **Go Code Review Comments** - https://go.dev/wiki/CodeReviewComments
5. **Uber Go Style Guide** - https://github.com/uber-go/guide/blob/master/style.md

---

## Training Notes for AI Agents

When generating Go code, prioritize:

1. **Correctness**: Handle all errors, avoid data races
2. **Clarity**: Keep code simple and readable
3. **Idiomatic**: Follow Go conventions and patterns
4. **Performance**: Pre-allocate when possible, avoid unnecessary allocations
5. **Maintainability**: Structure code for long-term maintenance

Always prefer standard library solutions over third-party dependencies unless there's a compelling reason.
