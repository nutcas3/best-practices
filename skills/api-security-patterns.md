# API Security Best Practices - Production-Ready Patterns

## Skill Metadata
- **Domain**: API Security, Authentication, Authorization
- **Skill Level**: Advanced
- **Last Updated**: 2026
- **Sources**: OWASP API Security Top 10, PCI DSS, Security Audit Best Practices

## Overview

This document provides comprehensive security patterns for building production-ready APIs in Go and Rust. Based on real-world security audits and industry standards, these patterns address the most critical vulnerabilities found in financial and enterprise applications.

---

## 1. Authentication Security

### Secure JWT Implementation (Go)

```go
// ❌ BAD: Missing algorithm validation
func ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte(os.Getenv("JWT_SECRET")), nil
    })
    // No algorithm check - vulnerable to "none" algorithm attack
}

// ✅ GOOD: Proper JWT validation with algorithm check
import (
    "errors"
    "github.com/golang-jwt/jwt/v5"
    "time"
)

type Claims struct {
    UserID string `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func ValidateToken(tokenString string) (*Claims, error) {
    claims := &Claims{}
    
    token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
        // Validate signing algorithm
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, errors.New("invalid signing method")
        }
        return []byte(os.Getenv("JWT_SECRET")), nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if !token.Valid {
        return nil, errors.New("invalid token")
    }
    
    // Additional validation
    if claims.ExpiresAt.Before(time.Now()) {
        return nil, errors.New("token expired")
    }
    
    return claims, nil
}

// ✅ GOOD: Token generation with proper claims
func GenerateToken(userID, role string) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    "vermi-api",
            Subject:   userID,
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(os.Getenv("JWT_SECRET")))
}
```

### Token Revocation and Logout

```go
// ✅ GOOD: Token blacklist implementation
type TokenBlacklist struct {
    mu     sync.RWMutex
    tokens map[string]time.Time
    redis  *redis.Client
}

func NewTokenBlacklist(redis *redis.Client) *TokenBlacklist {
    tb := &TokenBlacklist{
        tokens: make(map[string]time.Time),
        redis:  redis,
    }
    
    // Cleanup expired tokens every hour
    go tb.cleanupExpired()
    
    return tb
}

func (tb *TokenBlacklist) Revoke(tokenString string, expiresAt time.Time) error {
    // Store in Redis for distributed systems
    if tb.redis != nil {
        ttl := time.Until(expiresAt)
        return tb.redis.Set(context.Background(), "blacklist:"+tokenString, "1", ttl).Err()
    }
    
    // Fallback to in-memory
    tb.mu.Lock()
    defer tb.mu.Unlock()
    tb.tokens[tokenString] = expiresAt
    return nil
}

func (tb *TokenBlacklist) IsRevoked(tokenString string) bool {
    // Check Redis first
    if tb.redis != nil {
        val, err := tb.redis.Get(context.Background(), "blacklist:"+tokenString).Result()
        if err == nil && val == "1" {
            return true
        }
    }
    
    // Check in-memory
    tb.mu.RLock()
    defer tb.mu.RUnlock()
    _, exists := tb.tokens[tokenString]
    return exists
}

func (tb *TokenBlacklist) cleanupExpired() {
    ticker := time.NewTicker(1 * time.Hour)
    defer ticker.Stop()
    
    for range ticker.C {
        tb.mu.Lock()
        now := time.Now()
        for token, expiresAt := range tb.tokens {
            if now.After(expiresAt) {
                delete(tb.tokens, token)
            }
        }
        tb.mu.Unlock()
    }
}

// ✅ GOOD: Logout endpoint
func (h *AuthHandler) Logout(c *fiber.Ctx) error {
    tokenString := c.Get("Authorization")
    tokenString = strings.TrimPrefix(tokenString, "Bearer ")
    
    claims, err := ValidateToken(tokenString)
    if err != nil {
        return c.Status(401).JSON(fiber.Map{"error": "Invalid token"})
    }
    
    // Revoke token
    if err := h.blacklist.Revoke(tokenString, claims.ExpiresAt.Time); err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Failed to logout"})
    }
    
    return c.JSON(fiber.Map{"message": "Logged out successfully"})
}
```

### Strong Password/PIN Policy

```go
// ❌ BAD: Weak PIN validation
if len(pin) < 4 {
    return errors.New("pin must be at least 4 characters")
}

// ✅ GOOD: Strong PIN/password validation
type PasswordPolicy struct {
    MinLength      int
    RequireUpper   bool
    RequireLower   bool
    RequireDigit   bool
    RequireSpecial bool
    MaxAge         time.Duration
}

var DefaultPasswordPolicy = PasswordPolicy{
    MinLength:      8,
    RequireUpper:   true,
    RequireLower:   true,
    RequireDigit:   true,
    RequireSpecial: true,
    MaxAge:         90 * 24 * time.Hour,
}

func ValidatePassword(password string, policy PasswordPolicy) error {
    if len(password) < policy.MinLength {
        return fmt.Errorf("password must be at least %d characters", policy.MinLength)
    }
    
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
    
    if policy.RequireUpper && !hasUpper {
        return errors.New("password must contain at least one uppercase letter")
    }
    if policy.RequireLower && !hasLower {
        return errors.New("password must contain at least one lowercase letter")
    }
    if policy.RequireDigit && !hasDigit {
        return errors.New("password must contain at least one digit")
    }
    if policy.RequireSpecial && !hasSpecial {
        return errors.New("password must contain at least one special character")
    }
    
    return nil
}

// ✅ GOOD: PIN validation for financial apps
func ValidatePIN(pin string) error {
    // Minimum 6 digits
    if len(pin) < 6 {
        return errors.New("PIN must be at least 6 digits")
    }
    
    // Only digits allowed
    if !regexp.MustCompile(`^\d+$`).MatchString(pin) {
        return errors.New("PIN must contain only digits")
    }
    
    // Check for sequential numbers (123456, 654321)
    if isSequential(pin) {
        return errors.New("PIN cannot be sequential")
    }
    
    // Check for repeated digits (111111, 000000)
    if isRepeated(pin) {
        return errors.New("PIN cannot be all the same digit")
    }
    
    return nil
}

func isSequential(s string) bool {
    for i := 1; i < len(s); i++ {
        if s[i] != s[i-1]+1 && s[i] != s[i-1]-1 {
            return false
        }
    }
    return true
}

func isRepeated(s string) bool {
    for i := 1; i < len(s); i++ {
        if s[i] != s[0] {
            return false
        }
    }
    return true
}
```

### Account Lockout and Rate Limiting

```go
// ✅ GOOD: Account lockout implementation
type AccountLockout struct {
    mu            sync.RWMutex
    attempts      map[string]int
    lockouts      map[string]time.Time
    maxAttempts   int
    lockDuration  time.Duration
    resetDuration time.Duration
}

func NewAccountLockout() *AccountLockout {
    al := &AccountLockout{
        attempts:      make(map[string]int),
        lockouts:      make(map[string]time.Time),
        maxAttempts:   5,
        lockDuration:  15 * time.Minute,
        resetDuration: 5 * time.Minute,
    }
    
    go al.cleanup()
    return al
}

func (al *AccountLockout) RecordFailedAttempt(identifier string) error {
    al.mu.Lock()
    defer al.mu.Unlock()
    
    // Check if already locked
    if lockUntil, exists := al.lockouts[identifier]; exists {
        if time.Now().Before(lockUntil) {
            return fmt.Errorf("account locked until %v", lockUntil)
        }
        delete(al.lockouts, identifier)
    }
    
    al.attempts[identifier]++
    
    if al.attempts[identifier] >= al.maxAttempts {
        al.lockouts[identifier] = time.Now().Add(al.lockDuration)
        delete(al.attempts, identifier)
        return fmt.Errorf("account locked for %v due to too many failed attempts", al.lockDuration)
    }
    
    return nil
}

func (al *AccountLockout) RecordSuccessfulAttempt(identifier string) {
    al.mu.Lock()
    defer al.mu.Unlock()
    delete(al.attempts, identifier)
    delete(al.lockouts, identifier)
}

func (al *AccountLockout) IsLocked(identifier string) bool {
    al.mu.RLock()
    defer al.mu.RUnlock()
    
    if lockUntil, exists := al.lockouts[identifier]; exists {
        return time.Now().Before(lockUntil)
    }
    return false
}

func (al *AccountLockout) cleanup() {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()
    
    for range ticker.C {
        al.mu.Lock()
        now := time.Now()
        
        // Clean expired lockouts
        for id, lockUntil := range al.lockouts {
            if now.After(lockUntil) {
                delete(al.lockouts, id)
            }
        }
        
        al.mu.Unlock()
    }
}
```

### Multi-Factor Authentication (MFA)

```go
// ✅ GOOD: TOTP-based MFA implementation
import "github.com/pquerna/otp/totp"

type MFAService struct {
    issuer string
}

func NewMFAService(issuer string) *MFAService {
    return &MFAService{issuer: issuer}
}

func (m *MFAService) GenerateSecret(accountName string) (*MFASecret, error) {
    key, err := totp.Generate(totp.GenerateOpts{
        Issuer:      m.issuer,
        AccountName: accountName,
    })
    if err != nil {
        return nil, err
    }
    
    return &MFASecret{
        Secret: key.Secret(),
        URL:    key.URL(),
        QRCode: key.URL(), // Generate QR code from this
    }, nil
}

func (m *MFAService) ValidateCode(secret, code string) bool {
    return totp.Validate(code, secret)
}

// ✅ GOOD: SMS-based OTP with rate limiting
type OTPService struct {
    mu        sync.RWMutex
    otps      map[string]*OTPData
    smsClient SMSClient
}

type OTPData struct {
    Code      string
    ExpiresAt time.Time
    Attempts  int
}

func (o *OTPService) SendOTP(phoneNumber string) error {
    o.mu.Lock()
    defer o.mu.Unlock()
    
    // Generate 6-digit OTP
    code := fmt.Sprintf("%06d", rand.Intn(1000000))
    
    // Store OTP with 5-minute expiration
    o.otps[phoneNumber] = &OTPData{
        Code:      code,
        ExpiresAt: time.Now().Add(5 * time.Minute),
        Attempts:  0,
    }
    
    // Send via SMS
    return o.smsClient.Send(phoneNumber, fmt.Sprintf("Your OTP is: %s", code))
}

func (o *OTPService) VerifyOTP(phoneNumber, code string) error {
    o.mu.Lock()
    defer o.mu.Unlock()
    
    otpData, exists := o.otps[phoneNumber]
    if !exists {
        return errors.New("no OTP found for this number")
    }
    
    // Check expiration
    if time.Now().After(otpData.ExpiresAt) {
        delete(o.otps, phoneNumber)
        return errors.New("OTP expired")
    }
    
    // Check attempts
    if otpData.Attempts >= 3 {
        delete(o.otps, phoneNumber)
        return errors.New("too many failed attempts")
    }
    
    // Verify code
    if otpData.Code != code {
        otpData.Attempts++
        return errors.New("invalid OTP")
    }
    
    // Success - remove OTP
    delete(o.otps, phoneNumber)
    return nil
}
```

### Rust Authentication Patterns

```rust
// ✅ GOOD: JWT validation in Rust
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use chrono::{Duration, Utc};

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,
    pub role: String,
    pub exp: usize,
    pub iat: usize,
    pub nbf: usize,
}

pub fn generate_token(user_id: &str, role: &str, secret: &str) -> Result<String, Error> {
    let expiration = Utc::now()
        .checked_add_signed(Duration::hours(24))
        .expect("valid timestamp")
        .timestamp() as usize;
    
    let claims = Claims {
        sub: user_id.to_owned(),
        role: role.to_owned(),
        exp: expiration,
        iat: Utc::now().timestamp() as usize,
        nbf: Utc::now().timestamp() as usize,
    };
    
    encode(
        &Header::new(Algorithm::HS256),
        &claims,
        &EncodingKey::from_secret(secret.as_ref()),
    )
    .map_err(|e| Error::TokenGeneration(e.to_string()))
}

pub fn validate_token(token: &str, secret: &str) -> Result<Claims, Error> {
    let mut validation = Validation::new(Algorithm::HS256);
    validation.validate_exp = true;
    validation.validate_nbf = true;
    
    decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret.as_ref()),
        &validation,
    )
    .map(|data| data.claims)
    .map_err(|e| Error::InvalidToken(e.to_string()))
}

// ✅ GOOD: Password hashing with Argon2
use argon2::{
    password_hash::{rand_core::OsRng, PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};

pub fn hash_password(password: &str) -> Result<String, Error> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();
    
    argon2
        .hash_password(password.as_bytes(), &salt)
        .map(|hash| hash.to_string())
        .map_err(|e| Error::HashingFailed(e.to_string()))
}

pub fn verify_password(password: &str, hash: &str) -> Result<bool, Error> {
    let parsed_hash = PasswordHash::new(hash)
        .map_err(|e| Error::InvalidHash(e.to_string()))?;
    
    Ok(Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok())
}
```

---

## 2. Authorization and Access Control

### Role-Based Access Control (RBAC)

```go
// ✅ GOOD: RBAC implementation
type Permission string

const (
    PermissionReadUser   Permission = "user:read"
    PermissionWriteUser  Permission = "user:write"
    PermissionReadGroup  Permission = "group:read"
    PermissionWriteGroup Permission = "group:write"
    PermissionApprove    Permission = "loan:approve"
    PermissionAdmin      Permission = "admin:*"
)

type Role struct {
    Name        string
    Permissions []Permission
}

var Roles = map[string]Role{
    "user": {
        Name: "user",
        Permissions: []Permission{
            PermissionReadUser,
            PermissionReadGroup,
        },
    },
    "group_admin": {
        Name: "group_admin",
        Permissions: []Permission{
            PermissionReadUser,
            PermissionWriteUser,
            PermissionReadGroup,
            PermissionWriteGroup,
            PermissionApprove,
        },
    },
    "admin": {
        Name:        "admin",
        Permissions: []Permission{PermissionAdmin},
    },
}

func HasPermission(role string, permission Permission) bool {
    r, exists := Roles[role]
    if !exists {
        return false
    }
    
    // Admin has all permissions
    for _, p := range r.Permissions {
        if p == PermissionAdmin {
            return true
        }
        if p == permission {
            return true
        }
    }
    
    return false
}

// ✅ GOOD: Authorization middleware
func RequirePermission(permission Permission) fiber.Handler {
    return func(c *fiber.Ctx) error {
        claims := c.Locals("claims").(*Claims)
        
        if !HasPermission(claims.Role, permission) {
            return c.Status(403).JSON(fiber.Map{
                "error": "Insufficient permissions",
            })
        }
        
        return c.Next()
    }
}

// Usage
app.Get("/api/groups/:id", 
    AuthMiddleware(),
    RequirePermission(PermissionReadGroup),
    handler.GetGroup,
)
```

### Preventing IDOR (Insecure Direct Object References)

```go
// ❌ BAD: No authorization check
func GetGroup(c *fiber.Ctx) error {
    id := c.Params("id")
    group, err := db.GetGroupByID(id)
    if err != nil {
        return c.Status(404).JSON(fiber.Map{"error": "Group not found"})
    }
    return c.JSON(group)
}

// ✅ GOOD: Verify user has access to resource
func GetGroup(c *fiber.Ctx) error {
    claims := c.Locals("claims").(*Claims)
    groupID := c.Params("id")
    
    // Check if user is member of the group
    isMember, err := db.IsGroupMember(groupID, claims.UserID)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "Internal server error"})
    }
    
    if !isMember {
        return c.Status(403).JSON(fiber.Map{"error": "Access denied"})
    }
    
    group, err := db.GetGroupByID(groupID)
    if err != nil {
        return c.Status(404).JSON(fiber.Map{"error": "Group not found"})
    }
    
    return c.JSON(group)
}

// ✅ BETTER: Resource-based authorization
type ResourceAuthorizer struct {
    db *Database
}

func (ra *ResourceAuthorizer) CanAccessGroup(userID, groupID string) (bool, error) {
    // Check direct membership
    isMember, err := ra.db.IsGroupMember(groupID, userID)
    if err != nil {
        return false, err
    }
    if isMember {
        return true, nil
    }
    
    // Check if user is admin
    user, err := ra.db.GetUser(userID)
    if err != nil {
        return false, err
    }
    
    return user.Role == "admin", nil
}

func (ra *ResourceAuthorizer) CanApproveRequest(userID, requestID string) (bool, error) {
    request, err := ra.db.GetRequest(requestID)
    if err != nil {
        return false, err
    }
    
    // Check if user is the designated approver
    if request.ApproverID == userID {
        return true, nil
    }
    
    // Check if user is group admin
    isAdmin, err := ra.db.IsGroupAdmin(request.GroupID, userID)
    if err != nil {
        return false, err
    }
    
    return isAdmin, nil
}
```

### Field-Level Access Control

```go
// ✅ GOOD: Field-level permissions
type UserResponse struct {
    ID            string  `json:"id"`
    Name          string  `json:"name"`
    Email         string  `json:"email,omitempty"`
    Phone         string  `json:"phone,omitempty"`
    WalletBalance float64 `json:"wallet_balance,omitempty"`
    LoanLimit     float64 `json:"loan_limit,omitempty"`
}

func (h *UserHandler) GetUser(c *fiber.Ctx) error {
    claims := c.Locals("claims").(*Claims)
    targetUserID := c.Params("id")
    
    user, err := h.db.GetUser(targetUserID)
    if err != nil {
        return c.Status(404).JSON(fiber.Map{"error": "User not found"})
    }
    
    response := UserResponse{
        ID:   user.ID,
        Name: user.Name,
    }
    
    // Only include sensitive fields if viewing own profile or admin
    if claims.UserID == targetUserID || claims.Role == "admin" {
        response.Email = user.Email
        response.Phone = user.Phone
        response.WalletBalance = user.WalletBalance
        response.LoanLimit = user.LoanLimit
    }
    
    return c.JSON(response)
}
```

---

## 3. Input Validation and Sanitization

### Comprehensive Input Validation

```go
// ✅ GOOD: Input validation with go-playground/validator
import "github.com/go-playground/validator/v10"

type CreateUserRequest struct {
    Name     string  `json:"name" validate:"required,min=2,max=100"`
    Email    string  `json:"email" validate:"required,email"`
    Phone    string  `json:"phone" validate:"required,e164"`
    PIN      string  `json:"pin" validate:"required,min=6,max=6,numeric"`
    Amount   float64 `json:"amount,omitempty" validate:"omitempty,min=0,max=1000000"`
}

var validate = validator.New()

func ValidateRequest(req interface{}) error {
    if err := validate.Struct(req); err != nil {
        if validationErrors, ok := err.(validator.ValidationErrors); ok {
            errors := make(map[string]string)
            for _, e := range validationErrors {
                errors[e.Field()] = formatValidationError(e)
            }
            return &ValidationError{Errors: errors}
        }
        return err
    }
    return nil
}

func formatValidationError(e validator.FieldError) string {
    switch e.Tag() {
    case "required":
        return fmt.Sprintf("%s is required", e.Field())
    case "email":
        return "Invalid email format"
    case "min":
        return fmt.Sprintf("%s must be at least %s characters", e.Field(), e.Param())
    case "max":
        return fmt.Sprintf("%s must be at most %s characters", e.Field(), e.Param())
    case "numeric":
        return fmt.Sprintf("%s must contain only numbers", e.Field())
    default:
        return fmt.Sprintf("%s is invalid", e.Field())
    }
}
```

### SQL Injection Prevention

```go
// ❌ BAD: String concatenation (vulnerable to SQL injection)
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email)

// ✅ GOOD: Using GORM (ORM with parameterized queries)
var user User
result := db.Where("email = ?", email).First(&user)

// ✅ GOOD: Raw query with parameters
var users []User
db.Raw("SELECT * FROM users WHERE email = ? AND status = ?", email, "active").Scan(&users)

// ✅ GOOD: Named parameters
db.Raw("SELECT * FROM users WHERE email = @email AND status = @status", 
    sql.Named("email", email),
    sql.Named("status", "active"),
).Scan(&users)
```

### XSS Prevention

```go
// ✅ GOOD: HTML escaping
import "html"

func SanitizeHTML(input string) string {
    return html.EscapeString(input)
}

// ✅ GOOD: Using bluemonday for rich text
import "github.com/microcosm-cc/bluemonday"

var policy = bluemonday.UGCPolicy()

func SanitizeUserContent(input string) string {
    return policy.Sanitize(input)
}
```

### Amount and Decimal Validation

```go
// ✅ GOOD: Decimal handling for financial amounts
import "github.com/shopspring/decimal"

type TransactionRequest struct {
    Amount   string `json:"amount" validate:"required"`
    Currency string `json:"currency" validate:"required,len=3"`
}

func ValidateAmount(amountStr string) (decimal.Decimal, error) {
    amount, err := decimal.NewFromString(amountStr)
    if err != nil {
        return decimal.Zero, errors.New("invalid amount format")
    }
    
    // Check if amount is positive
    if amount.LessThanOrEqual(decimal.Zero) {
        return decimal.Zero, errors.New("amount must be greater than zero")
    }
    
    // Check maximum amount (e.g., 1 million)
    maxAmount := decimal.NewFromInt(1000000)
    if amount.GreaterThan(maxAmount) {
        return decimal.Zero, errors.New("amount exceeds maximum limit")
    }
    
    // Check decimal places (max 2 for currency)
    if amount.Exponent() < -2 {
        return decimal.Zero, errors.New("amount cannot have more than 2 decimal places")
    }
    
    return amount, nil
}
```

---

## 4. Network Security

### CORS Configuration

```go
// ❌ BAD: Wildcard CORS
app.Use(cors.New(cors.Config{
    AllowOrigins: "*",
}))

// ✅ GOOD: Specific origins from environment
func ConfigureCORS() fiber.Handler {
    allowedOrigins := strings.Split(os.Getenv("ALLOWED_ORIGINS"), ",")
    
    return cors.New(cors.Config{
        AllowOrigins:     strings.Join(allowedOrigins, ","),
        AllowMethods:     "GET,POST,PUT,DELETE,OPTIONS",
        AllowHeaders:     "Origin,Content-Type,Accept,Authorization",
        AllowCredentials: true,
        MaxAge:           86400,
    })
}

// ✅ BETTER: Dynamic origin validation
func ConfigureCORS() fiber.Handler {
    return cors.New(cors.Config{
        AllowOriginsFunc: func(origin string) bool {
            allowedOrigins := strings.Split(os.Getenv("ALLOWED_ORIGINS"), ",")
            for _, allowed := range allowedOrigins {
                if origin == strings.TrimSpace(allowed) {
                    return true
                }
            }
            return false
        },
        AllowMethods:     "GET,POST,PUT,DELETE,OPTIONS",
        AllowHeaders:     "Origin,Content-Type,Accept,Authorization",
        AllowCredentials: true,
        MaxAge:           86400,
    })
}
```

### HTTPS Enforcement

```go
// ✅ GOOD: HTTPS redirect middleware
func HTTPSRedirect() fiber.Handler {
    return func(c *fiber.Ctx) error {
        if c.Protocol() != "https" && os.Getenv("ENV") == "production" {
            return c.Redirect("https://"+c.Hostname()+c.OriginalURL(), 301)
        }
        return c.Next()
    }
}

// ✅ GOOD: Security headers middleware
func SecurityHeaders() fiber.Handler {
    return func(c *fiber.Ctx) error {
        c.Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains; preload")
        c.Set("X-Content-Type-Options", "nosniff")
        c.Set("X-Frame-Options", "DENY")
        c.Set("X-XSS-Protection", "1; mode=block")
        c.Set("Content-Security-Policy", "default-src 'self'")
        c.Set("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        
        return c.Next()
    }
}
```

### CSRF Protection

```go
// ✅ GOOD: CSRF token implementation
import "github.com/gofiber/fiber/v2/middleware/csrf"

func ConfigureCSRF() fiber.Handler {
    return csrf.New(csrf.Config{
        KeyLookup:      "header:X-CSRF-Token",
        CookieName:     "csrf_",
        CookieSameSite: "Strict",
        CookieSecure:   true,
        CookieHTTPOnly: true,
        Expiration:     1 * time.Hour,
        KeyGenerator:   utils.UUIDv4,
    })
}

// Usage
app.Use(ConfigureCSRF())

// In your frontend, include the CSRF token in requests
// The token is available in the cookie and should be sent in the X-CSRF-Token header
```

### Rate Limiting

```go
// ✅ GOOD: Rate limiting middleware
import "github.com/gofiber/fiber/v2/middleware/limiter"

func ConfigureRateLimiter() fiber.Handler {
    return limiter.New(limiter.Config{
        Max:        100,
        Expiration: 1 * time.Minute,
        KeyGenerator: func(c *fiber.Ctx) string {
            // Rate limit by IP
            return c.IP()
        },
        LimitReached: func(c *fiber.Ctx) error {
            return c.Status(429).JSON(fiber.Map{
                "error": "Too many requests",
            })
        },
    })
}

// ✅ BETTER: Different limits for different endpoints
func ConfigureAuthRateLimiter() fiber.Handler {
    return limiter.New(limiter.Config{
        Max:        5,
        Expiration: 15 * time.Minute,
        KeyGenerator: func(c *fiber.Ctx) string {
            // Rate limit by IP + endpoint
            return c.IP() + ":" + c.Path()
        },
        LimitReached: func(c *fiber.Ctx) error {
            return c.Status(429).JSON(fiber.Map{
                "error": "Too many login attempts. Please try again later.",
            })
        },
    })
}

// Usage
app.Post("/api/auth/login", ConfigureAuthRateLimiter(), handler.Login)
```

---

## 5. Business Logic Security

### Idempotency for Financial Operations

```go
// ❌ BAD: Optional idempotency
func IdempotencyMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        key := c.Get("Idempotency-Key")
        if key == "" {
            return c.Next() // Proceed without protection
        }
        // Check idempotency...
    }
}

// ✅ GOOD: Mandatory idempotency for financial operations
type IdempotencyStore struct {
    mu      sync.RWMutex
    store   map[string]*IdempotencyRecord
    redis   *redis.Client
}

type IdempotencyRecord struct {
    Response   []byte
    StatusCode int
    CreatedAt  time.Time
}

func (is *IdempotencyStore) RequireIdempotency() fiber.Handler {
    return func(c *fiber.Ctx) error {
        // Only for state-changing operations
        if c.Method() != "POST" && c.Method() != "PUT" && c.Method() != "DELETE" {
            return c.Next()
        }
        
        key := c.Get("Idempotency-Key")
        if key == "" {
            return c.Status(400).JSON(fiber.Map{
                "error": "Idempotency-Key header is required",
            })
        }
        
        // Validate key format (UUID)
        if !isValidUUID(key) {
            return c.Status(400).JSON(fiber.Map{
                "error": "Invalid Idempotency-Key format",
            })
        }
        
        // Check if request was already processed
        record, exists := is.Get(key)
        if exists {
            // Return cached response
            c.Status(record.StatusCode)
            return c.Send(record.Response)
        }
        
        // Store key in context for later
        c.Locals("idempotency_key", key)
        return c.Next()
    }
}

func (is *IdempotencyStore) StoreResponse(key string, response []byte, statusCode int) error {
    record := &IdempotencyRecord{
        Response:   response,
        StatusCode: statusCode,
        CreatedAt:  time.Now(),
    }
    
    // Store in Redis with 24-hour TTL
    if is.redis != nil {
        data, _ := json.Marshal(record)
        return is.redis.Set(context.Background(), "idempotency:"+key, data, 24*time.Hour).Err()
    }
    
    // Fallback to in-memory
    is.mu.Lock()
    defer is.mu.Unlock()
    is.store[key] = record
    return nil
}
```

### Race Condition Prevention

```go
// ✅ GOOD: Database-level locking for transactions
func (s *TransactionService) Transfer(ctx context.Context, from, to string, amount decimal.Decimal) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // Lock accounts in consistent order to prevent deadlocks
        accounts := []string{from, to}
        sort.Strings(accounts)
        
        var fromAccount, toAccount Account
        
        // Lock first account
        if err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            Where("id = ?", accounts[0]).First(&fromAccount).Error; err != nil {
            return err
        }
        
        // Lock second account
        if err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            Where("id = ?", accounts[1]).First(&toAccount).Error; err != nil {
            return err
        }
        
        // Determine which is from and which is to
        if fromAccount.ID != from {
            fromAccount, toAccount = toAccount, fromAccount
        }
        
        // Check balance
        if fromAccount.Balance.LessThan(amount) {
            return errors.New("insufficient balance")
        }
        
        // Perform transfer
        fromAccount.Balance = fromAccount.Balance.Sub(amount)
        toAccount.Balance = toAccount.Balance.Add(amount)
        
        if err := tx.Save(&fromAccount).Error; err != nil {
            return err
        }
        
        if err := tx.Save(&toAccount).Error; err != nil {
            return err
        }
        
        return nil
    })
}

// ✅ GOOD: Distributed locking with Redis
import "github.com/go-redsync/redsync/v4"

func (s *LoanService) ApproveLoan(ctx context.Context, loanID string) error {
    // Create distributed lock
    mutex := s.redsync.NewMutex("loan:"+loanID, 
        redsync.WithExpiry(30*time.Second),
        redsync.WithTries(3),
    )
    
    if err := mutex.Lock(); err != nil {
        return errors.New("could not acquire lock")
    }
    defer mutex.Unlock()
    
    // Process loan approval
    loan, err := s.db.GetLoan(loanID)
    if err != nil {
        return err
    }
    
    if loan.Status != "pending" {
        return errors.New("loan already processed")
    }
    
    loan.Status = "approved"
    loan.ApprovedAt = time.Now()
    
    return s.db.UpdateLoan(loan)
}
```

### Replay Attack Prevention

```go
// ✅ GOOD: Nonce and timestamp validation
type RequestValidator struct {
    nonceStore *NonceStore
    maxAge     time.Duration
}

func (rv *RequestValidator) ValidateRequest() fiber.Handler {
    return func(c *fiber.Ctx) error {
        timestamp := c.Get("X-Request-Timestamp")
        nonce := c.Get("X-Request-Nonce")
        
        if timestamp == "" || nonce == "" {
            return c.Status(400).JSON(fiber.Map{
                "error": "Missing timestamp or nonce",
            })
        }
        
        // Parse timestamp
        ts, err := time.Parse(time.RFC3339, timestamp)
        if err != nil {
            return c.Status(400).JSON(fiber.Map{
                "error": "Invalid timestamp format",
            })
        }
        
        // Check if request is too old
        if time.Since(ts) > rv.maxAge {
            return c.Status(400).JSON(fiber.Map{
                "error": "Request expired",
            })
        }
        
        // Check if request is from the future
        if ts.After(time.Now().Add(5 * time.Minute)) {
            return c.Status(400).JSON(fiber.Map{
                "error": "Invalid timestamp",
            })
        }
        
        // Check nonce uniqueness
        if rv.nonceStore.Exists(nonce) {
            return c.Status(400).JSON(fiber.Map{
                "error": "Duplicate request",
            })
        }
        
        // Store nonce
        rv.nonceStore.Add(nonce, rv.maxAge)
        
        return c.Next()
    }
}
```

### Business Rule Validation

```go
// ✅ GOOD: Comprehensive loan limit validation
func (s *LoanService) ValidateLoanRequest(ctx context.Context, userID string, amount decimal.Decimal) error {
    user, err := s.db.GetUser(userID)
    if err != nil {
        return err
    }
    
    // Check user's loan limit
    if amount.GreaterThan(user.LoanLimit) {
        return fmt.Errorf("amount exceeds loan limit of %s", user.LoanLimit)
    }
    
    // Get all active loans
    activeLoans, err := s.db.GetActiveLoans(userID)
    if err != nil {
        return err
    }
    
    totalActive := decimal.Zero
    for _, loan := range activeLoans {
        totalActive = totalActive.Add(loan.Amount)
    }
    
    // Get all pending loan requests
    pendingLoans, err := s.db.GetPendingLoans(userID)
    if err != nil {
        return err
    }
    
    totalPending := decimal.Zero
    for _, loan := range pendingLoans {
        totalPending = totalPending.Add(loan.Amount)
    }
    
    // Check total exposure
    totalExposure := totalActive.Add(totalPending).Add(amount)
    if totalExposure.GreaterThan(user.LoanLimit) {
        return fmt.Errorf("total loan exposure (%s) would exceed limit (%s)", 
            totalExposure, user.LoanLimit)
    }
    
    // Check repayment history
    defaultedLoans, err := s.db.GetDefaultedLoans(userID)
    if err != nil {
        return err
    }
    
    if len(defaultedLoans) > 0 {
        return errors.New("user has defaulted loans")
    }
    
    return nil
}
```

---

## 6. Audit Logging and Monitoring

### Comprehensive Audit Logging

```go
// ✅ GOOD: Structured audit logging
type AuditLogger struct {
    logger *zap.Logger
    db     *gorm.DB
}

type AuditLog struct {
    ID          string    `gorm:"primaryKey"`
    Timestamp   time.Time `gorm:"index"`
    UserID      string    `gorm:"index"`
    Action      string    `gorm:"index"`
    Resource    string
    ResourceID  string    `gorm:"index"`
    IPAddress   string
    UserAgent   string
    RequestID   string    `gorm:"index"`
    Status      string
    Changes     string    // JSON of before/after
    Metadata    string    // Additional context
}

func (al *AuditLogger) Log(ctx context.Context, entry AuditLog) error {
    entry.ID = uuid.New().String()
    entry.Timestamp = time.Now()
    
    // Log to structured logger
    al.logger.Info("audit_event",
        zap.String("audit_id", entry.ID),
        zap.String("user_id", entry.UserID),
        zap.String("action", entry.Action),
        zap.String("resource", entry.Resource),
        zap.String("resource_id", entry.ResourceID),
        zap.String("status", entry.Status),
    )
    
    // Store in database for compliance
    return al.db.Create(&entry).Error
}

// ✅ GOOD: Audit middleware
func AuditMiddleware(auditor *AuditLogger) fiber.Handler {
    return func(c *fiber.Ctx) error {
        start := time.Now()
        
        // Capture request body for audit
        var requestBody []byte
        if c.Method() == "POST" || c.Method() == "PUT" {
            requestBody = c.Body()
        }
        
        // Process request
        err := c.Next()
        
        // Get user from context
        claims := c.Locals("claims").(*Claims)
        
        // Create audit log
        entry := AuditLog{
            UserID:     claims.UserID,
            Action:     c.Method() + " " + c.Path(),
            Resource:   extractResource(c.Path()),
            ResourceID: c.Params("id"),
            IPAddress:  c.IP(),
            UserAgent:  c.Get("User-Agent"),
            RequestID:  c.Get("X-Request-ID"),
            Status:     fmt.Sprintf("%d", c.Response().StatusCode()),
            Metadata:   string(requestBody),
        }
        
        // Log asynchronously
        go auditor.Log(context.Background(), entry)
        
        return err
    }
}

// ✅ GOOD: Financial transaction audit
func (s *TransactionService) CreateTransaction(ctx context.Context, tx Transaction) error {
    // Get user from context
    userID := ctx.Value("user_id").(string)
    
    // Capture before state
    beforeState, _ := json.Marshal(tx)
    
    // Create transaction
    if err := s.db.Create(&tx).Error; err != nil {
        s.auditor.Log(ctx, AuditLog{
            UserID:     userID,
            Action:     "transaction.create",
            Resource:   "transaction",
            ResourceID: tx.ID,
            Status:     "failed",
            Metadata:   err.Error(),
        })
        return err
    }
    
    // Capture after state
    afterState, _ := json.Marshal(tx)
    
    // Log successful transaction
    changes := map[string]interface{}{
        "before": json.RawMessage(beforeState),
        "after":  json.RawMessage(afterState),
    }
    changesJSON, _ := json.Marshal(changes)
    
    s.auditor.Log(ctx, AuditLog{
        UserID:     userID,
        Action:     "transaction.create",
        Resource:   "transaction",
        ResourceID: tx.ID,
        Status:     "success",
        Changes:    string(changesJSON),
    })
    
    return nil
}
```

### Sensitive Data Sanitization in Logs

```go
// ✅ GOOD: Sanitize sensitive data before logging
type SanitizedUser struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
    // Exclude: Password, PIN, SSN, etc.
}

func SanitizeForLogging(user *User) *SanitizedUser {
    return &SanitizedUser{
        ID:    user.ID,
        Name:  user.Name,
        Email: maskEmail(user.Email),
    }
}

func maskEmail(email string) string {
    parts := strings.Split(email, "@")
    if len(parts) != 2 {
        return "***"
    }
    
    username := parts[0]
    if len(username) <= 2 {
        return "***@" + parts[1]
    }
    
    return username[:2] + "***@" + parts[1]
}

func maskPAN(pan string) string {
    if len(pan) < 8 {
        return "****"
    }
    return pan[:4] + "****" + pan[len(pan)-4:]
}

// ✅ GOOD: Custom logger that sanitizes
type SanitizingLogger struct {
    logger *zap.Logger
}

func (sl *SanitizingLogger) Info(msg string, fields ...zap.Field) {
    sanitized := make([]zap.Field, 0, len(fields))
    for _, field := range fields {
        if isSensitiveField(field.Key) {
            sanitized = append(sanitized, zap.String(field.Key, "***REDACTED***"))
        } else {
            sanitized = append(sanitized, field)
        }
    }
    sl.logger.Info(msg, sanitized...)
}

func isSensitiveField(key string) bool {
    sensitive := []string{"password", "pin", "ssn", "credit_card", "token", "secret"}
    key = strings.ToLower(key)
    for _, s := range sensitive {
        if strings.Contains(key, s) {
            return true
        }
    }
    return false
}
```

---

## 7. Data Protection and Privacy

### Encryption at Rest

```go
// ✅ GOOD: Field-level encryption
import "crypto/aes"
import "crypto/cipher"
import "crypto/rand"
import "encoding/base64"

type EncryptionService struct {
    key []byte
}

func NewEncryptionService(key string) (*EncryptionService, error) {
    keyBytes := []byte(key)
    if len(keyBytes) != 32 {
        return nil, errors.New("key must be 32 bytes")
    }
    return &EncryptionService{key: keyBytes}, nil
}

func (es *EncryptionService) Encrypt(plaintext string) (string, error) {
    block, err := aes.NewCipher(es.key)
    if err != nil {
        return "", err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }
    
    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return "", err
    }
    
    ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func (es *EncryptionService) Decrypt(ciphertext string) (string, error) {
    data, err := base64.StdEncoding.DecodeString(ciphertext)
    if err != nil {
        return "", err
    }
    
    block, err := aes.NewCipher(es.key)
    if err != nil {
        return "", err
    }
    
    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }
    
    nonceSize := gcm.NonceSize()
    if len(data) < nonceSize {
        return "", errors.New("ciphertext too short")
    }
    
    nonce, ciphertext := data[:nonceSize], data[nonceSize:]
    plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
    if err != nil {
        return "", err
    }
    
    return string(plaintext), nil
}

// Usage with GORM hooks
type User struct {
    ID       string
    Name     string
    SSN      string `gorm:"column:ssn_encrypted"`
    ssnPlain string `gorm:"-"`
}

func (u *User) BeforeSave(tx *gorm.DB) error {
    if u.ssnPlain != "" {
        encrypted, err := encryptionService.Encrypt(u.ssnPlain)
        if err != nil {
            return err
        }
        u.SSN = encrypted
    }
    return nil
}

func (u *User) AfterFind(tx *gorm.DB) error {
    if u.SSN != "" {
        decrypted, err := encryptionService.Decrypt(u.SSN)
        if err != nil {
            return err
        }
        u.ssnPlain = decrypted
    }
    return nil
}
```

### PII Data Handling

```go
// ✅ GOOD: PII data retention and deletion
type PIIManager struct {
    db *gorm.DB
}

func (pm *PIIManager) AnonymizeUser(userID string) error {
    return pm.db.Transaction(func(tx *gorm.DB) error {
        // Anonymize user data
        updates := map[string]interface{}{
            "name":         "User-" + userID[:8],
            "email":        fmt.Sprintf("deleted-%s@example.com", userID[:8]),
            "phone":        "",
            "ssn":          "",
            "address":      "",
            "anonymized":   true,
            "anonymized_at": time.Now(),
        }
        
        if err := tx.Model(&User{}).Where("id = ?", userID).Updates(updates).Error; err != nil {
            return err
        }
        
        // Delete related PII
        if err := tx.Where("user_id = ?", userID).Delete(&UserDocument{}).Error; err != nil {
            return err
        }
        
        // Audit the anonymization
        audit := AuditLog{
            Action:     "user.anonymize",
            ResourceID: userID,
            Status:     "success",
        }
        return tx.Create(&audit).Error
    })
}

// ✅ GOOD: Data retention policy
func (pm *PIIManager) EnforceRetentionPolicy() error {
    // Delete inactive accounts after 2 years
    cutoff := time.Now().AddDate(-2, 0, 0)
    
    var users []User
    if err := pm.db.Where("last_login < ? AND anonymized = ?", cutoff, false).
        Find(&users).Error; err != nil {
        return err
    }
    
    for _, user := range users {
        if err := pm.AnonymizeUser(user.ID); err != nil {
            log.Printf("Failed to anonymize user %s: %v", user.ID, err)
        }
    }
    
    return nil
}
```

---

## 8. Security Testing Checklist

### Pre-Deployment Security Checklist

```markdown
## Authentication & Authorization
- [ ] JWT algorithm validation implemented
- [ ] Token expiration properly enforced
- [ ] Token revocation/logout mechanism in place
- [ ] Password/PIN meets complexity requirements (min 6-8 chars)
- [ ] Account lockout after failed attempts (5 attempts, 15 min lockout)
- [ ] MFA available for sensitive operations
- [ ] RBAC properly implemented
- [ ] IDOR vulnerabilities tested and fixed
- [ ] Field-level access control implemented

## Network Security
- [ ] CORS configured with specific origins (no wildcards)
- [ ] HTTPS enforced in production
- [ ] HSTS header set (max-age=31536000)
- [ ] CSRF protection enabled
- [ ] Security headers configured (CSP, X-Frame-Options, etc.)
- [ ] Rate limiting on all endpoints
- [ ] Aggressive rate limiting on auth endpoints (5 req/15min)

## Input Validation
- [ ] All inputs validated with whitelist approach
- [ ] SQL injection prevented (using ORM/parameterized queries)
- [ ] XSS prevention (HTML escaping, CSP)
- [ ] Amount fields validated (min, max, decimal places)
- [ ] File uploads validated (if applicable)
- [ ] Request size limits enforced

## Business Logic
- [ ] Idempotency required for financial operations
- [ ] Race conditions prevented (database locking)
- [ ] Replay attacks prevented (nonce + timestamp)
- [ ] Business rules validated (loan limits, balances)
- [ ] Transaction atomicity ensured

## Data Protection
- [ ] Sensitive data encrypted at rest
- [ ] PII handling compliant with regulations
- [ ] Data retention policy implemented
- [ ] Secure data deletion/anonymization
- [ ] No sensitive data in logs
- [ ] No sensitive data in URLs/query params

## Audit & Monitoring
- [ ] Comprehensive audit logging for financial operations
- [ ] Sensitive data sanitized in logs
- [ ] Failed authentication attempts logged
- [ ] Authorization failures logged
- [ ] Anomaly detection configured
- [ ] Security alerts configured

## Infrastructure
- [ ] Secrets in environment variables/vault (not hardcoded)
- [ ] Debug endpoints removed/protected
- [ ] Swagger documentation protected
- [ ] Database credentials rotated regularly
- [ ] TLS 1.2+ enforced
- [ ] Dependency vulnerabilities scanned

## Compliance
- [ ] OWASP Top 10 vulnerabilities addressed
- [ ] PCI DSS requirements met (if applicable)
- [ ] GDPR compliance (if applicable)
- [ ] SOC 2 controls implemented (if applicable)
```

---

## Summary

This comprehensive API security guide covers:

1. **Authentication**: JWT validation, token revocation, strong passwords, account lockout, MFA
2. **Authorization**: RBAC, IDOR prevention, field-level access control
3. **Input Validation**: Comprehensive validation, injection prevention, amount handling
4. **Network Security**: CORS, HTTPS, CSRF, rate limiting, security headers
5. **Business Logic**: Idempotency, race condition prevention, replay attack prevention
6. **Audit Logging**: Comprehensive audit trails, sensitive data sanitization
7. **Data Protection**: Encryption at rest, PII handling, data retention
8. **Security Testing**: Pre-deployment checklist

## References

1. **OWASP API Security Top 10**: https://owasp.org/www-project-api-security/
2. **PCI DSS**: https://www.pcisecuritystandards.org/
3. **NIST Cybersecurity Framework**: https://www.nist.gov/cyberframework
4. **CWE Top 25**: https://cwe.mitre.org/top25/
5. **Go Security**: https://github.com/guardrailsio/awesome-golang-security

---

**Last Updated**: January 2026  
**Version**: 1.0  
**Maintainers**: Security Team
