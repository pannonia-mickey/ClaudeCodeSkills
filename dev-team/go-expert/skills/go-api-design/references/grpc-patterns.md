# gRPC Patterns

## Streaming RPCs

```protobuf
service ChatService {
    // Server streaming — server sends multiple responses
    rpc Subscribe(SubscribeRequest) returns (stream Event);
    // Client streaming — client sends multiple requests
    rpc Upload(stream Chunk) returns (UploadResponse);
    // Bidirectional streaming
    rpc Chat(stream Message) returns (stream Message);
}
```

```go
// Server streaming
func (s *chatServer) Subscribe(req *pb.SubscribeRequest, stream pb.ChatService_SubscribeServer) error {
    ch := s.broker.Subscribe(req.GetRoomId())
    defer s.broker.Unsubscribe(ch)

    for {
        select {
        case event := <-ch:
            if err := stream.Send(event); err != nil {
                return status.Errorf(codes.Internal, "send failed: %v", err)
            }
        case <-stream.Context().Done():
            return nil
        }
    }
}

// Client streaming
func (s *chatServer) Upload(stream pb.ChatService_UploadServer) error {
    var totalSize int64
    var buf bytes.Buffer

    for {
        chunk, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&pb.UploadResponse{
                Size: totalSize,
            })
        }
        if err != nil {
            return status.Errorf(codes.Internal, "recv failed: %v", err)
        }
        totalSize += int64(len(chunk.GetData()))
        buf.Write(chunk.GetData())
    }
}

// Bidirectional streaming
func (s *chatServer) Chat(stream pb.ChatService_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }

        reply := processMessage(msg)
        if err := stream.Send(reply); err != nil {
            return err
        }
    }
}
```

## Interceptors (Middleware)

```go
// Unary server interceptor
func loggingInterceptor(
    ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    duration := time.Since(start)

    slog.Info("gRPC request",
        "method", info.FullMethod,
        "duration_ms", duration.Milliseconds(),
        "error", err,
    )
    return resp, err
}

// Auth interceptor
func authInterceptor(
    ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (any, error) {
    // Skip auth for health checks
    if info.FullMethod == "/grpc.health.v1.Health/Check" {
        return handler(ctx, req)
    }

    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing token")
    }

    claims, err := validateToken(tokens[0])
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }

    ctx = context.WithValue(ctx, userClaimsKey{}, claims)
    return handler(ctx, req)
}

// Chain interceptors
srv := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        loggingInterceptor,
        authInterceptor,
        recoveryInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        streamLoggingInterceptor,
        streamAuthInterceptor,
    ),
)
```

## Health Checking

```go
import "google.golang.org/grpc/health"
import healthpb "google.golang.org/grpc/health/grpc_health_v1"

func setupHealthCheck(srv *grpc.Server) *health.Server {
    healthServer := health.NewServer()
    healthpb.RegisterHealthServer(srv, healthServer)

    // Set service status
    healthServer.SetServingStatus("myapp.UserService", healthpb.HealthCheckResponse_SERVING)

    return healthServer
}

// Kubernetes gRPC health probe (K8s 1.24+)
// livenessProbe:
//   grpc:
//     port: 50051
```

## Error Details

```go
import (
    "google.golang.org/genproto/googleapis/rpc/errdetails"
    "google.golang.org/grpc/status"
)

// Rich error responses
func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
    // Field-level validation errors
    var violations []*errdetails.BadRequest_FieldViolation

    if req.GetName() == "" {
        violations = append(violations, &errdetails.BadRequest_FieldViolation{
            Field:       "name",
            Description: "name is required",
        })
    }
    if !isValidEmail(req.GetEmail()) {
        violations = append(violations, &errdetails.BadRequest_FieldViolation{
            Field:       "email",
            Description: "invalid email format",
        })
    }

    if len(violations) > 0 {
        st := status.New(codes.InvalidArgument, "validation failed")
        detailed, _ := st.WithDetails(&errdetails.BadRequest{
            FieldViolations: violations,
        })
        return nil, detailed.Err()
    }

    // Business logic error with retry info
    if rateLimited {
        st := status.New(codes.ResourceExhausted, "rate limited")
        detailed, _ := st.WithDetails(&errdetails.RetryInfo{
            RetryDelay: durationpb.New(5 * time.Second),
        })
        return nil, detailed.Err()
    }

    return createUser(ctx, req)
}
```

## Deadline Propagation

```go
// Client sets deadline
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &pb.GetUserRequest{Id: "123"})
if err != nil {
    st := status.Convert(err)
    if st.Code() == codes.DeadlineExceeded {
        // Handle timeout
    }
}

// Server checks remaining deadline
func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    deadline, ok := ctx.Deadline()
    if ok && time.Until(deadline) < 100*time.Millisecond {
        return nil, status.Error(codes.DeadlineExceeded, "insufficient time remaining")
    }

    // Pass context to downstream calls — deadline propagates automatically
    user, err := s.repo.FindByID(ctx, req.GetId())
    if err != nil {
        return nil, err
    }
    return toProto(user), nil
}
```

## Reflection and Testing

```go
// Enable reflection for grpcurl/grpcui debugging
import "google.golang.org/grpc/reflection"

srv := grpc.NewServer()
pb.RegisterUserServiceServer(srv, &userServer{})
reflection.Register(srv) // Only in dev/staging

// grpcurl usage:
// grpcurl -plaintext localhost:50051 list
// grpcurl -plaintext -d '{"id":"123"}' localhost:50051 myapp.UserService/GetUser

// Testing with bufconn (in-memory connection)
import "google.golang.org/grpc/test/bufconn"

func setupTestServer(t *testing.T) *grpc.ClientConn {
    lis := bufconn.Listen(1024 * 1024)

    srv := grpc.NewServer()
    pb.RegisterUserServiceServer(srv, newTestServer())
    go srv.Serve(lis)

    t.Cleanup(func() { srv.Stop() })

    conn, err := grpc.NewClient("passthrough:///bufconn",
        grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
            return lis.DialContext(ctx)
        }),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    require.NoError(t, err)
    t.Cleanup(func() { conn.Close() })

    return conn
}

func TestGetUser(t *testing.T) {
    conn := setupTestServer(t)
    client := pb.NewUserServiceClient(conn)

    resp, err := client.GetUser(context.Background(), &pb.GetUserRequest{Id: "123"})
    require.NoError(t, err)
    assert.Equal(t, "123", resp.GetId())
}
```
