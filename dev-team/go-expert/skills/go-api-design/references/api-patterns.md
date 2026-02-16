# API Patterns

## Pagination

```go
// Cursor-based pagination (preferred for large datasets)
type PageRequest struct {
    Cursor string `json:"cursor"`
    Limit  int    `json:"limit"`
}

type PageResponse[T any] struct {
    Items      []T    `json:"items"`
    NextCursor string `json:"next_cursor,omitempty"`
    HasMore    bool   `json:"has_more"`
}

func (h *Handler) listUsers(w http.ResponseWriter, r *http.Request) {
    cursor := r.URL.Query().Get("cursor")
    limit := 20
    if l, err := strconv.Atoi(r.URL.Query().Get("limit")); err == nil && l > 0 && l <= 100 {
        limit = l
    }

    users, nextCursor, err := h.repo.ListUsers(r.Context(), cursor, limit+1)
    if err != nil {
        writeError(w, http.StatusInternalServerError, "failed to list users")
        return
    }

    hasMore := len(users) > limit
    if hasMore {
        users = users[:limit]
    }

    writeJSON(w, http.StatusOK, PageResponse[User]{
        Items:      users,
        NextCursor: nextCursor,
        HasMore:    hasMore,
    })
}

// Offset-based pagination (simpler, less efficient for deep pages)
type OffsetPage[T any] struct {
    Items      []T `json:"items"`
    Total      int `json:"total"`
    Page       int `json:"page"`
    PageSize   int `json:"page_size"`
    TotalPages int `json:"total_pages"`
}
```

## Request Validation

```go
import "github.com/go-playground/validator/v10"

var validate = validator.New()

type CreateUserRequest struct {
    Name     string `json:"name" validate:"required,min=2,max=100"`
    Email    string `json:"email" validate:"required,email"`
    Age      int    `json:"age" validate:"omitempty,gte=0,lte=150"`
    Role     string `json:"role" validate:"required,oneof=admin user moderator"`
    Password string `json:"password" validate:"required,min=8"`
}

func decodeAndValidate[T any](r *http.Request) (T, error) {
    var v T
    dec := json.NewDecoder(r.Body)
    dec.DisallowUnknownFields()

    if err := dec.Decode(&v); err != nil {
        return v, fmt.Errorf("invalid JSON: %w", err)
    }
    if err := validate.Struct(v); err != nil {
        return v, fmt.Errorf("validation failed: %w", err)
    }
    return v, nil
}

// Validation error formatting
func formatValidationErrors(err error) map[string]string {
    errors := make(map[string]string)
    var ve validator.ValidationErrors
    if stderrors.As(err, &ve) {
        for _, fe := range ve {
            field := strings.ToLower(fe.Field())
            switch fe.Tag() {
            case "required":
                errors[field] = fmt.Sprintf("%s is required", field)
            case "email":
                errors[field] = "invalid email format"
            case "min":
                errors[field] = fmt.Sprintf("%s must be at least %s characters", field, fe.Param())
            default:
                errors[field] = fmt.Sprintf("%s failed %s validation", field, fe.Tag())
            }
        }
    }
    return errors
}
```

## Error Responses

```go
// Standard error response
type ErrorResponse struct {
    Error   string            `json:"error"`
    Code    string            `json:"code,omitempty"`
    Details map[string]string `json:"details,omitempty"`
}

// Application errors mapped to HTTP status
type AppError struct {
    Code    int
    Message string
    Err     error
}

func (e *AppError) Error() string { return e.Message }
func (e *AppError) Unwrap() error { return e.Err }

var (
    ErrNotFound     = &AppError{Code: http.StatusNotFound, Message: "resource not found"}
    ErrUnauthorized = &AppError{Code: http.StatusUnauthorized, Message: "unauthorized"}
    ErrForbidden    = &AppError{Code: http.StatusForbidden, Message: "forbidden"}
    ErrConflict     = &AppError{Code: http.StatusConflict, Message: "resource already exists"}
)

// Handler wrapper that converts errors to HTTP responses
func handleError(w http.ResponseWriter, err error) {
    var appErr *AppError
    if errors.As(err, &appErr) {
        writeJSON(w, appErr.Code, ErrorResponse{Error: appErr.Message})
        return
    }
    slog.Error("unhandled error", "error", err)
    writeJSON(w, http.StatusInternalServerError, ErrorResponse{Error: "internal server error"})
}
```

## API Versioning

```go
// URL path versioning
func setupRoutes(mux *http.ServeMux) {
    // v1 endpoints
    mux.HandleFunc("GET /api/v1/users", v1ListUsers)
    mux.HandleFunc("GET /api/v1/users/{id}", v1GetUser)

    // v2 endpoints with breaking changes
    mux.HandleFunc("GET /api/v2/users", v2ListUsers)
    mux.HandleFunc("GET /api/v2/users/{id}", v2GetUser)
}

// Header-based versioning
func versionMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        version := r.Header.Get("API-Version")
        if version == "" {
            version = "2024-01-01" // Default to latest stable
        }
        ctx := context.WithValue(r.Context(), apiVersionKey{}, version)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## OpenAPI Generation

```go
// Using swaggo/swag for Gin
// Install: go install github.com/swaggo/swag/cmd/swag@latest

// @title           User API
// @version         1.0
// @description     User management service
// @BasePath        /api/v1

// @Summary      Get user by ID
// @Description  Retrieve a single user
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        id   path      string  true  "User ID"
// @Success      200  {object}  User
// @Failure      404  {object}  ErrorResponse
// @Router       /users/{id} [get]
func (h *UserHandler) Get(c *gin.Context) {
    // implementation
}

// Generate docs: swag init -g cmd/api/main.go
// Serve: r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
```

## Content Negotiation

```go
func writeResponse(w http.ResponseWriter, r *http.Request, status int, v any) {
    accept := r.Header.Get("Accept")
    switch {
    case strings.Contains(accept, "application/xml"):
        w.Header().Set("Content-Type", "application/xml")
        w.WriteHeader(status)
        xml.NewEncoder(w).Encode(v)
    default:
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(status)
        json.NewEncoder(w).Encode(v)
    }
}
```

## Rate Limiting

```go
import "golang.org/x/time/rate"

func rateLimitMiddleware(rps float64, burst int) func(http.Handler) http.Handler {
    limiter := rate.NewLimiter(rate.Limit(rps), burst)

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                w.Header().Set("Retry-After", "1")
                writeJSON(w, http.StatusTooManyRequests, ErrorResponse{
                    Error: "rate limit exceeded",
                })
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}

// Per-client rate limiting
type clientLimiter struct {
    mu       sync.Mutex
    clients  map[string]*rate.Limiter
    rps      rate.Limit
    burst    int
}

func (cl *clientLimiter) getLimiter(clientID string) *rate.Limiter {
    cl.mu.Lock()
    defer cl.mu.Unlock()
    if l, exists := cl.clients[clientID]; exists {
        return l
    }
    l := rate.NewLimiter(cl.rps, cl.burst)
    cl.clients[clientID] = l
    return l
}
```
