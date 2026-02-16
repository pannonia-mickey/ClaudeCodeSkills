---
name: Go Testing
description: This skill should be used when the user asks about "Go tests", "Go table-driven tests", "Go test fixtures", "gomock", "testify", "Go benchmarks", "Go fuzz testing", "Go integration tests", "Go test helpers", or "Go testing patterns". It covers table-driven tests, mocking, benchmarks, fuzz testing, and test organization.
---

# Go Testing

## Table-Driven Tests

```go
func TestParseAmount(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    int64
        wantErr bool
    }{
        {name: "valid dollars", input: "$10.50", want: 1050},
        {name: "valid no cents", input: "$100", want: 10000},
        {name: "zero", input: "$0.00", want: 0},
        {name: "negative", input: "-$5.00", want: -500},
        {name: "invalid format", input: "abc", wantErr: true},
        {name: "empty string", input: "", wantErr: true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseAmount(tt.input)
            if tt.wantErr {
                require.Error(t, err)
                return
            }
            require.NoError(t, err)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

## Testify Assertions and Suites

```go
import (
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/stretchr/testify/suite"
)

// require — stops test on failure (for preconditions)
// assert — continues test on failure (for multiple checks)

func TestUser(t *testing.T) {
    user, err := CreateUser(ctx, req)
    require.NoError(t, err) // Stop if this fails
    require.NotNil(t, user)

    assert.Equal(t, "Alice", user.Name)
    assert.Contains(t, user.Email, "@")
    assert.WithinDuration(t, time.Now(), user.CreatedAt, time.Second)
}

// Test suite with setup/teardown
type UserServiceSuite struct {
    suite.Suite
    db      *sql.DB
    service *UserService
}

func (s *UserServiceSuite) SetupSuite() {
    s.db = setupTestDB(s.T())
    s.service = NewUserService(s.db)
}

func (s *UserServiceSuite) TearDownSuite() {
    s.db.Close()
}

func (s *UserServiceSuite) SetupTest() {
    _, err := s.db.Exec("DELETE FROM users")
    s.Require().NoError(err)
}

func (s *UserServiceSuite) TestCreateUser() {
    user, err := s.service.Create(context.Background(), CreateUserRequest{
        Name:  "Alice",
        Email: "alice@example.com",
    })
    s.Require().NoError(err)
    s.Assert().Equal("Alice", user.Name)
}

func TestUserService(t *testing.T) {
    suite.Run(t, new(UserServiceSuite))
}
```

## Mocking with gomock

```go
//go:generate mockgen -destination=mocks/mock_repo.go -package=mocks . UserRepository

type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
}

func TestGetUser(t *testing.T) {
    ctrl := gomock.NewController(t)
    mockRepo := mocks.NewMockUserRepository(ctrl)

    mockRepo.EXPECT().
        FindByID(gomock.Any(), "123").
        Return(&User{ID: "123", Name: "Alice"}, nil)

    svc := NewUserService(mockRepo)
    user, err := svc.GetUser(context.Background(), "123")
    require.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}
```

## Benchmark Tests

```go
func BenchmarkJSONMarshal(b *testing.B) {
    user := User{ID: "123", Name: "Alice", Email: "alice@example.com"}

    b.ResetTimer()
    for b.Loop() {
        json.Marshal(user)
    }
}

func BenchmarkLookup(b *testing.B) {
    sizes := []int{100, 1000, 10000}
    for _, size := range sizes {
        data := generateData(size)
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            for b.Loop() {
                lookup(data, "target")
            }
        })
    }
}

// Run: go test -bench=. -benchmem ./...
// Output: BenchmarkJSONMarshal-8   5000000   240 ns/op   128 B/op   2 allocs/op
```

## Fuzz Testing (Go 1.18+)

```go
func FuzzParseAmount(f *testing.F) {
    // Seed corpus
    f.Add("$10.50")
    f.Add("$0.00")
    f.Add("-$5.00")
    f.Add("")

    f.Fuzz(func(t *testing.T, input string) {
        result, err := ParseAmount(input)
        if err != nil {
            return // Invalid input is fine
        }
        // Round-trip property: format then parse should match
        formatted := FormatAmount(result)
        reparsed, err := ParseAmount(formatted)
        require.NoError(t, err)
        assert.Equal(t, result, reparsed)
    })
}

// Run: go test -fuzz=FuzzParseAmount -fuzztime=30s ./...
```

## References

- [Test Organization](references/test-organization.md) — Test helpers, fixtures, golden files, testcontainers, build tags, httptest.
- [Advanced Testing](references/advanced-testing.md) — Property-based testing, snapshot testing, coverage analysis, race detection, test caching.
