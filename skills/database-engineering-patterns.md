# Database Engineering Patterns - Schema Design, Migrations & Optimization

## Skill Metadata
- **Domain**: Database Engineering, Schema Design, Query Optimization
- **Skill Level**: Advanced
- **Last Updated**: 2026
- **Sources**: PostgreSQL Documentation, MySQL Best Practices, Database Internals

## Overview

Production-ready patterns for database schema design, migrations, transactions, indexing, and optimization for Go and Rust applications.

---

## 1. Schema Design Best Practices

### Normalized Schema Design

```sql
-- ✅ GOOD: Properly normalized schema (3NF)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP NULL,
    
    CONSTRAINT email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    date_of_birth DATE,
    phone VARCHAR(20),
    avatar_url TEXT,
    bio TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    address_type VARCHAR(20) NOT NULL CHECK (address_type IN ('billing', 'shipping', 'home')),
    street_address VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(2) NOT NULL,
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE(user_id, address_type, is_default) WHERE is_default = TRUE
);

-- ✅ GOOD: Audit trail pattern
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    record_id UUID NOT NULL,
    action VARCHAR(10) NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    old_data JSONB,
    new_data JSONB,
    changed_by UUID REFERENCES users(id),
    changed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    INDEX idx_audit_table_record (table_name, record_id),
    INDEX idx_audit_changed_at (changed_at DESC)
);
```

### Denormalization for Performance

```sql
-- ✅ GOOD: Strategic denormalization with materialized views
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL,
    total_amount DECIMAL(10, 2) NOT NULL,
    item_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL,
    subtotal DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Materialized view for order summaries
CREATE MATERIALIZED VIEW order_summaries AS
SELECT 
    o.id,
    o.user_id,
    o.status,
    o.created_at,
    COUNT(oi.id) as item_count,
    SUM(oi.subtotal) as total_amount,
    ARRAY_AGG(p.name) as product_names
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
LEFT JOIN products p ON oi.product_id = p.id
GROUP BY o.id, o.user_id, o.status, o.created_at;

CREATE UNIQUE INDEX idx_order_summaries_id ON order_summaries(id);
CREATE INDEX idx_order_summaries_user ON order_summaries(user_id);

-- Refresh strategy
CREATE OR REPLACE FUNCTION refresh_order_summaries()
RETURNS TRIGGER AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY order_summaries;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER refresh_order_summaries_trigger
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_order_summaries();
```

### Polymorphic Associations

```sql
-- ✅ GOOD: Polymorphic associations with proper constraints
CREATE TABLE comments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    commentable_type VARCHAR(50) NOT NULL,
    commentable_id UUID NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    CHECK (
        (commentable_type = 'post' AND EXISTS (SELECT 1 FROM posts WHERE id = commentable_id)) OR
        (commentable_type = 'product' AND EXISTS (SELECT 1 FROM products WHERE id = commentable_id))
    )
);

CREATE INDEX idx_comments_polymorphic ON comments(commentable_type, commentable_id);
```

---

## 2. Database Migrations

### Migration Management (Go with golang-migrate)

```go
// ✅ GOOD: Migration setup with golang-migrate
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

type MigrationManager struct {
    migrate *migrate.Migrate
}

func NewMigrationManager(dbURL, migrationsPath string) (*MigrationManager, error) {
    m, err := migrate.New(
        fmt.Sprintf("file://%s", migrationsPath),
        dbURL,
    )
    if err != nil {
        return nil, err
    }
    
    return &MigrationManager{migrate: m}, nil
}

func (mm *MigrationManager) Up() error {
    if err := mm.migrate.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }
    return nil
}

func (mm *MigrationManager) Down() error {
    return mm.migrate.Down()
}

func (mm *MigrationManager) Version() (uint, bool, error) {
    return mm.migrate.Version()
}

func (mm *MigrationManager) Steps(n int) error {
    return mm.migrate.Steps(n)
}
```

### Migration Files

```sql
-- migrations/000001_create_users_table.up.sql
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);

-- migrations/000001_create_users_table.down.sql
DROP TABLE IF EXISTS users CASCADE;

-- migrations/000002_add_user_roles.up.sql
CREATE TYPE user_role AS ENUM ('user', 'admin', 'moderator');

ALTER TABLE users ADD COLUMN role user_role NOT NULL DEFAULT 'user';
CREATE INDEX idx_users_role ON users(role);

-- migrations/000002_add_user_roles.down.sql
ALTER TABLE users DROP COLUMN role;
DROP TYPE user_role;

-- migrations/000003_add_soft_delete.up.sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP NULL;
CREATE INDEX idx_users_deleted_at ON users(deleted_at) WHERE deleted_at IS NULL;

-- migrations/000003_add_soft_delete.down.sql
ALTER TABLE users DROP COLUMN deleted_at;
```

### Zero-Downtime Migrations

```sql
-- ✅ GOOD: Adding a column with default value (zero-downtime)
-- Step 1: Add column as nullable
ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;

-- Step 2: Backfill data in batches (application code)
-- UPDATE users SET phone = '' WHERE phone IS NULL LIMIT 1000;

-- Step 3: Add NOT NULL constraint
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- ✅ GOOD: Renaming a column (zero-downtime)
-- Step 1: Add new column
ALTER TABLE users ADD COLUMN email_address VARCHAR(255);

-- Step 2: Backfill data
UPDATE users SET email_address = email WHERE email_address IS NULL;

-- Step 3: Add constraints to new column
ALTER TABLE users ALTER COLUMN email_address SET NOT NULL;
CREATE UNIQUE INDEX CONCURRENTLY idx_users_email_address ON users(email_address);

-- Step 4: Update application to use new column

-- Step 5: Drop old column
ALTER TABLE users DROP COLUMN email;

-- ✅ GOOD: Adding an index without locking
CREATE INDEX CONCURRENTLY idx_users_created_at ON users(created_at DESC);
```

---

## 3. Query Optimization

### Indexing Strategies

```sql
-- ✅ GOOD: B-tree indexes for equality and range queries
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_orders_status ON orders(status);

-- ✅ GOOD: Composite indexes for multi-column queries
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);

-- ✅ GOOD: Partial indexes for filtered queries
CREATE INDEX idx_active_orders ON orders(user_id) WHERE status = 'active';
CREATE INDEX idx_pending_orders ON orders(created_at DESC) WHERE status = 'pending';

-- ✅ GOOD: Covering indexes (include columns)
CREATE INDEX idx_orders_covering ON orders(user_id, status) 
    INCLUDE (total_amount, created_at);

-- ✅ GOOD: GIN indexes for full-text search
ALTER TABLE products ADD COLUMN search_vector tsvector;

CREATE INDEX idx_products_search ON products USING GIN(search_vector);

UPDATE products SET search_vector = 
    to_tsvector('english', coalesce(name, '') || ' ' || coalesce(description, ''));

-- ✅ GOOD: Expression indexes
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
CREATE INDEX idx_orders_year_month ON orders(EXTRACT(YEAR FROM created_at), EXTRACT(MONTH FROM created_at));
```

### Query Optimization Patterns (Go)

```go
// ✅ GOOD: Efficient pagination with cursor-based approach
type PaginationCursor struct {
    ID        string
    CreatedAt time.Time
}

func (r *OrderRepository) ListOrders(userID string, cursor *PaginationCursor, limit int) ([]Order, *PaginationCursor, error) {
    query := `
        SELECT id, user_id, status, total_amount, created_at
        FROM orders
        WHERE user_id = $1
    `
    
    args := []interface{}{userID}
    argCount := 1
    
    if cursor != nil {
        argCount++
        query += fmt.Sprintf(" AND (created_at, id) < ($%d, $%d)", argCount, argCount+1)
        args = append(args, cursor.CreatedAt, cursor.ID)
        argCount++
    }
    
    argCount++
    query += fmt.Sprintf(" ORDER BY created_at DESC, id DESC LIMIT $%d", argCount)
    args = append(args, limit+1) // Fetch one extra to determine if there are more
    
    rows, err := r.db.Query(query, args...)
    if err != nil {
        return nil, nil, err
    }
    defer rows.Close()
    
    orders := make([]Order, 0, limit)
    for rows.Next() {
        var order Order
        if err := rows.Scan(&order.ID, &order.UserID, &order.Status, &order.TotalAmount, &order.CreatedAt); err != nil {
            return nil, nil, err
        }
        orders = append(orders, order)
    }
    
    var nextCursor *PaginationCursor
    if len(orders) > limit {
        lastOrder := orders[limit-1]
        nextCursor = &PaginationCursor{
            ID:        lastOrder.ID,
            CreatedAt: lastOrder.CreatedAt,
        }
        orders = orders[:limit]
    }
    
    return orders, nextCursor, nil
}

// ✅ GOOD: Batch loading to avoid N+1 queries
func (r *OrderRepository) LoadOrdersWithItems(orderIDs []string) (map[string][]OrderItem, error) {
    query := `
        SELECT order_id, id, product_id, quantity, unit_price, subtotal
        FROM order_items
        WHERE order_id = ANY($1)
        ORDER BY order_id, created_at
    `
    
    rows, err := r.db.Query(query, pq.Array(orderIDs))
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    result := make(map[string][]OrderItem)
    for rows.Next() {
        var item OrderItem
        if err := rows.Scan(&item.OrderID, &item.ID, &item.ProductID, &item.Quantity, &item.UnitPrice, &item.Subtotal); err != nil {
            return nil, err
        }
        result[item.OrderID] = append(result[item.OrderID], item)
    }
    
    return result, nil
}

// ✅ GOOD: Using EXPLAIN ANALYZE for query optimization
func (r *Repository) AnalyzeQuery(query string, args ...interface{}) (string, error) {
    explainQuery := "EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) " + query
    
    var result string
    err := r.db.QueryRow(explainQuery, args...).Scan(&result)
    if err != nil {
        return "", err
    }
    
    return result, nil
}
```

---

## 4. Transaction Management

### Transaction Isolation Levels

```go
// ✅ GOOD: Using appropriate isolation levels
import (
    "database/sql"
    "context"
)

type TransactionManager struct {
    db *sql.DB
}

func (tm *TransactionManager) WithTransaction(ctx context.Context, isolationLevel sql.IsolationLevel, fn func(*sql.Tx) error) error {
    tx, err := tm.db.BeginTx(ctx, &sql.TxOptions{
        Isolation: isolationLevel,
    })
    if err != nil {
        return err
    }
    
    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p)
        }
    }()
    
    if err := fn(tx); err != nil {
        tx.Rollback()
        return err
    }
    
    return tx.Commit()
}

// ✅ GOOD: Read Committed for most operations
func (s *OrderService) CreateOrder(ctx context.Context, order Order) error {
    return s.tm.WithTransaction(ctx, sql.LevelReadCommitted, func(tx *sql.Tx) error {
        // Insert order
        _, err := tx.ExecContext(ctx, `
            INSERT INTO orders (id, user_id, status, total_amount)
            VALUES ($1, $2, $3, $4)
        `, order.ID, order.UserID, order.Status, order.TotalAmount)
        if err != nil {
            return err
        }
        
        // Insert order items
        for _, item := range order.Items {
            _, err := tx.ExecContext(ctx, `
                INSERT INTO order_items (id, order_id, product_id, quantity, unit_price, subtotal)
                VALUES ($1, $2, $3, $4, $5, $6)
            `, item.ID, order.ID, item.ProductID, item.Quantity, item.UnitPrice, item.Subtotal)
            if err != nil {
                return err
            }
        }
        
        return nil
    })
}

// ✅ GOOD: Serializable for financial operations
func (s *PaymentService) ProcessPayment(ctx context.Context, payment Payment) error {
    return s.tm.WithTransaction(ctx, sql.LevelSerializable, func(tx *sql.Tx) error {
        // Lock account for update
        var balance decimal.Decimal
        err := tx.QueryRowContext(ctx, `
            SELECT balance FROM accounts WHERE id = $1 FOR UPDATE
        `, payment.AccountID).Scan(&balance)
        if err != nil {
            return err
        }
        
        // Check sufficient balance
        if balance.LessThan(payment.Amount) {
            return errors.New("insufficient balance")
        }
        
        // Deduct amount
        _, err = tx.ExecContext(ctx, `
            UPDATE accounts SET balance = balance - $1 WHERE id = $2
        `, payment.Amount, payment.AccountID)
        if err != nil {
            return err
        }
        
        // Record transaction
        _, err = tx.ExecContext(ctx, `
            INSERT INTO transactions (id, account_id, amount, type, status)
            VALUES ($1, $2, $3, $4, $5)
        `, payment.ID, payment.AccountID, payment.Amount, "debit", "completed")
        
        return err
    })
}
```

### Optimistic Locking

```go
// ✅ GOOD: Optimistic locking with version column
type Product struct {
    ID       string
    Name     string
    Stock    int
    Version  int
    UpdatedAt time.Time
}

func (r *ProductRepository) UpdateStock(ctx context.Context, productID string, quantity int, expectedVersion int) error {
    result, err := r.db.ExecContext(ctx, `
        UPDATE products
        SET stock = stock - $1,
            version = version + 1,
            updated_at = NOW()
        WHERE id = $2 AND version = $3 AND stock >= $1
    `, quantity, productID, expectedVersion)
    if err != nil {
        return err
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return errors.New("version mismatch or insufficient stock")
    }
    
    return nil
}
```

### Pessimistic Locking

```go
// ✅ GOOD: Pessimistic locking with SELECT FOR UPDATE
func (s *InventoryService) ReserveStock(ctx context.Context, productID string, quantity int) error {
    return s.tm.WithTransaction(ctx, sql.LevelReadCommitted, func(tx *sql.Tx) error {
        // Lock row for update
        var stock int
        err := tx.QueryRowContext(ctx, `
            SELECT stock FROM products WHERE id = $1 FOR UPDATE
        `, productID).Scan(&stock)
        if err != nil {
            return err
        }
        
        if stock < quantity {
            return errors.New("insufficient stock")
        }
        
        // Update stock
        _, err = tx.ExecContext(ctx, `
            UPDATE products SET stock = stock - $1 WHERE id = $2
        `, quantity, productID)
        
        return err
    })
}
```

---

## 5. Database Sharding and Partitioning

### Table Partitioning

```sql
-- ✅ GOOD: Range partitioning by date
CREATE TABLE events (
    id BIGSERIAL,
    user_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data JSONB,
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions for each month
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Indexes on partitions
CREATE INDEX idx_events_2024_01_user ON events_2024_01(user_id);
CREATE INDEX idx_events_2024_02_user ON events_2024_02(user_id);

-- ✅ GOOD: Hash partitioning for even distribution
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    session_data JSONB,
    created_at TIMESTAMP NOT NULL
) PARTITION BY HASH (user_id);

CREATE TABLE user_sessions_0 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE user_sessions_1 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE user_sessions_2 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE user_sessions_3 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Sharding Strategy (Go)

```go
// ✅ GOOD: Application-level sharding
type ShardManager struct {
    shards []*sql.DB
}

func NewShardManager(connectionStrings []string) (*ShardManager, error) {
    shards := make([]*sql.DB, len(connectionStrings))
    
    for i, connStr := range connectionStrings {
        db, err := sql.Open("postgres", connStr)
        if err != nil {
            return nil, err
        }
        shards[i] = db
    }
    
    return &ShardManager{shards: shards}, nil
}

func (sm *ShardManager) GetShard(userID string) *sql.DB {
    // Hash-based sharding
    hash := fnv.New32a()
    hash.Write([]byte(userID))
    shardIndex := hash.Sum32() % uint32(len(sm.shards))
    return sm.shards[shardIndex]
}

func (sm *ShardManager) GetShardByKey(key string) *sql.DB {
    hash := fnv.New32a()
    hash.Write([]byte(key))
    shardIndex := hash.Sum32() % uint32(len(sm.shards))
    return sm.shards[shardIndex]
}

// ✅ GOOD: Shard-aware repository
type UserRepository struct {
    shardManager *ShardManager
}

func (r *UserRepository) GetUser(userID string) (*User, error) {
    db := r.shardManager.GetShard(userID)
    
    var user User
    err := db.QueryRow(`
        SELECT id, email, username, created_at
        FROM users
        WHERE id = $1
    `, userID).Scan(&user.ID, &user.Email, &user.Username, &user.CreatedAt)
    
    if err != nil {
        return nil, err
    }
    
    return &user, nil
}

func (r *UserRepository) CreateUser(user *User) error {
    db := r.shardManager.GetShard(user.ID)
    
    _, err := db.Exec(`
        INSERT INTO users (id, email, username, password_hash, created_at)
        VALUES ($1, $2, $3, $4, $5)
    `, user.ID, user.Email, user.Username, user.PasswordHash, user.CreatedAt)
    
    return err
}
```

---

## 6. Connection Pooling

### Connection Pool Configuration (Go)

```go
// ✅ GOOD: Properly configured connection pool
func NewDatabase(dsn string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    
    // Set maximum number of open connections
    db.SetMaxOpenConns(25)
    
    // Set maximum number of idle connections
    db.SetMaxIdleConns(5)
    
    // Set maximum lifetime of a connection
    db.SetConnMaxLifetime(5 * time.Minute)
    
    // Set maximum idle time of a connection
    db.SetConnMaxIdleTime(10 * time.Minute)
    
    // Verify connection
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    return db, nil
}

// ✅ GOOD: Health check for database
func (db *Database) HealthCheck(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()
    
    if err := db.PingContext(ctx); err != nil {
        return fmt.Errorf("database ping failed: %w", err)
    }
    
    // Check if we can execute a simple query
    var result int
    err := db.QueryRowContext(ctx, "SELECT 1").Scan(&result)
    if err != nil {
        return fmt.Errorf("database query failed: %w", err)
    }
    
    return nil
}
```

### Rust Database Patterns

```rust
// ✅ GOOD: SQLx with connection pooling
use sqlx::{PgPool, postgres::PgPoolOptions};
use std::time::Duration;

pub async fn create_pool(database_url: &str) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(25)
        .min_connections(5)
        .max_lifetime(Duration::from_secs(300))
        .idle_timeout(Duration::from_secs(600))
        .acquire_timeout(Duration::from_secs(3))
        .connect(database_url)
        .await
}

// ✅ GOOD: Repository pattern with SQLx
pub struct UserRepository {
    pool: PgPool,
}

impl UserRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
    
    pub async fn find_by_id(&self, id: &Uuid) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as!(
            User,
            r#"
            SELECT id, email, username, created_at, updated_at
            FROM users
            WHERE id = $1 AND deleted_at IS NULL
            "#,
            id
        )
        .fetch_optional(&self.pool)
        .await
    }
    
    pub async fn create(&self, user: &NewUser) -> Result<User, sqlx::Error> {
        sqlx::query_as!(
            User,
            r#"
            INSERT INTO users (id, email, username, password_hash)
            VALUES ($1, $2, $3, $4)
            RETURNING id, email, username, created_at, updated_at
            "#,
            user.id,
            user.email,
            user.username,
            user.password_hash
        )
        .fetch_one(&self.pool)
        .await
    }
    
    pub async fn update(&self, id: &Uuid, updates: &UserUpdate) -> Result<User, sqlx::Error> {
        sqlx::query_as!(
            User,
            r#"
            UPDATE users
            SET email = COALESCE($2, email),
                username = COALESCE($3, username),
                updated_at = NOW()
            WHERE id = $1 AND deleted_at IS NULL
            RETURNING id, email, username, created_at, updated_at
            "#,
            id,
            updates.email,
            updates.username
        )
        .fetch_one(&self.pool)
        .await
    }
}

// ✅ GOOD: Transaction handling in Rust
pub async fn transfer_funds(
    pool: &PgPool,
    from_account: &Uuid,
    to_account: &Uuid,
    amount: Decimal,
) -> Result<(), Error> {
    let mut tx = pool.begin().await?;
    
    // Lock accounts
    let from_balance: Decimal = sqlx::query_scalar!(
        "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE",
        from_account
    )
    .fetch_one(&mut *tx)
    .await?;
    
    if from_balance < amount {
        return Err(Error::InsufficientBalance);
    }
    
    // Deduct from source
    sqlx::query!(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount,
        from_account
    )
    .execute(&mut *tx)
    .await?;
    
    // Add to destination
    sqlx::query!(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount,
        to_account
    )
    .execute(&mut *tx)
    .await?;
    
    tx.commit().await?;
    Ok(())
}
```

---

## Summary

This document covers:

1. **Schema Design**: Normalization, denormalization, polymorphic associations
2. **Migrations**: Zero-downtime migrations, version control
3. **Query Optimization**: Indexing strategies, pagination, N+1 prevention
4. **Transactions**: Isolation levels, optimistic/pessimistic locking
5. **Sharding**: Partitioning, application-level sharding
6. **Connection Pooling**: Configuration, health checks

## References

1. **PostgreSQL Documentation**: https://www.postgresql.org/docs/
2. **Database Internals**: https://www.databass.dev/
3. **Use The Index, Luke**: https://use-the-index-luke.com/
4. **golang-migrate**: https://github.com/golang-migrate/migrate

---

**Last Updated**: January 2026  
**Version**: 1.0
