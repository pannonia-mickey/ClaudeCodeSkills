# Standard Library

## slog — Structured Logging (Go 1.21+)

```go
import "log/slog"

// Configure structured logging
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
    AddSource: true,
}))
slog.SetDefault(logger)

// Structured log entries
slog.Info("request handled",
    "method", r.Method,
    "path", r.URL.Path,
    "status", status,
    "duration_ms", duration.Milliseconds(),
)

// Logger with persistent context
requestLogger := slog.With(
    "request_id", requestID,
    "user_id", userID,
)
requestLogger.Info("processing order", "order_id", orderID)

// Log groups
slog.Info("user created",
    slog.Group("user",
        slog.String("id", user.ID),
        slog.String("email", user.Email),
    ),
)
// Output: {"msg":"user created","user":{"id":"123","email":"a@b.com"}}
```

## net/http — HTTP Server (Go 1.22+)

```go
// Go 1.22+ enhanced routing with method and path params
mux := http.NewServeMux()

mux.HandleFunc("GET /api/users", listUsers)
mux.HandleFunc("POST /api/users", createUser)
mux.HandleFunc("GET /api/users/{id}", getUser)
mux.HandleFunc("PUT /api/users/{id}", updateUser)
mux.HandleFunc("DELETE /api/users/{id}", deleteUser)

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id") // Go 1.22+ path parameter
    // ...
}

// Middleware pattern
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        slog.Info("request",
            "method", r.Method,
            "path", r.URL.Path,
            "duration", time.Since(start),
        )
    })
}

// Chain middleware
handler := loggingMiddleware(recoveryMiddleware(mux))

// Server with graceful shutdown
srv := &http.Server{
    Addr:         ":8080",
    Handler:      handler,
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}

ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        slog.Error("server error", "error", err)
    }
}()

<-ctx.Done()
slog.Info("shutting down")

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
srv.Shutdown(shutdownCtx)
```

## encoding/json

```go
// Custom JSON marshaling
type Status int

const (
    StatusActive Status = iota
    StatusInactive
)

func (s Status) MarshalJSON() ([]byte, error) {
    switch s {
    case StatusActive:
        return json.Marshal("active")
    case StatusInactive:
        return json.Marshal("inactive")
    default:
        return nil, fmt.Errorf("unknown status: %d", s)
    }
}

// Struct tags for JSON control
type User struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    Password  string    `json:"-"`                  // Never serialize
    Name      string    `json:"name,omitempty"`      // Omit if empty
    CreatedAt time.Time `json:"created_at"`
}

// Decode with validation
func decodeJSON[T any](r io.Reader) (T, error) {
    var v T
    dec := json.NewDecoder(r)
    dec.DisallowUnknownFields() // Strict parsing
    if err := dec.Decode(&v); err != nil {
        return v, fmt.Errorf("decoding JSON: %w", err)
    }
    return v, nil
}
```

## context

```go
// Propagate cancellation, deadlines, and values
func (s *Service) ProcessOrder(ctx context.Context, orderID string) error {
    // Respect parent context cancellation
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    // Check cancellation before expensive operations
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }

    order, err := s.repo.FindOrder(ctx, orderID) // Pass ctx to all IO
    if err != nil {
        return err
    }

    // Context values for request-scoped data
    userID := ctx.Value(userIDKey{}).(string)
    _ = userID
    return nil
}

// Type-safe context keys
type userIDKey struct{}

func WithUserID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, userIDKey{}, id)
}

func UserIDFromContext(ctx context.Context) string {
    if v, ok := ctx.Value(userIDKey{}).(string); ok {
        return v
    }
    return ""
}
```

## sync

```go
// sync.Pool — reuse allocations
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func processRequest(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()
    buf.Write(data)
    // Use buf...
}

// sync.Once — lazy initialization
var (
    dbOnce sync.Once
    dbConn *sql.DB
)

func getDB() *sql.DB {
    dbOnce.Do(func() {
        var err error
        dbConn, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
        if err != nil {
            panic(err)
        }
    })
    return dbConn
}

// sync.Map — concurrent map (prefer regular map + mutex for most cases)
var cache sync.Map
cache.Store("key", value)
if v, ok := cache.Load("key"); ok { /* use v */ }
```
