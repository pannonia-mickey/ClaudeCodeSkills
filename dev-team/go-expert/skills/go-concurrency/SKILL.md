---
name: Go Concurrency
description: This skill should be used when the user asks about "goroutines", "Go channels", "Go select", "sync primitives", "Go context", "errgroup", "worker pools", "fan-out fan-in", "Go race conditions", or "Go concurrency patterns". It covers goroutines, channels, sync primitives, worker pools, and concurrent pipeline design.
---

# Go Concurrency

## Goroutines and Channels

```go
// Producer-consumer with typed channels
func producer(ctx context.Context, items []string) <-chan string {
    ch := make(chan string)
    go func() {
        defer close(ch)
        for _, item := range items {
            select {
            case ch <- item:
            case <-ctx.Done():
                return
            }
        }
    }()
    return ch
}

func consumer(ctx context.Context, in <-chan string) <-chan Result {
    out := make(chan Result)
    go func() {
        defer close(out)
        for item := range in {
            select {
            case out <- process(item):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

## Worker Pool

```go
func workerPool[T, R any](ctx context.Context, workers int, jobs <-chan T, fn func(T) R) <-chan R {
    results := make(chan R, workers)
    var wg sync.WaitGroup

    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                select {
                case results <- fn(job):
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

## errgroup — Concurrent Error Handling

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]Response, error) {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // Max 10 concurrent goroutines

    results := make([]Response, len(urls))

    for i, url := range urls {
        g.Go(func() error {
            resp, err := fetchURL(ctx, url)
            if err != nil {
                return fmt.Errorf("fetching %s: %w", url, err)
            }
            results[i] = resp
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err // First error cancels ctx, stops other goroutines
    }

    return results, nil
}
```

## Select with Timeout

```go
func processWithTimeout(ctx context.Context) error {
    resultCh := make(chan Result, 1)

    go func() {
        resultCh <- expensiveOperation()
    }()

    select {
    case result := <-resultCh:
        return handleResult(result)
    case <-ctx.Done():
        return ctx.Err()
    case <-time.After(5 * time.Second):
        return fmt.Errorf("operation timed out")
    }
}

// Ticker for periodic work
func periodicTask(ctx context.Context, interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            doWork()
        case <-ctx.Done():
            return
        }
    }
}
```

## References

- [Pipeline Patterns](references/pipeline-patterns.md) — Fan-out/fan-in, pipeline stages, rate limiting, bounded concurrency, graceful shutdown.
- [Sync Primitives](references/sync-primitives.md) — Mutex, RWMutex, WaitGroup, Once, Cond, atomic operations, race detector usage.
