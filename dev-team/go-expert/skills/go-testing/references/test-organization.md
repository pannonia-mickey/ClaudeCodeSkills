# Test Organization

## Test Helpers

```go
// t.Helper() marks function as test helper — errors report caller's line
func createTestUser(t *testing.T, db *sql.DB, name string) *User {
    t.Helper()
    user := &User{ID: uuid.New().String(), Name: name, Email: name + "@test.com"}
    _, err := db.Exec("INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
        user.ID, user.Name, user.Email)
    require.NoError(t, err)
    t.Cleanup(func() {
        db.Exec("DELETE FROM users WHERE id = $1", user.ID)
    })
    return user
}

// TestMain for package-level setup
func TestMain(m *testing.M) {
    db := setupTestDatabase()
    defer db.Close()
    os.Exit(m.Run())
}
```

## Golden Files

```go
import "github.com/sebdah/goldie/v2"

func TestRenderTemplate(t *testing.T) {
    g := goldie.New(t, goldie.WithFixtureDir("testdata"))

    result := renderTemplate(data)
    g.Assert(t, "expected_output", []byte(result))
}

// Manual golden file pattern
func TestJSONOutput(t *testing.T) {
    got := generateReport(testData)
    golden := filepath.Join("testdata", t.Name()+".golden")

    if *update {
        os.WriteFile(golden, got, 0644)
    }

    want, err := os.ReadFile(golden)
    require.NoError(t, err)
    assert.JSONEq(t, string(want), string(got))
}

var update = flag.Bool("update", false, "update golden files")
```

## Testcontainers

```go
import "github.com/testcontainers/testcontainers-go"
import "github.com/testcontainers/testcontainers-go/modules/postgres"

func setupPostgres(t *testing.T) *sql.DB {
    t.Helper()
    ctx := context.Background()

    pgContainer, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready").WithOccurrence(2),
        ),
    )
    require.NoError(t, err)
    t.Cleanup(func() { pgContainer.Terminate(ctx) })

    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    db, err := sql.Open("postgres", connStr)
    require.NoError(t, err)
    t.Cleanup(func() { db.Close() })

    // Run migrations
    runMigrations(t, db)
    return db
}

func setupRedis(t *testing.T) *redis.Client {
    t.Helper()
    ctx := context.Background()

    redisContainer, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "redis:7-alpine",
            ExposedPorts: []string{"6379/tcp"},
            WaitingFor:   wait.ForLog("Ready to accept connections"),
        },
        Started: true,
    })
    require.NoError(t, err)
    t.Cleanup(func() { redisContainer.Terminate(ctx) })

    endpoint, err := redisContainer.Endpoint(ctx, "")
    require.NoError(t, err)

    return redis.NewClient(&redis.Options{Addr: endpoint})
}
```

## Build Tags for Test Separation

```go
//go:build integration

package myapp_test

// Only runs with: go test -tags=integration ./...
func TestDatabaseIntegration(t *testing.T) {
    db := setupPostgres(t)
    repo := NewUserRepo(db)

    user, err := repo.Create(context.Background(), &User{Name: "Alice"})
    require.NoError(t, err)

    found, err := repo.FindByID(context.Background(), user.ID)
    require.NoError(t, err)
    assert.Equal(t, "Alice", found.Name)
}
```

## httptest — Testing HTTP Handlers

```go
func TestGetUser(t *testing.T) {
    // Setup
    handler := setupRouter()

    // Create request
    req := httptest.NewRequest(http.MethodGet, "/api/users/123", nil)
    req.Header.Set("Authorization", "Bearer test-token")
    w := httptest.NewRecorder()

    // Execute
    handler.ServeHTTP(w, req)

    // Assert
    assert.Equal(t, http.StatusOK, w.Code)

    var user User
    err := json.Unmarshal(w.Body.Bytes(), &user)
    require.NoError(t, err)
    assert.Equal(t, "123", user.ID)
}

func TestCreateUser(t *testing.T) {
    body := `{"name": "Alice", "email": "alice@example.com"}`
    req := httptest.NewRequest(http.MethodPost, "/api/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()

    handler.ServeHTTP(w, req)

    assert.Equal(t, http.StatusCreated, w.Code)
}

// Test server for client code
func TestAPIClient(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "/api/users/123", r.URL.Path)
        json.NewEncoder(w).Encode(User{ID: "123", Name: "Alice"})
    }))
    defer srv.Close()

    client := NewAPIClient(srv.URL)
    user, err := client.GetUser(context.Background(), "123")
    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}
```

## Test Fixtures and Data Builders

```go
// Builder pattern for test data
type UserBuilder struct {
    user User
}

func NewUserBuilder() *UserBuilder {
    return &UserBuilder{
        user: User{
            ID:    uuid.New().String(),
            Name:  "Test User",
            Email: "test@example.com",
            Role:  "user",
        },
    }
}

func (b *UserBuilder) WithName(name string) *UserBuilder {
    b.user.Name = name
    return b
}

func (b *UserBuilder) WithRole(role string) *UserBuilder {
    b.user.Role = role
    return b
}

func (b *UserBuilder) Build() User {
    return b.user
}

// Usage in tests
func TestAdminAccess(t *testing.T) {
    admin := NewUserBuilder().WithName("Admin").WithRole("admin").Build()
    assert.True(t, admin.HasPermission("manage_users"))
}
```
