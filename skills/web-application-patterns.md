# Web Application Patterns - Authentication, Sessions & State Management

## Skill Metadata
- **Domain**: Web Development, Authentication, Session Management
- **Skill Level**: Advanced
- **Last Updated**: 2026
- **Sources**: OAuth 2.0 RFC, JWT Best Practices, OWASP Session Management

## Overview

This document provides production-ready patterns for building secure web applications with proper authentication flows, session management, and state handling in Go and Rust.

---

## 1. Authentication Flows

### Session-Based Authentication (Go)

```go
// ✅ GOOD: Secure session-based authentication
import (
    "crypto/rand"
    "encoding/base64"
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/session"
    "time"
)

type SessionStore struct {
    store *session.Store
}

func NewSessionStore() *SessionStore {
    return &SessionStore{
        store: session.New(session.Config{
            Expiration:     24 * time.Hour,
            CookieHTTPOnly: true,
            CookieSecure:   true,
            CookieSameSite: "Lax",
            KeyGenerator: func() string {
                b := make([]byte, 32)
                rand.Read(b)
                return base64.URLEncoding.EncodeToString(b)
            },
        }),
    }
}

func (s *SessionStore) Login(c *fiber.Ctx, userID string) error {
    sess, err := s.store.Get(c)
    if err != nil {
        return err
    }
    
    sess.Set("user_id", userID)
    sess.Set("authenticated", true)
    sess.Set("created_at", time.Now())
    
    return sess.Save()
}

func (s *SessionStore) Logout(c *fiber.Ctx) error {
    sess, err := s.store.Get(c)
    if err != nil {
        return err
    }
    
    return sess.Destroy()
}

func (s *SessionStore) GetUserID(c *fiber.Ctx) (string, error) {
    sess, err := s.store.Get(c)
    if err != nil {
        return "", err
    }
    
    userID := sess.Get("user_id")
    if userID == nil {
        return "", errors.New("not authenticated")
    }
    
    return userID.(string), nil
}

// ✅ GOOD: Session middleware with validation
func (s *SessionStore) RequireAuth() fiber.Handler {
    return func(c *fiber.Ctx) error {
        sess, err := s.store.Get(c)
        if err != nil {
            return c.Status(401).JSON(fiber.Map{"error": "Unauthorized"})
        }
        
        authenticated := sess.Get("authenticated")
        if authenticated == nil || !authenticated.(bool) {
            return c.Status(401).JSON(fiber.Map{"error": "Not authenticated"})
        }
        
        // Check session age
        createdAt := sess.Get("created_at")
        if createdAt != nil {
            created := createdAt.(time.Time)
            if time.Since(created) > 24*time.Hour {
                sess.Destroy()
                return c.Status(401).JSON(fiber.Map{"error": "Session expired"})
            }
        }
        
        return c.Next()
    }
}
```

### JWT Authentication with Refresh Tokens (Go)

```go
// ✅ GOOD: JWT with refresh token implementation
import (
    "github.com/golang-jwt/jwt/v5"
    "github.com/google/uuid"
)

type TokenPair struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
    ExpiresIn    int64  `json:"expires_in"`
}

type RefreshTokenStore struct {
    redis *redis.Client
}

func (ts *TokenService) GenerateTokenPair(userID, role string) (*TokenPair, error) {
    // Generate access token (short-lived)
    accessToken, err := ts.generateAccessToken(userID, role, 15*time.Minute)
    if err != nil {
        return nil, err
    }
    
    // Generate refresh token (long-lived)
    refreshToken := uuid.New().String()
    
    // Store refresh token in Redis with 7-day expiration
    err = ts.redis.Set(context.Background(), 
        "refresh:"+refreshToken, 
        userID, 
        7*24*time.Hour,
    ).Err()
    if err != nil {
        return nil, err
    }
    
    return &TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
        ExpiresIn:    int64(15 * 60), // 15 minutes in seconds
    }, nil
}

func (ts *TokenService) RefreshAccessToken(refreshToken string) (*TokenPair, error) {
    // Validate refresh token
    userID, err := ts.redis.Get(context.Background(), "refresh:"+refreshToken).Result()
    if err != nil {
        return nil, errors.New("invalid refresh token")
    }
    
    // Get user to check if still active
    user, err := ts.db.GetUser(userID)
    if err != nil {
        return nil, err
    }
    
    if !user.Active {
        // Revoke refresh token
        ts.redis.Del(context.Background(), "refresh:"+refreshToken)
        return nil, errors.New("user account disabled")
    }
    
    // Generate new token pair
    return ts.GenerateTokenPair(user.ID, user.Role)
}

func (ts *TokenService) RevokeRefreshToken(refreshToken string) error {
    return ts.redis.Del(context.Background(), "refresh:"+refreshToken).Err()
}

// ✅ GOOD: Refresh token endpoint
func (h *AuthHandler) RefreshToken(c *fiber.Ctx) error {
    var req struct {
        RefreshToken string `json:"refresh_token" validate:"required"`
    }
    
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
    }
    
    tokens, err := h.tokenService.RefreshAccessToken(req.RefreshToken)
    if err != nil {
        return c.Status(401).JSON(fiber.Map{"error": "Invalid refresh token"})
    }
    
    return c.JSON(tokens)
}
```

### OAuth 2.0 Implementation (Go)

```go
// ✅ GOOD: OAuth 2.0 Authorization Code Flow
import (
    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

type OAuthConfig struct {
    GoogleConfig *oauth2.Config
    GithubConfig *oauth2.Config
}

func NewOAuthConfig() *OAuthConfig {
    return &OAuthConfig{
        GoogleConfig: &oauth2.Config{
            ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
            ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
            RedirectURL:  os.Getenv("GOOGLE_REDIRECT_URL"),
            Scopes: []string{
                "https://www.googleapis.com/auth/userinfo.email",
                "https://www.googleapis.com/auth/userinfo.profile",
            },
            Endpoint: google.Endpoint,
        },
        GithubConfig: &oauth2.Config{
            ClientID:     os.Getenv("GITHUB_CLIENT_ID"),
            ClientSecret: os.Getenv("GITHUB_CLIENT_SECRET"),
            RedirectURL:  os.Getenv("GITHUB_REDIRECT_URL"),
            Scopes:       []string{"user:email"},
            Endpoint: oauth2.Endpoint{
                AuthURL:  "https://github.com/login/oauth/authorize",
                TokenURL: "https://github.com/login/oauth/access_token",
            },
        },
    }
}

// ✅ GOOD: OAuth state management (CSRF protection)
type OAuthStateStore struct {
    redis *redis.Client
}

func (oss *OAuthStateStore) GenerateState(userSession string) (string, error) {
    state := uuid.New().String()
    
    // Store state with 10-minute expiration
    err := oss.redis.Set(context.Background(), 
        "oauth:state:"+state, 
        userSession, 
        10*time.Minute,
    ).Err()
    
    return state, err
}

func (oss *OAuthStateStore) ValidateState(state string) (string, error) {
    userSession, err := oss.redis.Get(context.Background(), "oauth:state:"+state).Result()
    if err != nil {
        return "", errors.New("invalid state")
    }
    
    // Delete state after use (one-time use)
    oss.redis.Del(context.Background(), "oauth:state:"+state)
    
    return userSession, nil
}

// ✅ GOOD: OAuth login handler
func (h *AuthHandler) GoogleLogin(c *fiber.Ctx) error {
    // Generate state for CSRF protection
    state, err := h.oauthState.GenerateState(c.Cookies("session_id"))
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to generate state"})
    }
    
    url := h.oauth.GoogleConfig.AuthCodeURL(state, oauth2.AccessTypeOffline)
    return c.Redirect(url)
}

// ✅ GOOD: OAuth callback handler
func (h *AuthHandler) GoogleCallback(c *fiber.Ctx) error {
    code := c.Query("code")
    state := c.Query("state")
    
    // Validate state
    _, err := h.oauthState.ValidateState(state)
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid state"})
    }
    
    // Exchange code for token
    token, err := h.oauth.GoogleConfig.Exchange(context.Background(), code)
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Failed to exchange token"})
    }
    
    // Get user info
    client := h.oauth.GoogleConfig.Client(context.Background(), token)
    resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo")
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to get user info"})
    }
    defer resp.Body.Close()
    
    var userInfo struct {
        ID      string `json:"id"`
        Email   string `json:"email"`
        Name    string `json:"name"`
        Picture string `json:"picture"`
    }
    
    if err := json.NewDecoder(resp.Body).Decode(&userInfo); err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to parse user info"})
    }
    
    // Find or create user
    user, err := h.userService.FindOrCreateOAuthUser(userInfo.Email, userInfo.Name, "google", userInfo.ID)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to create user"})
    }
    
    // Generate JWT tokens
    tokens, err := h.tokenService.GenerateTokenPair(user.ID, user.Role)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to generate tokens"})
    }
    
    return c.JSON(tokens)
}
```

### Rust Authentication Patterns

```rust
// ✅ GOOD: Session-based authentication in Rust
use actix_session::{Session, SessionMiddleware, storage::RedisSessionStore};
use actix_web::{web, HttpResponse, Result};
use actix_web::cookie::Key;

pub async fn login(
    session: Session,
    credentials: web::Json<LoginRequest>,
    user_service: web::Data<UserService>,
) -> Result<HttpResponse> {
    // Validate credentials
    let user = user_service
        .authenticate(&credentials.email, &credentials.password)
        .await
        .map_err(|_| actix_web::error::ErrorUnauthorized("Invalid credentials"))?;
    
    // Set session data
    session.insert("user_id", &user.id)?;
    session.insert("authenticated", true)?;
    session.insert("created_at", chrono::Utc::now().timestamp())?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "Login successful",
        "user": user
    })))
}

pub async fn logout(session: Session) -> Result<HttpResponse> {
    session.purge();
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "Logged out successfully"
    })))
}

// ✅ GOOD: Session middleware
pub async fn require_auth(
    session: Session,
    req: actix_web::HttpRequest,
    srv: actix_web::dev::Service<
        actix_web::dev::ServiceRequest,
        Response = actix_web::dev::ServiceResponse,
        Error = actix_web::Error,
    >,
) -> Result<actix_web::dev::ServiceResponse> {
    let authenticated = session.get::<bool>("authenticated")?;
    
    if authenticated != Some(true) {
        return Ok(req.into_response(
            HttpResponse::Unauthorized()
                .json(serde_json::json!({"error": "Not authenticated"}))
                .into_body()
        ));
    }
    
    // Check session age
    if let Some(created_at) = session.get::<i64>("created_at")? {
        let now = chrono::Utc::now().timestamp();
        if now - created_at > 86400 { // 24 hours
            session.purge();
            return Ok(req.into_response(
                HttpResponse::Unauthorized()
                    .json(serde_json::json!({"error": "Session expired"}))
                    .into_body()
            ));
        }
    }
    
    srv.call(req).await
}

// ✅ GOOD: JWT with refresh tokens in Rust
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,
    pub role: String,
    pub exp: usize,
    pub iat: usize,
}

pub struct TokenService {
    secret: String,
    redis: redis::Client,
}

impl TokenService {
    pub async fn generate_token_pair(&self, user_id: &str, role: &str) -> Result<TokenPair, Error> {
        // Generate access token (15 minutes)
        let access_token = self.generate_access_token(user_id, role, 900)?;
        
        // Generate refresh token
        let refresh_token = Uuid::new_v4().to_string();
        
        // Store refresh token in Redis (7 days)
        let mut conn = self.redis.get_async_connection().await?;
        redis::cmd("SETEX")
            .arg(format!("refresh:{}", refresh_token))
            .arg(604800) // 7 days
            .arg(user_id)
            .query_async(&mut conn)
            .await?;
        
        Ok(TokenPair {
            access_token,
            refresh_token,
            expires_in: 900,
        })
    }
    
    fn generate_access_token(&self, user_id: &str, role: &str, expires_in: i64) -> Result<String, Error> {
        let now = chrono::Utc::now().timestamp() as usize;
        let claims = Claims {
            sub: user_id.to_owned(),
            role: role.to_owned(),
            exp: now + expires_in as usize,
            iat: now,
        };
        
        encode(
            &Header::default(),
            &claims,
            &EncodingKey::from_secret(self.secret.as_ref()),
        )
        .map_err(|e| Error::TokenGeneration(e.to_string()))
    }
    
    pub async fn refresh_access_token(&self, refresh_token: &str) -> Result<TokenPair, Error> {
        // Validate refresh token
        let mut conn = self.redis.get_async_connection().await?;
        let user_id: String = redis::cmd("GET")
            .arg(format!("refresh:{}", refresh_token))
            .query_async(&mut conn)
            .await
            .map_err(|_| Error::InvalidRefreshToken)?;
        
        // Get user to check if still active
        let user = self.get_user(&user_id).await?;
        
        if !user.active {
            // Revoke refresh token
            redis::cmd("DEL")
                .arg(format!("refresh:{}", refresh_token))
                .query_async(&mut conn)
                .await?;
            return Err(Error::UserDisabled);
        }
        
        // Generate new token pair
        self.generate_token_pair(&user.id, &user.role).await
    }
}
```

---

## 2. Forms and Validation

### Server-Side Form Validation (Go)

```go
// ✅ GOOD: Comprehensive form validation
import "github.com/go-playground/validator/v10"

type RegisterForm struct {
    Email           string `json:"email" validate:"required,email"`
    Password        string `json:"password" validate:"required,min=8,max=72"`
    ConfirmPassword string `json:"confirm_password" validate:"required,eqfield=Password"`
    Name            string `json:"name" validate:"required,min=2,max=100"`
    Phone           string `json:"phone" validate:"omitempty,e164"`
    DateOfBirth     string `json:"date_of_birth" validate:"required,datetime=2006-01-02"`
    Terms           bool   `json:"terms" validate:"required,eq=true"`
}

type FormValidator struct {
    validate *validator.Validate
}

func NewFormValidator() *FormValidator {
    v := validator.New()
    
    // Register custom validators
    v.RegisterValidation("strong_password", validateStrongPassword)
    v.RegisterValidation("adult", validateAdult)
    
    return &FormValidator{validate: v}
}

func validateStrongPassword(fl validator.FieldLevel) bool {
    password := fl.Field().String()
    
    var hasUpper, hasLower, hasDigit, hasSpecial bool
    for _, char := range password {
        switch {
        case unicode.IsUpper(char):
            hasUpper = true
        case unicode.IsLower(char):
            hasLower = true
        case unicode.IsDigit(char):
            hasDigit = true
        case unicode.IsPunct(char) || unicode.IsSymbol(char):
            hasSpecial = true
        }
    }
    
    return hasUpper && hasLower && hasDigit && hasSpecial
}

func validateAdult(fl validator.FieldLevel) bool {
    dateStr := fl.Field().String()
    dob, err := time.Parse("2006-01-02", dateStr)
    if err != nil {
        return false
    }
    
    age := time.Now().Year() - dob.Year()
    return age >= 18
}

func (fv *FormValidator) ValidateStruct(s interface{}) map[string]string {
    errors := make(map[string]string)
    
    err := fv.validate.Struct(s)
    if err == nil {
        return nil
    }
    
    for _, err := range err.(validator.ValidationErrors) {
        field := err.Field()
        switch err.Tag() {
        case "required":
            errors[field] = fmt.Sprintf("%s is required", field)
        case "email":
            errors[field] = "Invalid email format"
        case "min":
            errors[field] = fmt.Sprintf("%s must be at least %s characters", field, err.Param())
        case "max":
            errors[field] = fmt.Sprintf("%s must be at most %s characters", field, err.Param())
        case "eqfield":
            errors[field] = fmt.Sprintf("%s must match %s", field, err.Param())
        case "strong_password":
            errors[field] = "Password must contain uppercase, lowercase, digit, and special character"
        case "adult":
            errors[field] = "You must be at least 18 years old"
        default:
            errors[field] = fmt.Sprintf("%s is invalid", field)
        }
    }
    
    return errors
}

// ✅ GOOD: Form handler with validation
func (h *AuthHandler) Register(c *fiber.Ctx) error {
    var form RegisterForm
    
    if err := c.BodyParser(&form); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid request body"})
    }
    
    // Validate form
    if errors := h.validator.ValidateStruct(form); errors != nil {
        return c.Status(422).JSON(fiber.Map{
            "error": "Validation failed",
            "fields": errors,
        })
    }
    
    // Check if email already exists
    exists, err := h.userService.EmailExists(form.Email)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Internal server error"})
    }
    if exists {
        return c.Status(422).JSON(fiber.Map{
            "error": "Validation failed",
            "fields": map[string]string{
                "Email": "Email already registered",
            },
        })
    }
    
    // Create user
    user, err := h.userService.CreateUser(form)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to create user"})
    }
    
    return c.Status(201).JSON(user)
}
```

### File Upload Handling (Go)

```go
// ✅ GOOD: Secure file upload with validation
type FileUploadConfig struct {
    MaxSize      int64
    AllowedTypes []string
    StoragePath  string
}

type FileUploadService struct {
    config FileUploadConfig
    s3     *s3.Client
}

func (fus *FileUploadService) UploadFile(c *fiber.Ctx) error {
    file, err := c.FormFile("file")
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "No file uploaded"})
    }
    
    // Validate file size
    if file.Size > fus.config.MaxSize {
        return c.Status(400).JSON(fiber.Map{
            "error": fmt.Sprintf("File size exceeds maximum of %d bytes", fus.config.MaxSize),
        })
    }
    
    // Open file
    src, err := file.Open()
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to open file"})
    }
    defer src.Close()
    
    // Detect actual MIME type (don't trust client)
    buffer := make([]byte, 512)
    _, err = src.Read(buffer)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to read file"})
    }
    src.Seek(0, 0) // Reset reader
    
    mimeType := http.DetectContentType(buffer)
    
    // Validate MIME type
    allowed := false
    for _, allowedType := range fus.config.AllowedTypes {
        if mimeType == allowedType {
            allowed = true
            break
        }
    }
    if !allowed {
        return c.Status(400).JSON(fiber.Map{
            "error": fmt.Sprintf("File type %s not allowed", mimeType),
        })
    }
    
    // Generate secure filename
    ext := filepath.Ext(file.Filename)
    filename := fmt.Sprintf("%s%s", uuid.New().String(), ext)
    
    // Upload to S3
    _, err = fus.s3.PutObject(context.Background(), &s3.PutObjectInput{
        Bucket:      aws.String("my-bucket"),
        Key:         aws.String(filename),
        Body:        src,
        ContentType: aws.String(mimeType),
    })
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to upload file"})
    }
    
    return c.JSON(fiber.Map{
        "filename": filename,
        "url":      fmt.Sprintf("https://cdn.example.com/%s", filename),
    })
}
```

---

## 3. WebSockets and Real-Time Communication

### WebSocket Implementation (Go)

```go
// ✅ GOOD: Production-ready WebSocket server
import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/websocket/v2"
)

type WSClient struct {
    ID     string
    UserID string
    Conn   *websocket.Conn
    Send   chan []byte
}

type WSHub struct {
    clients    map[string]*WSClient
    broadcast  chan []byte
    register   chan *WSClient
    unregister chan *WSClient
    mu         sync.RWMutex
}

func NewWSHub() *WSHub {
    return &WSHub{
        clients:    make(map[string]*WSClient),
        broadcast:  make(chan []byte, 256),
        register:   make(chan *WSClient),
        unregister: make(chan *WSClient),
    }
}

func (h *WSHub) Run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client.ID] = client
            h.mu.Unlock()
            
        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client.ID]; ok {
                delete(h.clients, client.ID)
                close(client.Send)
            }
            h.mu.Unlock()
            
        case message := <-h.broadcast:
            h.mu.RLock()
            for _, client := range h.clients {
                select {
                case client.Send <- message:
                default:
                    close(client.Send)
                    delete(h.clients, client.ID)
                }
            }
            h.mu.RUnlock()
        }
    }
}

func (h *WSHub) HandleWebSocket(c *websocket.Conn) {
    client := &WSClient{
        ID:   uuid.New().String(),
        Conn: c,
        Send: make(chan []byte, 256),
    }
    
    h.register <- client
    
    // Start writer goroutine
    go client.writePump()
    
    // Read messages in current goroutine
    client.readPump(h)
}

func (c *WSClient) readPump(hub *WSHub) {
    defer func() {
        hub.unregister <- c
        c.Conn.Close()
    }()
    
    c.Conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.Conn.SetPongHandler(func(string) error {
        c.Conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })
    
    for {
        _, message, err := c.Conn.ReadMessage()
        if err != nil {
            break
        }
        
        // Process message
        hub.broadcast <- message
    }
}

func (c *WSClient) writePump() {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.Conn.Close()
    }()
    
    for {
        select {
        case message, ok := <-c.Send:
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if !ok {
                c.Conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            
            if err := c.Conn.WriteMessage(websocket.TextMessage, message); err != nil {
                return
            }
            
        case <-ticker.C:
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.Conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// ✅ GOOD: WebSocket endpoint with authentication
func (h *Handler) WebSocketUpgrade(c *fiber.Ctx) error {
    // Validate JWT token
    token := c.Query("token")
    claims, err := h.tokenService.ValidateToken(token)
    if err != nil {
        return c.Status(401).JSON(fiber.Map{"error": "Unauthorized"})
    }
    
    if websocket.IsWebSocketUpgrade(c) {
        c.Locals("user_id", claims.UserID)
        return websocket.New(func(conn *websocket.Conn) {
            h.hub.HandleWebSocket(conn)
        })(c)
    }
    
    return c.Status(426).JSON(fiber.Map{"error": "Upgrade required"})
}
```

### Pub/Sub with Redis (Go)

```go
// ✅ GOOD: Redis pub/sub for distributed real-time messaging
type RedisPubSub struct {
    client *redis.Client
    pubsub *redis.PubSub
}

func NewRedisPubSub(client *redis.Client) *RedisPubSub {
    return &RedisPubSub{
        client: client,
    }
}

func (rps *RedisPubSub) Subscribe(channels ...string) error {
    rps.pubsub = rps.client.Subscribe(context.Background(), channels...)
    return nil
}

func (rps *RedisPubSub) Listen(handler func(channel, message string)) {
    ch := rps.pubsub.Channel()
    
    for msg := range ch {
        handler(msg.Channel, msg.Payload)
    }
}

func (rps *RedisPubSub) Publish(channel string, message interface{}) error {
    data, err := json.Marshal(message)
    if err != nil {
        return err
    }
    
    return rps.client.Publish(context.Background(), channel, data).Err()
}

// ✅ GOOD: Integrating pub/sub with WebSocket hub
type RealtimeService struct {
    hub    *WSHub
    pubsub *RedisPubSub
}

func (rs *RealtimeService) Start() {
    // Subscribe to channels
    rs.pubsub.Subscribe("notifications", "messages", "updates")
    
    // Listen for messages
    go rs.pubsub.Listen(func(channel, message string) {
        // Broadcast to WebSocket clients
        rs.hub.broadcast <- []byte(message)
    })
}

func (rs *RealtimeService) SendNotification(userID string, notification Notification) error {
    data := map[string]interface{}{
        "type":    "notification",
        "user_id": userID,
        "data":    notification,
    }
    
    return rs.pubsub.Publish("notifications", data)
}
```

---

## 4. State Management

### Client-Side State Patterns

```go
// ✅ GOOD: API for client state synchronization
type StateSync struct {
    redis *redis.Client
}

func (ss *StateSync) GetUserState(userID string) (map[string]interface{}, error) {
    data, err := ss.redis.Get(context.Background(), "state:"+userID).Result()
    if err == redis.Nil {
        return make(map[string]interface{}), nil
    }
    if err != nil {
        return nil, err
    }
    
    var state map[string]interface{}
    if err := json.Unmarshal([]byte(data), &state); err != nil {
        return nil, err
    }
    
    return state, nil
}

func (ss *StateSync) UpdateUserState(userID string, updates map[string]interface{}) error {
    // Get current state
    current, err := ss.GetUserState(userID)
    if err != nil {
        return err
    }
    
    // Merge updates
    for k, v := range updates {
        current[k] = v
    }
    
    // Save state
    data, err := json.Marshal(current)
    if err != nil {
        return err
    }
    
    return ss.redis.Set(context.Background(), "state:"+userID, data, 24*time.Hour).Err()
}

// ✅ GOOD: Optimistic locking for state updates
func (ss *StateSync) UpdateWithVersion(userID string, version int, updates map[string]interface{}) error {
    key := "state:" + userID
    versionKey := "state:version:" + userID
    
    // Watch keys for changes
    err := ss.redis.Watch(context.Background(), func(tx *redis.Tx) error {
        // Get current version
        currentVersion, err := tx.Get(context.Background(), versionKey).Int()
        if err != nil && err != redis.Nil {
            return err
        }
        
        // Check version
        if currentVersion != version {
            return errors.New("version mismatch")
        }
        
        // Get current state
        data, err := tx.Get(context.Background(), key).Result()
        if err != nil && err != redis.Nil {
            return err
        }
        
        var state map[string]interface{}
        if data != "" {
            if err := json.Unmarshal([]byte(data), &state); err != nil {
                return err
            }
        } else {
            state = make(map[string]interface{})
        }
        
        // Merge updates
        for k, v := range updates {
            state[k] = v
        }
        
        // Save state with new version
        newData, err := json.Marshal(state)
        if err != nil {
            return err
        }
        
        _, err = tx.TxPipelined(context.Background(), func(pipe redis.Pipeliner) error {
            pipe.Set(context.Background(), key, newData, 24*time.Hour)
            pipe.Incr(context.Background(), versionKey)
            return nil
        })
        
        return err
    }, versionKey, key)
    
    return err
}
```

---

## Summary

This document covers:

1. **Authentication Flows**: Session-based, JWT with refresh tokens, OAuth 2.0
2. **Forms & Validation**: Server-side validation, file uploads
3. **Real-Time**: WebSockets, Redis pub/sub
4. **State Management**: Client state sync, optimistic locking

## References

1. **OAuth 2.0 RFC**: https://datatracker.ietf.org/doc/html/rfc6749
2. **JWT Best Practices**: https://datatracker.ietf.org/doc/html/rfc8725
3. **OWASP Session Management**: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
4. **WebSocket RFC**: https://datatracker.ietf.org/doc/html/rfc6455

---

**Last Updated**: January 2026  
**Version**: 1.0
