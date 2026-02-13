---
name: C# Security
description: This skill should be used when the user asks about "ASP.NET security", "C# authentication", "JWT in .NET", "Data Protection API", "anti-forgery", ".NET security headers", "C# input validation", "secrets management .NET", ".NET authorization", "rate limiting .NET", or "CORS ASP.NET". It covers authentication, authorization, input validation, security headers, secrets management, and secure coding patterns for ASP.NET Core applications.
---

## JWT Authentication

### Configuration

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
            ClockSkew = TimeSpan.FromSeconds(30),
        };
    });

builder.Services.AddAuthorization();
```

### Token Generation Service

```csharp
public class TokenService(IConfiguration config, TimeProvider timeProvider)
{
    public string GenerateAccessToken(User user, IEnumerable<string> roles)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:Key"]!));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var now = timeProvider.GetUtcNow();

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var token = new JwtSecurityToken(
            issuer: config["Jwt:Issuer"],
            audience: config["Jwt:Audience"],
            claims: claims,
            notBefore: now.DateTime,
            expires: now.AddMinutes(15).DateTime,
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        return Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
    }
}
```

---

## Authorization

### Policy-Based Authorization

```csharp
// Program.cs
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"))
    .AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription", "premium", "enterprise"))
    .AddPolicy("MinimumAge", policy =>
        policy.AddRequirements(new MinimumAgeRequirement(18)));
```

### Custom Authorization Handler

```csharp
public record MinimumAgeRequirement(int MinimumAge) : IAuthorizationRequirement;

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dobClaim = context.User.FindFirst("date_of_birth");
        if (dobClaim is null) return Task.CompletedTask;

        if (DateOnly.TryParse(dobClaim.Value, out var dob))
        {
            var age = DateOnly.FromDateTime(DateTime.Today).Year - dob.Year;
            if (age >= requirement.MinimumAge)
            {
                context.Succeed(requirement);
            }
        }

        return Task.CompletedTask;
    }
}
```

### Resource-Based Authorization

```csharp
public class DocumentAuthorizationHandler
    : AuthorizationHandler<OperationAuthorizationRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);

        if (requirement.Name == Operations.Read.Name)
        {
            if (resource.OwnerId == userId || resource.IsPublic)
                context.Succeed(requirement);
        }
        else if (requirement.Name == Operations.Update.Name)
        {
            if (resource.OwnerId == userId)
                context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Usage in a controller or minimal API handler
app.MapPut("/documents/{id}", async (
    int id,
    UpdateDocumentRequest request,
    IAuthorizationService authService,
    ClaimsPrincipal user,
    AppDbContext db) =>
{
    var document = await db.Documents.FindAsync(id);
    if (document is null) return Results.NotFound();

    var result = await authService.AuthorizeAsync(user, document, Operations.Update);
    if (!result.Succeeded) return Results.Forbid();

    document.Title = request.Title;
    await db.SaveChangesAsync();
    return Results.Ok(document);
});
```

---

## Input Validation

### FluentValidation

```csharp
public record CreateProductRequest(string Name, decimal Price, string? Category);

public class CreateProductValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(200)
            .Matches(@"^[\w\s\-\.]+$").WithMessage("Name contains invalid characters");

        RuleFor(x => x.Price)
            .GreaterThan(0)
            .LessThanOrEqualTo(999_999.99m);

        RuleFor(x => x.Category)
            .MaximumLength(100)
            .When(x => x.Category is not null);
    }
}

// Register as endpoint filter
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductValidator>();
```

### Over-Posting Prevention

Never bind directly to entity models. Use DTOs:

```csharp
// DANGEROUS — allows setting IsAdmin via mass assignment
app.MapPost("/users", async (User user, AppDbContext db) => { ... });

// SAFE — DTO limits which fields can be set
public record CreateUserRequest(string Email, string Name, string Password);

app.MapPost("/users", async (CreateUserRequest request, AppDbContext db) =>
{
    var user = new User
    {
        Email = request.Email,
        Name = request.Name,
        PasswordHash = hasher.Hash(request.Password),
        IsAdmin = false, // explicitly set, never from client
    };
    db.Users.Add(user);
    await db.SaveChangesAsync();
    return Results.Created($"/users/{user.Id}", user.ToDto());
});
```

---

## Security Headers

### Security Headers Middleware

```csharp
app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;
    headers["X-Content-Type-Options"] = "nosniff";
    headers["X-Frame-Options"] = "DENY";
    headers["X-XSS-Protection"] = "0";
    headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
    headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";
    headers["Content-Security-Policy"] =
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data: https:; frame-ancestors 'none'";
    headers.Remove("Server");
    headers.Remove("X-Powered-By");
    await next();
});

// HSTS (in production only)
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}
app.UseHttpsRedirection();
```

---

## CORS Configuration

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
        policy
            .WithOrigins("https://app.example.com", "https://admin.example.com")
            .WithMethods("GET", "POST", "PUT", "PATCH", "DELETE")
            .WithHeaders("Authorization", "Content-Type", "X-Request-ID")
            .WithExposedHeaders("X-Request-ID", "X-Total-Count")
            .SetPreflightMaxAge(TimeSpan.FromHours(1))
            .AllowCredentials());

    options.AddPolicy("Development", policy =>
        policy
            .WithOrigins("http://localhost:3000", "http://localhost:5173")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials());
});

// Apply based on environment
app.UseCors(app.Environment.IsDevelopment() ? "Development" : "Production");
```

---

## Secrets Management

### Development — User Secrets

```bash
dotnet user-secrets init
dotnet user-secrets set "Jwt:Key" "your-secret-key-here"
dotnet user-secrets set "ConnectionStrings:Database" "Server=...;Password=..."
```

```csharp
// Automatically loaded in Development environment
builder.Configuration.AddUserSecrets<Program>();
```

### Production — Azure Key Vault

```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
    new DefaultAzureCredential());
```

### Never Hardcode Secrets

```csharp
// DANGEROUS
var connectionString = "Server=prod;Database=app;Password=secret123";

// SAFE — from configuration
var connectionString = builder.Configuration.GetConnectionString("Database");
```

---

## Rate Limiting

### Built-in Rate Limiting (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
    });

    options.AddSlidingWindowLimiter("auth", limiter =>
    {
        limiter.PermitLimit = 5;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.SegmentsPerWindow = 3;
        limiter.QueueLimit = 0;
    });

    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await context.HttpContext.Response.WriteAsJsonAsync(
            new { error = "Rate limit exceeded. Try again later." }, ct);
    };
});

app.UseRateLimiter();

// Apply to endpoints
app.MapPost("/auth/login", LoginHandler).RequireRateLimiting("auth");
app.MapGet("/api/products", ListProducts).RequireRateLimiting("api");
```

---

## References

- **[Authentication and Authorization](references/authentication-authorization.md)** — Complete JWT setup, custom authorization handlers, API key auth, multi-scheme authentication, OAuth2/OIDC, account lockout, and two-factor authentication.
- **[Secure Coding Practices](references/secure-coding.md)** — Data Protection API, anti-forgery tokens, HTTPS enforcement, SQL injection prevention, XSS output encoding, file upload security, and Azure Key Vault integration.
