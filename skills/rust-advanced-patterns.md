# Advanced Rust Patterns and Optimizations

## Memory and Performance Patterns

### Zero-Copy Parsing

```rust
// ✅ GOOD: Zero-copy string parsing
use std::borrow::Cow;

fn parse_identifier(input: &str) -> Cow<str> {
    if input.chars().all(|c| c.is_ascii_alphanumeric()) {
        // No allocation if already valid
        Cow::Borrowed(input)
    } else {
        // Allocate only when necessary
        Cow::Owned(input.chars()
            .filter(|c| c.is_ascii_alphanumeric())
            .collect())
    }
}
```

### Custom Allocators

```rust
// ✅ GOOD: Custom allocator for specialized use cases
use std::alloc::{GlobalAlloc, Layout, System};

struct CountingAllocator;

unsafe impl GlobalAlloc for CountingAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // Track allocation
        ALLOCATION_COUNT.fetch_add(1, Ordering::SeqCst);
        System.alloc(layout)
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // Track deallocation
        DEALLOCATION_COUNT.fetch_add(1, Ordering::SeqCst);
        System.dealloc(ptr, layout)
    }
}

#[global_allocator]
static ALLOCATOR: CountingAllocator = CountingAllocator;
```

### SIMD Optimizations

```rust
// ✅ GOOD: Using SIMD for parallel computations
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

#[cfg(target_arch = "x86_64")]
pub fn sum_array_simd(arr: &[f32]) -> f32 {
    let mut sum = 0.0;
    let len = arr.len();
    let mut i = 0;

    unsafe {
        // Process 4 elements at a time
        if len >= 4 {
            let mut sum_vec = _mm_setzero_ps();
            while i + 4 <= len {
                let v = _mm_loadu_ps(arr[i..].as_ptr());
                sum_vec = _mm_add_ps(sum_vec, v);
                i += 4;
            }
            // Extract sum from vector
            sum += _mm_cvtss_f32(sum_vec) +
                  _mm_cvtss_f32(_mm_shuffle_ps(sum_vec, sum_vec, 1)) +
                  _mm_cvtss_f32(_mm_shuffle_ps(sum_vec, sum_vec, 2)) +
                  _mm_cvtss_f32(_mm_shuffle_ps(sum_vec, sum_vec, 3));
        }
    }

    // Process remaining elements
    while i < len {
        sum += arr[i];
        i += 1;
    }
    sum
}
```

### Memory Pooling

```rust
// ✅ GOOD: Object pool for reusing allocations
use parking_lot::Mutex;

struct Pool<T> {
    items: Mutex<Vec<T>>,
}

impl<T: Default> Pool<T> {
    pub fn new() -> Self {
        Self {
            items: Mutex::new(Vec::new()),
        }
    }

    pub fn get(&self) -> T {
        let mut items = self.items.lock();
        items.pop().unwrap_or_default()
    }

    pub fn put(&self, item: T) {
        let mut items = self.items.lock();
        items.push(item);
    }
}

// Usage
let pool = Pool::<Vec<u8>>::new();
let mut buf = pool.get();
// Use buffer...
buf.clear();
pool.put(buf);
```

## Advanced Type System Patterns

### Type State Pattern

```rust
// ✅ GOOD: Compile-time state machine
struct Uninitialized;
struct Initialized;
struct Running;

struct Server<State> {
    config: Config,
    _state: std::marker::PhantomData<State>,
}

impl Server<Uninitialized> {
    pub fn new(config: Config) -> Self {
        Self {
            config,
            _state: std::marker::PhantomData,
        }
    }

    pub fn initialize(self) -> Result<Server<Initialized>, Error> {
        // Initialize server...
        Ok(Server {
            config: self.config,
            _state: std::marker::PhantomData,
        })
    }
}

impl Server<Initialized> {
    pub fn start(self) -> Result<Server<Running>, Error> {
        // Start server...
        Ok(Server {
            config: self.config,
            _state: std::marker::PhantomData,
        })
    }
}

impl Server<Running> {
    pub fn stop(self) -> Result<Server<Initialized>, Error> {
        // Stop server...
        Ok(Server {
            config: self.config,
            _state: std::marker::PhantomData,
        })
    }
}
```

### Const Generics

```rust
// ✅ GOOD: Fixed-size array types with const generics
#[derive(Debug)]
struct Matrix<const N: usize, const M: usize> {
    data: [[f64; M]; N],
}

impl<const N: usize, const M: usize> Matrix<N, M> {
    pub fn new() -> Self {
        Self {
            data: [[0.0; M]; N],
        }
    }

    pub fn transpose(&self) -> Matrix<M, N> {
        let mut result = Matrix::new();
        for i in 0..N {
            for j in 0..M {
                result.data[j][i] = self.data[i][j];
            }
        }
        result
    }
}
```

## Concurrency Patterns

### Actor Pattern

```rust
// ✅ GOOD: Actor-based concurrency
use tokio::sync::mpsc;

#[derive(Debug)]
enum Message {
    Process(Data),
    Stop,
}

struct Actor {
    rx: mpsc::Receiver<Message>,
    state: ActorState,
}

impl Actor {
    async fn run(&mut self) {
        while let Some(msg) = self.rx.recv().await {
            match msg {
                Message::Process(data) => {
                    self.process(data).await;
                }
                Message::Stop => break,
            }
        }
    }

    async fn process(&mut self, data: Data) {
        // Process data...
    }
}

// Usage
let (tx, rx) = mpsc::channel(100);
let mut actor = Actor::new(rx);
tokio::spawn(async move {
    actor.run().await;
});

// Send messages
tx.send(Message::Process(data)).await?;
```

### Work Stealing

```rust
// ✅ GOOD: Work-stealing scheduler
use crossbeam_deque::{Injector, Steal, Worker};
use std::sync::Arc;

struct WorkStealingPool {
    injector: Arc<Injector<Task>>,
    workers: Vec<Worker<Task>>,
}

impl WorkStealingPool {
    pub fn new(num_workers: usize) -> Self {
        let injector = Arc::new(Injector::new());
        let workers = (0..num_workers)
            .map(|_| Worker::new_fifo())
            .collect();

        Self { injector, workers }
    }

    pub fn execute(&self, task: Task) {
        self.injector.push(task);
    }

    fn steal_work(&self, worker: &Worker<Task>) -> Option<Task> {
        // Try to steal from other workers
        let mut rng = rand::thread_rng();
        let mut stealers: Vec<_> = self.workers.iter()
            .filter(|w| w != &worker)
            .collect();
        stealers.shuffle(&mut rng);

        for stealer in stealers {
            if let Steal::Success(task) = stealer.steal() {
                return Some(task);
            }
        }

        // Try to steal from the injector
        self.injector.steal().success()
    }
}
```

## Error Handling Patterns

### Error Context Chain

```rust
// ✅ GOOD: Rich error context chain
use snafu::{prelude::*, ResultExt};

#[derive(Debug, Snafu)]
enum Error {
    #[snafu(display("Failed to read config file: {}", source))]
    ConfigRead { source: std::io::Error },

    #[snafu(display("Invalid config at line {}: {}", line, message))]
    ConfigParse { line: usize, message: String },

    #[snafu(display("Database error: {}", source))]
    Database { source: sqlx::Error },
}

async fn load_config() -> Result<Config, Error> {
    let content = std::fs::read_to_string("config.toml")
        .context(ConfigReadSnafu)?;

    let config: Config = toml::from_str(&content)
        .map_err(|e| Error::ConfigParse {
            line: e.line().unwrap_or(0),
            message: e.message().to_string(),
        })?;

    Ok(config)
}
```

### Result Extension Traits

```rust
// ✅ GOOD: Custom result extensions
trait ResultExt<T, E> {
    fn on_error<F>(self, f: F) -> Result<T, E>
    where
        F: FnOnce(&E);
}

impl<T, E> ResultExt<T, E> for Result<T, E> {
    fn on_error<F>(self, f: F) -> Result<T, E>
    where
        F: FnOnce(&E),
    {
        if let Err(ref e) = self {
            f(e);
        }
        self
    }
}

// Usage
let result = operation()
    .on_error(|e| log::error!("Operation failed: {}", e))
    .on_error(|e| metrics::increment_counter("errors"));
```

## Testing Patterns

### Property-Based Testing

```rust
// ✅ GOOD: Property-based tests with strategies
use proptest::prelude::*;

#[derive(Debug, Clone)]
struct NonEmptyString(String);

prop_compose! {
    fn nonempty_string()(s in "[a-zA-Z][a-zA-Z0-9]*") -> NonEmptyString {
        NonEmptyString(s)
    }
}

proptest! {
    #[test]
    fn test_string_processing(s in nonempty_string()) {
        let processed = process_string(&s.0);
        prop_assert!(!processed.is_empty());
        prop_assert!(processed.len() >= s.0.len());
    }

    #[test]
    fn test_reversible_operation(vec in prop::collection::vec(0..100i32, 0..100)) {
        let processed = process_vec(&vec);
        let reversed = reverse_process(&processed);
        prop_assert_eq!(vec, reversed);
    }
}
```

### Test Utilities

```rust
// ✅ GOOD: Test helpers and fixtures
#[cfg(test)]
mod tests {
    use super::*;

    struct TestContext {
        db: MockDb,
        cache: MockCache,
        config: Config,
    }

    impl TestContext {
        async fn new() -> Self {
            let db = MockDb::new();
            let cache = MockCache::new();
            let config = Config::test_default();

            Self { db, cache, config }
        }

        async fn with_data(self) -> Self {
            self.db.insert_test_data().await;
            self.cache.prime().await;
            self
        }
    }

    #[tokio::test]
    async fn test_complex_operation() {
        let ctx = TestContext::new().await
            .with_data().await;

        let result = perform_operation(&ctx.db, &ctx.cache, &ctx.config).await;
        assert!(result.is_ok());
    }
}
```

## Macro Patterns

### Builder Macros

```rust
// ✅ GOOD: Derive macro for builder pattern
use proc_macro2::{TokenStream, Span};
use quote::{quote, format_ident};
use syn::{parse_macro_input, DeriveInput, Data};

#[proc_macro_derive(Builder)]
pub fn derive_builder(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    let builder_name = format_ident!("{}Builder", name);

    let fields = match input.data {
        Data::Struct(data) => data.fields,
        _ => panic!("Builder can only be derived for structs"),
    };

    // Generate builder methods...
    quote! {
        #[derive(Default)]
        pub struct #builder_name {
            // Field definitions...
        }

        impl #builder_name {
            // Builder methods...
        }
    }
}
```

### Debug Helper

```rust
// ✅ GOOD: Custom debug formatting
macro_rules! debug_fields {
    ($self:ident, $f:ident, $($field:ident),*) => {
        let mut debug = $f.debug_struct(stringify!($self));
        $(
            debug.field(stringify!($field), &$self.$field);
        )*
        debug.finish()
    };
}

struct ComplexType {
    field1: String,
    field2: Vec<i32>,
    field3: HashMap<String, String>,
}

impl Debug for ComplexType {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        debug_fields!(self, f, field1, field2, field3)
    }
}
```

## Summary

These advanced patterns focus on:
1. **Performance Optimization**: Zero-copy, SIMD, memory pooling
2. **Type Safety**: Type state pattern, const generics
3. **Concurrency**: Actors, work stealing
4. **Error Handling**: Rich context, custom extensions
5. **Testing**: Property-based testing, test utilities
6. **Macros**: Code generation, debug helpers

Remember to:
- Profile before optimizing
- Use advanced features judiciously
- Document complex patterns
- Consider maintenance implications
- Test thoroughly, especially with concurrent code

## References

1. **Rust Performance Book**: https://nnethercote.github.io/perf-book/
2. **Rust Design Patterns**: https://rust-unofficial.github.io/patterns/
3. **Async Book**: https://rust-lang.github.io/async-book/
4. **Apollo Rust Guidelines**: https://github.com/apollographql/rust-best-practices
