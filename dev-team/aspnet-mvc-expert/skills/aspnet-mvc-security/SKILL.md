---
name: aspnet-mvc-security
description: This skill should be used when the user asks to "add authentication", "implement authorization", "secure the API", "add JWT authentication", "implement Identity", "prevent XSS", "prevent SQL injection", "add CSRF protection", "configure CORS", "add security headers", "implement role-based access", "add policy-based authorization", or needs guidance on securing ASP.NET Core MVC applications against OWASP vulnerabilities.
---

# ASP.NET MVC Security

Comprehensive security guidance for ASP.NET Core MVC applications covering authentication, authorization, and OWASP Top 10 prevention.

## Authentication Setup

### JWT Bearer Authentication (APIs)

```csharp
// Program.cs
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
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
        ClockSkew = TimeSpan.Zero // Eliminate default 5-minute tolerance
    };
});
```

### Token Generation Service

```csharp
public class TokenService : ITokenService
{
    private readonly JwtSettings _settings;

    public TokenService(IOptions<JwtSettings> settings)
        => _settings = settings.Value;

    public string GenerateAccessToken(ApplicationUser user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id),
            new(ClaimTypes.Email, user.Email!),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_settings.Key));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _settings.Issuer,
            audience: _settings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_settings.ExpiryMinutes),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### Cookie Authentication (MVC Views)

```csharp
builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(options =>
    {
        options.LoginPath = "/Account/Login";
        options.AccessDeniedPath = "/Account/AccessDenied";
        options.ExpireTimeSpan = TimeSpan.FromHours(2);
        options.SlidingExpiration = true;
        options.Cookie.HttpOnly = true;
        options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        options.Cookie.SameSite = SameSiteMode.Strict;
    });
```

## Authorization

### Policy-Based Authorization (Preferred)

```csharp
// Registration
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("CanManageProducts", policy =>
        policy.RequireClaim("Permission", "Products.Manage"));

    options.AddPolicy("MinimumAge", policy =>
        policy.AddRequirements(new MinimumAgeRequirement(18)));

    // Fallback policy — require authentication for all endpoints by default
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// Custom requirement + handler
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }
    public MinimumAgeRequirement(int minimumAge) => MinimumAge = minimumAge;
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst("DateOfBirth");
        if (dateOfBirthClaim is null) return Task.CompletedTask;

        var dateOfBirth = DateOnly.Parse(dateOfBirthClaim.Value);
        var age = DateOnly.FromDateTime(DateTime.Today).Year - dateOfBirth.Year;

        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}
```

### Resource-Based Authorization

```csharp
// For authorizing access to specific resources (e.g., "user can edit own post")
public class PostAuthorizationHandler
    : AuthorizationHandler<OperationAuthorizationRequirement, Post>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        Post resource)
    {
        if (requirement.Name == Operations.Update.Name &&
            resource.AuthorId == context.User.FindFirstValue(ClaimTypes.NameIdentifier))
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}

// Usage in controller
public async Task<IActionResult> Edit(int id)
{
    var post = await _repository.GetByIdAsync(id);
    var result = await _authorizationService.AuthorizeAsync(User, post, Operations.Update);
    if (!result.Succeeded) return Forbid();
    // proceed with edit
}
```

## OWASP Top 10 Prevention

### 1. Injection Prevention

```csharp
// ALWAYS use parameterized queries — EF Core does this by default
var products = await _db.Products
    .Where(p => p.Name == searchTerm)  // Safe — parameterized
    .ToListAsync();

// If raw SQL is needed, use FromSqlInterpolated (parameterized)
var products = await _db.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Name = {searchTerm}")
    .ToListAsync();

// NEVER concatenate user input into SQL
// BAD: FromSqlRaw($"SELECT * FROM Products WHERE Name = '{searchTerm}'")
```

### 2. XSS Prevention

```csharp
// Razor automatically HTML-encodes output by default
<p>@Model.UserInput</p>  // Safe — encoded

// DANGER: @Html.Raw() bypasses encoding — avoid with user input
// If HTML content must be rendered, sanitize first:
services.AddSingleton<IHtmlSanitizer>(new HtmlSanitizer());
```

### 3. CSRF Protection

```csharp
// MVC forms: automatically included by Tag Helpers
<form asp-action="Create" asp-controller="Products">
    <!-- Anti-forgery token automatically included -->
</form>

// Validate on POST actions (auto-validated with [AutoValidateAntiforgeryToken] globally)
builder.Services.AddControllersWithViews(options =>
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute()));

// APIs using JWT do not need CSRF tokens (bearer tokens are not auto-attached)
```

### 4. Security Headers

```csharp
// Middleware to add security headers
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "0"); // Modern CSP replaces this
    context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Append("Permissions-Policy",
        "accelerometer=(), camera=(), geolocation=(), microphone=()");
    context.Response.Headers.Append("Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");
    await next();
});
```

### 5. CORS Configuration

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowedOrigins", policy =>
    {
        policy.WithOrigins("https://app.example.com", "https://admin.example.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Authorization", "Content-Type")
              .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });
});
// NEVER use AllowAnyOrigin() with AllowCredentials()
```

## Data Protection

```csharp
// Encrypt sensitive data at rest using Data Protection API
public class SecureDataService
{
    private readonly IDataProtector _protector;

    public SecureDataService(IDataProtectionProvider provider)
        => _protector = provider.CreateProtector("SecureData.v1");

    public string Protect(string plainText) => _protector.Protect(plainText);
    public string Unprotect(string cipherText) => _protector.Unprotect(cipherText);
}
```

## Input Validation

```csharp
// FluentValidation for complex validation rules
public class CreateUserCommandValidator : AbstractValidator<CreateUserCommand>
{
    public CreateUserCommandValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(256);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(12)
            .Matches("[A-Z]").WithMessage("Must contain uppercase letter")
            .Matches("[a-z]").WithMessage("Must contain lowercase letter")
            .Matches("[0-9]").WithMessage("Must contain digit")
            .Matches("[^a-zA-Z0-9]").WithMessage("Must contain special character");

        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(100)
            .Matches(@"^[\p{L}\s\-']+$").WithMessage("Contains invalid characters");
    }
}
```

## Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.PermitLimit = 100;
        opt.Window = TimeSpan.FromMinutes(1);
        opt.QueueLimit = 0;
    });

    options.AddFixedWindowLimiter("auth", opt =>
    {
        opt.PermitLimit = 5;
        opt.Window = TimeSpan.FromMinutes(15);
        opt.QueueLimit = 0;
    });

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

// Apply to endpoints
[EnableRateLimiting("auth")]
[HttpPost("login")]
public async Task<IActionResult> Login(LoginCommand command) { }
```

## Security Checklist

Before deploying any ASP.NET Core application:

- [ ] HTTPS enforced (`UseHttpsRedirection`, HSTS)
- [ ] Authentication configured (JWT or Cookie)
- [ ] Authorization on all endpoints (FallbackPolicy)
- [ ] Anti-forgery tokens for MVC forms
- [ ] Input validation on all user inputs
- [ ] Parameterized queries only (no string concatenation in SQL)
- [ ] Security headers configured (CSP, X-Frame-Options, etc.)
- [ ] CORS restricted to known origins
- [ ] Rate limiting on auth and sensitive endpoints
- [ ] Secrets stored in User Secrets / Key Vault (not appsettings.json)
- [ ] Structured logging without PII
- [ ] Data Protection API for sensitive data at rest

## Additional Resources

### Reference Files

For detailed security patterns, consult:

- **`references/owasp_prevention.md`** — Detailed OWASP Top 10 prevention strategies with ASP.NET Core-specific mitigations
- **`references/authentication_authorization.md`** — Advanced Identity configuration, external providers, refresh tokens, multi-tenancy authorization
