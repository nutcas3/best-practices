# Observability and Monitoring Patterns

## Go Observability Patterns

### Structured Logging with Context

```go
// ✅ GOOD: Structured logging with context propagation
import (
    "context"
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

type contextKey string

const loggerKey contextKey = "logger"

func WithLogger(ctx context.Context, logger *zap.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func LoggerFromContext(ctx context.Context) *zap.Logger {
    if logger, ok := ctx.Value(loggerKey).(*zap.Logger); ok {
        return logger
    }
    return zap.L() // Default logger
}

func ProcessRequest(ctx context.Context, req Request) error {
    logger := LoggerFromContext(ctx).With(
        zap.String("request_id", GetRequestID(ctx)),
        zap.String("user_id", GetUserID(ctx)),
        zap.String("operation", "process_request"),
    )

    logger.Info("Starting request processing")
    
    // Process request...
    
    logger.Info("Completed request processing",
        zap.Duration("duration", time.Since(start)),
    )
    return nil
}
```

### Distributed Tracing

```go
// ✅ GOOD: OpenTelemetry tracing
import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

func ProcessOrder(ctx context.Context, order Order) error {
    ctx, span := otel.Tracer("order-service").Start(ctx, "ProcessOrder")
    defer span.End()

    // Add span attributes
    span.SetAttributes(
        attribute.String("order.id", order.ID),
        attribute.String("order.customer", order.CustomerID),
        attribute.Float64("order.amount", order.Amount),
    )

    // Validate order
    if err := validateOrder(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "Order validation failed")
        return err
    }

    // Process payment
    if err := processPayment(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "Payment processing failed")
        return err
    }

    // Update inventory
    if err := updateInventory(ctx, order); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "Inventory update failed")
        return err
    }

    span.SetStatus(codes.Ok, "Order processed successfully")
    return nil
}

// ✅ GOOD: Tracing middleware
func TracingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()
        
        // Start span
        tracer := otel.Tracer("http-server")
        ctx, span := tracer.Start(ctx, r.URL.Path,
            trace.WithAttributes(
                attribute.String("http.method", r.Method),
                attribute.String("http.url", r.URL.String()),
                attribute.String("http.user_agent", r.UserAgent()),
            ),
        )
        defer span.End()

        // Continue with traced context
        r = r.WithContext(ctx)
        next.ServeHTTP(w, r)

        // Record response status
        span.SetAttributes(
            attribute.Int("http.status_code", w.Header().Get("Status")),
        )
    })
}
```

### Metrics Collection

```go
// ✅ GOOD: Prometheus metrics
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    requestCount = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )

    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
)

func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Wrap response writer to capture status
        rw := &responseWriter{ResponseWriter: w}
        
        next.ServeHTTP(rw, r)
        
        // Record metrics
        duration := time.Since(start).Seconds()
        status := fmt.Sprintf("%d", rw.statusCode)
        
        requestCount.WithLabelValues(r.Method, r.URL.Path, status).Inc()
        requestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

### Health Checks

```go
// ✅ GOOD: Comprehensive health checks
type HealthChecker struct {
    checks map[string]CheckFunc
    mu     sync.RWMutex
}

type CheckFunc func(ctx context.Context) error

type HealthStatus struct {
    Status  string            `json:"status"`
    Checks  map[string]string `json:"checks"`
    Uptime  time.Duration     `json:"uptime"`
    Version string            `json:"version"`
}

func NewHealthChecker() *HealthChecker {
    return &HealthChecker{
        checks: make(map[string]CheckFunc),
    }
}

func (h *HealthChecker) AddCheck(name string, fn CheckFunc) {
    h.mu.Lock()
    defer h.mu.Unlock()
    h.checks[name] = fn
}

func (h *HealthChecker) Check(ctx context.Context) HealthStatus {
    h.mu.RLock()
    defer h.mu.RUnlock()

    status := HealthStatus{
        Checks:  make(map[string]string),
        Uptime:  time.Since(startTime),
        Version: version,
    }

    allHealthy := true
    for name, check := range h.checks {
        checkCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
        defer cancel()
        
        if err := check(checkCtx); err != nil {
            status.Checks[name] = "unhealthy: " + err.Error()
            allHealthy = false
        } else {
            status.Checks[name] = "healthy"
        }
    }

    if allHealthy {
        status.Status = "healthy"
    } else {
        status.Status = "unhealthy"
    }

    return status
}

// Example health checks
func (s *Service) DatabaseHealthCheck(ctx context.Context) error {
    if err := s.db.PingContext(ctx); err != nil {
        return fmt.Errorf("database ping failed: %w", err)
    }
    return nil
}

func (s *Service) RedisHealthCheck(ctx context.Context) error {
    if err := s.redis.Ping(ctx).Err(); err != nil {
        return fmt.Errorf("redis ping failed: %w", err)
    }
    return nil
}
```

## Rust Observability Patterns

### Structured Logging with Tracing

```rust
// ✅ GOOD: Structured logging with tracing
use tracing::{info, warn, error, instrument, Span};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[instrument(skip(db))]
async fn process_user(db: &Database, user_id: u64) -> Result<User, Error> {
    info!("Processing user", user_id);

    let user = db.get_user(user_id).await
        .map_err(|e| {
            error!("Failed to get user", user_id, error = %e);
            e
        })?;

    if !user.is_active {
        warn!("User is not active", user_id);
        return Err(Error::InactiveUser);
    }

    info!("Successfully processed user", user_id);
    Ok(user)
}

// ✅ GOOD: Custom spans and fields
#[instrument(fields(user_id = %user_id))]
async fn update_user_profile(db: &Database, user_id: u64, profile: Profile) -> Result<(), Error> {
    let span = Span::current();
    span.record("profile_fields", &profile.fields_count());

    // Validate profile
    if let Err(e) = validate_profile(&profile) {
        span.record("validation_error", &e.to_string());
        return Err(e);
    }

    // Update in database
    db.update_user_profile(user_id, profile).await?;
    
    span.record("updated", true);
    info!("User profile updated", user_id);
    Ok(())
}
```

### Metrics with Prometheus

```rust
// ✅ GOOD: Prometheus metrics
use prometheus::{Counter, Histogram, Gauge, register_counter, register_histogram, register_gauge};

lazy_static! {
    static ref REQUESTS_TOTAL: Counter = register_counter!(
        "http_requests_total",
        "Total number of HTTP requests"
    ).unwrap();

    static ref REQUEST_DURATION: Histogram = register_histogram!(
        "http_request_duration_seconds",
        "HTTP request duration in seconds"
    ).unwrap();

    static ref ACTIVE_CONNECTIONS: Gauge = register_gauge!(
        "active_connections",
        "Number of active connections"
    ).unwrap();
}

#[instrument]
async fn handle_request(request: Request) -> Response {
    let timer = REQUEST_DURATION.start_timer();
    REQUESTS_TOTAL.inc();
    ACTIVE_CONNECTIONS.inc();

    let response = process_request(request).await;

    ACTIVE_CONNECTIONS.dec();
    timer.observe_duration();
    response
}
```

### Distributed Tracing

```rust
// ✅ GOOD: OpenTelemetry tracing
use opentelemetry::global;
use opentelemetry::trace::{Tracer, Span};
use opentelemetry::trace::TraceError;

#[instrument(skip(db))]
async fn process_order(db: &Database, order: Order) -> Result<(), Error> {
    let tracer = global::tracer("order-service");
    let mut span = tracer.start("process_order");
    span.set_attribute(KeyValue::new("order.id", order.id.clone()));
    span.set_attribute(KeyValue::new("order.amount", order.amount));

    // Validate order
    if let Err(e) = validate_order(&order) {
        span.set_status(Status::error("Order validation failed"));
        span.record_error(&e);
        return Err(e);
    }

    // Process payment
    if let Err(e) = process_payment(&order).await {
        span.set_status(Status::error("Payment processing failed"));
        span.record_error(&e);
        return Err(e);
    }

    span.set_status(Status::Ok);
    Ok(())
}

// ✅ GOOD: Tracing middleware for web frameworks
async fn tracing_middleware(
    req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let tracer = global::tracer("http-server");
    let method = req.method().as_str();
    let path = req.uri().path();
    
    let mut span = tracer.start(format!("{} {}", method, path));
    span.set_attribute(KeyValue::new("http.method", method));
    span.set_attribute(KeyValue::new("http.url", req.uri().to_string()));

    let response = next.run(req).await;

    span.set_attribute(KeyValue::new("http.status_code", response.status().as_u16() as i64));
    span.end();

    Ok(response)
}
```

### Error Tracking

```rust
// ✅ GOOD: Error tracking with Sentry
use sentry::integrations::tracing::event_from_error;
use sentry::Level;

#[instrument]
async fn process_data(data: Vec<u8>) -> Result<ProcessedData, Error> {
    match parse_data(&data) {
        Ok(parsed) => {
            info!("Data parsed successfully", data_size = data.len());
            Ok(process_parsed_data(parsed).await?)
        }
        Err(e) => {
            error!("Failed to parse data", error = %e);
            
            // Send to Sentry
            sentry::capture_error(&e);
            
            // Create tracing event
            event_from_error(&e, Some(Level::Error));
            
            Err(e)
        }
    }
}

// ✅ GOOD: Custom error types with context
#[derive(thiserror::Error, Debug)]
#[error("Failed to process {operation}: {source}")]
pub struct ProcessingError {
    pub operation: String,
    #[source]
    pub source: Error,
    pub context: ErrorContext,
}

#[derive(Debug)]
pub struct ErrorContext {
    pub user_id: Option<String>,
    pub request_id: Option<String>,
    pub metadata: HashMap<String, String>,
}

impl ProcessingError {
    pub fn new(operation: impl Into<String>, source: Error) -> Self {
        Self {
            operation: operation.into(),
            source,
            context: ErrorContext::default(),
        }
    }

    pub fn with_user_id(mut self, user_id: impl Into<String>) -> Self {
        self.context.user_id = Some(user_id.into());
        self
    }

    pub fn with_request_id(mut self, request_id: impl Into<String>) -> Self {
        self.context.request_id = Some(request_id.into());
        self
    }
}
```

## Security Patterns

### Go Security Patterns

```go
// ✅ GOOD: Rate limiting with token bucket
type RateLimiter struct {
    tokens     map[string]*tokenBucket
    mu         sync.RWMutex
    refillRate time.Duration
    capacity   int
}

type tokenBucket struct {
    tokens    int
    lastRefill time.Time
    mu        sync.Mutex
}

func (rl *RateLimiter) Allow(key string) bool {
    rl.mu.RLock()
    bucket, exists := rl.tokens[key]
    if !exists {
        rl.mu.RUnlock()
        rl.mu.Lock()
        bucket = &tokenBucket{
            tokens:    rl.capacity,
            lastRefill: time.Now(),
        }
        rl.tokens[key] = bucket
        rl.mu.Unlock()
    } else {
        rl.mu.RUnlock()
    }

    bucket.mu.Lock()
    defer bucket.mu.Unlock()

    // Refill tokens
    now := time.Now()
    elapsed := now.Sub(bucket.lastRefill)
    tokensToAdd := int(elapsed / rl.refillRate)
    if tokensToAdd > 0 {
        bucket.tokens = min(bucket.tokens+tokensToAdd, rl.capacity)
        bucket.lastRefill = now
    }

    if bucket.tokens > 0 {
        bucket.tokens--
        return true
    }

    return false
}

// ✅ GOOD: Input validation and sanitization
type Validator struct {
    sanitizer *bluemonday.Policy
}

func NewValidator() *Validator {
    p := bluemonday.StrictPolicy()
    p.AllowElements("b", "i", "em", "strong")
    
    return &Validator{
        sanitizer: p,
    }
}

func (v *Validator) ValidateAndSanitize(input string) (string, error) {
    // Check length
    if len(input) > 1000 {
        return "", errors.New("input too long")
    }

    // Sanitize HTML
    sanitized := v.sanitizer.Sanitize(input)

    // Additional validation
    if strings.TrimSpace(sanitized) == "" {
        return "", errors.New("input cannot be empty")
    }

    return sanitized, nil
}

// ✅ GOOD: Secure headers middleware
func SecurityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        
        next.ServeHTTP(w, r)
    })
}
```

### Rust Security Patterns

```rust
// ✅ GOOD: Rate limiting with governor
use governor::{Quota, RateLimiter};
use std::num::NonZeroU32;
use std::time::Duration;

type AppRateLimiter = RateLimiter<governor::keyed::DefaultKeyHasher>;

fn create_rate_limiter() -> AppRateLimiter {
    RateLimiter::keyed(
        Quota::per_second(NonZeroU32::new(10).unwrap()),
        &Default::default(),
    )
}

// ✅ GOOD: Input validation with serde
use serde::{Deserialize, Validate};
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(length(min = 3, max = 50))]
    pub username: String,
    
    #[validate(email)]
    pub email: String,
    
    #[validate(length(min = 8))]
    #[validate(custom = "validate_password")]
    pub password: String,
}

fn validate_password(password: &str) -> Result<(), validator::ValidationError> {
    if !password.chars().any(|c| c.is_uppercase()) {
        return Err(validator::ValidationError::new("must contain uppercase"));
    }
    if !password.chars().any(|c| c.is_lowercase()) {
        return Err(validator::ValidationError::new("must contain lowercase"));
    }
    if !password.chars().any(|c| c.is_numeric()) {
        return Err(validator::ValidationError::new("must contain number"));
    }
    Ok(())
}

// ✅ GOOD: Secure headers middleware
async fn security_headers_middleware(
    req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let mut response = next.run(req).await;
    
    let headers = response.headers_mut();
    headers.insert("X-Content-Type-Options", "nosniff".parse().unwrap());
    headers.insert("X-Frame-Options", "DENY".parse().unwrap());
    headers.insert("X-XSS-Protection", "1; mode=block".parse().unwrap());
    headers.insert("Strict-Transport-Security", "max-age=31536000; includeSubDomains".parse().unwrap());
    headers.insert("Content-Security-Policy", "default-src 'self'".parse().unwrap());
    headers.insert("Referrer-Policy", "strict-origin-when-cross-origin".parse().unwrap());

    Ok(response)
}

// ✅ GOOD: Authentication middleware
use jsonwebtoken::{decode, Validation, DecodingKey};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct Claims {
    pub sub: String,
    pub exp: usize,
    pub role: String,
}

pub async fn auth_middleware(
    req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let auth_header = req.headers().get("Authorization");
    
    if auth_header.is_none() {
        return Err(StatusCode::UNAUTHORIZED);
    }

    let auth_header = auth_header.unwrap().to_str().unwrap();
    let token = auth_header.strip_prefix("Bearer ").unwrap_or("");

    let validation = Validation::new(jsonwebtoken::Algorithm::HS256);
    let decoding_key = DecodingKey::from_secret("your-secret-key".as_ref());

    match decode::<Claims>(token, &decoding_key, &validation) {
        Ok(token_data) => {
            // Add claims to request extensions
            let mut req = req;
            req.extensions_mut().insert(token_data.claims);
            Ok(next.run(req).await)
        }
        Err(_) => Err(StatusCode::UNAUTHORIZED),
    }
}
```

## Summary

These observability and security patterns provide:

1. **Observability**: Structured logging, distributed tracing, metrics, health checks
2. **Security**: Rate limiting, input validation, secure headers, authentication
3. **Integration**: Seamless integration with existing monitoring systems
4. **Performance**: Low-overhead instrumentation
5. **Standards**: OpenTelemetry, Prometheus, Sentry integration

Remember to:
- Instrument early in development
- Use structured data for better querying
- Set appropriate sampling rates for tracing
- Monitor the monitoring system itself
- Keep security configurations up to date

## References

1. **OpenTelemetry**: https://opentelemetry.io/
2. **Prometheus**: https://prometheus.io/
3. **Tracing in Go**: https://opentelemetry.io/docs/instrumentation/go/
4. **Tracing in Rust**: https://github.com/open-telemetry/opentelemetry-rust
