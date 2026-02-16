# Pipeline Patterns

## Fan-Out / Fan-In

```go
// Fan-out: distribute work to multiple goroutines
// Fan-in: merge results from multiple channels into one

func fanOut[T any](in <-chan T, workers int) []<-chan T {
    channels := make([]<-chan T, workers)
    for i := 0; i < workers; i++ {
        channels[i] = filterWork(in)
    }
    return channels
}

func fanIn[T any](channels ...<-chan T) <-chan T {
    merged := make(chan T)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan T) {
            defer wg.Done()
            for v := range c {
                merged <- v
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}

// Complete pipeline example
func processPipeline(ctx context.Context, input []string) ([]Result, error) {
    // Stage 1: Generate work
    source := generator(ctx, input)

    // Stage 2: Fan-out to workers
    const numWorkers = 5
    workers := make([]<-chan Result, numWorkers)
    for i := 0; i < numWorkers; i++ {
        workers[i] = worker(ctx, source)
    }

    // Stage 3: Fan-in results
    merged := fanIn(workers...)

    // Stage 4: Collect
    var results []Result
    for r := range merged {
        results = append(results, r)
    }
    return results, nil
}
```

## Rate-Limited Pipeline

```go
import "golang.org/x/time/rate"

func rateLimitedWorker(ctx context.Context, limiter *rate.Limiter, jobs <-chan Job) <-chan Result {
    results := make(chan Result)
    go func() {
        defer close(results)
        for job := range jobs {
            if err := limiter.Wait(ctx); err != nil {
                return // Context cancelled
            }
            results <- processJob(job)
        }
    }()
    return results
}

// Usage: 100 requests per second with burst of 10
limiter := rate.NewLimiter(rate.Limit(100), 10)
results := rateLimitedWorker(ctx, limiter, jobs)
```

## Bounded Concurrency with Semaphore

```go
import "golang.org/x/sync/semaphore"

func processWithLimit(ctx context.Context, items []Item, maxConcurrent int64) error {
    sem := semaphore.NewWeighted(maxConcurrent)
    g, ctx := errgroup.WithContext(ctx)

    for _, item := range items {
        if err := sem.Acquire(ctx, 1); err != nil {
            return err
        }
        g.Go(func() error {
            defer sem.Release(1)
            return processItem(ctx, item)
        })
    }

    return g.Wait()
}
```

## Pipeline with Backpressure

```go
// Buffered channels provide natural backpressure
// When buffer is full, producers block until consumers catch up

func pipeline(ctx context.Context) error {
    // Stage 1 → Stage 2: buffer of 100
    stage1Out := make(chan RawData, 100)
    // Stage 2 → Stage 3: buffer of 50
    stage2Out := make(chan ProcessedData, 50)
    // Stage 3 → output: buffer of 25
    stage3Out := make(chan FinalResult, 25)

    g, ctx := errgroup.WithContext(ctx)

    // Stage 1: Read data
    g.Go(func() error {
        defer close(stage1Out)
        return readData(ctx, stage1Out)
    })

    // Stage 2: Process (multiple workers)
    g.Go(func() error {
        defer close(stage2Out)
        return processData(ctx, stage1Out, stage2Out, 5)
    })

    // Stage 3: Write results
    g.Go(func() error {
        defer close(stage3Out)
        return writeResults(ctx, stage2Out, stage3Out)
    })

    // Collect final results
    g.Go(func() error {
        for result := range stage3Out {
            slog.Info("completed", "id", result.ID)
        }
        return nil
    })

    return g.Wait()
}
```

## Graceful Shutdown with Drain

```go
func runWorkers(ctx context.Context, jobs <-chan Job) {
    var wg sync.WaitGroup

    for i := 0; i < runtime.NumCPU(); i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for {
                select {
                case job, ok := <-jobs:
                    if !ok {
                        return // Channel closed
                    }
                    processJob(job)
                case <-ctx.Done():
                    // Drain remaining jobs before exiting
                    for job := range jobs {
                        processJob(job)
                    }
                    return
                }
            }
        }(i)
    }

    wg.Wait()
    slog.Info("all workers stopped")
}
```
