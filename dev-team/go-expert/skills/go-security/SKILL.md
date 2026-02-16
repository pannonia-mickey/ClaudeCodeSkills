---
name: Go Security
description: This skill should be used when the user asks about "Go security", "Go input validation", "Go SQL injection", "Go crypto", "Go secure HTTP", "Go dependency scanning", "Go CORS", "Go JWT", "Go authentication", or "Go secrets management". It covers input validation, crypto, secure HTTP, dependency scanning, and authentication.
---

# Go Security

## Input Validation

```go
import "github.com/go-playground/validator/v10"

var validate = validator.New()

type CreateOrderRequest struct {
    UserID   string  `json:"user_id" validate:"required,uuid4"`
    Amount   float64 `json:"amount" validate:"required,gt=0,lte=1000000"`
    Currency string  `json:"currency" validate:"required,iso4217"`
    Items    []Item  `json:"items" validate:"required,min=1,max=100,dive"`
}

type Item struct {
    ProductID string `json:"product_id" validate:"required,alphanum,max=64"`
    Quantity  int    `json:"quantity" validate:"required,gte=1,lte=9999"`
}

func (h *Handler) createOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(io.LimitReader(r.Body, 1<<20)).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid request body")
        return
    }

    if err := validate.Struct(req); err != nil {
        writeError(w, http.StatusBadRequest, "validation failed")
        return
    }

    // Safe to use req — all fields validated
}
```

## SQL Injection Prevention

```go
// SAFE — parameterized queries (always use this)
func (r *UserRepo) FindByEmail(ctx context.Context, email string) (*User, error) {
    var user User
    err := r.db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE email = $1", email,
    ).Scan(&user.ID, &user.Name, &user.Email)
    return &user, err
}

// SAFE — using sqlx named queries
func (r *UserRepo) Search(ctx context.Context, filter Filter) ([]User, error) {
    query, args, err := sqlx.Named(
        "SELECT * FROM users WHERE status = :status AND role = :role",
        filter,
    )
    if err != nil {
        return nil, err
    }
    var users []User
    return users, r.db.SelectContext(ctx, &users, r.db.Rebind(query), args...)
}

// DANGEROUS — never concatenate user input into SQL
// BAD: fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", name)
```

## Secure HTTP Server

```go
func newSecureServer(handler http.Handler) *http.Server {
    return &http.Server{
        Addr:              ":8443",
        Handler:           handler,
        ReadTimeout:       5 * time.Second,
        ReadHeaderTimeout: 2 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       120 * time.Second,
        MaxHeaderBytes:    1 << 20, // 1 MB
        TLSConfig: &tls.Config{
            MinVersion: tls.VersionTLS12,
            CurvePreferences: []tls.CurveID{
                tls.X25519,
                tls.CurveP256,
            },
            CipherSuites: []uint16{
                tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
                tls.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
                tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,
                tls.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,
            },
        },
    }
}

// Security headers middleware
func securityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "0")
        w.Header().Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains")
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        next.ServeHTTP(w, r)
    })
}
```

## JWT Authentication

```go
import "github.com/golang-jwt/jwt/v5"

type Claims struct {
    UserID string `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func generateToken(userID, role string, secret []byte) (string, error) {
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "myapp",
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secret)
}

func validateToken(tokenString string, secret []byte) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{},
        func(token *jwt.Token) (any, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
            }
            return secret, nil
        },
    )
    if err != nil {
        return nil, err
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }
    return claims, nil
}
```

## Password Hashing

```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(hash), err
}

func checkPassword(hash, password string) bool {
    return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

## References

- [Crypto Patterns](references/crypto-patterns.md) — AES-GCM encryption, key derivation, secure random, digital signatures, secrets management.
- [Security Hardening](references/security-hardening.md) — Dependency scanning, CORS configuration, rate limiting, CSRF protection, secure file handling.
