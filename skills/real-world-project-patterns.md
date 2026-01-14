# Real-World Project Patterns - Payment Systems, Order Management & Background Jobs

## Skill Metadata
- **Domain**: Real-World Applications, Payment Processing, State Machines, Job Queues
- **Skill Level**: Advanced
- **Last Updated**: 2026
- **Sources**: Stripe API Documentation, Event Sourcing Patterns, Background Job Best Practices

## Overview

Production-ready patterns for building real-world applications including payment processing with idempotency, order management with state machines, background job processors with retries, and event-driven architectures.

---

## 1. Payment Workflow Simulator

### Idempotent Payment Processing

```go
// ✅ GOOD: Idempotent payment processing with retries and webhooks
type PaymentService struct {
    db              *Database
    paymentGateway  *PaymentGateway
    idempotencyStore *IdempotencyStore
    eventBus        *EventBus
    redis           *redis.Client
}

type Payment struct {
    ID              string          `json:"id"`
    OrderID         string          `json:"order_id"`
    Amount          decimal.Decimal `json:"amount"`
    Currency        string          `json:"currency"`
    Status          string          `json:"status"`
    IdempotencyKey  string          `json:"idempotency_key"`
    ProviderID      string          `json:"provider_id,omitempty"`
    ProviderStatus  string          `json:"provider_status,omitempty"`
    Metadata        map[string]string `json:"metadata"`
    CreatedAt       time.Time       `json:"created_at"`
    UpdatedAt       time.Time       `json:"updated_at"`
}

func (ps *PaymentService) ProcessPayment(ctx context.Context, req *PaymentRequest) (*Payment, error) {
    // Check idempotency - prevent duplicate processing
    if cached, exists := ps.idempotencyStore.Get(req.IdempotencyKey); exists {
        return cached.(*Payment), nil
    }
    
    // Validate amount
    if req.Amount.LessThanOrEqual(decimal.Zero) {
        return nil, errors.New("invalid amount")
    }
    
    // Create payment record in pending state
    payment := &Payment{
        ID:             uuid.New().String(),
        OrderID:        req.OrderID,
        Amount:         req.Amount,
        Currency:       req.Currency,
        Status:         "pending",
        IdempotencyKey: req.IdempotencyKey,
        Metadata:       req.Metadata,
        CreatedAt:      time.Now(),
    }
    
    // Save to database first
    if err := ps.db.CreatePayment(payment); err != nil {
        return nil, fmt.Errorf("failed to create payment: %w", err)
    }
    
    // Process with payment gateway (with retries)
    result, err := ps.processWithRetry(ctx, payment, 3)
    if err != nil {
        payment.Status = "failed"
        payment.UpdatedAt = time.Now()
        ps.db.UpdatePayment(payment)
        
        // Publish failure event
        ps.eventBus.Publish("payment.failed", payment)
        
        return nil, fmt.Errorf("payment processing failed: %w", err)
    }
    
    // Update payment with gateway response
    payment.Status = result.Status
    payment.ProviderID = result.ProviderID
    payment.ProviderStatus = result.ProviderStatus
    payment.UpdatedAt = time.Now()
    
    if err := ps.db.UpdatePayment(payment); err != nil {
        return nil, fmt.Errorf("failed to update payment: %w", err)
    }
    
    // Store in idempotency cache (24 hours)
    ps.idempotencyStore.Set(req.IdempotencyKey, payment, 24*time.Hour)
    
    // Publish success event
    ps.eventBus.Publish("payment.succeeded", payment)
    
    return payment, nil
}

func (ps *PaymentService) processWithRetry(ctx context.Context, payment *Payment, maxRetries int) (*PaymentResult, error) {
    var lastErr error
    
    for attempt := 0; attempt < maxRetries; attempt++ {
        if attempt > 0 {
            // Exponential backoff: 1s, 2s, 4s
            backoff := time.Duration(math.Pow(2, float64(attempt))) * time.Second
            select {
            case <-time.After(backoff):
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }
        
        result, err := ps.paymentGateway.Charge(ctx, &ChargeRequest{
            Amount:         payment.Amount,
            Currency:       payment.Currency,
            IdempotencyKey: payment.IdempotencyKey,
            Metadata:       payment.Metadata,
        })
        
        if err == nil {
            return result, nil
        }
        
        // Check if error is retryable
        if !isRetryableError(err) {
            return nil, err
        }
        
        lastErr = err
        log.Printf("Payment attempt %d failed: %v", attempt+1, err)
    }
    
    return nil, fmt.Errorf("max retries exceeded: %w", lastErr)
}

func isRetryableError(err error) bool {
    // Network errors and timeouts are retryable
    if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
        return true
    }
    
    // 5xx errors are retryable
    if httpErr, ok := err.(*HTTPError); ok {
        return httpErr.StatusCode >= 500
    }
    
    // Rate limit errors are retryable
    if strings.Contains(err.Error(), "rate_limit") {
        return true
    }
    
    return false
}
```

### Webhook Handling with Signature Verification

```go
// ✅ GOOD: Secure webhook handling
type WebhookHandler struct {
    secret         string
    paymentService *PaymentService
    eventStore     *EventStore
    redis          *redis.Client
}

func (wh *WebhookHandler) HandleWebhook(c *fiber.Ctx) error {
    // Verify signature
    signature := c.Get("X-Webhook-Signature")
    timestamp := c.Get("X-Webhook-Timestamp")
    
    if !wh.verifySignature(c.Body(), signature, timestamp) {
        log.Printf("Invalid webhook signature from IP: %s", c.IP())
        return c.Status(401).JSON(fiber.Map{"error": "Invalid signature"})
    }
    
    // Check timestamp to prevent replay attacks (5 minute window)
    ts, err := strconv.ParseInt(timestamp, 10, 64)
    if err != nil || time.Since(time.Unix(ts, 0)) > 5*time.Minute {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid timestamp"})
    }
    
    var event WebhookEvent
    if err := c.BodyParser(&event); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid payload"})
    }
    
    // Check for duplicate events (idempotency)
    eventKey := fmt.Sprintf("webhook:event:%s", event.ID)
    exists, _ := wh.redis.SetNX(context.Background(), eventKey, "1", 24*time.Hour).Result()
    if !exists {
        log.Printf("Duplicate webhook event: %s", event.ID)
        return c.Status(200).JSON(fiber.Map{"status": "already processed"})
    }
    
    // Store event for audit trail
    if err := wh.eventStore.Save(event); err != nil {
        log.Printf("Failed to store webhook event: %v", err)
    }
    
    // Process event asynchronously
    go wh.processEvent(event)
    
    // Return 200 immediately to acknowledge receipt
    return c.Status(200).JSON(fiber.Map{"status": "received"})
}

func (wh *WebhookHandler) verifySignature(payload []byte, signature, timestamp string) bool {
    // Construct signed payload: timestamp.payload
    signedPayload := fmt.Sprintf("%s.%s", timestamp, string(payload))
    
    // Compute HMAC
    mac := hmac.New(sha256.New, []byte(wh.secret))
    mac.Write([]byte(signedPayload))
    expectedSignature := hex.EncodeToString(mac.Sum(nil))
    
    // Constant-time comparison
    return hmac.Equal([]byte(signature), []byte(expectedSignature))
}

func (wh *WebhookHandler) processEvent(event WebhookEvent) {
    ctx := context.Background()
    
    switch event.Type {
    case "payment.succeeded":
        wh.handlePaymentSucceeded(ctx, event)
    case "payment.failed":
        wh.handlePaymentFailed(ctx, event)
    case "refund.created":
        wh.handleRefundCreated(ctx, event)
    case "dispute.created":
        wh.handleDisputeCreated(ctx, event)
    default:
        log.Printf("Unknown webhook event type: %s", event.Type)
    }
}

func (wh *WebhookHandler) handlePaymentSucceeded(ctx context.Context, event WebhookEvent) {
    payment := event.Data.(*Payment)
    
    // Update payment status
    if err := wh.paymentService.UpdatePaymentStatus(ctx, payment.ID, "succeeded"); err != nil {
        log.Printf("Failed to update payment status: %v", err)
        return
    }
    
    // Complete order
    if err := wh.paymentService.CompleteOrder(ctx, payment.OrderID); err != nil {
        log.Printf("Failed to complete order: %v", err)
    }
}
```

### Refund Processing

```go
// ✅ GOOD: Refund processing with partial refunds
type RefundService struct {
    db             *Database
    paymentGateway *PaymentGateway
    eventBus       *EventBus
}

type Refund struct {
    ID        string          `json:"id"`
    PaymentID string          `json:"payment_id"`
    Amount    decimal.Decimal `json:"amount"`
    Reason    string          `json:"reason"`
    Status    string          `json:"status"`
    CreatedAt time.Time       `json:"created_at"`
    UpdatedAt time.Time       `json:"updated_at"`
}

func (rs *RefundService) CreateRefund(ctx context.Context, paymentID string, amount decimal.Decimal, reason string) (*Refund, error) {
    // Get payment
    payment, err := rs.db.GetPayment(paymentID)
    if err != nil {
        return nil, fmt.Errorf("payment not found: %w", err)
    }
    
    // Validate payment status
    if payment.Status != "succeeded" {
        return nil, errors.New("can only refund succeeded payments")
    }
    
    // Calculate total refunded amount
    totalRefunded, err := rs.db.GetTotalRefunded(paymentID)
    if err != nil {
        return nil, err
    }
    
    // Validate refund amount
    if totalRefunded.Add(amount).GreaterThan(payment.Amount) {
        return nil, errors.New("refund amount exceeds payment amount")
    }
    
    // Create refund record
    refund := &Refund{
        ID:        uuid.New().String(),
        PaymentID: paymentID,
        Amount:    amount,
        Reason:    reason,
        Status:    "pending",
        CreatedAt: time.Now(),
    }
    
    if err := rs.db.CreateRefund(refund); err != nil {
        return nil, fmt.Errorf("failed to create refund: %w", err)
    }
    
    // Process refund with gateway
    result, err := rs.paymentGateway.Refund(ctx, &RefundRequest{
        PaymentID: payment.ProviderID,
        Amount:    amount,
        Reason:    reason,
    })
    
    if err != nil {
        refund.Status = "failed"
        refund.UpdatedAt = time.Now()
        rs.db.UpdateRefund(refund)
        return nil, fmt.Errorf("refund processing failed: %w", err)
    }
    
    refund.Status = "succeeded"
    refund.UpdatedAt = time.Now()
    rs.db.UpdateRefund(refund)
    
    // Publish event
    rs.eventBus.Publish("refund.succeeded", refund)
    
    return refund, nil
}
```

---

## 2. Order Management System with State Machine

### State Machine Implementation

```go
// ✅ GOOD: Order state machine with strict validation
type OrderStatus string

const (
    OrderStatusPending    OrderStatus = "pending"
    OrderStatusConfirmed  OrderStatus = "confirmed"
    OrderStatusProcessing OrderStatus = "processing"
    OrderStatusShipped    OrderStatus = "shipped"
    OrderStatusDelivered  OrderStatus = "delivered"
    OrderStatusCancelled  OrderStatus = "cancelled"
    OrderStatusRefunded   OrderStatus = "refunded"
)

type OrderStateMachine struct {
    transitions map[OrderStatus][]OrderStatus
}

func NewOrderStateMachine() *OrderStateMachine {
    return &OrderStateMachine{
        transitions: map[OrderStatus][]OrderStatus{
            OrderStatusPending: {
                OrderStatusConfirmed,
                OrderStatusCancelled,
            },
            OrderStatusConfirmed: {
                OrderStatusProcessing,
                OrderStatusCancelled,
            },
            OrderStatusProcessing: {
                OrderStatusShipped,
                OrderStatusCancelled,
            },
            OrderStatusShipped: {
                OrderStatusDelivered,
            },
            OrderStatusDelivered: {
                OrderStatusRefunded,
            },
            // Terminal states have no transitions
            OrderStatusCancelled: {},
            OrderStatusRefunded:  {},
        },
    }
}

func (osm *OrderStateMachine) CanTransition(from, to OrderStatus) bool {
    allowed, exists := osm.transitions[from]
    if !exists {
        return false
    }
    
    for _, status := range allowed {
        if status == to {
            return true
        }
    }
    
    return false
}

func (osm *OrderStateMachine) GetAllowedTransitions(from OrderStatus) []OrderStatus {
    return osm.transitions[from]
}

type OrderService struct {
    db           *Database
    stateMachine *OrderStateMachine
    eventBus     *EventBus
    inventory    *InventoryService
}

func (os *OrderService) TransitionOrder(ctx context.Context, orderID string, newStatus OrderStatus, metadata map[string]string) error {
    // Use transaction with pessimistic locking
    return os.db.WithTransaction(ctx, func(tx *sql.Tx) error {
        // Lock order for update
        order, err := os.db.GetOrderForUpdate(tx, orderID)
        if err != nil {
            return fmt.Errorf("failed to get order: %w", err)
        }
        
        currentStatus := OrderStatus(order.Status)
        
        // Validate transition
        if !os.stateMachine.CanTransition(currentStatus, newStatus) {
            return fmt.Errorf("invalid transition from %s to %s", currentStatus, newStatus)
        }
        
        // Execute pre-transition hooks
        if err := os.executePreTransitionHook(ctx, tx, order, newStatus); err != nil {
            return fmt.Errorf("pre-transition hook failed: %w", err)
        }
        
        // Update order status
        oldStatus := order.Status
        order.Status = string(newStatus)
        order.UpdatedAt = time.Now()
        
        if err := os.db.UpdateOrderInTx(tx, order); err != nil {
            return fmt.Errorf("failed to update order: %w", err)
        }
        
        // Create state transition record for audit trail
        transition := &OrderStateTransition{
            ID:        uuid.New().String(),
            OrderID:   orderID,
            FromState: oldStatus,
            ToState:   string(newStatus),
            Metadata:  metadata,
            CreatedAt: time.Now(),
        }
        
        if err := os.db.CreateTransitionInTx(tx, transition); err != nil {
            return fmt.Errorf("failed to create transition record: %w", err)
        }
        
        // Execute post-transition hooks
        if err := os.executePostTransitionHook(ctx, tx, order, OrderStatus(oldStatus), newStatus); err != nil {
            return fmt.Errorf("post-transition hook failed: %w", err)
        }
        
        // Publish event (after transaction commits)
        go os.eventBus.Publish("order.status_changed", map[string]interface{}{
            "order_id":   orderID,
            "old_status": oldStatus,
            "new_status": newStatus,
            "metadata":   metadata,
            "timestamp":  time.Now(),
        })
        
        return nil
    })
}

func (os *OrderService) executePreTransitionHook(ctx context.Context, tx *sql.Tx, order *Order, newStatus OrderStatus) error {
    switch newStatus {
    case OrderStatusConfirmed:
        // Reserve inventory
        for _, item := range order.Items {
            if err := os.inventory.ReserveStock(ctx, item.ProductID, item.Quantity); err != nil {
                return fmt.Errorf("failed to reserve inventory: %w", err)
            }
        }
    case OrderStatusCancelled:
        // Release inventory reservations
        for _, item := range order.Items {
            os.inventory.ReleaseReservation(ctx, item.ProductID, item.Quantity)
        }
    }
    return nil
}

func (os *OrderService) executePostTransitionHook(ctx context.Context, tx *sql.Tx, order *Order, oldStatus, newStatus OrderStatus) error {
    switch newStatus {
    case OrderStatusShipped:
        // Send shipping notification
        go os.sendShippingNotification(order)
    case OrderStatusDelivered:
        // Commit inventory
        for _, item := range order.Items {
            os.inventory.CommitReservation(ctx, item.ProductID, item.Quantity)
        }
    }
    return nil
}
```

### Inventory Management with Concurrency Control

```go
// ✅ GOOD: Inventory reservation with optimistic locking
type InventoryService struct {
    db *Database
}

type InventoryItem struct {
    ProductID string          `json:"product_id"`
    Quantity  int             `json:"quantity"`
    Reserved  int             `json:"reserved"`
    Version   int             `json:"version"`
    UpdatedAt time.Time       `json:"updated_at"`
}

func (is *InventoryService) ReserveStock(ctx context.Context, productID string, quantity int) error {
    maxRetries := 5
    var lastErr error
    
    for attempt := 0; attempt < maxRetries; attempt++ {
        // Get current inventory
        item, err := is.db.GetInventory(productID)
        if err != nil {
            return fmt.Errorf("failed to get inventory: %w", err)
        }
        
        // Check available quantity
        available := item.Quantity - item.Reserved
        if available < quantity {
            return fmt.Errorf("insufficient stock: available=%d, requested=%d", available, quantity)
        }
        
        // Try to update with version check (optimistic locking)
        result, err := is.db.Exec(`
            UPDATE inventory
            SET reserved = reserved + $1,
                version = version + 1,
                updated_at = NOW()
            WHERE product_id = $2 
              AND version = $3
              AND (quantity - reserved) >= $1
        `, quantity, productID, item.Version)
        
        if err != nil {
            return fmt.Errorf("failed to update inventory: %w", err)
        }
        
        rowsAffected, _ := result.RowsAffected()
        if rowsAffected > 0 {
            return nil // Success
        }
        
        // Version mismatch - retry with exponential backoff
        lastErr = errors.New("version mismatch")
        backoff := time.Duration(attempt*10) * time.Millisecond
        time.Sleep(backoff)
    }
    
    return fmt.Errorf("failed to reserve stock after %d attempts: %w", maxRetries, lastErr)
}

func (is *InventoryService) ReleaseReservation(ctx context.Context, productID string, quantity int) error {
    _, err := is.db.Exec(`
        UPDATE inventory
        SET reserved = GREATEST(0, reserved - $1),
            version = version + 1,
            updated_at = NOW()
        WHERE product_id = $2
    `, quantity, productID)
    
    return err
}

func (is *InventoryService) CommitReservation(ctx context.Context, productID string, quantity int) error {
    _, err := is.db.Exec(`
        UPDATE inventory
        SET quantity = quantity - $1,
            reserved = GREATEST(0, reserved - $1),
            version = version + 1,
            updated_at = NOW()
        WHERE product_id = $2
    `, quantity, productID)
    
    return err
}
```

### Saga Pattern for Distributed Transactions

```go
// ✅ GOOD: Saga pattern for order creation
type OrderSaga struct {
    orderService     *OrderService
    paymentService   *PaymentService
    inventoryService *InventoryService
    shippingService  *ShippingService
}

type SagaStep struct {
    Name       string
    Execute    func(ctx context.Context, data interface{}) error
    Compensate func(ctx context.Context, data interface{}) error
}

type OrderSagaData struct {
    OrderID     string
    UserID      string
    Items       []OrderItem
    TotalAmount decimal.Decimal
    PaymentID   string
}

func (os *OrderSaga) CreateOrder(ctx context.Context, req *CreateOrderRequest) error {
    sagaData := &OrderSagaData{
        OrderID:     uuid.New().String(),
        UserID:      req.UserID,
        Items:       req.Items,
        TotalAmount: req.TotalAmount,
    }
    
    steps := []SagaStep{
        {
            Name: "create_order",
            Execute: func(ctx context.Context, data interface{}) error {
                d := data.(*OrderSagaData)
                order, err := os.orderService.CreateOrder(ctx, &Order{
                    ID:          d.OrderID,
                    UserID:      d.UserID,
                    Items:       d.Items,
                    TotalAmount: d.TotalAmount,
                    Status:      "pending",
                })
                if err != nil {
                    return err
                }
                d.OrderID = order.ID
                return nil
            },
            Compensate: func(ctx context.Context, data interface{}) error {
                d := data.(*OrderSagaData)
                return os.orderService.CancelOrder(ctx, d.OrderID)
            },
        },
        {
            Name: "reserve_inventory",
            Execute: func(ctx context.Context, data interface{}) error {
                d := data.(*OrderSagaData)
                for _, item := range d.Items {
                    if err := os.inventoryService.ReserveStock(ctx, item.ProductID, item.Quantity); err != nil {
                        return fmt.Errorf("failed to reserve %s: %w", item.ProductID, err)
                    }
                }
                return nil
            },
            Compensate: func(ctx context.Context, data interface{}) error {
                d := data.(*OrderSagaData)
                for _, item := range d.Items {
                    os.inventoryService.ReleaseReservation(ctx, item.ProductID, item.Quantity)
                }
                return nil
            },
        },
        {
            Name: "process_payment",
            Execute: func(ctx context.Context, data interface{}) error {
                d := data.(*OrderSagaData)
                payment, err := os.paymentService.ProcessPayment(ctx, &PaymentRequest{
                    OrderID:        d.OrderID,
                    Amount:         d.TotalAmount,
                    Currency:       "USD",
                    IdempotencyKey: d.OrderID,
                })
                if err != nil {
                    return fmt.Errorf("payment failed: %w", err)
                }
                d.PaymentID = payment.ID
                return nil
            },
            Compensate: func(ctx context.Context, data interface{}) error {
                d := data.(*OrderSagaData)
                if d.PaymentID != "" {
                    return os.paymentService.RefundPayment(ctx, d.PaymentID, d.TotalAmount)
                }
                return nil
            },
        },
        {
            Name: "commit_inventory",
            Execute: func(ctx context.Context, data interface{}) error {
                d := data.(*OrderSagaData)
                for _, item := range d.Items {
                    if err := os.inventoryService.CommitReservation(ctx, item.ProductID, item.Quantity); err != nil {
                        return fmt.Errorf("failed to commit inventory: %w", err)
                    }
                }
                return nil
            },
            Compensate: func(ctx context.Context, data interface{}) error {
                // Cannot uncommit - would need to add back to inventory
                return nil
            },
        },
    }
    
    // Execute saga
    executedSteps := []SagaStep{}
    
    for _, step := range steps {
        log.Printf("Executing saga step: %s", step.Name)
        
        if err := step.Execute(ctx, sagaData); err != nil {
            log.Printf("Saga step %s failed: %v", step.Name, err)
            
            // Compensate in reverse order
            for i := len(executedSteps) - 1; i >= 0; i-- {
                log.Printf("Compensating step: %s", executedSteps[i].Name)
                if compErr := executedSteps[i].Compensate(ctx, sagaData); compErr != nil {
                    log.Printf("Compensation failed for step %s: %v", executedSteps[i].Name, compErr)
                }
            }
            
            return fmt.Errorf("saga failed at step %s: %w", step.Name, err)
        }
        
        executedSteps = append(executedSteps, step)
    }
    
    log.Printf("Saga completed successfully for order: %s", sagaData.OrderID)
    return nil
}
```

---

## 3. Background Job Processor

### Job Queue with Redis

```go
// ✅ GOOD: Redis-based job queue with priorities and retries
type JobQueue struct {
    redis *redis.Client
}

type Job struct {
    ID          string                 `json:"id"`
    Type        string                 `json:"type"`
    Payload     map[string]interface{} `json:"payload"`
    Priority    int                    `json:"priority"`
    MaxRetries  int                    `json:"max_retries"`
    Attempts    int                    `json:"attempts"`
    CreatedAt   time.Time              `json:"created_at"`
    ScheduledAt time.Time              `json:"scheduled_at"`
    LastError   string                 `json:"last_error,omitempty"`
}

func (jq *JobQueue) Enqueue(ctx context.Context, job *Job) error {
    if job.ID == "" {
        job.ID = uuid.New().String()
    }
    job.CreatedAt = time.Now()
    
    if job.ScheduledAt.IsZero() {
        job.ScheduledAt = time.Now()
    }
    
    data, err := json.Marshal(job)
    if err != nil {
        return fmt.Errorf("failed to marshal job: %w", err)
    }
    
    // Add to sorted set with priority and scheduled time
    // Score = scheduled_timestamp * 1000 - priority (lower score = higher priority)
    score := float64(job.ScheduledAt.Unix()*1000 - int64(job.Priority))
    
    queueKey := fmt.Sprintf("queue:priority:%d", job.Priority)
    return jq.redis.ZAdd(ctx, queueKey, redis.Z{
        Score:  score,
        Member: data,
    }).Err()
}

func (jq *JobQueue) Dequeue(ctx context.Context, priorities []int) (*Job, error) {
    now := time.Now().Unix() * 1000
    
    // Check queues in priority order (highest priority first)
    for _, priority := range priorities {
        queueKey := fmt.Sprintf("queue:priority:%d", priority)
        
        // Get jobs ready to run (score <= now)
        results, err := jq.redis.ZRangeByScore(ctx, queueKey, &redis.ZRangeBy{
            Min:    "-inf",
            Max:    fmt.Sprintf("%d", now),
            Offset: 0,
            Count:  1,
        }).Result()
        
        if err != nil || len(results) == 0 {
            continue
        }
        
        // Remove from queue atomically
        removed, err := jq.redis.ZRem(ctx, queueKey, results[0]).Result()
        if err != nil || removed == 0 {
            continue
        }
        
        var job Job
        if err := json.Unmarshal([]byte(results[0]), &job); err != nil {
            log.Printf("Failed to unmarshal job: %v", err)
            continue
        }
        
        return &job, nil
    }
    
    return nil, nil
}

func (jq *JobQueue) Retry(ctx context.Context, job *Job, err error, delay time.Duration) error {
    job.Attempts++
    job.LastError = err.Error()
    
    if job.Attempts >= job.MaxRetries {
        // Move to dead letter queue
        return jq.moveToDeadLetter(ctx, job)
    }
    
    // Re-enqueue with delay
    job.ScheduledAt = time.Now().Add(delay)
    return jq.Enqueue(ctx, job)
}

func (jq *JobQueue) moveToDeadLetter(ctx context.Context, job *Job) error {
    data, _ := json.Marshal(job)
    return jq.redis.LPush(ctx, "queue:dead_letter", data).Err()
}

func (jq *JobQueue) GetQueueStats(ctx context.Context) (map[string]int64, error) {
    stats := make(map[string]int64)
    
    priorities := []int{0, 1, 2, 3}
    for _, priority := range priorities {
        queueKey := fmt.Sprintf("queue:priority:%d", priority)
        count, err := jq.redis.ZCard(ctx, queueKey).Result()
        if err != nil {
            return nil, err
        }
        stats[queueKey] = count
    }
    
    deadLetterCount, _ := jq.redis.LLen(ctx, "queue:dead_letter").Result()
    stats["dead_letter"] = deadLetterCount
    
    return stats, nil
}
```

### Worker Pool with Graceful Shutdown

```go
// ✅ GOOD: Worker pool with graceful shutdown
type WorkerPool struct {
    queue       *JobQueue
    handlers    map[string]JobHandler
    workerCount int
    wg          sync.WaitGroup
    ctx         context.Context
    cancel      context.CancelFunc
}

type JobHandler func(ctx context.Context, job *Job) error

func NewWorkerPool(queue *JobQueue, workerCount int) *WorkerPool {
    ctx, cancel := context.WithCancel(context.Background())
    
    return &WorkerPool{
        queue:       queue,
        handlers:    make(map[string]JobHandler),
        workerCount: workerCount,
        ctx:         ctx,
        cancel:      cancel,
    }
}

func (wp *WorkerPool) RegisterHandler(jobType string, handler JobHandler) {
    wp.handlers[jobType] = handler
}

func (wp *WorkerPool) Start() {
    priorities := []int{3, 2, 1, 0} // High to low priority
    
    for i := 0; i < wp.workerCount; i++ {
        wp.wg.Add(1)
        go wp.worker(i, priorities)
    }
    
    log.Printf("Started %d workers", wp.workerCount)
}

func (wp *WorkerPool) worker(id int, priorities []int) {
    defer wp.wg.Done()
    
    log.Printf("Worker %d started", id)
    
    for {
        select {
        case <-wp.ctx.Done():
            log.Printf("Worker %d shutting down", id)
            return
        default:
            job, err := wp.queue.Dequeue(wp.ctx, priorities)
            if err != nil {
                log.Printf("Worker %d: failed to dequeue: %v", id, err)
                time.Sleep(1 * time.Second)
                continue
            }
            
            if job == nil {
                // No jobs available, wait before polling again
                time.Sleep(100 * time.Millisecond)
                continue
            }
            
            wp.processJob(id, job)
        }
    }
}

func (wp *WorkerPool) processJob(workerID int, job *Job) {
    log.Printf("Worker %d processing job %s (type: %s, attempt: %d)", 
        workerID, job.ID, job.Type, job.Attempts+1)
    
    handler, exists := wp.handlers[job.Type]
    if !exists {
        log.Printf("No handler for job type: %s", job.Type)
        wp.queue.moveToDeadLetter(wp.ctx, job)
        return
    }
    
    // Create timeout context for job execution
    ctx, cancel := context.WithTimeout(wp.ctx, 5*time.Minute)
    defer cancel()
    
    // Execute job
    err := handler(ctx, job)
    
    if err != nil {
        log.Printf("Worker %d: job %s failed: %v", workerID, job.ID, err)
        
        // Calculate retry delay with exponential backoff
        delay := time.Duration(math.Pow(2, float64(job.Attempts))) * time.Second
        if delay > 1*time.Hour {
            delay = 1 * time.Hour
        }
        
        if retryErr := wp.queue.Retry(wp.ctx, job, err, delay); retryErr != nil {
            log.Printf("Failed to retry job: %v", retryErr)
        }
    } else {
        log.Printf("Worker %d: job %s completed successfully", workerID, job.ID)
    }
}

func (wp *WorkerPool) Shutdown(timeout time.Duration) error {
    log.Println("Shutting down worker pool...")
    
    // Signal workers to stop
    wp.cancel()
    
    // Wait for workers to finish with timeout
    done := make(chan struct{})
    go func() {
        wp.wg.Wait()
        close(done)
    }()
    
    select {
    case <-done:
        log.Println("All workers shut down gracefully")
        return nil
    case <-time.After(timeout):
        return errors.New("shutdown timeout exceeded")
    }
}
```

### Example Job Handlers

```go
// ✅ GOOD: Example job handlers
func SendEmailHandler(ctx context.Context, job *Job) error {
    to := job.Payload["to"].(string)
    subject := job.Payload["subject"].(string)
    body := job.Payload["body"].(string)
    
    // Send email
    return sendEmail(ctx, to, subject, body)
}

func ProcessImageHandler(ctx context.Context, job *Job) error {
    imageURL := job.Payload["image_url"].(string)
    
    // Download image
    img, err := downloadImage(ctx, imageURL)
    if err != nil {
        return fmt.Errorf("failed to download image: %w", err)
    }
    
    // Process image (resize, optimize, etc.)
    processed, err := processImage(img)
    if err != nil {
        return fmt.Errorf("failed to process image: %w", err)
    }
    
    // Upload processed image
    newURL, err := uploadImage(ctx, processed)
    if err != nil {
        return fmt.Errorf("failed to upload image: %w", err)
    }
    
    log.Printf("Image processed: %s -> %s", imageURL, newURL)
    return nil
}

func GenerateReportHandler(ctx context.Context, job *Job) error {
    reportType := job.Payload["report_type"].(string)
    startDate := job.Payload["start_date"].(string)
    endDate := job.Payload["end_date"].(string)
    
    // Generate report
    report, err := generateReport(ctx, reportType, startDate, endDate)
    if err != nil {
        return fmt.Errorf("failed to generate report: %w", err)
    }
    
    // Store report
    reportURL, err := storeReport(ctx, report)
    if err != nil {
        return fmt.Errorf("failed to store report: %w", err)
    }
    
    // Notify user
    userID := job.Payload["user_id"].(string)
    notifyUser(ctx, userID, reportURL)
    
    return nil
}
```

---

## Summary

This document covers:

1. **Payment Workflows**: Idempotent processing, retries, webhook handling, refunds
2. **Order Management**: State machines, inventory management, saga pattern
3. **Background Jobs**: Redis-based queue, worker pool, retry logic, graceful shutdown

## References

1. **Stripe API Documentation**: https://stripe.com/docs/api
2. **Event Sourcing**: https://martinfowler.com/eaaDev/EventSourcing.html
3. **Saga Pattern**: https://microservices.io/patterns/data/saga.html
4. **Background Jobs**: https://github.com/hibiken/asynq

---

**Last Updated**: January 2026  
**Version**: 1.0
