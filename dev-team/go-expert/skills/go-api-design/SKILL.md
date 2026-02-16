---
name: Go API Design
description: This skill should be used when the user asks about "Go HTTP handler", "Go middleware", "Gin", "Echo", "Go REST API", "Go gRPC", "Go OpenAPI", "Go JSON handling", "Go validation", or "Go error responses". It covers HTTP handlers, middleware, framework patterns, gRPC, and API best practices.
---

# Go API Design

## net/http Handlers (Go 1.22+)

```go
func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("GET /api/users", listUsers)
    mux.HandleFunc("POST /api/users", createUser)
    mux.HandleFunc("GET /api/users/{id}", getUser)
    mux.HandleFunc("PUT /api/users/{id}", updateUser)
    mux.HandleFunc("DELETE /api/users/{id}", deleteUser)

    handler := middleware.Chain(mux,
        middleware.RequestID,
        middleware.Logger,
        middleware.Recovery,
        middleware.CORS,
    )

    srv := &http.Server{Addr: ":8080", Handler: handler}
    log.Fatal(srv.ListenAndServe())
}

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    user, err := userService.FindByID(r.Context(), id)
    if errors.Is(err, ErrNotFound) {
        writeJSON(w, http.StatusNotFound, ErrorResponse{Error: "user not found"})
        return
    }
    if err != nil {
        writeJSON(w, http.StatusInternalServerError, ErrorResponse{Error: "internal error"})
        return
    }

    writeJSON(w, http.StatusOK, user)
}

func writeJSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(v)
}
```

## Gin Framework

```go
func setupRouter(userHandler *UserHandler) *gin.Engine {
    r := gin.New()
    r.Use(gin.Recovery(), gin.Logger())

    api := r.Group("/api")
    api.Use(authMiddleware())
    {
        users := api.Group("/users")
        users.GET("", userHandler.List)
        users.POST("", userHandler.Create)
        users.GET("/:id", userHandler.Get)
        users.PUT("/:id", userHandler.Update)
    }

    return r
}

type UserHandler struct {
    service UserService
}

func (h *UserHandler) Create(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := h.service.Create(c.Request.Context(), req)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal error"})
        return
    }

    c.JSON(http.StatusCreated, user)
}

func authMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        claims, err := validateToken(token)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            return
        }
        c.Set("user_id", claims.UserID)
        c.Next()
    }
}
```

## Middleware Pattern

```go
type Middleware func(http.Handler) http.Handler

func Chain(handler http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        handler = middlewares[i](handler)
    }
    return handler
}

func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = uuid.New().String()
        }
        ctx := context.WithValue(r.Context(), requestIDKey{}, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## gRPC Service

```protobuf
// proto/user/v1/user.proto
syntax = "proto3";
package user.v1;
option go_package = "myapp/gen/user/v1";

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
    rpc CreateUser(CreateUserRequest) returns (User);
}

message User {
    string id = 1;
    string name = 2;
    string email = 3;
}
```

```go
type userServer struct {
    userv1.UnimplementedUserServiceServer
    repo UserRepository
}

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.User, error) {
    user, err := s.repo.FindByID(ctx, req.GetId())
    if errors.Is(err, ErrNotFound) {
        return nil, status.Error(codes.NotFound, "user not found")
    }
    if err != nil {
        return nil, status.Error(codes.Internal, "internal error")
    }
    return toProto(user), nil
}
```

## References

- [API Patterns](references/api-patterns.md) — Pagination, filtering, error responses, versioning, OpenAPI generation, request validation.
- [gRPC Patterns](references/grpc-patterns.md) — Streaming, interceptors, health checking, reflection, error details, deadline propagation.
