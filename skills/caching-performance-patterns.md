# Caching and Performance Patterns - Redis, CDN & Optimization

## Skill Metadata
- **Domain**: Caching, Performance Optimization, CDN
- **Skill Level**: Advanced
- **Last Updated**: 2026
- **Sources**: Redis Documentation, CDN Best Practices, High Performance Browser Networking

## Overview

Production-ready patterns for implementing caching strategies, CDN integration, rate limiting, and performance optimization in Go and Rust applications.

---

## 1. Redis Caching Patterns

### Cache-Aside (Lazy Loading)

```go
// ✅ GOOD: Cache-aside pattern with Redis
import (
    "context"
    "encoding/json"
    "github.com/redis/go-redis/v9"
    "time"
)

type CacheService struct {
    redis *redis.Client
    db    *Database
}

func (cs *CacheService) GetUser(ctx context.Context, userID string) (*User, error) {
    // Try to get from cache
    cacheKey := fmt.Sprintf("user:%s", userID)
    cached, err := cs.redis.Get(ctx, cacheKey).Result()
    
    if err == nil {
        // Cache hit
        var user User
        if err := json.Unmarshal([]byte(cached), &user); err == nil {
            return &user, nil
        }
    }
    
    // Cache miss - fetch from database
    user, err := cs.db.GetUser(userID)
    if err != nil {
        return nil, err
    }
    
    // Store in cache
    data, _ := json.Marshal(user)
    cs.redis.Set(ctx, cacheKey, data, 1*time.Hour)
    
    return user, nil
}

func (cs *CacheService) UpdateUser(ctx context.Context, user *User) error {
    // Update database
    if err := cs.db.UpdateUser(user); err != nil {
        return err
    }
    
    // Invalidate cache
    cacheKey := fmt.Sprintf("user:%s", user.ID)
    cs.redis.Del(ctx, cacheKey)
    
    return nil
}
```

### Write-Through Cache

```go
// ✅ GOOD: Write-through caching pattern
func (cs *CacheService) CreateUser(ctx context.Context, user *User) error {
    // Write to database first
    if err := cs.db.CreateUser(user); err != nil {
        return err
    }
    
    // Write to cache
    cacheKey := fmt.Sprintf("user:%s", user.ID)
    data, _ := json.Marshal(user)
    cs.redis.Set(ctx, cacheKey, data, 1*time.Hour)
    
    return nil
}
```

### Write-Behind (Write-Back) Cache

```go
// ✅ GOOD: Write-behind caching with background sync
type WriteBehindCache struct {
    redis      *redis.Client
    db         *Database
    writeQueue chan *CacheWrite
}

type CacheWrite struct {
    Key   string
    Value interface{}
}

func NewWriteBehindCache(redis *redis.Client, db *Database) *WriteBehindCache {
    wbc := &WriteBehindCache{
        redis:      redis,
        db:         db,
        writeQueue: make(chan *CacheWrite, 1000),
    }
    
    // Start background workers
    for i := 0; i < 5; i++ {
        go wbc.worker()
    }
    
    return wbc
}

func (wbc *WriteBehindCache) worker() {
    for write := range wbc.writeQueue {
        // Write to database
        if err := wbc.db.Save(write.Key, write.Value); err != nil {
            log.Printf("Failed to write to database: %v", err)
            // Retry logic here
        }
    }
}

func (wbc *WriteBehindCache) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error {
    // Write to cache immediately
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    if err := wbc.redis.Set(ctx, key, data, ttl).Err(); err != nil {
        return err
    }
    
    // Queue for database write
    select {
    case wbc.writeQueue <- &CacheWrite{Key: key, Value: value}:
    default:
        log.Printf("Write queue full, dropping write for key: %s", key)
    }
    
    return nil
}
```

### Cache Invalidation Strategies

```go
// ✅ GOOD: Tag-based cache invalidation
type TaggedCache struct {
    redis *redis.Client
}

func (tc *TaggedCache) Set(ctx context.Context, key string, value interface{}, tags []string, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    
    pipe := tc.redis.Pipeline()
    
    // Set the cache value
    pipe.Set(ctx, key, data, ttl)
    
    // Add key to each tag set
    for _, tag := range tags {
        tagKey := fmt.Sprintf("tag:%s", tag)
        pipe.SAdd(ctx, tagKey, key)
        pipe.Expire(ctx, tagKey, ttl)
    }
    
    _, err = pipe.Exec(ctx)
    return err
}

func (tc *TaggedCache) InvalidateByTag(ctx context.Context, tag string) error {
    tagKey := fmt.Sprintf("tag:%s", tag)
    
    // Get all keys with this tag
    keys, err := tc.redis.SMembers(ctx, tagKey).Result()
    if err != nil {
        return err
    }
    
    if len(keys) == 0 {
        return nil
    }
    
    // Delete all keys and the tag set
    keys = append(keys, tagKey)
    return tc.redis.Del(ctx, keys...).Err()
}

// Usage example
func (cs *CacheService) GetUserOrders(ctx context.Context, userID string) ([]Order, error) {
    cacheKey := fmt.Sprintf("user:%s:orders", userID)
    
    // Try cache
    cached, err := cs.cache.Get(ctx, cacheKey).Result()
    if err == nil {
        var orders []Order
        json.Unmarshal([]byte(cached), &orders)
        return orders, nil
    }
    
    // Fetch from database
    orders, err := cs.db.GetUserOrders(userID)
    if err != nil {
        return nil, err
    }
    
    // Cache with tags
    tags := []string{
        fmt.Sprintf("user:%s", userID),
        "orders",
    }
    cs.taggedCache.Set(ctx, cacheKey, orders, tags, 10*time.Minute)
    
    return orders, nil
}

func (cs *CacheService) CreateOrder(ctx context.Context, order *Order) error {
    if err := cs.db.CreateOrder(order); err != nil {
        return err
    }
    
    // Invalidate user's order cache
    cs.taggedCache.InvalidateByTag(ctx, fmt.Sprintf("user:%s", order.UserID))
    
    return nil
}
```

### Cache Stampede Prevention

```go
// ✅ GOOD: Preventing cache stampede with singleflight
import "golang.org/x/sync/singleflight"

type StampedeSafeCache struct {
    redis *redis.Client
    db    *Database
    group singleflight.Group
}

func (ssc *StampedeSafeCache) GetUser(ctx context.Context, userID string) (*User, error) {
    cacheKey := fmt.Sprintf("user:%s", userID)
    
    // Use singleflight to ensure only one goroutine fetches the data
    result, err, _ := ssc.group.Do(cacheKey, func() (interface{}, error) {
        // Try cache first
        cached, err := ssc.redis.Get(ctx, cacheKey).Result()
        if err == nil {
            var user User
            if err := json.Unmarshal([]byte(cached), &user); err == nil {
                return &user, nil
            }
        }
        
        // Fetch from database
        user, err := ssc.db.GetUser(userID)
        if err != nil {
            return nil, err
        }
        
        // Cache the result
        data, _ := json.Marshal(user)
        ssc.redis.Set(ctx, cacheKey, data, 1*time.Hour)
        
        return user, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    return result.(*User), nil
}

// ✅ GOOD: Probabilistic early expiration to prevent stampede
func (ssc *StampedeSafeCache) GetWithProbabilisticExpiry(ctx context.Context, key string, ttl time.Duration, fetchFn func() (interface{}, error)) (interface{}, error) {
    // Try to get from cache
    cached, err := ssc.redis.Get(ctx, key).Result()
    if err == nil {
        // Get TTL
        remainingTTL, _ := ssc.redis.TTL(ctx, key).Result()
        
        // Probabilistically refresh before expiry
        // Probability increases as TTL decreases
        delta := ttl - remainingTTL
        probability := delta.Seconds() / ttl.Seconds()
        
        if rand.Float64() < probability {
            // Refresh in background
            go func() {
                result, err := fetchFn()
                if err == nil {
                    data, _ := json.Marshal(result)
                    ssc.redis.Set(context.Background(), key, data, ttl)
                }
            }()
        }
        
        var result interface{}
        json.Unmarshal([]byte(cached), &result)
        return result, nil
    }
    
    // Cache miss - fetch and store
    result, err := fetchFn()
    if err != nil {
        return nil, err
    }
    
    data, _ := json.Marshal(result)
    ssc.redis.Set(ctx, key, data, ttl)
    
    return result, nil
}
```

---

## 2. Distributed Caching Patterns

### Redis Cluster Configuration

```go
// ✅ GOOD: Redis cluster setup
func NewRedisCluster(addrs []string) *redis.ClusterClient {
    return redis.NewClusterClient(&redis.ClusterOptions{
        Addrs:        addrs,
        MaxRetries:   3,
        PoolSize:     10,
        MinIdleConns: 5,
        DialTimeout:  5 * time.Second,
        ReadTimeout:  3 * time.Second,
        WriteTimeout: 3 * time.Second,
        PoolTimeout:  4 * time.Second,
    })
}

// ✅ GOOD: Consistent hashing for cache distribution
type ConsistentHashCache struct {
    clients []*redis.Client
    ring    *hashring.HashRing
}

func NewConsistentHashCache(addrs []string) (*ConsistentHashCache, error) {
    clients := make([]*redis.Client, len(addrs))
    nodes := make([]string, len(addrs))
    
    for i, addr := range addrs {
        clients[i] = redis.NewClient(&redis.Options{
            Addr: addr,
        })
        nodes[i] = addr
    }
    
    ring := hashring.New(nodes)
    
    return &ConsistentHashCache{
        clients: clients,
        ring:    ring,
    }, nil
}

func (chc *ConsistentHashCache) getClient(key string) *redis.Client {
    node, _ := chc.ring.GetNode(key)
    for i, client := range chc.clients {
        if chc.ring.GetNodes()[i] == node {
            return client
        }
    }
    return chc.clients[0]
}

func (chc *ConsistentHashCache) Get(ctx context.Context, key string) (string, error) {
    client := chc.getClient(key)
    return client.Get(ctx, key).Result()
}

func (chc *ConsistentHashCache) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error {
    client := chc.getClient(key)
    return client.Set(ctx, key, value, ttl).Err()
}
```

---

## 3. CDN Integration

### CDN Cache Control Headers

```go
// ✅ GOOD: Setting appropriate cache headers
func SetCacheHeaders(c *fiber.Ctx, cacheType string) {
    switch cacheType {
    case "static":
        // Static assets (images, CSS, JS) - cache for 1 year
        c.Set("Cache-Control", "public, max-age=31536000, immutable")
        c.Set("Expires", time.Now().Add(365*24*time.Hour).Format(http.TimeFormat))
        
    case "dynamic":
        // Dynamic content - cache for 5 minutes
        c.Set("Cache-Control", "public, max-age=300, must-revalidate")
        
    case "private":
        // User-specific content - cache in browser only
        c.Set("Cache-Control", "private, max-age=300")
        
    case "no-cache":
        // Sensitive content - no caching
        c.Set("Cache-Control", "no-store, no-cache, must-revalidate, proxy-revalidate")
        c.Set("Pragma", "no-cache")
        c.Set("Expires", "0")
    }
}

// ✅ GOOD: ETag support for conditional requests
func (h *Handler) GetResource(c *fiber.Ctx) error {
    resource, err := h.service.GetResource(c.Params("id"))
    if err != nil {
        return c.Status(404).JSON(fiber.Map{"error": "Not found"})
    }
    
    // Generate ETag
    etag := fmt.Sprintf(`"%s-%d"`, resource.ID, resource.UpdatedAt.Unix())
    
    // Check If-None-Match header
    if c.Get("If-None-Match") == etag {
        return c.SendStatus(304) // Not Modified
    }
    
    // Set ETag and cache headers
    c.Set("ETag", etag)
    SetCacheHeaders(c, "dynamic")
    
    return c.JSON(resource)
}

// ✅ GOOD: Last-Modified support
func (h *Handler) GetFile(c *fiber.Ctx) error {
    file, err := h.storage.GetFile(c.Params("id"))
    if err != nil {
        return c.Status(404).JSON(fiber.Map{"error": "Not found"})
    }
    
    // Check If-Modified-Since header
    if modifiedSince := c.Get("If-Modified-Since"); modifiedSince != "" {
        t, _ := time.Parse(http.TimeFormat, modifiedSince)
        if !file.ModifiedAt.After(t) {
            return c.SendStatus(304)
        }
    }
    
    // Set Last-Modified header
    c.Set("Last-Modified", file.ModifiedAt.Format(http.TimeFormat))
    SetCacheHeaders(c, "static")
    
    return c.SendFile(file.Path)
}
```

### CDN Purge/Invalidation

```go
// ✅ GOOD: CDN cache invalidation
type CDNService struct {
    apiKey    string
    apiSecret string
    baseURL   string
}

func (cdn *CDNService) PurgeCache(paths []string) error {
    payload := map[string]interface{}{
        "files": paths,
    }
    
    data, _ := json.Marshal(payload)
    
    req, err := http.NewRequest("POST", cdn.baseURL+"/purge", bytes.NewBuffer(data))
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-API-Key", cdn.apiKey)
    
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != 200 {
        return fmt.Errorf("CDN purge failed with status: %d", resp.StatusCode)
    }
    
    return nil
}

// ✅ GOOD: Automatic CDN purge on content update
func (s *ContentService) UpdateContent(ctx context.Context, content *Content) error {
    // Update content
    if err := s.db.UpdateContent(content); err != nil {
        return err
    }
    
    // Invalidate cache
    cacheKey := fmt.Sprintf("content:%s", content.ID)
    s.cache.Del(ctx, cacheKey)
    
    // Purge CDN
    paths := []string{
        fmt.Sprintf("/api/content/%s", content.ID),
        fmt.Sprintf("/content/%s", content.Slug),
    }
    
    go func() {
        if err := s.cdn.PurgeCache(paths); err != nil {
            log.Printf("CDN purge failed: %v", err)
        }
    }()
    
    return nil
}
```

---

## 4. Rate Limiting and Throttling

### Token Bucket Rate Limiter

```go
// ✅ GOOD: Token bucket rate limiter with Redis
type TokenBucketLimiter struct {
    redis *redis.Client
}

func (tbl *TokenBucketLimiter) Allow(ctx context.Context, key string, rate int, burst int) (bool, error) {
    now := time.Now().Unix()
    
    script := redis.NewScript(`
        local key = KEYS[1]
        local rate = tonumber(ARGV[1])
        local burst = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        
        local bucket = redis.call('HMGET', key, 'tokens', 'last_update')
        local tokens = tonumber(bucket[1]) or burst
        local last_update = tonumber(bucket[2]) or now
        
        local elapsed = now - last_update
        tokens = math.min(burst, tokens + elapsed * rate)
        
        if tokens >= 1 then
            tokens = tokens - 1
            redis.call('HMSET', key, 'tokens', tokens, 'last_update', now)
            redis.call('EXPIRE', key, 60)
            return 1
        else
            return 0
        end
    `)
    
    result, err := script.Run(ctx, tbl.redis, []string{key}, rate, burst, now).Int()
    if err != nil {
        return false, err
    }
    
    return result == 1, nil
}

// ✅ GOOD: Rate limiting middleware
func RateLimitMiddleware(limiter *TokenBucketLimiter) fiber.Handler {
    return func(c *fiber.Ctx) error {
        // Use IP address as key
        key := fmt.Sprintf("ratelimit:%s", c.IP())
        
        // Allow 100 requests per second with burst of 200
        allowed, err := limiter.Allow(c.Context(), key, 100, 200)
        if err != nil {
            return c.Status(500).JSON(fiber.Map{"error": "Rate limit check failed"})
        }
        
        if !allowed {
            c.Set("Retry-After", "1")
            return c.Status(429).JSON(fiber.Map{"error": "Too many requests"})
        }
        
        return c.Next()
    }
}
```

### Sliding Window Rate Limiter

```go
// ✅ GOOD: Sliding window rate limiter
type SlidingWindowLimiter struct {
    redis *redis.Client
}

func (swl *SlidingWindowLimiter) Allow(ctx context.Context, key string, limit int, window time.Duration) (bool, error) {
    now := time.Now()
    windowStart := now.Add(-window)
    
    pipe := swl.redis.Pipeline()
    
    // Remove old entries
    pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%d", windowStart.UnixNano()))
    
    // Count current entries
    countCmd := pipe.ZCard(ctx, key)
    
    // Add current request
    pipe.ZAdd(ctx, key, redis.Z{
        Score:  float64(now.UnixNano()),
        Member: fmt.Sprintf("%d", now.UnixNano()),
    })
    
    // Set expiration
    pipe.Expire(ctx, key, window)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    count := countCmd.Val()
    return count < int64(limit), nil
}

// ✅ GOOD: Per-user rate limiting
func (h *Handler) RateLimitByUser() fiber.Handler {
    return func(c *fiber.Ctx) error {
        claims := c.Locals("claims").(*Claims)
        key := fmt.Sprintf("ratelimit:user:%s", claims.UserID)
        
        // Allow 1000 requests per hour
        allowed, err := h.limiter.Allow(c.Context(), key, 1000, 1*time.Hour)
        if err != nil {
            return c.Status(500).JSON(fiber.Map{"error": "Rate limit check failed"})
        }
        
        if !allowed {
            return c.Status(429).JSON(fiber.Map{
                "error": "Rate limit exceeded",
                "retry_after": 3600,
            })
        }
        
        return c.Next()
    }
}
```

### Distributed Rate Limiting

```go
// ✅ GOOD: Distributed rate limiting with Redis
type DistributedRateLimiter struct {
    redis *redis.Client
}

func (drl *DistributedRateLimiter) AllowN(ctx context.Context, key string, limit int, window time.Duration, cost int) (bool, error) {
    script := redis.NewScript(`
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local cost = tonumber(ARGV[3])
        local now = tonumber(ARGV[4])
        
        local current = redis.call('GET', key)
        if current == false then
            redis.call('SETEX', key, window, cost)
            return 1
        end
        
        current = tonumber(current)
        if current + cost <= limit then
            redis.call('INCRBY', key, cost)
            return 1
        end
        
        return 0
    `)
    
    result, err := script.Run(ctx, drl.redis, 
        []string{key}, 
        limit, 
        int(window.Seconds()), 
        cost, 
        time.Now().Unix(),
    ).Int()
    
    if err != nil {
        return false, err
    }
    
    return result == 1, nil
}
```

---

## 5. Performance Optimization

### Response Compression

```go
// ✅ GOOD: Compression middleware
import "github.com/gofiber/fiber/v2/middleware/compress"

func SetupCompression() fiber.Handler {
    return compress.New(compress.Config{
        Level: compress.LevelBestSpeed,
        // Only compress responses larger than 1KB
        Next: func(c *fiber.Ctx) bool {
            return c.Response().Header.ContentLength() < 1024
        },
    })
}
```

### Database Query Caching

```go
// ✅ GOOD: Query result caching
type QueryCache struct {
    redis *redis.Client
}

func (qc *QueryCache) Query(ctx context.Context, query string, args []interface{}, dest interface{}) error {
    // Generate cache key from query and args
    cacheKey := qc.generateCacheKey(query, args)
    
    // Try cache
    cached, err := qc.redis.Get(ctx, cacheKey).Result()
    if err == nil {
        return json.Unmarshal([]byte(cached), dest)
    }
    
    // Execute query
    if err := qc.db.Query(query, args, dest); err != nil {
        return err
    }
    
    // Cache result
    data, _ := json.Marshal(dest)
    qc.redis.Set(ctx, cacheKey, data, 5*time.Minute)
    
    return nil
}

func (qc *QueryCache) generateCacheKey(query string, args []interface{}) string {
    h := sha256.New()
    h.Write([]byte(query))
    for _, arg := range args {
        h.Write([]byte(fmt.Sprintf("%v", arg)))
    }
    return fmt.Sprintf("query:%x", h.Sum(nil))
}
```

### Batch Operations

```go
// ✅ GOOD: Batch loading with DataLoader pattern
type DataLoader struct {
    batchFn  func([]string) (map[string]interface{}, error)
    cache    map[string]interface{}
    batch    []string
    mu       sync.Mutex
    wait     time.Duration
    maxBatch int
}

func NewDataLoader(batchFn func([]string) (map[string]interface{}, error)) *DataLoader {
    dl := &DataLoader{
        batchFn:  batchFn,
        cache:    make(map[string]interface{}),
        batch:    make([]string, 0),
        wait:     10 * time.Millisecond,
        maxBatch: 100,
    }
    
    go dl.processBatches()
    
    return dl
}

func (dl *DataLoader) Load(key string) (interface{}, error) {
    dl.mu.Lock()
    
    // Check cache
    if val, ok := dl.cache[key]; ok {
        dl.mu.Unlock()
        return val, nil
    }
    
    // Add to batch
    dl.batch = append(dl.batch, key)
    
    // Trigger batch if full
    if len(dl.batch) >= dl.maxBatch {
        dl.mu.Unlock()
        return dl.executeBatch()
    }
    
    dl.mu.Unlock()
    
    // Wait for batch
    time.Sleep(dl.wait)
    return dl.Load(key)
}

func (dl *DataLoader) processBatches() {
    ticker := time.NewTicker(dl.wait)
    defer ticker.Stop()
    
    for range ticker.C {
        dl.executeBatch()
    }
}

func (dl *DataLoader) executeBatch() (interface{}, error) {
    dl.mu.Lock()
    if len(dl.batch) == 0 {
        dl.mu.Unlock()
        return nil, nil
    }
    
    batch := dl.batch
    dl.batch = make([]string, 0)
    dl.mu.Unlock()
    
    // Execute batch function
    results, err := dl.batchFn(batch)
    if err != nil {
        return nil, err
    }
    
    // Update cache
    dl.mu.Lock()
    for k, v := range results {
        dl.cache[k] = v
    }
    dl.mu.Unlock()
    
    return results, nil
}
```

---

## 6. Rust Caching Patterns

```rust
// ✅ GOOD: Redis caching in Rust
use redis::{Client, Commands, AsyncCommands};
use serde::{Serialize, Deserialize};

pub struct CacheService {
    redis: Client,
}

impl CacheService {
    pub async fn get_user(&self, user_id: &str) -> Result<Option<User>, Error> {
        let mut conn = self.redis.get_async_connection().await?;
        let cache_key = format!("user:{}", user_id);
        
        // Try cache
        let cached: Option<String> = conn.get(&cache_key).await?;
        if let Some(data) = cached {
            if let Ok(user) = serde_json::from_str::<User>(&data) {
                return Ok(Some(user));
            }
        }
        
        // Fetch from database
        let user = self.db.get_user(user_id).await?;
        
        // Cache result
        let data = serde_json::to_string(&user)?;
        conn.set_ex(&cache_key, data, 3600).await?;
        
        Ok(Some(user))
    }
    
    pub async fn invalidate_user(&self, user_id: &str) -> Result<(), Error> {
        let mut conn = self.redis.get_async_connection().await?;
        let cache_key = format!("user:{}", user_id);
        conn.del(&cache_key).await?;
        Ok(())
    }
}

// ✅ GOOD: Moka cache for in-memory caching
use moka::future::Cache;

pub struct InMemoryCache {
    cache: Cache<String, User>,
}

impl InMemoryCache {
    pub fn new() -> Self {
        let cache = Cache::builder()
            .max_capacity(10_000)
            .time_to_live(Duration::from_secs(3600))
            .time_to_idle(Duration::from_secs(600))
            .build();
        
        Self { cache }
    }
    
    pub async fn get_or_fetch<F, Fut>(&self, key: String, fetch_fn: F) -> Result<User, Error>
    where
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<User, Error>>,
    {
        if let Some(user) = self.cache.get(&key).await {
            return Ok(user);
        }
        
        let user = fetch_fn().await?;
        self.cache.insert(key, user.clone()).await;
        Ok(user)
    }
}
```

---

## Summary

This document covers:

1. **Redis Caching**: Cache-aside, write-through, write-behind, invalidation
2. **Distributed Caching**: Cluster setup, consistent hashing
3. **CDN Integration**: Cache headers, ETag, purging
4. **Rate Limiting**: Token bucket, sliding window, distributed limiting
5. **Performance**: Compression, query caching, batch operations
6. **Rust Patterns**: Redis and in-memory caching

## References

1. **Redis Documentation**: https://redis.io/docs/
2. **CDN Best Practices**: https://www.cloudflare.com/learning/cdn/
3. **Caching Strategies**: https://aws.amazon.com/caching/best-practices/

---

**Last Updated**: January 2026  
**Version**: 1.0
