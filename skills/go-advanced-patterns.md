# Advanced Go Patterns and Architectures

## Advanced Concurrency Patterns

### Fan-Out Fan-In Pattern

```go
// ✅ GOOD: Parallel processing with controlled concurrency
func fanOutFanIn(tasks []Task, workers int) []Result {
    // Create channels
    taskCh := make(chan Task, len(tasks))
    resultCh := make(chan Result, len(tasks))
    doneCh := make(chan struct{})

    // Fan Out: Start workers
    for i := 0; i < workers; i++ {
        go func() {
            for task := range taskCh {
                result := process(task)
                resultCh <- result
            }
        }()
    }

    // Send tasks
    go func() {
        for _, task := range tasks {
            taskCh <- task
        }
        close(taskCh)
    }()

    // Fan In: Collect results
    var results []Result
    go func() {
        for i := 0; i < len(tasks); i++ {
            result := <-resultCh
            results = append(results, result)
        }
        close(doneCh)
    }()

    <-doneCh
    return results
}
```

### Adaptive Worker Pool

```go
// ✅ GOOD: Worker pool that adapts to load
type AdaptivePool struct {
    tasks    chan Task
    results  chan Result
    workers  int32
    minSize  int32
    maxSize  int32
    metrics  *Metrics
}

func (p *AdaptivePool) adjustWorkers() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for range ticker.C {
        queueSize := len(p.tasks)
        currentWorkers := atomic.LoadInt32(&p.workers)

        switch {
        case queueSize > 100 && currentWorkers < p.maxSize:
            // Add workers if queue is backing up
            p.addWorker()
        case queueSize < 10 && currentWorkers > p.minSize:
            // Remove workers if queue is small
            p.removeWorker()
        }
    }
}

func (p *AdaptivePool) addWorker() {
    atomic.AddInt32(&p.workers, 1)
    go p.worker()
}

func (p *AdaptivePool) removeWorker() {
    atomic.AddInt32(&p.workers, -1)
}

func (p *AdaptivePool) worker() {
    for task := range p.tasks {
        if atomic.LoadInt32(&p.workers) <= p.minSize {
            break
        }
        result := task.Process()
        p.results <- result
        p.metrics.RecordProcessingTime(time.Since(task.StartTime))
    }
}
```

### Circuit Breaker Pattern

```go
// ✅ GOOD: Circuit breaker for external service calls
type CircuitBreaker struct {
    mu           sync.RWMutex
    failureCount int
    lastFailure  time.Time
    state        State
    threshold    int
    timeout      time.Duration
}

type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

func (cb *CircuitBreaker) Execute(fn func() error) error {
    if !cb.allowRequest() {
        return ErrCircuitOpen
    }

    err := fn()
    cb.recordResult(err)
    return err
}

func (cb *CircuitBreaker) allowRequest() bool {
    cb.mu.RLock()
    defer cb.mu.RUnlock()

    switch cb.state {
    case StateClosed:
        return true
    case StateOpen:
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.mu.RUnlock()
            cb.mu.Lock()
            cb.state = StateHalfOpen
            cb.mu.Unlock()
            cb.mu.RLock()
            return true
        }
        return false
    case StateHalfOpen:
        return true
    default:
        return false
    }
}

func (cb *CircuitBreaker) recordResult(err error) {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    if err != nil {
        cb.failureCount++
        cb.lastFailure = time.Now()
        if cb.failureCount >= cb.threshold {
            cb.state = StateOpen
        }
    } else {
        if cb.state == StateHalfOpen {
            cb.state = StateClosed
        }
        cb.failureCount = 0
    }
}
```

## Microservices Patterns

### Service Mesh Pattern

```go
// ✅ GOOD: Service mesh proxy implementation
type ServiceProxy struct {
    target      string
    retries     int
    timeout     time.Duration
    circuitBreaker *CircuitBreaker
    loadBalancer  *LoadBalancer
    metrics      *Metrics
    tracer       *Tracer
}

func (p *ServiceProxy) Call(ctx context.Context, req *Request) (*Response, error) {
    start := time.Now()
    span := p.tracer.StartSpan(ctx, "service_call")
    defer span.End()

    for i := 0; i <= p.retries; i++ {
        endpoint := p.loadBalancer.Next()
        
        ctx, cancel := context.WithTimeout(ctx, p.timeout)
        defer cancel()

        resp, err := p.circuitBreaker.Execute(func() error {
            return p.doCall(ctx, endpoint, req)
        })

        if err == nil {
            p.metrics.RecordSuccess(time.Since(start))
            return resp, nil
        }

        p.metrics.RecordError(err)
        span.RecordError(err)

        if !isRetryable(err) || i == p.retries {
            return nil, err
        }

        // Exponential backoff
        time.Sleep(time.Duration(i*i) * 100 * time.Millisecond)
    }

    return nil, ErrMaxRetriesExceeded
}
```

### Event Sourcing Pattern

```go
// ✅ GOOD: Event sourcing with CQRS
type Event struct {
    ID        string
    Type      string
    Data      interface{}
    Timestamp time.Time
    Version   int64
}

type EventStore interface {
    Append(ctx context.Context, events ...*Event) error
    Get(ctx context.Context, aggregateID string) ([]*Event, error)
}

type Aggregate struct {
    ID       string
    Version  int64
    events   []*Event
    snapshot interface{}
}

func (a *Aggregate) Apply(event *Event) error {
    switch e := event.Data.(type) {
    case *UserCreated:
        return a.applyUserCreated(e)
    case *UserUpdated:
        return a.applyUserUpdated(e)
    default:
        return fmt.Errorf("unknown event type: %T", e)
    }
}

func (a *Aggregate) Replay(events []*Event) error {
    for _, event := range events {
        if err := a.Apply(event); err != nil {
            return err
        }
        a.Version = event.Version
    }
    return nil
}
```

### API Gateway Pattern

```go
// ✅ GOOD: API Gateway with rate limiting and auth
type Gateway struct {
    router       *mux.Router
    rateLimiter  *RateLimiter
    auth         *Authenticator
    services     map[string]*ServiceClient
    cache        *Cache
}

func (g *Gateway) Handle(pattern string, handler http.Handler) {
    // Wrap handler with middleware chain
    wrapped := handler
    wrapped = g.withMetrics(wrapped)
    wrapped = g.withRateLimit(wrapped)
    wrapped = g.withAuth(wrapped)
    wrapped = g.withCache(wrapped)
    wrapped = g.withTimeout(wrapped)
    wrapped = g.withRetry(wrapped)
    wrapped = g.withCircuitBreaker(wrapped)

    g.router.Handle(pattern, wrapped)
}

type RateLimiter struct {
    mu       sync.RWMutex
    windows  map[string]*SlidingWindow
    limit    rate.Limit
    capacity int
}

func (rl *RateLimiter) Allow(key string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    window, exists := rl.windows[key]
    if !exists {
        window = NewSlidingWindow(rl.capacity)
        rl.windows[key] = window
    }

    return window.Allow()
}
```

## Advanced Design Patterns

### Options Pattern with Validation

```go
// ✅ GOOD: Options pattern with validation
type ServerOption func(*Server) error

type Server struct {
    addr     string
    port     int
    tls      *tls.Config
    timeout  time.Duration
    handlers map[string]http.Handler
    metrics  *Metrics
    logger   *zap.Logger
}

func WithAddress(addr string) ServerOption {
    return func(s *Server) error {
        if addr == "" {
            return errors.New("address cannot be empty")
        }
        s.addr = addr
        return nil
    }
}

func WithTLS(cert, key string) ServerOption {
    return func(s *Server) error {
        tlsConfig, err := loadTLSConfig(cert, key)
        if err != nil {
            return fmt.Errorf("load TLS config: %w", err)
        }
        s.tls = tlsConfig
        return nil
    }
}

func NewServer(opts ...ServerOption) (*Server, error) {
    s := &Server{
        addr:     ":8080",
        timeout:  30 * time.Second,
        handlers: make(map[string]http.Handler),
        logger:   zap.NewNop(),
    }

    for _, opt := range opts {
        if err := opt(s); err != nil {
            return nil, err
        }
    }

    return s, nil
}
```

### Decorator Pattern with Generics

```go
// ✅ GOOD: Generic decorator pattern
type Handler[T any] interface {
    Handle(ctx context.Context, req T) error
}

type LoggingDecorator[T any] struct {
    next   Handler[T]
    logger *zap.Logger
}

func (d *LoggingDecorator[T]) Handle(ctx context.Context, req T) error {
    start := time.Now()
    err := d.next.Handle(ctx, req)
    d.logger.Info("handled request",
        zap.String("type", fmt.Sprintf("%T", req)),
        zap.Duration("duration", time.Since(start)),
        zap.Error(err),
    )
    return err
}

type MetricsDecorator[T any] struct {
    next    Handler[T]
    metrics *Metrics
}

func (d *MetricsDecorator[T]) Handle(ctx context.Context, req T) error {
    start := time.Now()
    err := d.next.Handle(ctx, req)
    d.metrics.RecordLatency(fmt.Sprintf("%T", req), time.Since(start))
    if err != nil {
        d.metrics.RecordError(fmt.Sprintf("%T", req))
    }
    return err
}
```

### Domain-Driven Design Patterns

```go
// ✅ GOOD: DDD patterns in Go
type AggregateRoot struct {
    ID        uuid.UUID
    Version   int
    Events    []Event
    CreatedAt time.Time
    UpdatedAt time.Time
}

func (ar *AggregateRoot) ApplyEvent(event Event) {
    ar.Events = append(ar.Events, event)
    ar.Version++
    ar.UpdatedAt = time.Now()
}

type Repository[T AggregateRoot] interface {
    Save(ctx context.Context, aggregate T) error
    FindByID(ctx context.Context, id uuid.UUID) (T, error)
}

type EventSourcedRepository[T AggregateRoot] struct {
    eventStore EventStore
    factory    func() T
}

func (r *EventSourcedRepository[T]) Save(ctx context.Context, aggregate T) error {
    return r.eventStore.Append(ctx, aggregate.Events...)
}

func (r *EventSourcedRepository[T]) FindByID(ctx context.Context, id uuid.UUID) (T, error) {
    events, err := r.eventStore.Get(ctx, id.String())
    if err != nil {
        return r.factory(), err
    }

    aggregate := r.factory()
    for _, event := range events {
        aggregate.ApplyEvent(event)
    }
    return aggregate, nil
}
```

## Performance Patterns

### Memory Pool with Size Classes

```go
// ✅ GOOD: Memory pool with size classes for better memory usage
type SizeClass struct {
    size  int
    pool  sync.Pool
    stats *PoolStats
}

type PoolStats struct {
    hits   uint64
    misses uint64
}

type MemoryPool struct {
    classes []SizeClass
    maxSize int
}

func NewMemoryPool(maxSize int) *MemoryPool {
    // Create size classes: 64B, 256B, 1KB, 4KB, 16KB, 64KB
    sizes := []int{64, 256, 1024, 4096, 16384, 65536}
    classes := make([]SizeClass, len(sizes))
    
    for i, size := range sizes {
        classes[i] = SizeClass{
            size: size,
            pool: sync.Pool{
                New: func() interface{} {
                    return make([]byte, size)
                },
            },
            stats: &PoolStats{},
        }
    }

    return &MemoryPool{
        classes: classes,
        maxSize: maxSize,
    }
}

func (p *MemoryPool) Get(size int) []byte {
    // Find appropriate size class
    for i := range p.classes {
        if p.classes[i].size >= size {
            buf := p.classes[i].pool.Get().([]byte)
            atomic.AddUint64(&p.classes[i].stats.hits, 1)
            return buf[:size]
        }
    }
    
    // If too large, allocate directly
    atomic.AddUint64(&p.classes[len(p.classes)-1].stats.misses, 1)
    return make([]byte, size)
}

func (p *MemoryPool) Put(buf []byte) {
    size := cap(buf)
    for i := range p.classes {
        if p.classes[i].size == size {
            p.classes[i].pool.Put(buf)
            return
        }
    }
}
```

### Lock-Free Queue

```go
// ✅ GOOD: Lock-free queue implementation
type Node[T any] struct {
    value T
    next  atomic.Pointer[Node[T]]
}

type Queue[T any] struct {
    head atomic.Pointer[Node[T]]
    tail atomic.Pointer[Node[T]]
}

func NewQueue[T any]() *Queue[T] {
    node := &Node[T]{}
    q := &Queue[T]{}
    q.head.Store(node)
    q.tail.Store(node)
    return q
}

func (q *Queue[T]) Enqueue(value T) {
    node := &Node[T]{value: value}
    for {
        tail := q.tail.Load()
        next := tail.next.Load()
        if tail == q.tail.Load() {
            if next == nil {
                if tail.next.CompareAndSwap(nil, node) {
                    q.tail.CompareAndSwap(tail, node)
                    return
                }
            } else {
                q.tail.CompareAndSwap(tail, next)
            }
        }
    }
}

func (q *Queue[T]) Dequeue() (T, bool) {
    for {
        head := q.head.Load()
        tail := q.tail.Load()
        next := head.next.Load()
        if head == q.head.Load() {
            if head == tail {
                if next == nil {
                    var zero T
                    return zero, false
                }
                q.tail.CompareAndSwap(tail, next)
            } else {
                value := next.value
                if q.head.CompareAndSwap(head, next) {
                    return value, true
                }
            }
        }
    }
}
```

## Testing Patterns

### Table-Driven Tests with Subtests

```go
// ✅ GOOD: Advanced table-driven tests
func TestComplexOperation(t *testing.T) {
    tests := []struct {
        name        string
        input       Input
        setup       func(t *testing.T) (*MockDB, *MockCache)
        validate    func(t *testing.T, result Result, err error)
        wantErr     bool
        errorType   error
        cleanup     func(t *testing.T, db *MockDB, cache *MockCache)
    }{
        {
            name: "successful operation",
            input: Input{
                ID:   "test",
                Data: []byte("test data"),
            },
            setup: func(t *testing.T) (*MockDB, *MockCache) {
                db := NewMockDB(t)
                db.ExpectBegin()
                db.ExpectExec("INSERT").WillReturnResult(sqlmock.NewResult(1, 1))
                db.ExpectCommit()

                cache := NewMockCache(t)
                cache.EXPECT().Set(mock.Anything, mock.Anything).Return(nil)

                return db, cache
            },
            validate: func(t *testing.T, result Result, err error) {
                assert.NoError(t, err)
                assert.NotEmpty(t, result.ID)
                assert.True(t, result.Success)
            },
            cleanup: func(t *testing.T, db *MockDB, cache *MockCache) {
                assert.NoError(t, db.ExpectationsWereMet())
            },
        },
        // More test cases...
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            db, cache := tt.setup(t)
            svc := NewService(db, cache)

            // Execute
            result, err := svc.ComplexOperation(context.Background(), tt.input)

            // Validate
            tt.validate(t, result, err)
            if tt.wantErr {
                assert.Error(t, err)
                assert.ErrorIs(t, err, tt.errorType)
            }

            // Cleanup
            if tt.cleanup != nil {
                tt.cleanup(t, db, cache)
            }
        })
    }
}
```

### Fuzzing with Custom Types

```go
// ✅ GOOD: Advanced fuzzing
func FuzzComplexParser(f *testing.F) {
    // Add seed corpus
    f.Add([]byte("valid input"))
    f.Add([]byte(""))
    f.Add([]byte("{\"}"))

    f.Fuzz(func(t *testing.T, data []byte) {
        // Ensure we don't spend too much time on large inputs
        if len(data) > 1024 {
            t.Skip()
        }

        // Track coverage and mutations
        defer func() {
            if r := recover(); r != nil {
                t.Errorf("parser panicked: %v", r)
            }
        }()

        result, err := Parse(data)
        if err != nil {
            // Verify error conditions make sense
            if len(data) == 0 {
                assert.ErrorIs(t, err, ErrEmptyInput)
            }
            return
        }

        // Verify invariants
        assert.NotNil(t, result)
        assert.True(t, validateResult(result))
    })
}
```

## Summary

These advanced patterns focus on:
1. **Concurrency Control**: Fan-out/fan-in, adaptive pools, circuit breakers
2. **Microservices**: Service mesh, event sourcing, API gateways
3. **Design Patterns**: Options validation, generic decorators, DDD
4. **Performance**: Memory pools, lock-free structures
5. **Testing**: Advanced table tests, fuzzing

Remember to:
- Use patterns judiciously based on actual needs
- Consider maintenance implications
- Document complex implementations
- Include comprehensive tests
- Monitor performance impacts

## References

1. **Go Concurrency Patterns**: https://github.com/lotusirous/go-concurrency-patterns
2. **Microservices Patterns**: https://microservices.io/patterns/
3. **Go Design Patterns**: https://github.com/tmrts/go-patterns
4. **Performance Patterns**: https://github.com/dgryski/go-perfbook
