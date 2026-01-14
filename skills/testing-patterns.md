# Advanced Testing Patterns and Strategies

## Go Testing Patterns

### Property-Based Testing

```go
// ✅ GOOD: Property-based tests with testify
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "pgregory.net/rapid"
)

func TestAdditionProperties(t *testing.T) {
    t.Run("commutative", func(t *testing.T) {
        rapid.Check(t, func(t *rapid.T) {
            a := rapid.Int().Draw(t, "a")
            b := rapid.Int().Draw(t, "b")
            
            result1 := add(a, b)
            result2 := add(b, a)
            
            assert.Equal(t, result1, result2)
        })
    })

    t.Run("associative", func(t *testing.T) {
        rapid.Check(t, func(t *rapid.T) {
            a := rapid.Int().Draw(t, "a")
            b := rapid.Int().Draw(t, "b")
            c := rapid.Int().Draw(t, "c")
            
            result1 := add(add(a, b), c)
            result2 := add(a, add(b, c))
            
            assert.Equal(t, result1, result2)
        })
    })

    t.Run("identity", func(t *testing.T) {
        rapid.Check(t, func(t *rapid.T) {
            x := rapid.Int().Draw(t, "x")
            
            result1 := add(x, 0)
            result2 := add(0, x)
            
            assert.Equal(t, x, result1)
            assert.Equal(t, x, result2)
        })
    })
}
```

### Contract Testing

```go
// ✅ GOOD: Contract tests for API contracts
import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    "github.com/stretchr/testify/assert"
)

type ContractTest struct {
    name           string
    request        Request
    expectedStatus int
    expectedBody   interface{}
    setup          func(*testing.T) *Service
}

func TestAPIContracts(t *testing.T) {
    tests := []ContractTest{
        {
            name: "create user with valid data",
            request: Request{
                Method: "POST",
                Path:   "/api/users",
                Body: User{
                    Name:  "John Doe",
                    Email: "john@example.com",
                },
            },
            expectedStatus: http.StatusCreated,
            expectedBody: User{
                ID:    "generated-id",
                Name:  "John Doe",
                Email: "john@example.com",
            },
            setup: func(t *testing.T) *Service {
                return NewTestService(t)
            },
        },
        {
            name: "create user with invalid email",
            request: Request{
                Method: "POST",
                Path:   "/api/users",
                Body: User{
                    Name:  "John Doe",
                    Email: "invalid-email",
                },
            },
            expectedStatus: http.StatusBadRequest,
            expectedBody: ErrorResponse{
                Code:    "INVALID_INPUT",
                Message: "Invalid email format",
            },
            setup: func(t *testing.T) *Service {
                return NewTestService(t)
            },
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            service := tt.setup(t)
            handler := NewHandler(service)

            body, _ := json.Marshal(tt.request.Body)
            req := httptest.NewRequest(tt.request.Method, tt.request.Path, bytes.NewBuffer(body))
            w := httptest.NewRecorder()

            handler.ServeHTTP(w, req)

            assert.Equal(t, tt.expectedStatus, w.Code)
            
            var actual interface{}
            json.Unmarshal(w.Body.Bytes(), &actual)
            assert.Equal(t, tt.expectedBody, actual)
        })
    }
}
```

### Integration Testing with Test Containers

```go
// ✅ GOOD: Integration tests with test containers
import (
    "context"
    "database/sql"
    "testing"
    "time"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/wait"
    _ "github.com/lib/pq"
)

func TestUserRepositoryIntegration(t *testing.T) {
    // Start PostgreSQL container
    ctx := context.Background()
    req := testcontainers.ContainerRequest{
        Image:        "postgres:15",
        ExposedPorts: []string{"5432/tcp"},
        Env: map[string]string{
            "POSTGRES_DB":       "testdb",
            "POSTGRES_USER":     "test",
            "POSTGRES_PASSWORD": "test",
        },
        WaitingFor: wait.ForLog("database system is ready to accept connections"),
    }

    container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: req,
        Started:          true,
    })
    require.NoError(t, err)
    defer container.Terminate(ctx)

    // Get connection details
    host, err := container.Host(ctx)
    require.NoError(t, err)

    port, err := container.MappedPort(ctx, "5432")
    require.NoError(t, err)

    // Connect to database
    dbURL := fmt.Sprintf("postgres://test:test@%s:%s/testdb?sslmode=disable", host, port.Port())
    db, err := sql.Open("postgres", dbURL)
    require.NoError(t, err)
    defer db.Close()

    // Wait for database to be ready
    require.Eventually(t, func() bool {
        return db.Ping() == nil
    }, 30*time.Second, time.Second)

    // Run migrations
    err = runMigrations(db)
    require.NoError(t, err)

    // Test repository
    repo := NewUserRepository(db)
    
    user := &User{
        Name:  "Test User",
        Email: "test@example.com",
    }
    
    // Create user
    err = repo.Create(user)
    require.NoError(t, err)
    assert.NotEmpty(t, user.ID)

    // Get user
    found, err := repo.GetByID(user.ID)
    require.NoError(t, err)
    assert.Equal(t, user.Name, found.Name)
    assert.Equal(t, user.Email, found.Email)

    // Update user
    user.Name = "Updated Name"
    err = repo.Update(user)
    require.NoError(t, err)

    // Verify update
    found, err = repo.GetByID(user.ID)
    require.NoError(t, err)
    assert.Equal(t, "Updated Name", found.Name)

    // Delete user
    err = repo.Delete(user.ID)
    require.NoError(t, err)

    // Verify deletion
    found, err = repo.GetByID(user.ID)
    assert.Error(t, err)
    assert.Nil(t, found)
}
```

### Load Testing

```go
// ✅ GOOD: Load testing with vegeta
import (
    "fmt"
    "net/http"
    "testing"
    "time"
    "github.com/tsenart/vegeta/v12/lib"
)

func TestAPIPerformance(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping performance test in short mode")
    }

    rate := vegeta.Rate{Freq: 100, Per: time.Second}
    duration := 30 * time.Second
    targets := []vegeta.Target{
        {
            Method: "GET",
            URL:    "http://localhost:8080/api/users",
            Header: http.Header{
                "Content-Type": []string{"application/json"},
            },
        },
        {
            Method: "POST",
            URL:    "http://localhost:8080/api/users",
            Body:   []byte(`{"name":"Test User","email":"test@example.com"}`),
            Header: http.Header{
                "Content-Type": []string{"application/json"},
            },
        },
    }

    attacker := vegeta.NewAttacker()
    var metrics vegeta.Metrics

    for _, target := range targets {
        results := attacker.Attack(vegeta.NewStaticTargeter(target), rate, duration, fmt.Sprintf("Load test %s", target.URL))
        for res := range results {
            metrics.Add(res)
        }
    }

    metrics.Close()

    // Assert performance requirements
    assert.Less(t, metrics.Latencies.P95, 100*time.Millisecond, "95th percentile latency should be < 100ms")
    assert.Less(t, metrics.Latencies.P99, 500*time.Millisecond, "99th percentile latency should be < 500ms")
    assert.Greater(t, metrics.Success, 0.95, "Success rate should be > 95%")
    assert.Less(t, metrics.Errors, 0.05, "Error rate should be < 5%")

    // Print detailed metrics
    t.Logf("Performance metrics:")
    t.Logf("  Requests: %d", metrics.Requests)
    t.Logf("  Success rate: %.2f%%", metrics.Success*100)
    t.Logf("  Latency P50: %v", metrics.Latencies.P50)
    t.Logf("  Latency P95: %v", metrics.Latencies.P95)
    t.Logf("  Latency P99: %v", metrics.Latencies.P99)
    t.Logf("  Throughput: %.2f req/s", metrics.Throughput)
}
```

### Chaos Testing

```go
// ✅ GOOD: Chaos testing for resilience
import (
    "context"
    "testing"
    "time"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

type ChaosInjector interface {
    InjectLatency(ctx context.Context, duration time.Duration) error
    InjectError(ctx context.Context, errorRate float64) error
    Stop() error
}

func TestServiceResilience(t *testing.T) {
    tests := []struct {
        name           string
        chaos          func(ctx context.Context, injector ChaosInjector)
        expectedResult string
        shouldSucceed  bool
    }{
        {
            name: "normal operation",
            chaos: func(ctx context.Context, injector ChaosInjector) {
                // No chaos injection
            },
            expectedResult: "success",
            shouldSucceed:  true,
        },
        {
            name: "database latency",
            chaos: func(ctx context.Context, injector ChaosInjector) {
                injector.InjectLatency(ctx, 2*time.Second)
            },
            expectedResult: "success",
            shouldSucceed:  true,
        },
        {
            name: "database errors",
            chaos: func(ctx context.Context, injector ChaosInjector) {
                injector.InjectError(ctx, 0.3) // 30% error rate
            },
            expectedResult: "success",
            shouldSucceed:  true,
        },
        {
            name: "high latency and errors",
            chaos: func(ctx context.Context, injector ChaosInjector) {
                injector.InjectLatency(ctx, 5*time.Second)
                injector.InjectError(ctx, 0.5) // 50% error rate
            },
            expectedResult: "error",
            shouldSucceed:  false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup chaos injector
            injector := NewChaosInjector(t)
            defer injector.Stop()

            // Setup service with circuit breaker and retries
            service := NewServiceWithResilience(injector)

            // Inject chaos
            ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
            defer cancel()

            tt.chaos(ctx, injector)

            // Test service
            result, err := service.ProcessRequest(ctx, "test-request")

            if tt.shouldSucceed {
                require.NoError(t, err)
                assert.Equal(t, tt.expectedResult, result)
            } else {
                assert.Error(t, err)
            }
        })
    }
}
```

## Rust Testing Patterns

### Property-Based Testing

```rust
// ✅ GOOD: Property-based tests with proptest
use proptest::prelude::*;
use std::collections::HashMap;

proptest! {
    #[test]
    fn test_map_operations_preserve_invariants(
        // Generate a map with 0-100 entries
        map in prop::collection::hash_map(".*", 0..1000u32, 0..100)
    ) {
        // Test that iteration count equals map length
        let iter_count = map.iter().count();
        prop_assert_eq!(iter_count, map.len());

        // Test that cloning preserves equality
        let cloned = map.clone();
        prop_assert_eq!(map, cloned);

        // Test that clear makes map empty
        let mut cleared = map.clone();
        cleared.clear();
        prop_assert!(cleared.is_empty());
        prop_assert_eq!(cleared.len(), 0);
    }

    #[test]
    fn test_sorting_properties(
        // Generate a vector with 0-100 elements
        vec in prop::collection::vec(0..1000i32, 0..100)
    ) {
        let mut sorted = vec.clone();
        sorted.sort();

        // Test that sorting preserves length
        prop_assert_eq!(vec.len(), sorted.len());

        // Test that sorted vector is actually sorted
        for window in sorted.windows(2) {
            prop_assert!(window[0] <= window[1]);
        }

        // Test that sorting is stable for equal elements
        let mut with_indices: Vec<_> = vec.iter().enumerate().collect();
        with_indices.sort_by_key(|(_, &val)| val);
        
        for window in with_indices.windows(2) {
            if window[0].1 == window[1].1 {
                prop_assert!(window[0].0 < window[1].0);
            }
        }
    }

    #[test]
    fn test_compression_roundtrip(
        data in prop::collection::vec(0..255u8, 0..1000)
    ) {
        let compressed = compress(&data);
        let decompressed = decompress(&compressed);
        
        prop_assert_eq!(data, decompressed);
    }
}
```

### Mock Testing with Mockall

```rust
// ✅ GOOD: Mock testing with mockall
use mockall::{mock, predicate::*};
use tokio_test;

#[mockall::automock]
trait UserRepository {
    async fn get_user(&self, id: u64) -> Result<User, Error>;
    async fn save_user(&self, user: &User) -> Result<(), Error>;
}

#[tokio::test]
async fn test_user_service() {
    let mut mock_repo = MockUserRepository::new();
    
    // Setup mock expectations
    let expected_user = User {
        id: 1,
        name: "Test User".to_string(),
        email: "test@example.com".to_string(),
    };

    mock_repo
        .expect_get_user()
        .with(eq(1))
        .times(1)
        .returning(move |_| Ok(expected_user.clone()));

    mock_repo
        .expect_save_user()
        .with(predicate::always())
        .times(1)
        .returning(|_| Ok(()));

    // Test service
    let service = UserService::new(mock_repo);
    let result = service.update_user_name(1, "Updated Name".to_string()).await;

    assert!(result.is_ok());
}

#[tokio::test]
async fn test_user_service_error_handling() {
    let mut mock_repo = MockUserRepository::new();
    
    mock_repo
        .expect_get_user()
        .with(eq(999))
        .times(1)
        .returning(|_| Err(Error::UserNotFound));

    let service = UserService::new(mock_repo);
    let result = service.update_user_name(999, "Updated Name".to_string()).await;

    assert!(result.is_err());
    assert!(matches!(result.unwrap_err(), Error::UserNotFound));
}
```

### Integration Testing with Test Containers

```rust
// ✅ GOOD: Integration tests with testcontainers
use testcontainers::{clients::Cli, Container, GenericImage};
use testcontainers::core::WaitFor;
use sqlx::postgres::PgPoolOptions;

#[tokio::test]
async fn test_user_repository_integration() -> Result<(), Box<dyn std::error::Error>> {
    let docker = Cli::default();
    
    // Start PostgreSQL container
    let container = docker.run(GenericImage::new("postgres", "15")
        .with_env_var("POSTGRES_DB", "testdb")
        .with_env_var("POSTGRES_USER", "test")
        .with_env_var("POSTGRES_PASSWORD", "test")
        .with_exposed_port(5432)
        .with_wait_for(WaitFor::message_on_stdout("database system is ready to accept connections")));

    // Get connection details
    let port = container.get_host_port_ipv4(5432)?;
    let connection_string = format!(
        "postgres://test:test@localhost:{}/testdb",
        port
    );

    // Create connection pool
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&connection_string)
        .await?;

    // Run migrations
    sqlx::migrate!("./migrations").run(&pool).await?;

    // Test repository
    let repo = UserRepository::new(pool);
    
    // Create user
    let user = User {
        id: 0,
        name: "Test User".to_string(),
        email: "test@example.com".to_string(),
    };
    
    let created_user = repo.create_user(&user).await?;
    assert!(created_user.id > 0);

    // Get user
    let found_user = repo.get_user_by_id(created_user.id).await?;
    assert_eq!(created_user.id, found_user.id);
    assert_eq!(created_user.name, found_user.name);

    // Update user
    let mut updated_user = found_user;
    updated_user.name = "Updated Name".to_string();
    repo.update_user(&updated_user).await?;

    // Verify update
    let found_user = repo.get_user_by_id(updated_user.id).await?;
    assert_eq!("Updated Name", found_user.name);

    // Delete user
    repo.delete_user(updated_user.id).await?;

    // Verify deletion
    let result = repo.get_user_by_id(updated_user.id).await;
    assert!(result.is_err());

    Ok(())
}
```

### Load Testing with Criterion

```rust
// ✅ GOOD: Performance benchmarks with criterion
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 1,
        1 => 1,
        n => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn fibonacci_iterative(n: u64) -> u64 {
    let (mut a, mut b) = (1, 1);
    for _ in 1..n {
        let next = a + b;
        a = b;
        b = next;
    }
    b
}

fn bench_fibonacci(c: &mut Criterion) {
    let mut group = c.benchmark_group("fibonacci");
    
    for n in [10, 20, 30].iter() {
        group.bench_with_input(
            BenchmarkId::new("recursive", n),
            n,
            |b, n| b.iter(|| fibonacci(black_box(*n))),
        );
        
        group.bench_with_input(
            BenchmarkId::new("iterative", n),
            n,
            |b, n| b.iter(|| fibonacci_iterative(black_box(*n))),
        );
    }
    
    group.finish();
}

fn bench_string_operations(c: &mut Criterion) {
    let text = "The quick brown fox jumps over the lazy dog";
    
    c.bench_function("string_concat", |b| {
        b.iter(|| {
            let mut result = String::new();
            for _ in 0..100 {
                result.push_str(black_box(text));
            }
            result
        })
    });
    
    c.bench_function("string_format", |b| {
        b.iter(|| {
            let mut result = String::new();
            for _ in 0..100 {
                result = format!("{}{}", result, black_box(text));
            }
            result
        })
    });
}

criterion_group!(benches, bench_fibonacci, bench_string_operations);
criterion_main!(benches);
```

### Fuzz Testing

```rust
// ✅ GOOD: Fuzz testing with cargo-fuzz
use arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;

#[derive(Arbitrary, Debug)]
struct FuzzInput {
    data: Vec<u8>,
    offset: usize,
    length: usize,
}

fuzz_target!(|input: FuzzInput| {
    // Ensure offset and length are within bounds
    if input.offset >= input.data.len() {
        return;
    }
    
    let end = std::cmp::min(input.offset + input.length, input.data.len());
    let slice = &input.data[input.offset..end];
    
    // Test parsing function
    let result = parse_data(slice);
    
    // Verify invariants
    match result {
        Ok(parsed) => {
            assert!(!parsed.is_empty());
            assert!(parsed.len() <= slice.len());
        }
        Err(_) => {
            // Error is acceptable for invalid input
        }
    }
});

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_parse_data_edge_cases() {
        // Test empty input
        let result = parse_data(&[]);
        assert!(result.is_err());
        
        // Test single byte
        let result = parse_data(&[0x01]);
        assert!(result.is_ok());
        
        // Test maximum valid input
        let large_input = vec![0xFF; 1024];
        let result = parse_data(&large_input);
        assert!(result.is_ok());
    }
}
```

## Summary

These testing patterns provide:

1. **Property-Based Testing**: Verify invariants across many inputs
2. **Contract Testing**: Ensure API contracts are maintained
3. **Integration Testing**: Test real components with test containers
4. **Load Testing**: Verify performance under load
5. **Chaos Testing**: Test system resilience
6. **Mock Testing**: Isolate components for unit testing
7. **Fuzz Testing**: Find edge cases and security issues

Remember to:
- Test both happy paths and error cases
- Use realistic test data
- Test performance characteristics
- Include security testing
- Automate testing in CI/CD
- Monitor test coverage
- Keep tests maintainable

## References

1. **Go Testing**: https://golang.org/pkg/testing/
2. **Testify**: https://github.com/stretchr/testify
3. **Proptest**: https://github.com/BurntSushi/ripgrep
4. **Mockall**: https://github.com/asomers/mockall
5. **Criterion**: https://github.com/bheisler/criterion.rs
