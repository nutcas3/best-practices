# Agent Skills - Programming Best Practices

A comprehensive collection of production-ready best practices for training AI agents to write high-quality, maintainable code across multiple programming languages.

## Overview

This repository contains detailed skill documents that codify industry-standard best practices, common pitfalls to avoid, and idiomatic patterns for professional software development. These skills are designed to be used as training material for AI coding agents to ensure they generate production-ready code.

## Purpose

Modern AI coding agents need to understand not just syntax, but the deeper principles of software engineering:

- **Safety**: Write code that prevents bugs at compile time
- **Performance**: Optimize for efficiency without premature optimization
- **Maintainability**: Create code that's easy to understand and modify
- **Reliability**: Handle errors gracefully and prevent resource leaks
- **Idiomatic**: Follow language-specific conventions and patterns

## Available Skills

### Core Language Best Practices

#### [Go Best Practices](skills/golang-best-practices.md)

Comprehensive guide covering:
- Project organization following the standard Go layout
- Idiomatic code patterns and control structures
- Concurrency best practices (goroutines, channels, sync primitives)
- Memory management and performance optimization
- Error handling and reliability patterns
- Testing strategies (table-driven tests, race detection)
- Production-ready HTTP clients and servers
- Modern Go features (Go 1.18+): generics, `any`, `slog`

**Key Topics:**
- The "happy path" left-aligned pattern
- WaitGroup and errgroup usage
- Context propagation
- Slice and map memory management
- Functional options pattern
- Graceful shutdown

**Sources**: 100 Go Mistakes and How to Avoid Them, Effective Go, Go Project Layout

#### [Rust Best Practices](skills/rust-best-practices.md)

Comprehensive guide covering:
- Cargo project structure and module organization
- Ownership, borrowing, and lifetime management
- Error handling with thiserror and anyhow
- Type system patterns (newtype, builder, type state)
- Async/await and concurrent programming with Tokio
- Performance optimization and zero-cost abstractions
- Memory safety patterns and smart pointers

**Key Topics:**
- Ownership patterns and smart pointers (Rc, Arc, Box)
- Custom error types with thiserror
- Async programming with tokio
- Iterator chains and zero-cost abstractions
- Pin and self-referential structs
- Property-based testing with proptest

**Sources**: Rust Style Guide, Apollo Rust Best Practices, Rust API Guidelines

### Advanced Patterns

#### [Go Advanced Patterns](skills/go-advanced-patterns.md)

Advanced concurrency and design patterns:
- Fan-out/fan-in patterns
- Adaptive worker pools
- Circuit breaker implementation
- Service mesh patterns
- Event sourcing and CQRS
- API gateway patterns
- Domain-driven design in Go
- Lock-free data structures
- Memory pools with size classes

#### [Rust Advanced Patterns](skills/rust-advanced-patterns.md)

Advanced Rust patterns and optimizations:
- Zero-copy parsing
- Custom allocators and SIMD
- Type state pattern
- Const generics
- Actor pattern
- Work stealing schedulers
- Error context chains
- Property-based testing
- Procedural macros

### Web Application Development

#### [Web Application Patterns](skills/web-application-patterns.md)

**Complete guide to building modern web applications:**

**Authentication Flows:**
- Session-based authentication with secure cookies
- JWT with refresh tokens (access + refresh pattern)
- OAuth 2.0 implementation (Google, GitHub)
- State management for CSRF protection
- Token revocation and logout mechanisms

**Forms & Validation:**
- Server-side validation with go-playground/validator
- Custom validators (strong passwords, adult verification)
- Secure file upload handling with MIME type detection
- File size and type validation

**Real-Time Communication:**
- Production-ready WebSocket server with graceful shutdown
- Redis pub/sub for distributed messaging
- Connection pooling and heartbeat mechanisms
- Broadcasting to specific users/channels

**State Management:**
- Client state synchronization with Redis
- Optimistic locking with version control
- Conflict resolution strategies

**Sources**: OAuth 2.0 RFC, JWT Best Practices, WebSocket RFC, OWASP Session Management

#### [Database Engineering Patterns](skills/database-engineering-patterns.md) 💾

**Production-ready database design and optimization:**

**Schema Design:**
- Normalized schema design (3NF) with constraints
- Strategic denormalization with materialized views
- Polymorphic associations with proper validation
- Audit trail patterns

**Migrations:**
- Zero-downtime migration strategies
- golang-migrate integration
- Adding columns, indexes, and constraints safely
- Backfilling data in batches

**Query Optimization:**
- Indexing strategies (B-tree, GIN, partial, covering)
- Cursor-based pagination for large datasets
- N+1 query prevention with batch loading
- EXPLAIN ANALYZE for performance tuning

**Transaction Management:**
- Isolation levels (Read Committed, Serializable)
- Optimistic locking with version columns
- Pessimistic locking with SELECT FOR UPDATE
- Deadlock prevention strategies

**Sharding & Partitioning:**
- Range and hash partitioning
- Application-level sharding with consistent hashing
- Shard-aware repository patterns

**Connection Pooling:**
- Proper pool configuration (max connections, idle timeout)
- Health checks and connection validation

**Sources**: PostgreSQL Documentation, Database Internals, Use The Index Luke

#### [Caching & Performance Patterns](skills/caching-performance-patterns.md) ⚡

**High-performance caching strategies:**

**Redis Caching:**
- Cache-aside (lazy loading) pattern
- Write-through and write-behind caching
- Tag-based cache invalidation
- Cache stampede prevention with singleflight
- Probabilistic early expiration

**Distributed Caching:**
- Redis cluster configuration
- Consistent hashing for cache distribution
- Cache replication and failover

**CDN Integration:**
- Cache-Control headers (static, dynamic, private)
- ETag and Last-Modified support
- CDN purge/invalidation strategies
- Automatic cache invalidation on content updates

**Rate Limiting:**
- Token bucket algorithm with Redis
- Sliding window rate limiter
- Distributed rate limiting
- Per-user and per-IP limits

**Performance Optimization:**
- Response compression middleware
- Database query result caching
- Batch operations with DataLoader pattern
- In-memory caching with Moka (Rust)

**Sources**: Redis Documentation, CDN Best Practices, Caching Strategies

### Security & Observability

#### [API Security Patterns](skills/api-security-patterns.md) 🔒

**Production-ready security patterns based on real-world audits:**

**Authentication & Authorization:**
- JWT validation with algorithm verification
- Token revocation and logout mechanisms
- Strong password/PIN policies (6+ digits)
- Account lockout (5 attempts, 15-min lockout)
- Multi-factor authentication (TOTP, SMS OTP)
- Role-based access control (RBAC)
- IDOR prevention
- Field-level access control

**Network Security:**
- CORS configuration (no wildcards)
- HTTPS enforcement with HSTS
- CSRF protection
- Rate limiting (general and auth-specific)
- Security headers (CSP, X-Frame-Options, etc.)

**Input Validation:**
- Comprehensive validation patterns
- SQL injection prevention
- XSS prevention
- Decimal/amount validation for financial apps

**Business Logic Security:**
- Mandatory idempotency for financial operations
- Race condition prevention (database locking)
- Replay attack prevention (nonce + timestamp)
- Business rule validation

**Data Protection:**
- Field-level encryption at rest
- PII handling and anonymization
- Data retention policies
- Audit logging with sensitive data sanitization

**Sources**: OWASP API Security Top 10, PCI DSS, Real-world Security Audits

#### [Web Security Attacks Guide](skills/web-security-attacks-guide.md) 🛡️

**Comprehensive guide to 100 common web security vulnerabilities:**

**Injection Attacks (1-18):**
- SQL Injection (union-based, error-based, blind, time-based)
- NoSQL Injection
- Command Injection & OS Argument Injection
- LDAP, XPath, XML Injection
- Server-Side Template Injection (SSTI)
- GraphQL Injection

**XSS Attacks (7-18):**
- Reflected, Stored, DOM-based XSS
- Blind XSS and Mutation XSS
- HTML, Attribute, JavaScript Context Injection
- CSS Injection and DOM Clobbering
- Prototype Pollution

**CSRF & Session Attacks (19-35):**
- Cross-Site Request Forgery
- Login CSRF
- Session Fixation and Hijacking
- Weak Session ID Entropy

**Authentication Vulnerabilities (21-30):**
- Brute Force and Password Spraying
- Credential Stuffing
- Account Enumeration
- OTP Prediction and Reuse
- MFA Bypass techniques

**JWT Vulnerabilities (36-39):**
- "none" Algorithm Attack
- HS256/RS256 Confusion
- Weak Secret Brute-forcing
- Claim Tampering

**Access Control (40-50):**
- IDOR (Insecure Direct Object References)
- Horizontal/Vertical Privilege Escalation
- Mass Assignment
- Multi-Tenant Isolation Bypass

**File Upload Attacks (51-60):**
- Unrestricted File Upload
- Double Extension, MIME-Type Spoofing
- Path Traversal, Zip Slip
- Upload to RCE/XSS/SSRF

**SSRF & Deserialization (61-73):**
- Cloud Metadata SSRF (AWS/GCP/Azure)
- SSRF via Redirects and Webhooks
- Insecure Deserialization (Java, .NET, PHP, Python)

**Logic Flaws & HTTP Attacks (74-78):**
- Race Conditions / Double Spend
- Workflow Bypass
- HTTP Request Smuggling
- Cache Poisoning

**Each attack includes:**
- Attack description and examples
- Detection methods
- Mitigation strategies in Go and Rust
- Security testing patterns

**Sources**: OWASP Top 10, PortSwigger Web Security, HackerOne Reports

#### [API Design Patterns](skills/api-design-patterns.md) 🔌

**Professional API design and distributed systems:**

**RESTful API Design:**
- Resource naming conventions (collections, nested resources)
- Proper HTTP status codes (200, 201, 400, 401, 403, 404, 422, 500)
- Pagination (cursor-based and offset-based)
- Filtering, sorting, and searching
- API versioning strategies (URL, header, deprecation)
- HATEOAS (Hypermedia links)

**GraphQL:**
- Schema design with proper types and connections
- Resolver implementation in Go
- DataLoader pattern for N+1 query prevention
- Pagination with Relay cursor connections
- Input validation and error handling

**OpenAPI 3.0:**
- Comprehensive API specification
- Request/response schemas
- Security schemes (Bearer, OAuth2)
- Parameter validation
- Error response standards

**Microservices Patterns:**
- Service discovery with Consul
- API Gateway with routing and load balancing
- Circuit breaker implementation
- Retry and backoff strategies
- Health checks and monitoring

**Event-Driven Architecture:**
- Kafka producer/consumer patterns
- Event sourcing
- CQRS (Command Query Responsibility Segregation)
- Message ordering and idempotency

**Sources**: REST API Guidelines, GraphQL Best Practices, OpenAPI Spec, Microservices Patterns

#### [Observability Patterns](skills/observability-patterns.md)

Monitoring and security patterns:
- Structured logging with context propagation
- Distributed tracing with OpenTelemetry
- Metrics collection with Prometheus
- Health checks and monitoring
- Rate limiting implementations
- Input validation and sanitization
- Security headers and authentication

#### [Testing Patterns](skills/testing-patterns.md)

Advanced testing strategies:
- Property-based testing (rapid, proptest)
- Contract testing for APIs
- Integration testing with test containers
- Load testing (vegeta, criterion)
- Chaos testing for resilience
- Mock testing with proper isolation
- Fuzz testing for security

## Quick Start

### For AI Agent Training

1. **Load the skill documents** into your agent's context or training data
2. **Reference specific sections** when generating code for particular patterns
3. **Use the checklists** to validate generated code before deployment

### For Developers

1. **Clone the repository**:
   ```bash
   git clone https://github.com/nutcas3/best-practices.git
   cd best-practices
   ```

2. **Browse the skills** in the `skills/` directory
3. **Check the examples** in the `examples/` directory for reference implementations

## Repository Structure

```
/agent-skills
├── skills/                          # Skill documents
│   ├── golang-best-practices.md    # Go best practices
│   └── rust-best-practices.md      # Rust best practices
├── examples/                        # Reference implementations
│   ├── go-api-example/             # Go API example with best practices
│   │   ├── .github/workflows/      # CI/CD configuration
│   │   ├── .golangci.yml          # Linter configuration
│   │   └── Makefile               # Build automation
│   └── rust-api-example/          # Rust API example with best practices
│       ├── .github/workflows/     # CI/CD configuration
│       ├── rustfmt.toml          # Formatter configuration
│       ├── .clippy.toml          # Clippy configuration
│       └── Cargo.toml            # Package manifest
└── README.md                      # This file
```

## How to Use These Skills

### For Go Development

1. **Project Setup**: Follow the standard layout in `/cmd`, `/internal`, `/pkg`
2. **Error Handling**: Always wrap errors with `%w` for context
3. **Concurrency**: Use `errgroup` for coordinated goroutines
4. **Testing**: Run tests with `-race` flag to catch data races
5. **Production**: Configure timeouts on all HTTP clients and servers

### For Rust Development

1. **Project Setup**: Use Cargo's standard structure with clear module boundaries
2. **Error Handling**: Use `thiserror` for libraries, `anyhow` for applications
3. **Async**: Use `tokio` runtime with proper task lifecycle management
4. **Testing**: Include unit tests, property-based tests, and doc tests
5. **Production**: Enable all Clippy lints and address warnings

## Pre-Deployment Checklists

### Go Checklist

- [ ] All errors handled or explicitly ignored
- [ ] Resources closed with `defer`
- [ ] Goroutines have clear lifecycle management
- [ ] `wg.Add()` called before spawning goroutines
- [ ] Context propagated through call chains
- [ ] HTTP clients/servers have timeouts
- [ ] Tests pass with `-race` flag
- [ ] Linter passes with no warnings

### Rust Checklist

- [ ] All `unwrap()` calls reviewed and justified
- [ ] Error handling with proper context
- [ ] Clippy warnings addressed
- [ ] Tests passing (unit, integration, doc)
- [ ] Security audit passed (`cargo audit`)
- [ ] Documentation complete
- [ ] Graceful shutdown implemented
- [ ] Resource limits configured

## CI/CD Integration

Both skill documents include production-ready CI/CD configurations:

### Go (GitHub Actions)
- Linting with golangci-lint
- Testing with race detector
- Code coverage reporting
- Build verification

### Rust (GitHub Actions)
- Formatting checks with rustfmt
- Linting with Clippy
- Security audits with cargo-audit
- Code coverage with tarpaulin
- Release builds

## Key Principles

### Universal Best Practices

1. **Correctness First**: Write code that works correctly before optimizing
2. **Handle All Errors**: Never silently ignore errors
3. **Test Thoroughly**: Unit tests, integration tests, and edge cases
4. **Document Intent**: Clear comments and documentation
5. **Profile Before Optimizing**: Measure performance before making changes

### Language-Specific Principles

**Go**:
- Simplicity over cleverness
- Composition over inheritance
- Explicit over implicit
- Concurrency is not parallelism

**Rust**:
- Safety without garbage collection
- Zero-cost abstractions
- Ownership and borrowing
- Fearless concurrency

## Contributing

Contributions are welcome! To add new skills or improve existing ones:

1. Fork the repository
2. Create a feature branch
3. Follow the existing skill document format
4. Include practical examples and anti-patterns
5. Add references to authoritative sources
6. Submit a pull request

### Skill Document Format

Each skill document should include:
- **Metadata**: Domain, skill level, sources
- **Overview**: Purpose and scope
- **Sections**: Organized by topic with examples
- **Anti-patterns**: What NOT to do (marked with ❌)
- **Best practices**: What TO do (marked with ✅)
- **Checklists**: Pre-deployment verification
- **References**: Links to authoritative sources

## Additional Resources

### Go Resources
- [100 Go Mistakes](https://100go.co/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Project Layout](https://github.com/golang-standards/project-layout)
- [Uber Go Style Guide](https://github.com/uber-go/guide)

### Rust Resources
- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust Style Guide](https://doc.rust-lang.org/style-guide/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Apollo Rust Best Practices](https://github.com/apollographql/rust-best-practices)

## Roadmap

Future skills to be added:
- [ ] Python best practices
- [ ] TypeScript/JavaScript best practices
- [ ] Database design patterns
- [ ] API design principles
- [ ] Security best practices
- [ ] Cloud-native patterns
- [ ] Microservices architecture

## License

This repository is provided as educational material for training AI agents and developers. Feel free to use, modify, and distribute with attribution.

## Acknowledgments

These skills are compiled from industry-standard resources and the collective wisdom of the software engineering community:

- Teiva Harsanyi (100 Go Mistakes)
- The Go Team (Effective Go)
- The Rust Team (Rust Book, Style Guide)
- Apollo GraphQL (Rust Best Practices)
- The broader open-source community

---

**Built with ❤️ for better code generation by AI agents**

For questions or suggestions, please open an issue or submit a pull request.
