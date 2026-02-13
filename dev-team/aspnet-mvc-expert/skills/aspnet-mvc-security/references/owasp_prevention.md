# OWASP Top 10 Prevention in ASP.NET Core

Detailed mitigation strategies for each OWASP Top 10 vulnerability category, specific to ASP.NET Core MVC applications.

## A01:2021 — Broken Access Control

The most common web application vulnerability.

### Prevention Strategies

**1. Default-deny authorization**
```csharp
// Require authentication globally, opt-out selectively
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// Only allow anonymous where explicitly needed
[AllowAnonymous]
[HttpGet("public/health")]
public IActionResult HealthCheck() => Ok();
```

**2. Resource-based authorization**
```csharp
// Never trust client-side IDs without server-side ownership checks
[HttpPut("orders/{id}")]
public async Task<IActionResult> UpdateOrder(int id, UpdateOrderCommand command)
{
    var order = await _orderRepository.GetByIdAsync(id);
    if (order is null) return NotFound();

    // Verify ownership
    var authResult = await _authorizationService
        .AuthorizeAsync(User, order, Operations.Update);
    if (!authResult.Succeeded) return Forbid();

    // Proceed with update
}
```

**3. Prevent IDOR (Insecure Direct Object References)**
```csharp
// Filter data by authenticated user context
public async Task<IReadOnlyList<Order>> GetUserOrdersAsync(CancellationToken ct)
{
    var userId = _currentUserService.UserId
        ?? throw new UnauthorizedAccessException();

    return await _db.Orders
        .Where(o => o.UserId == userId) // Always filter by current user
        .ToListAsync(ct);
}
```

**4. Disable directory listing and restrict file access**
```csharp
app.UseStaticFiles(); // Only serves from wwwroot by default
// Never: app.UseDirectoryBrowser()
```

## A02:2021 — Cryptographic Failures

### Prevention Strategies

**1. Use Data Protection API for encryption**
```csharp
// Register
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("/keys"))
    .SetApplicationName("MyApp")
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));

// Use for sensitive data
public class SensitiveDataService
{
    private readonly IDataProtector _protector;

    public SensitiveDataService(IDataProtectionProvider provider)
        => _protector = provider.CreateProtector("SensitiveData.v1");

    public string Encrypt(string value) => _protector.Protect(value);
    public string Decrypt(string value) => _protector.Unprotect(value);
}
```

**2. Enforce HTTPS everywhere**
```csharp
builder.Services.AddHttpsRedirection(options =>
{
    options.HttpsPort = 443;
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
});

builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});
```

**3. Secure password hashing (handled by ASP.NET Identity)**
```csharp
// ASP.NET Identity uses PBKDF2 with HMAC-SHA256 by default
// Override for stronger hashing if needed:
builder.Services.Configure<PasswordHasherOptions>(options =>
{
    options.IterationCount = 600_000; // OWASP 2023 recommendation
});
```

**4. Never log sensitive data**
```csharp
// BAD
_logger.LogInformation("User login: {Email}, Password: {Password}", email, password);

// GOOD
_logger.LogInformation("User login attempt for: {Email}", email);
```

## A03:2021 — Injection

### SQL Injection Prevention

```csharp
// SAFE — EF Core parameterizes automatically
var user = await _db.Users.FirstOrDefaultAsync(u => u.Email == email);

// SAFE — FromSqlInterpolated parameterizes
var results = await _db.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE CategoryId = {categoryId}")
    .ToListAsync();

// DANGER — FromSqlRaw with concatenation
// NEVER: _db.Products.FromSqlRaw($"SELECT * FROM Products WHERE Name = '{name}'")

// SAFE — If using ADO.NET directly, always parameterize
using var command = connection.CreateCommand();
command.CommandText = "SELECT * FROM Users WHERE Email = @email";
command.Parameters.AddWithValue("@email", email);
```

### Command Injection Prevention

```csharp
// NEVER pass user input to Process.Start without sanitization
// If external process execution is absolutely necessary:
var processInfo = new ProcessStartInfo
{
    FileName = "/usr/bin/convert",   // Fixed executable path
    Arguments = $"\"{SanitizePath(inputFile)}\" \"{SanitizePath(outputFile)}\"",
    UseShellExecute = false,
    CreateNoWindow = true
};

private static string SanitizePath(string path)
{
    // Whitelist allowed characters
    if (!Regex.IsMatch(path, @"^[\w\-./\\]+$"))
        throw new ArgumentException("Invalid path characters");
    return path;
}
```

### LDAP Injection Prevention

```csharp
// Encode special LDAP characters
public static string LdapEscape(string input)
{
    return input
        .Replace("\\", "\\5c")
        .Replace("*", "\\2a")
        .Replace("(", "\\28")
        .Replace(")", "\\29")
        .Replace("\0", "\\00");
}
```

## A04:2021 — Insecure Design

### Prevention Strategies

- Apply threat modeling during design phase
- Use established design patterns (MediatR, CQRS, Clean Architecture)
- Implement rate limiting on all public endpoints
- Apply input validation at every boundary
- Use the Result pattern instead of exceptions for expected failures
- Implement circuit breakers for external service calls

## A05:2021 — Security Misconfiguration

### Prevention Strategies

```csharp
// Remove server header
builder.WebHost.ConfigureKestrel(options =>
{
    options.AddServerHeader = false;
});

// Disable detailed errors in production
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

// Configure security headers
app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;
    headers.Append("X-Content-Type-Options", "nosniff");
    headers.Append("X-Frame-Options", "DENY");
    headers.Append("X-XSS-Protection", "0");
    headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    headers.Append("Permissions-Policy",
        "accelerometer=(), camera=(), geolocation=(), microphone=()");
    headers.Remove("X-Powered-By");
    headers.Remove("Server");
    await next();
});
```

## A06:2021 — Vulnerable and Outdated Components

### Prevention Strategies

```bash
# Regularly check for vulnerable NuGet packages
dotnet list package --vulnerable

# Keep packages updated
dotnet outdated  # Using dotnet-outdated-tool
```

## A07:2021 — Identification and Authentication Failures

### Prevention Strategies

```csharp
// Configure strong password policy
builder.Services.Configure<IdentityOptions>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequiredLength = 12;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireLowercase = true;

    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;

    options.User.RequireUniqueEmail = true;
});

// Rate limit authentication endpoints
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("auth", opt =>
    {
        opt.PermitLimit = 5;
        opt.Window = TimeSpan.FromMinutes(15);
    });
});
```

## A08:2021 — Software and Data Integrity Failures

### Prevention Strategies

- Verify NuGet package signatures
- Use lock files (`packages.lock.json`) in CI/CD
- Validate deserialization with known types only

```csharp
// SAFE — System.Text.Json with known types
var options = new JsonSerializerOptions
{
    PropertyNameCaseInsensitive = true,
    // Do not use TypeNameHandling equivalent — it enables type injection
};
var product = JsonSerializer.Deserialize<ProductDto>(json, options);

// DANGER with Newtonsoft.Json:
// NEVER: JsonConvert.DeserializeObject(json, new JsonSerializerSettings
// { TypeNameHandling = TypeNameHandling.All }); // Remote code execution risk
```

## A09:2021 — Security Logging and Monitoring Failures

### Prevention Strategies

```csharp
// Structured logging with Serilog
builder.Host.UseSerilog((context, config) =>
{
    config.ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .WriteTo.Console()
        .WriteTo.Seq("http://localhost:5341"); // Centralized logging
});

// Log security events
public class SecurityEventLogger
{
    private readonly ILogger<SecurityEventLogger> _logger;

    public void LogFailedLogin(string email, string ipAddress)
        => _logger.LogWarning(
            "Failed login attempt for {Email} from {IpAddress}",
            email, ipAddress);

    public void LogUnauthorizedAccess(string userId, string resource)
        => _logger.LogWarning(
            "Unauthorized access attempt by {UserId} to {Resource}",
            userId, resource);

    public void LogSuspiciousActivity(string userId, string activity)
        => _logger.LogCritical(
            "Suspicious activity detected: {UserId} performed {Activity}",
            userId, activity);
}
```

## A10:2021 — Server-Side Request Forgery (SSRF)

### Prevention Strategies

```csharp
// Validate and whitelist URLs before making server-side requests
public class SafeHttpClient
{
    private static readonly HashSet<string> AllowedHosts = new()
    {
        "api.trusted-service.com",
        "cdn.example.com"
    };

    public async Task<string> FetchAsync(string url)
    {
        var uri = new Uri(url);

        // Block internal network addresses
        if (IPAddress.TryParse(uri.Host, out var ip))
        {
            if (IsPrivateIp(ip))
                throw new SecurityException("Internal addresses are not allowed");
        }

        // Whitelist check
        if (!AllowedHosts.Contains(uri.Host))
            throw new SecurityException($"Host {uri.Host} is not allowed");

        // Proceed with request
    }

    private static bool IsPrivateIp(IPAddress ip)
    {
        var bytes = ip.GetAddressBytes();
        return bytes[0] switch
        {
            10 => true,
            172 => bytes[1] >= 16 && bytes[1] <= 31,
            192 => bytes[1] == 168,
            127 => true,
            _ => false
        };
    }
}
```
