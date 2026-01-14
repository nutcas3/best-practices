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

### Security & Observability

#### [API Security Patterns](skills/api-security-patterns.md) 

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
