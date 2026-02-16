# Go Patterns

## Functional Options

```go
// Flexible, extensible configuration
type Server struct {
    addr         string
    readTimeout  time.Duration
    writeTimeout time.Duration
    logger       *slog.Logger
}

type Option func(*Server)

func WithAddr(addr string) Option {
    return func(s *Server) { s.addr = addr }
}

func WithTimeouts(read, write time.Duration) Option {
    return func(s *Server) {
        s.readTimeout = read
        s.writeTimeout = write
    }
}

func WithLogger(logger *slog.Logger) Option {
    return func(s *Server) { s.logger = logger }
}

func NewServer(opts ...Option) *Server {
    s := &Server{
        addr:         ":8080",
        readTimeout:  5 * time.Second,
        writeTimeout: 10 * time.Second,
        logger:       slog.Default(),
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage
srv := NewServer(
    WithAddr(":3000"),
    WithTimeouts(10*time.Second, 30*time.Second),
    WithLogger(logger),
)
```

## Type Embedding

```go
// Embed types for composition (not inheritance)
type BaseModel struct {
    ID        string    `json:"id" db:"id"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

type User struct {
    BaseModel
    Name  string `json:"name" db:"name"`
    Email string `json:"email" db:"email"`
}

// Embed interfaces for delegation
type LoggingRepository struct {
    UserRepository  // embedded interface
    logger *slog.Logger
}

func (r *LoggingRepository) FindByID(ctx context.Context, id string) (*User, error) {
    r.logger.Info("finding user", "id", id)
    user, err := r.UserRepository.FindByID(ctx, id)
    if err != nil {
        r.logger.Error("find user failed", "id", id, "error", err)
    }
    return user, err
}
```

## Dependency Injection (Manual)

```go
// Wire dependencies in main.go — no framework needed
func main() {
    // Infrastructure
    db := setupDatabase()
    defer db.Close()
    cache := setupRedis()
    defer cache.Close()

    // Repositories
    userRepo := repository.NewUserRepository(db)
    orderRepo := repository.NewOrderRepository(db)

    // Services
    userService := service.NewUserService(userRepo, cache)
    orderService := service.NewOrderService(orderRepo, userRepo)

    // Handlers
    userHandler := handler.NewUserHandler(userService)
    orderHandler := handler.NewOrderHandler(orderService)

    // Router
    mux := http.NewServeMux()
    userHandler.RegisterRoutes(mux)
    orderHandler.RegisterRoutes(mux)

    // Server
    srv := &http.Server{
        Addr:    ":8080",
        Handler: middleware.Chain(mux, middleware.Logger, middleware.Recovery),
    }
    // ...
}
```

## Configuration with Viper

```go
type Config struct {
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
    Redis    RedisConfig    `mapstructure:"redis"`
}

type ServerConfig struct {
    Port         int           `mapstructure:"port"`
    ReadTimeout  time.Duration `mapstructure:"read_timeout"`
    WriteTimeout time.Duration `mapstructure:"write_timeout"`
}

func LoadConfig() (*Config, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    viper.AddConfigPath("/etc/myservice")

    // Environment variables override config file
    viper.SetEnvPrefix("APP")
    viper.AutomaticEnv()
    viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))

    // Defaults
    viper.SetDefault("server.port", 8080)
    viper.SetDefault("server.read_timeout", "5s")

    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, fmt.Errorf("reading config: %w", err)
        }
    }

    var cfg Config
    if err := viper.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("unmarshaling config: %w", err)
    }

    return &cfg, nil
}
```

## Result Pattern

```go
// Generic result type for operations that can fail
type Result[T any] struct {
    Value T
    Err   error
}

func Ok[T any](value T) Result[T] {
    return Result[T]{Value: value}
}

func Fail[T any](err error) Result[T] {
    return Result[T]{Err: err}
}

// Channel of results for concurrent operations
func processItems(items []Item) <-chan Result[ProcessedItem] {
    results := make(chan Result[ProcessedItem], len(items))
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()
            processed, err := process(item)
            if err != nil {
                results <- Fail[ProcessedItem](err)
                return
            }
            results <- Ok(processed)
        }(item)
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```
