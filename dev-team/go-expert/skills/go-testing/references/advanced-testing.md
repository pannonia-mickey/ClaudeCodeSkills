# Advanced Testing

## Property-Based Testing with rapid

```go
import "pgregory.net/rapid"

func TestSortProperties(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        // Generate random slice
        input := rapid.SliceOf(rapid.Int()).Draw(t, "input")

        sorted := SortInts(input)

        // Property 1: Output length equals input length
        assert.Equal(t, len(input), len(sorted))

        // Property 2: Output is sorted
        for i := 1; i < len(sorted); i++ {
            assert.LessOrEqual(t, sorted[i-1], sorted[i])
        }

        // Property 3: Output is a permutation of input
        inputCopy := make([]int, len(input))
        copy(inputCopy, input)
        sort.Ints(inputCopy)
        assert.Equal(t, inputCopy, sorted)
    })
}

func TestParseFormatRoundTrip(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        // Generate valid amounts
        cents := rapid.Int64Range(0, 999999999).Draw(t, "cents")

        formatted := FormatCents(cents)
        parsed, err := ParseCents(formatted)

        require.NoError(t, err)
        assert.Equal(t, cents, parsed)
    })
}
```

## Coverage Analysis

```bash
# Generate coverage report
go test -coverprofile=coverage.out ./...

# View HTML report
go tool cover -html=coverage.out -o coverage.html

# Coverage by function
go tool cover -func=coverage.out

# Enforce minimum coverage in CI
go test -coverprofile=coverage.out ./...
COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
if (( $(echo "$COVERAGE < 80" | bc -l) )); then
    echo "Coverage $COVERAGE% is below 80% threshold"
    exit 1
fi

# Coverage for specific packages
go test -coverprofile=coverage.out -coverpkg=./internal/... ./...
```

## Race Detection in Tests

```go
// Always run with -race in CI: go test -race ./...

func TestConcurrentMapAccess(t *testing.T) {
    cache := NewCache()
    var wg sync.WaitGroup

    // Concurrent writes
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            cache.Set(fmt.Sprintf("key-%d", n), n)
        }(i)
    }

    // Concurrent reads
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            cache.Get(fmt.Sprintf("key-%d", n))
        }(i)
    }

    wg.Wait()
}
```

## Snapshot Testing

```go
import "github.com/bradleyjkemp/cupaloy/v2"

func TestComplexOutput(t *testing.T) {
    result := generateReport(testInput)
    cupaloy.SnapshotT(t, result)
}

// Update snapshots: UPDATE_SNAPSHOTS=true go test ./...
// Snapshots stored in .snapshots/ directory

// Custom snapshot options
func TestWithOptions(t *testing.T) {
    snapshotter := cupaloy.New(
        cupaloy.SnapshotSubdirectory("testdata/snapshots"),
        cupaloy.SnapshotFileExtension(".json"),
    )
    snapshotter.SnapshotT(t, result)
}
```

## Test Caching and Parallelism

```go
// Mark tests as parallel to run concurrently
func TestUserCreation(t *testing.T) {
    t.Parallel() // Runs in parallel with other parallel tests

    tests := []struct {
        name string
        req  CreateUserRequest
    }{
        {name: "valid", req: CreateUserRequest{Name: "Alice"}},
        {name: "admin", req: CreateUserRequest{Name: "Admin", Role: "admin"}},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // Subtests also run in parallel
            user, err := createUser(tt.req)
            require.NoError(t, err)
            assert.Equal(t, tt.req.Name, user.Name)
        })
    }
}

// Cache busting — force re-run
// go test -count=1 ./...  (disables test caching)

// Verbose output with test timing
// go test -v -count=1 ./...

// Run specific tests
// go test -run TestUser ./internal/user/...
// go test -run TestUser/valid ./internal/user/...
```

## Testing with Interfaces and Fakes

```go
// Prefer interfaces at consumer, not producer
type UserStore interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, user *User) error
}

// Simple fake (often preferred over mocks for readability)
type fakeUserStore struct {
    users map[string]*User
    err   error
}

func newFakeUserStore() *fakeUserStore {
    return &fakeUserStore{users: make(map[string]*User)}
}

func (f *fakeUserStore) FindByID(_ context.Context, id string) (*User, error) {
    if f.err != nil {
        return nil, f.err
    }
    user, ok := f.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return user, nil
}

func (f *fakeUserStore) Save(_ context.Context, user *User) error {
    if f.err != nil {
        return f.err
    }
    f.users[user.ID] = user
    return nil
}

func TestServiceWithFake(t *testing.T) {
    store := newFakeUserStore()
    store.users["123"] = &User{ID: "123", Name: "Alice"}

    svc := NewUserService(store)
    user, err := svc.GetUser(context.Background(), "123")
    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}

// Test error paths
func TestServiceErrorHandling(t *testing.T) {
    store := newFakeUserStore()
    store.err = errors.New("db connection lost")

    svc := NewUserService(store)
    _, err := svc.GetUser(context.Background(), "123")
    assert.Error(t, err)
}
```

## Testing Time-Dependent Code

```go
// Inject a clock interface
type Clock interface {
    Now() time.Time
}

type realClock struct{}
func (realClock) Now() time.Time { return time.Now() }

type fakeClock struct {
    current time.Time
}
func (c *fakeClock) Now() time.Time { return c.current }
func (c *fakeClock) Advance(d time.Duration) { c.current = c.current.Add(d) }

type TokenService struct {
    clock Clock
    ttl   time.Duration
}

func (s *TokenService) IsExpired(token *Token) bool {
    return s.clock.Now().After(token.ExpiresAt)
}

func TestTokenExpiry(t *testing.T) {
    clock := &fakeClock{current: time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)}
    svc := &TokenService{clock: clock, ttl: time.Hour}

    token := &Token{ExpiresAt: clock.Now().Add(time.Hour)}

    assert.False(t, svc.IsExpired(token))

    clock.Advance(2 * time.Hour)
    assert.True(t, svc.IsExpired(token))
}
```
