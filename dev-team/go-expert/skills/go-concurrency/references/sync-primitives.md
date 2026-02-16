# Sync Primitives

## Mutex and RWMutex

```go
// Protect shared state with Mutex
type SafeCounter struct {
    mu    sync.Mutex
    count map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count[key]++
}

func (c *SafeCounter) Get(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count[key]
}

// RWMutex for read-heavy workloads
type Cache struct {
    mu    sync.RWMutex
    items map[string]any
}

func (c *Cache) Get(key string) (any, bool) {
    c.mu.RLock()         // Multiple readers allowed
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}

func (c *Cache) Set(key string, value any) {
    c.mu.Lock()          // Exclusive write access
    defer c.mu.Unlock()
    c.items[key] = value
}
```

## Atomic Operations

```go
import "sync/atomic"

// Atomic counter — faster than mutex for simple counters
type Metrics struct {
    requestCount  atomic.Int64
    errorCount    atomic.Int64
    totalDuration atomic.Int64
}

func (m *Metrics) RecordRequest(duration time.Duration, hasError bool) {
    m.requestCount.Add(1)
    m.totalDuration.Add(duration.Nanoseconds())
    if hasError {
        m.errorCount.Add(1)
    }
}

func (m *Metrics) Stats() (requests, errors int64, avgDuration time.Duration) {
    requests = m.requestCount.Load()
    errors = m.errorCount.Load()
    total := m.totalDuration.Load()
    if requests > 0 {
        avgDuration = time.Duration(total / requests)
    }
    return
}

// Atomic value for config hot-reload
var currentConfig atomic.Value // stores *Config

func init() {
    currentConfig.Store(loadConfig())
}

func getConfig() *Config {
    return currentConfig.Load().(*Config)
}

func reloadConfig() {
    newCfg := loadConfig()
    currentConfig.Store(newCfg) // Atomic swap — no locks needed
}
```

## WaitGroup Patterns

```go
// Basic WaitGroup
func processAll(items []Item) {
    var wg sync.WaitGroup
    for _, item := range items {
        wg.Add(1)
        go func(it Item) {
            defer wg.Done()
            process(it)
        }(item)
    }
    wg.Wait()
}

// WaitGroup with error collection
func processAllWithErrors(items []Item) []error {
    var (
        wg   sync.WaitGroup
        mu   sync.Mutex
        errs []error
    )

    for _, item := range items {
        wg.Add(1)
        go func(it Item) {
            defer wg.Done()
            if err := process(it); err != nil {
                mu.Lock()
                errs = append(errs, err)
                mu.Unlock()
            }
        }(item)
    }

    wg.Wait()
    return errs
}
```

## sync.Once Patterns

```go
// Lazy initialization
type DBPool struct {
    once sync.Once
    pool *pgxpool.Pool
}

func (d *DBPool) Get(ctx context.Context) (*pgxpool.Pool, error) {
    var initErr error
    d.once.Do(func() {
        d.pool, initErr = pgxpool.New(ctx, os.Getenv("DATABASE_URL"))
    })
    if initErr != nil {
        return nil, initErr
    }
    return d.pool, nil
}

// OnceValue (Go 1.21+) — simpler lazy init
var getDB = sync.OnceValue(func() *sql.DB {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        panic(err) // Or use sync.OnceValues for error return
    }
    return db
})

// sync.OnceValues — with error
var loadConfig = sync.OnceValues(func() (*Config, error) {
    data, err := os.ReadFile("config.yaml")
    if err != nil {
        return nil, err
    }
    var cfg Config
    return &cfg, yaml.Unmarshal(data, &cfg)
})
```

## Race Detector

```bash
# Run tests with race detector
go test -race ./...

# Run binary with race detector
go run -race main.go

# Build with race detector (for CI)
go build -race -o myapp ./cmd/myapp

# Common races detected:
# - Map read/write without synchronization
# - Struct field access from multiple goroutines
# - Slice append from multiple goroutines
# - Closing a channel from multiple goroutines
```

## sync.Cond — Waiting for Conditions

```go
// Wait until a condition is met
type Queue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []Item
}

func NewQueue() *Queue {
    q := &Queue{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Push(item Item) {
    q.mu.Lock()
    q.items = append(q.items, item)
    q.mu.Unlock()
    q.cond.Signal() // Wake one waiting consumer
}

func (q *Queue) Pop() Item {
    q.mu.Lock()
    defer q.mu.Unlock()
    for len(q.items) == 0 {
        q.cond.Wait() // Release lock, sleep, reacquire on wake
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```
