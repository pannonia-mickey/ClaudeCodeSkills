# Security Hardening

## Dependency Scanning

```bash
# govulncheck — official Go vulnerability scanner
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...

# Output shows only vulnerabilities that affect your code paths
# (not just all vulnerabilities in dependencies)

# CI integration
govulncheck -format json ./... | jq '.vulns // empty'

# Keep dependencies updated
go get -u ./...
go mod tidy

# Audit go.sum for integrity
go mod verify
```

```yaml
# GitHub Actions security scanning
- name: Run govulncheck
  uses: golang/govulncheck-action@v1
  with:
    go-version-input: "1.22"
    go-package: ./...
```

## CORS Configuration

```go
func corsMiddleware(allowedOrigins []string) func(http.Handler) http.Handler {
    originSet := make(map[string]bool)
    for _, o := range allowedOrigins {
        originSet[o] = true
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            origin := r.Header.Get("Origin")

            if originSet[origin] {
                w.Header().Set("Access-Control-Allow-Origin", origin)
                w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
                w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
                w.Header().Set("Access-Control-Max-Age", "86400")
                w.Header().Set("Access-Control-Allow-Credentials", "true")
                w.Header().Set("Vary", "Origin")
            }

            if r.Method == http.MethodOptions {
                w.WriteHeader(http.StatusNoContent)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

// Using rs/cors library
import "github.com/rs/cors"

c := cors.New(cors.Options{
    AllowedOrigins:   []string{"https://app.example.com"},
    AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE"},
    AllowedHeaders:   []string{"Authorization", "Content-Type"},
    AllowCredentials: true,
    MaxAge:           86400,
})
handler := c.Handler(mux)
```

## Rate Limiting Per Client

```go
import "golang.org/x/time/rate"

type IPRateLimiter struct {
    mu       sync.RWMutex
    limiters map[string]*rateLimiterEntry
    rate     rate.Limit
    burst    int
}

type rateLimiterEntry struct {
    limiter  *rate.Limiter
    lastSeen time.Time
}

func NewIPRateLimiter(rps float64, burst int) *IPRateLimiter {
    rl := &IPRateLimiter{
        limiters: make(map[string]*rateLimiterEntry),
        rate:     rate.Limit(rps),
        burst:    burst,
    }
    // Cleanup old entries every minute
    go rl.cleanup()
    return rl
}

func (rl *IPRateLimiter) Allow(ip string) bool {
    rl.mu.Lock()
    entry, exists := rl.limiters[ip]
    if !exists {
        entry = &rateLimiterEntry{
            limiter: rate.NewLimiter(rl.rate, rl.burst),
        }
        rl.limiters[ip] = entry
    }
    entry.lastSeen = time.Now()
    rl.mu.Unlock()
    return entry.limiter.Allow()
}

func (rl *IPRateLimiter) cleanup() {
    for range time.Tick(time.Minute) {
        rl.mu.Lock()
        for ip, entry := range rl.limiters {
            if time.Since(entry.lastSeen) > 3*time.Minute {
                delete(rl.limiters, ip)
            }
        }
        rl.mu.Unlock()
    }
}

func rateLimitMiddleware(limiter *IPRateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := realIP(r)
            if !limiter.Allow(ip) {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

func realIP(r *http.Request) string {
    if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
        return strings.TrimSpace(strings.Split(xff, ",")[0])
    }
    ip, _, _ := net.SplitHostPort(r.RemoteAddr)
    return ip
}
```

## CSRF Protection

```go
import "github.com/gorilla/csrf"

// Token-based CSRF protection
func setupCSRF(handler http.Handler) http.Handler {
    csrfKey := []byte(os.Getenv("CSRF_KEY")) // 32 bytes
    return csrf.Protect(csrfKey,
        csrf.Secure(true),
        csrf.HttpOnly(true),
        csrf.SameSite(csrf.SameSiteStrictMode),
        csrf.Path("/"),
    )(handler)
}

// For APIs: use SameSite cookies + custom header check
func csrfAPIMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method != http.MethodGet && r.Method != http.MethodHead {
            // Require custom header that can't be set by forms
            if r.Header.Get("X-Requested-With") != "XMLHttpRequest" {
                http.Error(w, "CSRF validation failed", http.StatusForbidden)
                return
            }
        }
        next.ServeHTTP(w, r)
    })
}
```

## Secure File Handling

```go
// Limit upload size
func uploadHandler(w http.ResponseWriter, r *http.Request) {
    // Limit total request body to 10 MB
    r.Body = http.MaxBytesReader(w, r.Body, 10<<20)

    if err := r.ParseMultipartForm(10 << 20); err != nil {
        http.Error(w, "File too large", http.StatusRequestEntityTooLarge)
        return
    }

    file, header, err := r.FormFile("upload")
    if err != nil {
        http.Error(w, "Invalid file", http.StatusBadRequest)
        return
    }
    defer file.Close()

    // Validate content type by reading magic bytes (not trusting headers)
    buf := make([]byte, 512)
    n, _ := file.Read(buf)
    contentType := http.DetectContentType(buf[:n])

    allowedTypes := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/gif":  true,
        "application/pdf": true,
    }

    if !allowedTypes[contentType] {
        http.Error(w, "File type not allowed", http.StatusBadRequest)
        return
    }

    // Reset file reader
    file.Seek(0, io.SeekStart)

    // Generate safe filename — never use user-provided filename directly
    ext := filepath.Ext(header.Filename)
    safeFilename := uuid.New().String() + ext

    // Ensure no path traversal
    destPath := filepath.Join("/uploads", filepath.Base(safeFilename))
    // ...
}

// Path traversal prevention
func safePath(base, userInput string) (string, error) {
    // Clean and resolve the path
    cleaned := filepath.Clean(userInput)
    full := filepath.Join(base, cleaned)

    // Verify it's still under base directory
    if !strings.HasPrefix(full, filepath.Clean(base)+string(os.PathSeparator)) {
        return "", errors.New("path traversal detected")
    }
    return full, nil
}
```

## Request Body Limits and Timeouts

```go
// Always limit request body size
func limitBody(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1 MB default
        next.ServeHTTP(w, r)
    })
}

// Context timeout per request
func timeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.TimeoutHandler(next, timeout, `{"error":"request timeout"}`)
    }
}
```
