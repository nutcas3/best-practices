# API Design Patterns - REST, GraphQL & Distributed Systems

## Skill Metadata
- **Domain**: API Design, REST, GraphQL, Microservices
- **Skill Level**: Advanced
- **Last Updated**: 2026
- **Sources**: OpenAPI 3.0, REST API Guidelines, GraphQL Best Practices

## Overview

Production-ready patterns for designing robust APIs including REST, GraphQL, versioning, pagination, and distributed systems architecture.

---

## 1. RESTful API Design

### Resource Naming Conventions

```go
// ✅ GOOD: RESTful resource naming
// Collections (plural nouns)
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/{id}
PUT    /api/v1/users/{id}
PATCH  /api/v1/users/{id}
DELETE /api/v1/users/{id}

// Nested resources
GET    /api/v1/users/{id}/orders
POST   /api/v1/users/{id}/orders
GET    /api/v1/users/{id}/orders/{orderId}

// Actions (use POST for non-CRUD operations)
POST   /api/v1/users/{id}/activate
POST   /api/v1/orders/{id}/cancel
POST   /api/v1/payments/{id}/refund

// ❌ BAD: Avoid verbs in URLs
GET    /api/v1/getUser/{id}
POST   /api/v1/createUser
DELETE /api/v1/deleteUser/{id}
```

### HTTP Status Codes

```go
// ✅ GOOD: Proper status code usage
func (h *Handler) CreateUser(c *fiber.Ctx) error {
    var req CreateUserRequest
    
    // 400 Bad Request - Invalid input
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{
            "error": "Invalid request body",
            "details": err.Error(),
        })
    }
    
    // 422 Unprocessable Entity - Validation errors
    if errors := h.validator.Validate(req); errors != nil {
        return c.Status(422).JSON(fiber.Map{
            "error": "Validation failed",
            "fields": errors,
        })
    }
    
    // 409 Conflict - Resource already exists
    if h.db.UserExists(req.Email) {
        return c.Status(409).JSON(fiber.Map{
            "error": "User already exists",
        })
    }
    
    user, err := h.service.CreateUser(req)
    if err != nil {
        // 500 Internal Server Error
        return c.Status(500).JSON(fiber.Map{
            "error": "Failed to create user",
        })
    }
    
    // 201 Created - Resource created successfully
    return c.Status(201).
        Header("Location", fmt.Sprintf("/api/v1/users/%s", user.ID)).
        JSON(user)
}

func (h *Handler) GetUser(c *fiber.Ctx) error {
    userID := c.Params("id")
    
    user, err := h.service.GetUser(userID)
    if err != nil {
        // 404 Not Found
        return c.Status(404).JSON(fiber.Map{
            "error": "User not found",
        })
    }
    
    // 200 OK
    return c.JSON(user)
}

func (h *Handler) UpdateUser(c *fiber.Ctx) error {
    // 401 Unauthorized - Not authenticated
    if !c.Locals("authenticated").(bool) {
        return c.Status(401).JSON(fiber.Map{
            "error": "Authentication required",
        })
    }
    
    // 403 Forbidden - Authenticated but not authorized
    if !h.canUpdate(c) {
        return c.Status(403).JSON(fiber.Map{
            "error": "Insufficient permissions",
        })
    }
    
    // 204 No Content - Success with no response body
    return c.SendStatus(204)
}
```

### Pagination Patterns

```go
// ✅ GOOD: Cursor-based pagination (recommended for large datasets)
type PaginationResponse struct {
    Data       []interface{} `json:"data"`
    NextCursor *string       `json:"next_cursor,omitempty"`
    PrevCursor *string       `json:"prev_cursor,omitempty"`
    HasMore    bool          `json:"has_more"`
}

func (h *Handler) ListUsers(c *fiber.Ctx) error {
    cursor := c.Query("cursor", "")
    limit := c.QueryInt("limit", 20)
    
    if limit > 100 {
        limit = 100 // Max limit
    }
    
    users, nextCursor, err := h.service.ListUsers(cursor, limit)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to fetch users"})
    }
    
    response := PaginationResponse{
        Data:       users,
        NextCursor: nextCursor,
        HasMore:    nextCursor != nil,
    }
    
    return c.JSON(response)
}

// ✅ GOOD: Offset-based pagination (simpler, but less efficient)
type OffsetPaginationResponse struct {
    Data       []interface{} `json:"data"`
    Page       int           `json:"page"`
    PerPage    int           `json:"per_page"`
    Total      int64         `json:"total"`
    TotalPages int           `json:"total_pages"`
}

func (h *Handler) ListUsersOffset(c *fiber.Ctx) error {
    page := c.QueryInt("page", 1)
    perPage := c.QueryInt("per_page", 20)
    
    if perPage > 100 {
        perPage = 100
    }
    
    users, total, err := h.service.ListUsersOffset(page, perPage)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to fetch users"})
    }
    
    response := OffsetPaginationResponse{
        Data:       users,
        Page:       page,
        PerPage:    perPage,
        Total:      total,
        TotalPages: int(math.Ceil(float64(total) / float64(perPage))),
    }
    
    return c.JSON(response)
}
```

### Filtering and Sorting

```go
// ✅ GOOD: Query parameter filtering
func (h *Handler) ListProducts(c *fiber.Ctx) error {
    filters := ProductFilters{
        Category:  c.Query("category"),
        MinPrice:  c.QueryFloat("min_price", 0),
        MaxPrice:  c.QueryFloat("max_price", 0),
        InStock:   c.QueryBool("in_stock", false),
        SortBy:    c.Query("sort_by", "created_at"),
        SortOrder: c.Query("sort_order", "desc"),
    }
    
    // Validate sort fields
    allowedSortFields := []string{"created_at", "price", "name", "popularity"}
    if !contains(allowedSortFields, filters.SortBy) {
        return c.Status(400).JSON(fiber.Map{
            "error": "Invalid sort field",
        })
    }
    
    products, err := h.service.ListProducts(filters)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to fetch products"})
    }
    
    return c.JSON(products)
}
```

### API Versioning

```go
// ✅ GOOD: URL versioning (recommended)
app.Group("/api/v1", func(v1 fiber.Router) {
    v1.Get("/users", handler.ListUsersV1)
    v1.Post("/users", handler.CreateUserV1)
})

app.Group("/api/v2", func(v2 fiber.Router) {
    v2.Get("/users", handler.ListUsersV2)
    v2.Post("/users", handler.CreateUserV2)
})

// ✅ GOOD: Header versioning (alternative)
func VersionMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        version := c.Get("API-Version", "v1")
        c.Locals("api_version", version)
        return c.Next()
    }
}

// ✅ GOOD: Deprecation headers
func DeprecationMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        if strings.HasPrefix(c.Path(), "/api/v1") {
            c.Set("Deprecation", "true")
            c.Set("Sunset", "2026-12-31")
            c.Set("Link", `</api/v2>; rel="successor-version"`)
        }
        return c.Next()
    }
}
```

### HATEOAS (Hypermedia)

```go
// ✅ GOOD: Including links in responses
type UserResponse struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
    Links Links  `json:"_links"`
}

type Links struct {
    Self    Link `json:"self"`
    Orders  Link `json:"orders,omitempty"`
    Profile Link `json:"profile,omitempty"`
}

type Link struct {
    Href   string `json:"href"`
    Method string `json:"method,omitempty"`
}

func (h *Handler) GetUser(c *fiber.Ctx) error {
    user, _ := h.service.GetUser(c.Params("id"))
    
    response := UserResponse{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
        Links: Links{
            Self: Link{
                Href:   fmt.Sprintf("/api/v1/users/%s", user.ID),
                Method: "GET",
            },
            Orders: Link{
                Href:   fmt.Sprintf("/api/v1/users/%s/orders", user.ID),
                Method: "GET",
            },
            Profile: Link{
                Href:   fmt.Sprintf("/api/v1/users/%s/profile", user.ID),
                Method: "GET",
            },
        },
    }
    
    return c.JSON(response)
}
```

---

## 2. GraphQL API Design

### Schema Design

```graphql
# ✅ GOOD: Well-structured GraphQL schema
type Query {
  user(id: ID!): User
  users(
    first: Int
    after: String
    filter: UserFilter
    orderBy: UserOrderBy
  ): UserConnection!
  
  product(id: ID!): Product
  products(
    first: Int
    after: String
    filter: ProductFilter
  ): ProductConnection!
}

type Mutation {
  createUser(input: CreateUserInput!): CreateUserPayload!
  updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
  deleteUser(id: ID!): DeleteUserPayload!
}

type Subscription {
  userUpdated(id: ID!): User!
  orderStatusChanged(orderId: ID!): Order!
}

type User {
  id: ID!
  email: String!
  name: String!
  createdAt: DateTime!
  orders(first: Int, after: String): OrderConnection!
}

type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

input UserFilter {
  email: String
  nameContains: String
  createdAfter: DateTime
}

input UserOrderBy {
  field: UserOrderField!
  direction: OrderDirection!
}

enum UserOrderField {
  CREATED_AT
  NAME
  EMAIL
}

enum OrderDirection {
  ASC
  DESC
}
```

### GraphQL Resolver (Go)

```go
// ✅ GOOD: GraphQL resolver implementation
import "github.com/graphql-go/graphql"

type Resolver struct {
    userService    *UserService
    productService *ProductService
}

func (r *Resolver) UserResolver() *graphql.Field {
    return &graphql.Field{
        Type: userType,
        Args: graphql.FieldConfigArgument{
            "id": &graphql.ArgumentConfig{
                Type: graphql.NewNonNull(graphql.ID),
            },
        },
        Resolve: func(p graphql.ResolveParams) (interface{}, error) {
            id := p.Args["id"].(string)
            return r.userService.GetUser(p.Context, id)
        },
    }
}

func (r *Resolver) UsersResolver() *graphql.Field {
    return &graphql.Field{
        Type: userConnectionType,
        Args: graphql.FieldConfigArgument{
            "first": &graphql.ArgumentConfig{
                Type:         graphql.Int,
                DefaultValue: 20,
            },
            "after": &graphql.ArgumentConfig{
                Type: graphql.String,
            },
            "filter": &graphql.ArgumentConfig{
                Type: userFilterInputType,
            },
        },
        Resolve: func(p graphql.ResolveParams) (interface{}, error) {
            first := p.Args["first"].(int)
            after, _ := p.Args["after"].(string)
            
            var filter *UserFilter
            if f, ok := p.Args["filter"]; ok {
                filter = parseUserFilter(f)
            }
            
            return r.userService.ListUsers(p.Context, first, after, filter)
        },
    }
}

// ✅ GOOD: DataLoader to prevent N+1 queries
type UserLoader struct {
    userService *UserService
    loader      *dataloader.Loader
}

func NewUserLoader(userService *UserService) *UserLoader {
    batchFn := func(ctx context.Context, keys dataloader.Keys) []*dataloader.Result {
        userIDs := keys.Keys()
        users, err := userService.GetUsersByIDs(ctx, userIDs)
        
        results := make([]*dataloader.Result, len(keys))
        userMap := make(map[string]*User)
        
        for _, user := range users {
            userMap[user.ID] = user
        }
        
        for i, key := range userIDs {
            if user, ok := userMap[key]; ok {
                results[i] = &dataloader.Result{Data: user}
            } else {
                results[i] = &dataloader.Result{Error: errors.New("user not found")}
            }
        }
        
        return results
    }
    
    return &UserLoader{
        userService: userService,
        loader:      dataloader.NewBatchedLoader(batchFn),
    }
}
```

---

## 3. OpenAPI 3.0 Specification

```yaml
# ✅ GOOD: Comprehensive OpenAPI spec
openapi: 3.0.3
info:
  title: User Management API
  version: 1.0.0
  description: API for managing users and their resources
  contact:
    name: API Support
    email: support@example.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging-api.example.com/v1
    description: Staging server

security:
  - bearerAuth: []

paths:
  /users:
    get:
      summary: List users
      operationId: listUsers
      tags:
        - Users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
            minimum: 1
        - name: per_page
          in: query
          schema:
            type: integer
            default: 20
            minimum: 1
            maximum: 100
        - name: sort_by
          in: query
          schema:
            type: string
            enum: [created_at, name, email]
            default: created_at
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserListResponse'
        '401':
          $ref: '#/components/responses/UnauthorizedError'
        '500':
          $ref: '#/components/responses/InternalServerError'
    
    post:
      summary: Create a new user
      operationId: createUser
      tags:
        - Users
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created successfully
          headers:
            Location:
              schema:
                type: string
              description: URL of the created user
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/BadRequestError'
        '422':
          $ref: '#/components/responses/ValidationError'

  /users/{userId}:
    parameters:
      - name: userId
        in: path
        required: true
        schema:
          type: string
          format: uuid
    
    get:
      summary: Get user by ID
      operationId: getUser
      tags:
        - Users
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFoundError'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    User:
      type: object
      required:
        - id
        - email
        - name
        - created_at
      properties:
        id:
          type: string
          format: uuid
          example: "123e4567-e89b-12d3-a456-426614174000"
        email:
          type: string
          format: email
          example: "user@example.com"
        name:
          type: string
          minLength: 2
          maxLength: 100
          example: "John Doe"
        created_at:
          type: string
          format: date-time
          example: "2024-01-15T10:30:00Z"
    
    CreateUserRequest:
      type: object
      required:
        - email
        - name
        - password
      properties:
        email:
          type: string
          format: email
        name:
          type: string
          minLength: 2
          maxLength: 100
        password:
          type: string
          format: password
          minLength: 8
    
    Error:
      type: object
      required:
        - error
      properties:
        error:
          type: string
        details:
          type: object

  responses:
    UnauthorizedError:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    NotFoundError:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    ValidationError:
      description: Validation failed
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: string
              fields:
                type: object
                additionalProperties:
                  type: string
```

---

## 4. Microservices Patterns

### Service Discovery

```go
// ✅ GOOD: Service registry with health checks
type ServiceRegistry struct {
    consul *consulapi.Client
}

func (sr *ServiceRegistry) Register(service *Service) error {
    registration := &consulapi.AgentServiceRegistration{
        ID:      service.ID,
        Name:    service.Name,
        Address: service.Address,
        Port:    service.Port,
        Tags:    service.Tags,
        Check: &consulapi.AgentServiceCheck{
            HTTP:                           fmt.Sprintf("http://%s:%d/health", service.Address, service.Port),
            Interval:                       "10s",
            Timeout:                        "3s",
            DeregisterCriticalServiceAfter: "30s",
        },
    }
    
    return sr.consul.Agent().ServiceRegister(registration)
}

func (sr *ServiceRegistry) Discover(serviceName string) ([]*Service, error) {
    services, _, err := sr.consul.Health().Service(serviceName, "", true, nil)
    if err != nil {
        return nil, err
    }
    
    result := make([]*Service, len(services))
    for i, service := range services {
        result[i] = &Service{
            ID:      service.Service.ID,
            Name:    service.Service.Service,
            Address: service.Service.Address,
            Port:    service.Service.Port,
        }
    }
    
    return result, nil
}
```

### API Gateway Pattern

```go
// ✅ GOOD: API Gateway with routing and load balancing
type APIGateway struct {
    router   *fiber.App
    registry *ServiceRegistry
    cache    *redis.Client
}

func (gw *APIGateway) ProxyRequest(c *fiber.Ctx) error {
    serviceName := c.Params("service")
    
    // Get service instances
    services, err := gw.registry.Discover(serviceName)
    if err != nil || len(services) == 0 {
        return c.Status(503).JSON(fiber.Map{
            "error": "Service unavailable",
        })
    }
    
    // Load balancing - round robin
    service := services[rand.Intn(len(services))]
    
    // Build target URL
    targetURL := fmt.Sprintf("http://%s:%d%s", 
        service.Address, 
        service.Port, 
        c.Path(),
    )
    
    // Forward request
    req := fasthttp.AcquireRequest()
    resp := fasthttp.AcquireResponse()
    defer fasthttp.ReleaseRequest(req)
    defer fasthttp.ReleaseResponse(resp)
    
    req.SetRequestURI(targetURL)
    req.Header.SetMethod(c.Method())
    req.SetBody(c.Body())
    
    // Copy headers
    c.Request().Header.VisitAll(func(key, value []byte) {
        req.Header.SetBytesKV(key, value)
    })
    
    // Make request with timeout
    client := &fasthttp.Client{
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 5 * time.Second,
    }
    
    if err := client.Do(req, resp); err != nil {
        return c.Status(502).JSON(fiber.Map{
            "error": "Bad gateway",
        })
    }
    
    // Copy response
    c.Status(resp.StatusCode())
    resp.Header.VisitAll(func(key, value []byte) {
        c.Set(string(key), string(value))
    })
    
    return c.Send(resp.Body())
}
```

### Circuit Breaker

```go
// ✅ GOOD: Circuit breaker implementation
import "github.com/sony/gobreaker"

type ServiceClient struct {
    breaker *gobreaker.CircuitBreaker
    client  *http.Client
}

func NewServiceClient(serviceName string) *ServiceClient {
    settings := gobreaker.Settings{
        Name:        serviceName,
        MaxRequests: 3,
        Interval:    time.Minute,
        Timeout:     30 * time.Second,
        ReadyToTrip: func(counts gobreaker.Counts) bool {
            failureRatio := float64(counts.TotalFailures) / float64(counts.Requests)
            return counts.Requests >= 3 && failureRatio >= 0.6
        },
        OnStateChange: func(name string, from gobreaker.State, to gobreaker.State) {
            log.Printf("Circuit breaker %s: %s -> %s", name, from, to)
        },
    }
    
    return &ServiceClient{
        breaker: gobreaker.NewCircuitBreaker(settings),
        client:  &http.Client{Timeout: 5 * time.Second},
    }
}

func (sc *ServiceClient) Call(url string) ([]byte, error) {
    result, err := sc.breaker.Execute(func() (interface{}, error) {
        resp, err := sc.client.Get(url)
        if err != nil {
            return nil, err
        }
        defer resp.Body.Close()
        
        if resp.StatusCode >= 500 {
            return nil, errors.New("server error")
        }
        
        return io.ReadAll(resp.Body)
    })
    
    if err != nil {
        return nil, err
    }
    
    return result.([]byte), nil
}
```

---

## 5. Event-Driven Architecture

### Event Bus with Kafka

```go
// ✅ GOOD: Kafka producer
import "github.com/segmentio/kafka-go"

type EventProducer struct {
    writer *kafka.Writer
}

func NewEventProducer(brokers []string, topic string) *EventProducer {
    return &EventProducer{
        writer: &kafka.Writer{
            Addr:         kafka.TCP(brokers...),
            Topic:        topic,
            Balancer:     &kafka.LeastBytes{},
            RequiredAcks: kafka.RequireAll,
            Compression:  kafka.Snappy,
        },
    }
}

func (ep *EventProducer) Publish(ctx context.Context, event *Event) error {
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }
    
    return ep.writer.WriteMessages(ctx, kafka.Message{
        Key:   []byte(event.AggregateID),
        Value: data,
        Headers: []kafka.Header{
            {Key: "event-type", Value: []byte(event.Type)},
            {Key: "event-version", Value: []byte(event.Version)},
        },
    })
}

// ✅ GOOD: Kafka consumer
type EventConsumer struct {
    reader  *kafka.Reader
    handler EventHandler
}

func NewEventConsumer(brokers []string, topic, groupID string, handler EventHandler) *EventConsumer {
    return &EventConsumer{
        reader: kafka.NewReader(kafka.ReaderConfig{
            Brokers:        brokers,
            Topic:          topic,
            GroupID:        groupID,
            MinBytes:       10e3,
            MaxBytes:       10e6,
            CommitInterval: time.Second,
        }),
        handler: handler,
    }
}

func (ec *EventConsumer) Start(ctx context.Context) error {
    for {
        msg, err := ec.reader.ReadMessage(ctx)
        if err != nil {
            return err
        }
        
        var event Event
        if err := json.Unmarshal(msg.Value, &event); err != nil {
            log.Printf("Failed to unmarshal event: %v", err)
            continue
        }
        
        if err := ec.handler.Handle(ctx, &event); err != nil {
            log.Printf("Failed to handle event: %v", err)
            // Implement retry logic or dead letter queue
        }
    }
}
```

---

## Summary

This document covers:

1. **RESTful API Design**: Resource naming, status codes, pagination, filtering, versioning, HATEOAS
2. **GraphQL**: Schema design, resolvers, DataLoader for N+1 prevention
3. **OpenAPI 3.0**: Comprehensive API specification
4. **Microservices**: Service discovery, API gateway, circuit breaker
5. **Event-Driven**: Kafka producer/consumer patterns

## References

1. **REST API Guidelines**: https://restfulapi.net/
2. **GraphQL Best Practices**: https://graphql.org/learn/best-practices/
3. **OpenAPI Specification**: https://swagger.io/specification/
4. **Microservices Patterns**: https://microservices.io/patterns/

---

**Last Updated**: January 2026  
**Version**: 1.0
