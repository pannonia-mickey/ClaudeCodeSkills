---
name: Go Mastery
description: This skill should be used when the user asks about "Go interfaces", "Go error handling", "Go struct embedding", "Go generics", "Go modules", "Go standard library", "idiomatic Go", "Go project structure", or "Go best practices". It covers core Go patterns, interfaces, error handling, generics, and project organization.
---

# Go Mastery

## Interfaces and Composition

```go
// Small, focused interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}

// Accept interfaces, return structs
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}

type userRepo struct {
    db *sqlx.DB
}

func NewUserRepository(db *sqlx.DB) UserRepository {
    return &userRepo{db: db}
}

func (r *userRepo) FindByID(ctx context.Context, id string) (*User, error) {
    var user User
    err := r.db.GetContext(ctx, &user, "SELECT * FROM users WHERE id = $1", id)
    if errors.Is(err, sql.ErrNoRows) {
        return nil, ErrNotFound
    }
    return &user, err
}
```

## Error Handling

```go
// Sentinel errors for expected cases
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrConflict     = errors.New("conflict")
)

// Wrap errors with context
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    user, err := s.users.FindByID(ctx, req.UserID)
    if err != nil {
        return nil, fmt.Errorf("finding user %s: %w", req.UserID, err)
    }

    order, err := s.orders.Save(ctx, newOrder(user, req))
    if err != nil {
        return nil, fmt.Errorf("saving order: %w", err)
    }

    return order, nil
}

// Check wrapped errors
if errors.Is(err, ErrNotFound) {
    http.Error(w, "not found", http.StatusNotFound)
    return
}

// Custom error types
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation: %s: %s", e.Field, e.Message)
}

var ve *ValidationError
if errors.As(err, &ve) {
    // Handle validation error specifically
}
```

## Generics (Go 1.18+)

```go
// Generic data structures
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable](items ...T) *Set[T] {
    s := &Set[T]{items: make(map[T]struct{}, len(items))}
    for _, item := range items {
        s.items[item] = struct{}{}
    }
    return s
}

func (s *Set[T]) Add(item T) { s.items[item] = struct{}{} }
func (s *Set[T]) Has(item T) bool { _, ok := s.items[item]; return ok }

// Generic utility functions
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

func Filter[T any](slice []T, fn func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}
```

## Project Structure

```
myservice/
├── cmd/
│   └── myservice/
│       └── main.go          # Entry point
├── internal/
│   ├── handler/             # HTTP/gRPC handlers
│   │   └── user.go
│   ├── service/             # Business logic
│   │   └── user.go
│   ├── repository/          # Data access
│   │   └── user.go
│   ├── middleware/           # HTTP middleware
│   │   ├── auth.go
│   │   └── logging.go
│   └── model/               # Domain types
│       └── user.go
├── pkg/                     # Public libraries (if any)
├── migrations/              # Database migrations
├── go.mod
├── go.sum
└── Makefile
```

## References

- [Go Patterns](references/go-patterns.md) — Functional options, builder pattern, type embedding, dependency injection, configuration management.
- [Standard Library](references/standard-library.md) — slog, net/http, encoding/json, context, sync, io, os/exec, text/template.
