---
name: .NET Security
description: This skill should be used when the user asks about "ASP.NET security", "JWT in .NET", "authentication .NET", "authorization .NET", "Data Protection API", "anti-forgery", "security headers .NET", "input validation C#", "secrets management .NET", "rate limiting .NET", "CORS ASP.NET", or "OWASP .NET". It covers authentication, authorization, input validation, security headers, secrets management, and secure coding for ASP.NET Core.
---

# .NET Security

## JWT Authentication

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true, ValidateAudience = true, ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
            ClockSkew = TimeSpan.FromSeconds(30),
        };
    });
```

### Token Generation

```csharp
public class TokenService(IConfiguration config, TimeProvider timeProvider)
{
    public string GenerateAccessToken(User user, IEnumerable<string> roles)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:Key"]!));
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var token = new JwtSecurityToken(
            issuer: config["Jwt:Issuer"], audience: config["Jwt:Audience"], claims: claims,
            notBefore: timeProvider.GetUtcNow().DateTime,
            expires: timeProvider.GetUtcNow().AddMinutes(15).DateTime,
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));
        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken() => Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
}
```

## Policy-Based Authorization

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"))
    .AddPolicy("PremiumUser", policy => policy.RequireClaim("subscription", "premium", "enterprise"))
    .AddPolicy("MinimumAge", policy => policy.AddRequirements(new MinimumAgeRequirement(18)));

// Custom handler
public record MinimumAgeRequirement(int MinimumAge) : IAuthorizationRequirement;

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext context, MinimumAgeRequirement req)
    {
        var dob = context.User.FindFirst("date_of_birth");
        if (dob is not null && DateOnly.TryParse(dob.Value, out var date))
        {
            if (DateOnly.FromDateTime(DateTime.Today).Year - date.Year >= req.MinimumAge)
                context.Succeed(req);
        }
        return Task.CompletedTask;
    }
}
```

## Resource-Based Authorization

```csharp
public class DocumentAuthorizationHandler : AuthorizationHandler<OperationAuthorizationRequirement, Document>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext context,
        OperationAuthorizationRequirement req, Document resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (req.Name == Operations.Update.Name && resource.OwnerId == userId)
            context.Succeed(req);
        return Task.CompletedTask;
    }
}
```

## Input Validation

```csharp
public class CreateProductValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200)
            .Matches(@"^[\w\s\-\.]+$").WithMessage("Name contains invalid characters");
        RuleFor(x => x.Price).GreaterThan(0).LessThanOrEqualTo(999_999.99m);
    }
}

// Over-posting prevention — always use DTOs, never bind directly to entities
public record CreateUserRequest(string Email, string Name, string Password);
```

## Security Headers

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
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; frame-ancestors 'none'";
    headers.Remove("Server");
    headers.Remove("X-Powered-By");
    await next();
});
```

## CORS

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy => policy
        .WithOrigins("https://app.example.com")
        .WithMethods("GET", "POST", "PUT", "PATCH", "DELETE")
        .WithHeaders("Authorization", "Content-Type")
        .SetPreflightMaxAge(TimeSpan.FromHours(1))
        .AllowCredentials());
});
// NEVER use AllowAnyOrigin() with AllowCredentials()
```

## Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", l => { l.PermitLimit = 100; l.Window = TimeSpan.FromMinutes(1); });
    options.AddSlidingWindowLimiter("auth", l => { l.PermitLimit = 5; l.Window = TimeSpan.FromMinutes(1); l.SegmentsPerWindow = 3; });
    options.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await ctx.HttpContext.Response.WriteAsJsonAsync(new { error = "Rate limit exceeded." }, ct);
    };
});

app.MapPost("/auth/login", LoginHandler).RequireRateLimiting("auth");
```

## Secrets Management

```bash
# Development
dotnet user-secrets init
dotnet user-secrets set "Jwt:Key" "your-secret-key-here"
```

```csharp
// Production — Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
    new DefaultAzureCredential());
```

## Security Checklist

- [ ] HTTPS enforced (UseHttpsRedirection, HSTS)
- [ ] Authentication configured (JWT or Cookie)
- [ ] Authorization on all endpoints (FallbackPolicy)
- [ ] Input validation on all user inputs
- [ ] Parameterized queries only
- [ ] Security headers configured
- [ ] CORS restricted to known origins
- [ ] Rate limiting on auth endpoints
- [ ] Secrets in Key Vault, not source

## References

- [Authentication and Authorization](references/authentication-authorization.md) — JWT, API key auth, OAuth2/OIDC, refresh token rotation, multi-scheme auth.
- [Secure Coding Practices](references/secure-coding.md) — Data Protection API, anti-forgery, XSS, SQL injection, file upload security, Key Vault.
