# Rust Best Practices - Production-Ready System Programming

## Skill Metadata
- **Domain**: Rust Programming Language
- **Skill Level**: Intermediate to Advanced
- **Last Updated**: 2024
- **Sources**: Rust Style Guide, Apollo Rust Best Practices, Rust for System Programming

## Overview

This skill document provides comprehensive best practices for building production-ready Rust applications. It covers ownership patterns, error handling, concurrency, performance optimization, and idiomatic Rust code that leverages the language's safety guarantees.

---

## 1. Project Organization & Structure

### Standard Cargo Project Layout

```
/project-root
├── src/
│   ├── main.rs              # Binary entry point
│   ├── lib.rs               # Library root (if building a library)
│   ├── bin/                 # Additional binaries
│   │   ├── server.rs
│   │   └── worker.rs
│   ├── models/              # Data models and domain types
│   │   ├── mod.rs
│   │   └── user.rs
│   ├── services/            # Business logic layer
│   │   ├── mod.rs
│   │   └── user_service.rs
│   ├── handlers/            # HTTP/API handlers
│   │   ├── mod.rs
│   │   └── user_handler.rs
│   └── utils/               # Utility functions
│       └── mod.rs
├── tests/                   # Integration tests
│   └── integration_test.rs
├── benches/                 # Benchmarks
│   └── benchmark.rs
├── examples/                # Example usage
│   └── basic_usage.rs
├── Cargo.toml              # Package manifest
├── Cargo.lock              # Dependency lock file
├── .rustfmt.toml           # Formatter configuration
├── .clippy.toml            # Clippy linter configuration
└── README.md
```

### Module Organization Best Practices

```rust
// ✅ GOOD: Clear module hierarchy in lib.rs
pub mod models;
pub mod services;
pub mod handlers;
pub mod error;

// Re-export commonly used types
pub use error::{Error, Result};
pub use models::User;
```

**Key Principles:**

**✅ DO:**
- Use `mod.rs` or the new `module_name.rs` pattern for module organization
- Keep related functionality together in modules
- Use `pub use` to create convenient re-exports
- Separate library code (`lib.rs`) from binary code (`main.rs`)

**❌ DON'T:**
- Create deeply nested module hierarchies
- Mix business logic with infrastructure code
- Export internal implementation details

---

## 2. Ownership, Borrowing & Lifetimes

### Ownership Patterns

```rust
// ❌ BAD: Unnecessary cloning
fn process_data(data: Vec<String>) -> Vec<String> {
    let mut result = Vec::new();
    for item in data.clone() {  // Unnecessary clone!
        result.push(item.to_uppercase());
    }
    result
}

// ✅ GOOD: Consume the input
fn process_data(data: Vec<String>) -> Vec<String> {
    data.into_iter()
        .map(|s| s.to_uppercase())
        .collect()
}

// ✅ ALSO GOOD: Borrow if you need to keep the original
fn process_data(data: &[String]) -> Vec<String> {
    data.iter()
        .map(|s| s.to_uppercase())
        .collect()
}
```

### Borrowing Best Practices

```rust
// ❌ BAD: Mutable and immutable borrows conflict
fn bad_example(data: &mut Vec<i32>) {
    let first = &data[0];  // Immutable borrow
    data.push(10);         // ERROR: Mutable borrow while immutable exists
    println!("{}", first);
}

// ✅ GOOD: Separate borrow scopes
fn good_example(data: &mut Vec<i32>) {
    let first = data.first().copied();  // Copy the value
    data.push(10);
    if let Some(val) = first {
        println!("{}", val);
    }
}

// ✅ BETTER: Restructure to avoid the issue
fn better_example(data: &mut Vec<i32>) -> Option<i32> {
    let first = data.first().copied();
    data.push(10);
    first
}
```

### Lifetime Annotations

```rust
// ✅ GOOD: Explicit lifetime when needed
struct Config<'a> {
    name: &'a str,
    value: &'a str,
}

impl<'a> Config<'a> {
    fn new(name: &'a str, value: &'a str) -> Self {
        Self { name, value }
    }
}

// ✅ GOOD: Lifetime elision works in simple cases
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}

// ✅ GOOD: Multiple lifetimes when references are independent
fn longest<'a, 'b>(x: &'a str, y: &'b str) -> &'a str 
where
    'b: 'a,  // 'b outlives 'a
{
    if x.len() > y.len() { x } else { x }
}
```

### Smart Pointers

```rust
use std::rc::Rc;
use std::sync::Arc;
use std::cell::RefCell;

// ✅ GOOD: Use Rc for single-threaded shared ownership
fn single_threaded_sharing() {
    let data = Rc::new(vec![1, 2, 3]);
    let data_clone = Rc::clone(&data);
    
    println!("Count: {}", Rc::strong_count(&data));  // 2
}

// ✅ GOOD: Use Arc for multi-threaded shared ownership
fn multi_threaded_sharing() {
    let data = Arc::new(vec![1, 2, 3]);
    let data_clone = Arc::clone(&data);
    
    std::thread::spawn(move || {
        println!("{:?}", data_clone);
    });
}

// ✅ GOOD: RefCell for interior mutability (single-threaded)
struct Cache {
    data: RefCell<HashMap<String, String>>,
}

impl Cache {
    fn get_or_insert(&self, key: &str, value: String) -> String {
        let mut data = self.data.borrow_mut();
        data.entry(key.to_string())
            .or_insert(value)
            .clone()
    }
}
```

---

## 3. Error Handling

### Custom Error Types with thiserror

```rust
use thiserror::Error;

// ✅ GOOD: Structured error types
#[derive(Error, Debug)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Configuration error: {0}")]
    Config(String),
    
    #[error("User not found: {id}")]
    UserNotFound { id: i64 },
    
    #[error("Validation failed: {field} - {message}")]
    Validation { field: String, message: String },
    
    #[error(transparent)]
    Other(#[from] anyhow::Error),
}

pub type Result<T> = std::result::Result<T, AppError>;
```

### Error Propagation

```rust
// ❌ BAD: Swallowing errors
fn load_config() -> Config {
    let content = std::fs::read_to_string("config.toml")
        .unwrap_or_default();  // Lost error information!
    toml::from_str(&content).unwrap_or_default()
}

// ✅ GOOD: Propagate errors with context
fn load_config() -> Result<Config> {
    let content = std::fs::read_to_string("config.toml")
        .map_err(|e| AppError::Config(format!("Failed to read config: {}", e)))?;
    
    toml::from_str(&content)
        .map_err(|e| AppError::Config(format!("Failed to parse config: {}", e)))
}

// ✅ BETTER: Use anyhow for applications
use anyhow::{Context, Result};

fn load_config() -> Result<Config> {
    let content = std::fs::read_to_string("config.toml")
        .context("Failed to read config.toml")?;
    
    toml::from_str(&content)
        .context("Failed to parse TOML configuration")
}
```

### Result Combinators

```rust
// ✅ GOOD: Use combinators instead of match
fn process_user(id: i64) -> Result<String> {
    fetch_user(id)?
        .validate()
        .map(|user| user.name)
        .ok_or(AppError::UserNotFound { id })
}

// ✅ GOOD: and_then for chaining Results
fn get_user_email(id: i64) -> Result<String> {
    fetch_user(id)
        .and_then(|user| user.email.ok_or(AppError::Validation {
            field: "email".to_string(),
            message: "Email not set".to_string(),
        }))
}

// ✅ GOOD: map_err for error transformation
fn read_file(path: &str) -> Result<String> {
    std::fs::read_to_string(path)
        .map_err(|e| AppError::Config(format!("Cannot read {}: {}", path, e)))
}
```

---

## 4. Type System & Traits

### Newtype Pattern

```rust
// ✅ GOOD: Type safety with newtypes
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct UserId(i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct OrderId(i64);

impl UserId {
    pub fn new(id: i64) -> Self {
        Self(id)
    }
    
    pub fn value(&self) -> i64 {
        self.0
    }
}

// Now you can't accidentally mix user IDs and order IDs
fn fetch_user(id: UserId) -> Result<User> {
    // ...
}

// ❌ This won't compile:
// fetch_user(OrderId::new(123));
```

### Builder Pattern

```rust
// ✅ GOOD: Builder pattern for complex construction
#[derive(Debug)]
pub struct Server {
    host: String,
    port: u16,
    timeout: Duration,
    max_connections: usize,
}

pub struct ServerBuilder {
    host: String,
    port: u16,
    timeout: Option<Duration>,
    max_connections: Option<usize>,
}

impl ServerBuilder {
    pub fn new(host: impl Into<String>, port: u16) -> Self {
        Self {
            host: host.into(),
            port,
            timeout: None,
            max_connections: None,
        }
    }
    
    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.timeout = Some(timeout);
        self
    }
    
    pub fn max_connections(mut self, max: usize) -> Self {
        self.max_connections = Some(max);
        self
    }
    
    pub fn build(self) -> Server {
        Server {
            host: self.host,
            port: self.port,
            timeout: self.timeout.unwrap_or(Duration::from_secs(30)),
            max_connections: self.max_connections.unwrap_or(100),
        }
    }
}

// Usage
let server = ServerBuilder::new("localhost", 8080)
    .timeout(Duration::from_secs(60))
    .max_connections(200)
    .build();
```

### Trait Implementation Best Practices

```rust
// ✅ GOOD: Implement common traits
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct User {
    pub id: UserId,
    pub name: String,
    pub email: String,
}

// ✅ GOOD: Custom Display implementation
impl std::fmt::Display for User {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} ({})", self.name, self.email)
    }
}

// ✅ GOOD: From/Into for conversions
impl From<UserDto> for User {
    fn from(dto: UserDto) -> Self {
        Self {
            id: UserId::new(dto.id),
            name: dto.name,
            email: dto.email,
        }
    }
}

// ✅ GOOD: Define traits for abstraction
pub trait Repository<T> {
    type Error;
    
    fn find_by_id(&self, id: i64) -> Result<Option<T>, Self::Error>;
    fn save(&self, entity: &T) -> Result<(), Self::Error>;
    fn delete(&self, id: i64) -> Result<(), Self::Error>;
}
```

---

## 5. Concurrency & Async Programming

### Async/Await Best Practices

```rust
use tokio;

// ✅ GOOD: Async function with proper error handling
async fn fetch_user(id: UserId) -> Result<User> {
    let response = reqwest::get(&format!("https://api.example.com/users/{}", id.value()))
        .await
        .context("Failed to fetch user")?;
    
    response.json::<User>()
        .await
        .context("Failed to parse user JSON")
}

// ✅ GOOD: Concurrent execution with join
async fn fetch_multiple_users(ids: Vec<UserId>) -> Result<Vec<User>> {
    let futures = ids.into_iter()
        .map(|id| fetch_user(id))
        .collect::<Vec<_>>();
    
    let results = futures::future::join_all(futures).await;
    
    results.into_iter().collect()
}

// ✅ BETTER: Use try_join for early exit on error
async fn fetch_user_and_orders(user_id: UserId) -> Result<(User, Vec<Order>)> {
    tokio::try_join!(
        fetch_user(user_id),
        fetch_orders(user_id)
    )
}
```

### Channels for Communication

```rust
use tokio::sync::{mpsc, oneshot};

// ✅ GOOD: mpsc for multiple producers
async fn worker_pool() {
    let (tx, mut rx) = mpsc::channel::<Task>(100);
    
    // Spawn workers
    for i in 0..4 {
        let mut rx = rx.clone();
        tokio::spawn(async move {
            while let Some(task) = rx.recv().await {
                process_task(task).await;
            }
        });
    }
    
    // Send tasks
    for task in generate_tasks() {
        tx.send(task).await.unwrap();
    }
}

// ✅ GOOD: oneshot for single response
async fn request_response_pattern() -> Result<String> {
    let (tx, rx) = oneshot::channel();
    
    tokio::spawn(async move {
        let result = expensive_computation().await;
        let _ = tx.send(result);
    });
    
    rx.await.map_err(|_| anyhow::anyhow!("Worker died"))
}
```

### Shared State with Arc and Mutex

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

// ✅ GOOD: Shared mutable state
#[derive(Clone)]
struct AppState {
    cache: Arc<Mutex<HashMap<String, String>>>,
    db: Arc<Database>,
}

impl AppState {
    async fn get_cached(&self, key: &str) -> Option<String> {
        let cache = self.cache.lock().await;
        cache.get(key).cloned()
    }
    
    async fn set_cached(&self, key: String, value: String) {
        let mut cache = self.cache.lock().await;
        cache.insert(key, value);
    }
}

// ✅ BETTER: Use RwLock for read-heavy workloads
use tokio::sync::RwLock;

struct Cache {
    data: Arc<RwLock<HashMap<String, String>>>,
}

impl Cache {
    async fn get(&self, key: &str) -> Option<String> {
        let data = self.data.read().await;
        data.get(key).cloned()
    }
    
    async fn set(&self, key: String, value: String) {
        let mut data = self.data.write().await;
        data.insert(key, value);
    }
}
```

### Spawning Tasks

```rust
// ❌ BAD: Spawning without handling the result
tokio::spawn(async {
    process_data().await;
});

// ✅ GOOD: Handle task results
let handle = tokio::spawn(async {
    process_data().await
});

match handle.await {
    Ok(result) => println!("Task completed: {:?}", result),
    Err(e) => eprintln!("Task panicked: {}", e),
}

// ✅ GOOD: Spawn with error handling inside
tokio::spawn(async {
    if let Err(e) = process_data().await {
        eprintln!("Error processing data: {}", e);
    }
});
```

### Select for Racing Futures

```rust
use tokio::select;

// ✅ GOOD: Timeout pattern
async fn fetch_with_timeout(url: &str) -> Result<String> {
    let timeout = tokio::time::sleep(Duration::from_secs(5));
    let fetch = reqwest::get(url);
    
    select! {
        result = fetch => {
            result?.text().await.map_err(Into::into)
        }
        _ = timeout => {
            Err(anyhow::anyhow!("Request timed out"))
        }
    }
}

// ✅ GOOD: Graceful shutdown
async fn run_server(mut shutdown: tokio::sync::broadcast::Receiver<()>) {
    loop {
        select! {
            Some(request) = receive_request() => {
                handle_request(request).await;
            }
            _ = shutdown.recv() => {
                println!("Shutting down gracefully");
                break;
            }
        }
    }
}
```

---

## 6. Performance Optimization

### Avoid Unnecessary Allocations

```rust
// ❌ BAD: Unnecessary String allocations
fn format_name(first: &str, last: &str) -> String {
    let mut result = String::new();
    result.push_str(first);
    result.push_str(" ");
    result.push_str(last);
    result
}

// ✅ GOOD: Pre-allocate capacity
fn format_name(first: &str, last: &str) -> String {
    let mut result = String::with_capacity(first.len() + last.len() + 1);
    result.push_str(first);
    result.push(' ');
    result.push_str(last);
    result
}

// ✅ BETTER: Use format! macro for simple cases
fn format_name(first: &str, last: &str) -> String {
    format!("{} {}", first, last)
}
```

### Use Iterators Effectively

```rust
// ❌ BAD: Collecting unnecessarily
fn sum_even_squares(numbers: &[i32]) -> i32 {
    let evens: Vec<i32> = numbers.iter()
        .filter(|&&n| n % 2 == 0)
        .copied()
        .collect();
    
    let squares: Vec<i32> = evens.iter()
        .map(|&n| n * n)
        .collect();
    
    squares.iter().sum()
}

// ✅ GOOD: Chain iterators without intermediate collections
fn sum_even_squares(numbers: &[i32]) -> i32 {
    numbers.iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * n)
        .sum()
}
```

### Zero-Cost Abstractions

```rust
// ✅ GOOD: Iterators are zero-cost
fn process_items(items: &[Item]) {
    items.iter()
        .filter(|item| item.is_active)
        .map(|item| item.process())
        .for_each(|result| handle(result));
}

// Compiles to the same code as:
fn process_items_manual(items: &[Item]) {
    for item in items {
        if item.is_active {
            let result = item.process();
            handle(result);
        }
    }
}
```

### Use Cow for Conditional Cloning

```rust
use std::borrow::Cow;

// ✅ GOOD: Avoid cloning when not needed
fn process_string(s: &str) -> Cow<str> {
    if s.contains("special") {
        Cow::Owned(s.replace("special", "SPECIAL"))
    } else {
        Cow::Borrowed(s)
    }
}

// Usage - no clone unless necessary
let result = process_string("normal text");  // No allocation
let result = process_string("special text"); // Allocates only when needed
```

### Inline Hints

```rust
// ✅ GOOD: Inline hot paths
#[inline]
pub fn fast_operation(&self) -> i32 {
    self.value * 2
}

// ✅ GOOD: Always inline trivial getters
#[inline(always)]
pub fn id(&self) -> UserId {
    self.id
}

// ✅ GOOD: Never inline large functions
#[inline(never)]
pub fn complex_operation(&self) -> Result<Data> {
    // Large function body
}
```

---

## 7. Memory Safety Patterns

### Avoid Unsafe Unless Necessary

```rust
// ❌ BAD: Unnecessary unsafe
unsafe fn get_value(ptr: *const i32) -> i32 {
    *ptr
}

// ✅ GOOD: Use safe abstractions
fn get_value(value: &i32) -> i32 {
    *value
}

// ✅ ACCEPTABLE: Unsafe when needed, with safety comments
/// # Safety
/// The caller must ensure that `ptr` is valid and properly aligned
unsafe fn read_raw(ptr: *const u8, len: usize) -> Vec<u8> {
    std::slice::from_raw_parts(ptr, len).to_vec()
}
```

### Pin for Self-Referential Structs

```rust
use std::pin::Pin;

// ✅ GOOD: Use Pin for async futures
async fn process_data(data: Pin<&mut Data>) {
    // Safe to work with pinned data
}

// ✅ GOOD: Pin in struct definitions when needed
struct SelfReferential {
    data: String,
    ptr: *const String,
}

impl SelfReferential {
    fn new(data: String) -> Pin<Box<Self>> {
        let mut boxed = Box::pin(Self {
            data,
            ptr: std::ptr::null(),
        });
        
        let ptr = &boxed.data as *const String;
        unsafe {
            let mut_ref = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).ptr = ptr;
        }
        
        boxed
    }
}
```

---

## 8. Testing Best Practices

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // ✅ GOOD: Descriptive test names
    #[test]
    fn test_user_creation_with_valid_email() {
        let user = User::new("test@example.com");
        assert!(user.is_ok());
    }

    #[test]
    fn test_user_creation_with_invalid_email_returns_error() {
        let user = User::new("invalid-email");
        assert!(user.is_err());
    }

    // ✅ GOOD: Use should_panic for expected panics
    #[test]
    #[should_panic(expected = "Email cannot be empty")]
    fn test_empty_email_panics() {
        User::new("").unwrap();
    }

    // ✅ GOOD: Test error types
    #[test]
    fn test_validation_error_type() {
        let result = validate_email("");
        match result {
            Err(AppError::Validation { field, .. }) => {
                assert_eq!(field, "email");
            }
            _ => panic!("Expected validation error"),
        }
    }
}
```

### Property-Based Testing

```rust
use proptest::prelude::*;

// ✅ GOOD: Property-based tests
proptest! {
    #[test]
    fn test_reversing_twice_gives_original(s in ".*") {
        let reversed = s.chars().rev().collect::<String>();
        let double_reversed = reversed.chars().rev().collect::<String>();
        prop_assert_eq!(s, double_reversed);
    }

    #[test]
    fn test_addition_is_commutative(a in 0..1000i32, b in 0..1000i32) {
        prop_assert_eq!(a + b, b + a);
    }
}
```

### Async Tests

```rust
// ✅ GOOD: Async tests with tokio
#[tokio::test]
async fn test_fetch_user() {
    let user = fetch_user(UserId::new(1)).await;
    assert!(user.is_ok());
}

// ✅ GOOD: Mock async dependencies
#[tokio::test]
async fn test_user_service_with_mock() {
    let mock_repo = MockUserRepository::new();
    let service = UserService::new(mock_repo);
    
    let result = service.get_user(UserId::new(1)).await;
    assert!(result.is_ok());
}
```

### Integration Tests

```rust
// tests/integration_test.rs

// ✅ GOOD: Integration tests in separate directory
use my_app::{Server, Config};

#[tokio::test]
async fn test_full_user_flow() {
    let config = Config::test_config();
    let server = Server::new(config).await.unwrap();
    
    // Create user
    let response = server.create_user("test@example.com").await;
    assert_eq!(response.status(), 201);
    
    // Fetch user
    let user_id = response.json::<User>().await.unwrap().id;
    let response = server.get_user(user_id).await;
    assert_eq!(response.status(), 200);
}
```

---

## 9. Documentation & Code Style

### Documentation Comments

```rust
/// Fetches a user from the database by their ID.
///
/// # Arguments
///
/// * `id` - The unique identifier for the user
///
/// # Returns
///
/// Returns `Ok(User)` if the user exists, or an error if:
/// - The user is not found
/// - A database error occurs
///
/// # Examples
///
/// ```
/// use my_app::{fetch_user, UserId};
///
/// # async fn example() -> Result<(), Box<dyn std::error::Error>> {
/// let user = fetch_user(UserId::new(1)).await?;
/// println!("User: {}", user.name);
/// # Ok(())
/// # }
/// ```
pub async fn fetch_user(id: UserId) -> Result<User> {
    // Implementation
}

/// A user in the system.
///
/// # Fields
///
/// * `id` - Unique identifier
/// * `name` - User's full name
/// * `email` - User's email address (must be unique)
#[derive(Debug, Clone)]
pub struct User {
    pub id: UserId,
    pub name: String,
    pub email: String,
}
```

### Code Formatting

```rust
// ✅ GOOD: Use rustfmt
// .rustfmt.toml
max_width = 100
tab_spaces = 4
edition = "2021"
use_small_heuristics = "Max"
```

### Clippy Lints

```rust
// ✅ GOOD: Enable strict lints
#![warn(clippy::all)]
#![warn(clippy::pedantic)]
#![warn(clippy::nursery)]
#![allow(clippy::module_name_repetitions)]

// .clippy.toml
cognitive-complexity-threshold = 30
```

---

## 10. Dependency Management

### Cargo.toml Best Practices

```toml
[package]
name = "my-app"
version = "0.1.0"
edition = "2021"
rust-version = "1.70"

[dependencies]
# Use specific versions for stability
tokio = { version = "1.35", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
anyhow = "1.0"
thiserror = "1.0"

# Async runtime
tokio = { version = "1.35", features = ["rt-multi-thread", "macros"] }

# HTTP client
reqwest = { version = "0.11", features = ["json"] }

[dev-dependencies]
proptest = "1.4"
mockall = "0.12"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
strip = true

[profile.dev]
opt-level = 0
debug = true
```

### Feature Flags

```toml
[features]
default = ["postgres"]
postgres = ["sqlx/postgres"]
mysql = ["sqlx/mysql"]
full = ["postgres", "mysql", "redis"]
```

```rust
// Use conditional compilation
#[cfg(feature = "postgres")]
mod postgres_impl;

#[cfg(feature = "mysql")]
mod mysql_impl;
```

---

## 11. CI/CD Configuration

### GitHub Actions for Rust

```yaml
name: Rust CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  CARGO_TERM_COLOR: always

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Install Rust
      uses: dtolnay/rust-toolchain@stable
      with:
        components: rustfmt, clippy
    
    - name: Cache cargo registry
      uses: actions/cache@v3
      with:
        path: ~/.cargo/registry
        key: ${{ runner.os }}-cargo-registry-${{ hashFiles('**/Cargo.lock') }}
    
    - name: Cache cargo index
      uses: actions/cache@v3
      with:
        path: ~/.cargo/git
        key: ${{ runner.os }}-cargo-git-${{ hashFiles('**/Cargo.lock') }}
    
    - name: Cache target directory
      uses: actions/cache@v3
      with:
        path: target
        key: ${{ runner.os }}-target-${{ hashFiles('**/Cargo.lock') }}
    
    - name: Check formatting
      run: cargo fmt -- --check
    
    - name: Run clippy
      run: cargo clippy -- -D warnings
    
    - name: Run tests
      run: cargo test --all-features --verbose
    
    - name: Run doc tests
      run: cargo test --doc
    
    - name: Check documentation
      run: cargo doc --no-deps --all-features

  security:
    name: Security Audit
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run cargo-audit
      uses: actions-rs/audit-check@v1
      with:
        token: ${{ secrets.GITHUB_TOKEN }}

  coverage:
    name: Code Coverage
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Install Rust
      uses: dtolnay/rust-toolchain@stable
    
    - name: Install tarpaulin
      run: cargo install cargo-tarpaulin
    
    - name: Generate coverage
      run: cargo tarpaulin --out Xml --all-features
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./cobertura.xml

  build:
    name: Build Release
    runs-on: ubuntu-latest
    needs: [test, security]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Install Rust
      uses: dtolnay/rust-toolchain@stable
    
    - name: Build release
      run: cargo build --release --all-features
    
    - name: Upload artifact
      uses: actions/upload-artifact@v3
      with:
        name: binary
        path: target/release/my-app
```

---

## 12. Common Patterns & Idioms

### The Newtype Pattern for Type Safety

```rust
// ✅ GOOD: Prevent mixing different ID types
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(i64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ProductId(i64);

// Can't accidentally use ProductId where UserId is expected
```

### Extension Traits

```rust
// ✅ GOOD: Extend existing types
pub trait StringExt {
    fn truncate_to(&self, max_len: usize) -> String;
}

impl StringExt for String {
    fn truncate_to(&self, max_len: usize) -> String {
        if self.len() <= max_len {
            self.clone()
        } else {
            format!("{}...", &self[..max_len - 3])
        }
    }
}

// Usage
let text = "Very long text".to_string();
let short = text.truncate_to(10);
```

### Type State Pattern

```rust
// ✅ GOOD: Enforce state transitions at compile time
struct Disconnected;
struct Connected;

struct Connection<State> {
    state: PhantomData<State>,
}

impl Connection<Disconnected> {
    fn new() -> Self {
        Self { state: PhantomData }
    }
    
    fn connect(self) -> Connection<Connected> {
        // Perform connection
        Connection { state: PhantomData }
    }
}

impl Connection<Connected> {
    fn send(&self, data: &[u8]) {
        // Can only send when connected
    }
    
    fn disconnect(self) -> Connection<Disconnected> {
        Connection { state: PhantomData }
    }
}

// Usage - enforced at compile time
let conn = Connection::new();
// conn.send(&data); // ERROR: Can't send when disconnected
let conn = conn.connect();
conn.send(&data); // OK
```

---

## 13. Production Readiness Checklist

### Before Deployment

- [ ] All `unwrap()` and `expect()` calls reviewed and justified
- [ ] Error handling comprehensive with proper context
- [ ] Logging configured with appropriate levels
- [ ] Metrics and monitoring instrumented
- [ ] Configuration externalized (environment variables or config files)
- [ ] Secrets managed securely (not hardcoded)
- [ ] Database migrations tested
- [ ] Graceful shutdown implemented
- [ ] Health check endpoints exposed
- [ ] Resource limits configured (connection pools, timeouts)
- [ ] All tests passing (unit, integration, property-based)
- [ ] Clippy warnings addressed
- [ ] Documentation complete
- [ ] Security audit passed (`cargo audit`)
- [ ] Performance benchmarks run
- [ ] Docker image optimized (multi-stage builds)

### Performance Checklist

- [ ] Hot paths profiled and optimized
- [ ] Unnecessary allocations removed
- [ ] Iterators used instead of intermediate collections
- [ ] String operations optimized (capacity pre-allocation)
- [ ] Database queries optimized (indexes, batch operations)
- [ ] Connection pooling configured
- [ ] Caching strategy implemented where appropriate
- [ ] Async operations used for I/O-bound tasks

---

## Summary: Key Rust Principles

### Ownership & Borrowing
1. Prefer borrowing over cloning
2. Use `&T` for read-only access, `&mut T` for exclusive write access
3. Return owned values when transferring ownership
4. Use smart pointers (`Rc`, `Arc`, `Box`) judiciously

### Error Handling
1. Use `Result<T, E>` for recoverable errors
2. Use `thiserror` for library errors, `anyhow` for applications
3. Provide context when propagating errors
4. Never silently ignore errors

### Concurrency
1. Use async/await for I/O-bound operations
2. Protect shared state with `Arc<Mutex<T>>` or `Arc<RwLock<T>>`
3. Use channels for communication between tasks
4. Handle task cancellation with `select!` and shutdown signals

### Performance
1. Avoid unnecessary allocations
2. Use iterators for zero-cost abstractions
3. Pre-allocate collections when size is known
4. Profile before optimizing

### Type Safety
1. Use the newtype pattern for domain types
2. Leverage the type system to prevent invalid states
3. Implement common traits (`Debug`, `Clone`, `Display`)
4. Use enums for sum types, structs for product types

---

## References

1. **Rust Style Guide** - https://doc.rust-lang.org/style-guide/
2. **Apollo Rust Best Practices** - https://github.com/apollographql/rust-best-practices
3. **Rust for System Programming** - https://medium.com/@enravishjeni411/rust-for-system-programming-best-practices
4. **The Rust Book** - https://doc.rust-lang.org/book/
5. **Rust API Guidelines** - https://rust-lang.github.io/api-guidelines/
6. **Async Book** - https://rust-lang.github.io/async-book/

---

## Training Notes for AI Agents

When generating Rust code, prioritize:

1. **Safety**: Leverage the type system and borrow checker
2. **Clarity**: Write explicit, readable code over clever tricks
3. **Idiomatic**: Follow Rust conventions and patterns
4. **Performance**: Use zero-cost abstractions effectively
5. **Robustness**: Handle all error cases explicitly

Always prefer safe Rust over unsafe unless there's a compelling performance reason and safety can be guaranteed.
