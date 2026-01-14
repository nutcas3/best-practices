# 100 Web Security Attacks - Detection and Mitigation Guide

## Skill Metadata
- **Domain**: Web Security, Penetration Testing, Vulnerability Assessment
- **Skill Level**: Advanced
- **Last Updated**: 2026
- **Sources**: OWASP Top 10, Bug Bounty Best Practices, CVE Database

## Overview

Comprehensive guide covering 100 common web application security vulnerabilities with detection methods and mitigation strategies for Go and Rust applications.

---

## INJECTION ATTACKS

### 1. SQL Injection

**Attack**: Injecting malicious SQL code through user inputs.

```sql
-- Attack example
' OR '1'='1' --
'; DROP TABLE users; --
```

**Mitigation (Go)**:
```go
// ❌ BAD: String concatenation
query := "SELECT * FROM users WHERE email = '" + email + "'"

// ✅ GOOD: Parameterized queries
db.Query("SELECT * FROM users WHERE email = $1", email)

// ✅ GOOD: Using ORM
db.Where("email = ?", email).First(&user)
```

**Mitigation (Rust)**:
```rust
// ✅ GOOD: SQLx with compile-time checked queries
sqlx::query_as!(User, "SELECT * FROM users WHERE email = $1", email)
    .fetch_one(&pool)
    .await?;
```

### 2. NoSQL Injection

**Attack**: Exploiting NoSQL query syntax.

```javascript
// Attack payload
{"$ne": null}
{"$gt": ""}
```

**Mitigation (Go)**:
```go
// ✅ GOOD: Validate and sanitize input
func validateMongoQuery(input map[string]interface{}) error {
    for key := range input {
        if strings.HasPrefix(key, "$") {
            return errors.New("operator not allowed")
        }
    }
    return nil
}
```

### 3. Command Injection

**Attack**: Executing arbitrary system commands.

```bash
; rm -rf /
| cat /etc/passwd
```

**Mitigation (Go)**:
```go
// ❌ BAD: Using shell
cmd := exec.Command("sh", "-c", "ping "+userInput)

// ✅ GOOD: Direct command with validated args
if !isValidHostname(userInput) {
    return errors.New("invalid hostname")
}
cmd := exec.Command("ping", "-c", "4", userInput)
```

### 4. LDAP Injection

**Mitigation**:
```go
// ✅ GOOD: Escape LDAP special characters
func escapeLDAP(input string) string {
    replacer := strings.NewReplacer(
        "\\", "\\5c",
        "*", "\\2a",
        "(", "\\28",
        ")", "\\29",
        "\x00", "\\00",
    )
    return replacer.Replace(input)
}
```

### 5. XML Injection / XXE

**Mitigation (Go)**:
```go
// ✅ GOOD: Disable external entities
decoder := xml.NewDecoder(reader)
decoder.Entity = xml.HTMLEntity
decoder.Strict = false
decoder.AutoClose = xml.HTMLAutoClose
```

### 6. Server-Side Template Injection (SSTI)

**Mitigation (Go)**:
```go
// ✅ GOOD: Use safe template rendering
tmpl := template.Must(template.New("page").Parse(templateString))
tmpl.Execute(w, safeData)

// ✅ GOOD: Escape user input
template.HTMLEscapeString(userInput)
```

---

## XSS ATTACKS

### 7-16. Cross-Site Scripting (All Types)

**Attack Vectors**:
- Reflected XSS
- Stored XSS
- DOM-based XSS
- Mutation XSS

**Mitigation (Go)**:
```go
// ✅ GOOD: Content Security Policy
func SetSecurityHeaders(c *fiber.Ctx) error {
    c.Set("Content-Security-Policy", 
        "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'")
    c.Set("X-Content-Type-Options", "nosniff")
    c.Set("X-Frame-Options", "DENY")
    c.Set("X-XSS-Protection", "1; mode=block")
    return c.Next()
}

// ✅ GOOD: HTML escaping
import "html"
safeOutput := html.EscapeString(userInput)

// ✅ GOOD: Using bluemonday for rich content
import "github.com/microcosm-cc/bluemonday"
policy := bluemonday.UGCPolicy()
clean := policy.Sanitize(userInput)
```

### 17. DOM Clobbering

**Mitigation**: Use unique IDs, avoid global variables, validate DOM access.

### 18. Prototype Pollution

**Mitigation (JavaScript/TypeScript)**:
```javascript
// ✅ GOOD: Freeze prototypes
Object.freeze(Object.prototype);

// ✅ GOOD: Use Map instead of objects
const safeMap = new Map();
```

---

## CSRF & REQUEST FORGERY

### 19. Cross-Site Request Forgery

**Mitigation (Go)**:
```go
// ✅ GOOD: CSRF token middleware
import "github.com/gofiber/fiber/v2/middleware/csrf"

app.Use(csrf.New(csrf.Config{
    KeyLookup:      "header:X-CSRF-Token",
    CookieName:     "csrf_",
    CookieSameSite: "Strict",
    CookieSecure:   true,
    CookieHTTPOnly: true,
    Expiration:     1 * time.Hour,
}))
```

### 20. Login CSRF

**Mitigation**: Require CSRF token on login forms.

---

## AUTHENTICATION ATTACKS

### 21-30. Authentication Vulnerabilities

**Brute Force Protection**:
```go
// ✅ GOOD: Account lockout
type LoginAttemptTracker struct {
    redis *redis.Client
}

func (lat *LoginAttemptTracker) RecordFailedAttempt(email string) error {
    key := fmt.Sprintf("login:attempts:%s", email)
    
    count, _ := lat.redis.Incr(context.Background(), key).Result()
    lat.redis.Expire(context.Background(), key, 15*time.Minute)
    
    if count >= 5 {
        lat.redis.Set(context.Background(), 
            fmt.Sprintf("login:locked:%s", email),
            "1",
            15*time.Minute,
        )
        return errors.New("account locked")
    }
    
    return nil
}
```

**Password Policy**:
```go
// ✅ GOOD: Strong password validation
func ValidatePassword(password string) error {
    if len(password) < 12 {
        return errors.New("password must be at least 12 characters")
    }
    
    var hasUpper, hasLower, hasDigit, hasSpecial bool
    for _, char := range password {
        switch {
        case unicode.IsUpper(char): hasUpper = true
        case unicode.IsLower(char): hasLower = true
        case unicode.IsDigit(char): hasDigit = true
        case unicode.IsPunct(char) || unicode.IsSymbol(char): hasSpecial = true
        }
    }
    
    if !hasUpper || !hasLower || !hasDigit || !hasSpecial {
        return errors.New("password must contain uppercase, lowercase, digit, and special character")
    }
    
    return nil
}
```

**MFA Bypass Prevention**:
```go
// ✅ GOOD: Enforce MFA for sensitive operations
func RequireMFA() fiber.Handler {
    return func(c *fiber.Ctx) error {
        claims := c.Locals("claims").(*Claims)
        
        if !claims.MFAVerified {
            return c.Status(403).JSON(fiber.Map{
                "error": "MFA verification required",
            })
        }
        
        return c.Next()
    }
}
```

---

## SESSION ATTACKS

### 31-35. Session Management Vulnerabilities

**Session Fixation Prevention**:
```go
// ✅ GOOD: Regenerate session ID after login
func (s *SessionService) RegenerateSession(c *fiber.Ctx) error {
    oldSession, _ := s.store.Get(c)
    userData := oldSession.Get("user_data")
    
    // Destroy old session
    oldSession.Destroy()
    
    // Create new session
    newSession, _ := s.store.Get(c)
    newSession.Set("user_data", userData)
    newSession.Set("created_at", time.Now())
    
    return newSession.Save()
}
```

**Secure Session Configuration**:
```go
// ✅ GOOD: Secure session settings
store := session.New(session.Config{
    Expiration:     24 * time.Hour,
    CookieHTTPOnly: true,
    CookieSecure:   true,
    CookieSameSite: "Strict",
    KeyGenerator: func() string {
        b := make([]byte, 32)
        rand.Read(b)
        return base64.URLEncoding.EncodeToString(b)
    },
})
```

---

## JWT ATTACKS

### 36-39. JWT Vulnerabilities

**"none" Algorithm Attack Prevention**:
```go
// ✅ GOOD: Validate algorithm
func ValidateToken(tokenString string) (*Claims, error) {
    claims := &Claims{}
    
    token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
        // Validate algorithm
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return []byte(os.Getenv("JWT_SECRET")), nil
    })
    
    if err != nil || !token.Valid {
        return nil, errors.New("invalid token")
    }
    
    return claims, nil
}
```

**HS256/RS256 Confusion Prevention**:
```go
// ✅ GOOD: Explicitly specify algorithm
validation := jwt.NewParser(jwt.WithValidMethods([]string{"HS256"}))
```

---

## ACCESS CONTROL

### 40-50. Authorization Vulnerabilities

**IDOR Prevention**:
```go
// ✅ GOOD: Verify resource ownership
func (h *Handler) GetResource(c *fiber.Ctx) error {
    claims := c.Locals("claims").(*Claims)
    resourceID := c.Params("id")
    
    resource, err := h.db.GetResource(resourceID)
    if err != nil {
        return c.Status(404).JSON(fiber.Map{"error": "Not found"})
    }
    
    // Verify ownership
    if resource.OwnerID != claims.UserID && claims.Role != "admin" {
        return c.Status(403).JSON(fiber.Map{"error": "Access denied"})
    }
    
    return c.JSON(resource)
}
```

**Mass Assignment Prevention**:
```go
// ✅ GOOD: Explicit field updates
type UpdateUserRequest struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    // Do NOT include: Role, IsAdmin, etc.
}

func (h *Handler) UpdateUser(c *fiber.Ctx) error {
    var req UpdateUserRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
    }
    
    // Only update allowed fields
    updates := map[string]interface{}{
        "name":  req.Name,
        "email": req.Email,
    }
    
    return h.db.UpdateUser(userID, updates)
}
```

---

## FILE UPLOAD ATTACKS

### 51-60. File Upload Vulnerabilities

**Secure File Upload**:
```go
// ✅ GOOD: Comprehensive file upload validation
func (h *Handler) UploadFile(c *fiber.Ctx) error {
    file, err := c.FormFile("file")
    if err != nil {
        return c.Status(400).JSON(fiber.Map{"error": "No file uploaded"})
    }
    
    // 1. Validate file size
    if file.Size > 10*1024*1024 { // 10MB
        return c.Status(400).JSON(fiber.Map{"error": "File too large"})
    }
    
    // 2. Validate file type by content (not extension)
    src, _ := file.Open()
    defer src.Close()
    
    buffer := make([]byte, 512)
    src.Read(buffer)
    src.Seek(0, 0)
    
    mimeType := http.DetectContentType(buffer)
    allowedTypes := []string{"image/jpeg", "image/png", "image/gif"}
    
    if !contains(allowedTypes, mimeType) {
        return c.Status(400).JSON(fiber.Map{"error": "Invalid file type"})
    }
    
    // 3. Generate secure filename
    ext := filepath.Ext(file.Filename)
    filename := fmt.Sprintf("%s%s", uuid.New().String(), ext)
    
    // 4. Store outside web root
    uploadPath := "/var/uploads/" + filename
    
    // 5. Scan for malware (if applicable)
    // scanForMalware(uploadPath)
    
    return c.SaveFile(file, uploadPath)
}
```

**Path Traversal Prevention**:
```go
// ✅ GOOD: Validate file paths
func ValidateFilePath(userPath string) error {
    cleanPath := filepath.Clean(userPath)
    
    if strings.Contains(cleanPath, "..") {
        return errors.New("path traversal detected")
    }
    
    if !strings.HasPrefix(cleanPath, "/safe/directory/") {
        return errors.New("access denied")
    }
    
    return nil
}
```

---

## SSRF ATTACKS

### 61-68. Server-Side Request Forgery

**SSRF Prevention**:
```go
// ✅ GOOD: Validate and restrict URLs
func ValidateURL(urlStr string) error {
    u, err := url.Parse(urlStr)
    if err != nil {
        return err
    }
    
    // Block private IPs
    host := u.Hostname()
    ip := net.ParseIP(host)
    
    if ip != nil {
        if ip.IsPrivate() || ip.IsLoopback() || ip.IsLinkLocalUnicast() {
            return errors.New("private IP not allowed")
        }
    }
    
    // Whitelist allowed domains
    allowedDomains := []string{"api.example.com", "cdn.example.com"}
    if !contains(allowedDomains, host) {
        return errors.New("domain not allowed")
    }
    
    return nil
}

// ✅ GOOD: Safe HTTP client
func MakeSafeRequest(urlStr string) (*http.Response, error) {
    if err := ValidateURL(urlStr); err != nil {
        return nil, err
    }
    
    client := &http.Client{
        Timeout: 5 * time.Second,
        Transport: &http.Transport{
            DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
                // Additional IP validation
                host, _, _ := net.SplitHostPort(addr)
                ip := net.ParseIP(host)
                
                if ip != nil && (ip.IsPrivate() || ip.IsLoopback()) {
                    return nil, errors.New("private IP blocked")
                }
                
                return (&net.Dialer{
                    Timeout: 5 * time.Second,
                }).DialContext(ctx, network, addr)
            },
        },
    }
    
    return client.Get(urlStr)
}
```

---

## DESERIALIZATION ATTACKS

### 69-73. Insecure Deserialization

**Safe Deserialization (Go)**:
```go
// ✅ GOOD: Validate before deserializing
func SafeDeserialize(data []byte) (*User, error) {
    var user User
    
    // Use JSON instead of gob for untrusted data
    if err := json.Unmarshal(data, &user); err != nil {
        return nil, err
    }
    
    // Validate deserialized data
    if err := validateUser(&user); err != nil {
        return nil, err
    }
    
    return &user, nil
}
```

---

## LOGIC FLAWS

### 74-76. Business Logic Vulnerabilities

**Race Condition Prevention**:
```go
// ✅ GOOD: Use database transactions with locking
func (s *Service) Transfer(from, to string, amount decimal.Decimal) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // Lock accounts in consistent order
        accounts := []string{from, to}
        sort.Strings(accounts)
        
        var acc1, acc2 Account
        tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            Where("id = ?", accounts[0]).First(&acc1)
        tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            Where("id = ?", accounts[1]).First(&acc2)
        
        // Perform transfer logic
        return nil
    })
}
```

**Workflow Bypass Prevention**:
```go
// ✅ GOOD: Validate state transitions
func (s *OrderService) SkipToShipped(orderID string) error {
    order, _ := s.db.GetOrder(orderID)
    
    // Enforce workflow
    if order.Status != "processing" {
        return errors.New("order must be in processing state")
    }
    
    // Additional validations
    if !order.PaymentConfirmed {
        return errors.New("payment not confirmed")
    }
    
    if !order.InventoryReserved {
        return errors.New("inventory not reserved")
    }
    
    return s.db.UpdateOrderStatus(orderID, "shipped")
}
```

---

## HTTP ATTACKS

### 77-78. HTTP Request Smuggling & Cache Poisoning

**Prevention**:
```go
// ✅ GOOD: Normalize headers
func NormalizeHeaders(c *fiber.Ctx) error {
    // Remove conflicting headers
    c.Request().Header.Del("Transfer-Encoding")
    c.Request().Header.Del("Content-Length")
    
    // Set proper content length
    c.Request().Header.Set("Content-Length", 
        fmt.Sprintf("%d", len(c.Body())))
    
    return c.Next()
}

// ✅ GOOD: Cache control
func SetCacheHeaders(c *fiber.Ctx) error {
    // Prevent cache poisoning
    c.Set("Cache-Control", "private, no-cache, no-store, must-revalidate")
    c.Set("Vary", "Accept-Encoding, Accept-Language")
    
    return c.Next()
}
```

---

## ADDITIONAL VULNERABILITIES

### 79-100. Other Critical Vulnerabilities

**Security Headers**:
```go
// ✅ GOOD: Comprehensive security headers
func SecurityHeaders() fiber.Handler {
    return func(c *fiber.Ctx) error {
        c.Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains; preload")
        c.Set("X-Content-Type-Options", "nosniff")
        c.Set("X-Frame-Options", "DENY")
        c.Set("X-XSS-Protection", "1; mode=block")
        c.Set("Content-Security-Policy", "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'")
        c.Set("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        
        return c.Next()
    }
}
```

**Input Validation**:
```go
// ✅ GOOD: Comprehensive validation
import "github.com/go-playground/validator/v10"

type Request struct {
    Email    string `validate:"required,email"`
    Username string `validate:"required,alphanum,min=3,max=20"`
    Age      int    `validate:"required,min=18,max=120"`
    URL      string `validate:"omitempty,url"`
}

var validate = validator.New()

func ValidateRequest(req interface{}) error {
    return validate.Struct(req)
}
```

**Secrets Management**:
```go
// ✅ GOOD: Use environment variables or secret managers
func LoadSecrets() {
    // Never hardcode secrets
    apiKey := os.Getenv("API_KEY")
    dbPassword := os.Getenv("DB_PASSWORD")
    
    // Or use secret manager
    // secret, _ := secretManager.GetSecret("api-key")
}
```

---

## Security Testing Checklist

### Automated Testing

```go
// ✅ GOOD: Security tests
func TestSQLInjection(t *testing.T) {
    payloads := []string{
        "' OR '1'='1",
        "'; DROP TABLE users; --",
        "1' UNION SELECT * FROM users--",
    }
    
    for _, payload := range payloads {
        _, err := GetUser(payload)
        assert.Error(t, err, "Should reject SQL injection")
    }
}

func TestXSS(t *testing.T) {
    payloads := []string{
        "<script>alert('XSS')</script>",
        "<img src=x onerror=alert('XSS')>",
        "javascript:alert('XSS')",
    }
    
    for _, payload := range payloads {
        output := SanitizeHTML(payload)
        assert.NotContains(t, output, "<script>")
        assert.NotContains(t, output, "javascript:")
    }
}
```

---

## Summary

This guide covers 100 common web security vulnerabilities:

1. **Injection Attacks** (1-18): SQL, NoSQL, Command, LDAP, XML, SSTI
2. **XSS Attacks** (7-18): Reflected, Stored, DOM-based, Mutation
3. **CSRF** (19-20): Request forgery, Login CSRF
4. **Authentication** (21-30): Brute force, weak passwords, MFA bypass
5. **Session Management** (31-35): Fixation, hijacking, weak entropy
6. **JWT Vulnerabilities** (36-39): Algorithm attacks, weak secrets
7. **Access Control** (40-50): IDOR, privilege escalation, mass assignment
8. **File Upload** (51-60): Unrestricted upload, path traversal
9. **SSRF** (61-68): Cloud metadata, internal access
10. **Deserialization** (69-73): RCE via deserialization
11. **Logic Flaws** (74-76): Race conditions, workflow bypass
12. **HTTP Attacks** (77-78): Request smuggling, cache poisoning
13. **Additional** (79-100): Security headers, input validation, secrets

## References

1. **OWASP Top 10**: https://owasp.org/www-project-top-ten/
2. **OWASP API Security**: https://owasp.org/www-project-api-security/
3. **PortSwigger Web Security**: https://portswigger.net/web-security
4. **HackerOne Reports**: https://hackerone.com/hacktivity

---

**Last Updated**: January 2026  
**Version**: 1.0
