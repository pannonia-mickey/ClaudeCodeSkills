# Secure Coding Practices for ASP.NET Core

Comprehensive reference covering Data Protection API, anti-forgery, HTTPS enforcement, SQL injection prevention, XSS output encoding, file upload security, rate limiting, and Azure Key Vault integration.

---

## Data Protection API

### Encrypting Sensitive Data

```csharp
public class SensitiveDataService(IDataProtectionProvider provider)
{
    private readonly IDataProtector _protector = provider.CreateProtector("SensitiveData.v1");

    public string Protect(string plainText) => _protector.Protect(plainText);

    public string Unprotect(string protectedText) => _protector.Unprotect(protectedText);
}

// Time-limited protection (auto-expires)
public class TimeLimitedTokenService(IDataProtectionProvider provider)
{
    private readonly ITimeLimitedDataProtector _protector =
        provider.CreateProtector("Tokens.v1").ToTimeLimitedDataProtector();

    public string CreateToken(string data, TimeSpan lifetime)
        => _protector.Protect(data, lifetime);

    public string? ValidateToken(string token)
    {
        try
        {
            return _protector.Unprotect(token);
        }
        catch (CryptographicException)
        {
            return null;
        }
    }
}
```

### Configuration for Production

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(connectionString, "dataprotection", "keys.xml")
    .ProtectKeysWithAzureKeyVault(new Uri("https://myvault.vault.azure.net/keys/dataprotection"), new DefaultAzureCredential())
    .SetApplicationName("MyApp")
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));
```

---

## Anti-Forgery Tokens

### API Anti-Forgery Configuration

```csharp
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-XSRF-TOKEN";
    options.Cookie.Name = "XSRF-TOKEN";
    options.Cookie.HttpOnly = false; // Must be readable by JS
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
});

// Endpoint to get the anti-forgery token
app.MapGet("/antiforgery/token", (IAntiforgery antiforgery, HttpContext context) =>
{
    var tokens = antiforgery.GetAndStoreTokens(context);
    return Results.Ok(new { token = tokens.RequestToken });
});

// Validate on state-changing endpoints
app.MapPost("/api/orders", async (
    CreateOrderRequest request,
    IAntiforgery antiforgery,
    HttpContext context) =>
{
    await antiforgery.ValidateRequestAsync(context);
    // Process the order...
}).DisableAntiforgery(); // Disable default, validate manually
```

---

## HTTPS Enforcement

```csharp
// Program.cs
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();

// Configure HSTS in services
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});

// Force HTTPS redirection
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = 443;
});
```

---

## SQL Injection Prevention

### EF Core — Safe by Default

```csharp
// SAFE — EF Core parameterizes automatically
var products = await db.Products
    .Where(p => p.Name.Contains(searchTerm))
    .ToListAsync();
```

### Raw SQL — Always Parameterize

```csharp
// DANGEROUS — string interpolation in raw SQL
var sql = $"SELECT * FROM Products WHERE Name = '{searchTerm}'";
var products = await db.Products.FromSqlRaw(sql).ToListAsync();

// SAFE — parameterized with FromSqlInterpolated
var products = await db.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Name = {searchTerm}")
    .ToListAsync();

// SAFE — explicit parameters with FromSqlRaw
var products = await db.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Name = {0}", searchTerm)
    .ToListAsync();
```

### Dapper — Always Use Parameters

```csharp
// DANGEROUS
var sql = $"SELECT * FROM Users WHERE Email = '{email}'";

// SAFE
var sql = "SELECT * FROM Users WHERE Email = @Email";
var users = await connection.QueryAsync<User>(sql, new { Email = email });
```

---

## XSS Prevention and Output Encoding

### HTML Encoding

```csharp
using System.Text.Encodings.Web;

public class SafeOutputService(HtmlEncoder htmlEncoder, JavaScriptEncoder jsEncoder)
{
    public string EncodeForHtml(string input) => htmlEncoder.Encode(input);
    public string EncodeForJs(string input) => jsEncoder.Encode(input);
}

// In Razor Pages / Views, encoding is automatic:
// @Model.UserName — automatically HTML-encoded
// @Html.Raw(Model.Content) — DANGEROUS, bypasses encoding
```

### JSON Response Security

ASP.NET Core's `System.Text.Json` automatically encodes HTML-significant characters in JSON:

```csharp
// '<' becomes '\u003C', '>' becomes '\u003E', etc.
// This prevents XSS when JSON is embedded in HTML
builder.Services.Configure<JsonOptions>(options =>
{
    options.SerializerOptions.Encoder = JavaScriptEncoder.Default; // safe by default
});
```

---

## File Upload Security

### Secure File Upload Endpoint

```csharp
app.MapPost("/api/upload", async (
    IFormFile file,
    IConfiguration config) =>
{
    // 1. Validate file size
    const long maxSize = 10 * 1024 * 1024; // 10 MB
    if (file.Length > maxSize)
        return Results.BadRequest("File too large");

    // 2. Validate file extension
    var allowedExtensions = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
        { ".jpg", ".jpeg", ".png", ".gif", ".pdf" };
    var extension = Path.GetExtension(file.FileName);
    if (!allowedExtensions.Contains(extension))
        return Results.BadRequest("File type not allowed");

    // 3. Validate MIME type matches extension
    var allowedMimeTypes = new Dictionary<string, string[]>(StringComparer.OrdinalIgnoreCase)
    {
        [".jpg"] = ["image/jpeg"],
        [".jpeg"] = ["image/jpeg"],
        [".png"] = ["image/png"],
        [".gif"] = ["image/gif"],
        [".pdf"] = ["application/pdf"],
    };

    if (!allowedMimeTypes.TryGetValue(extension, out var mimeTypes) ||
        !mimeTypes.Contains(file.ContentType, StringComparer.OrdinalIgnoreCase))
        return Results.BadRequest("MIME type does not match extension");

    // 4. Generate safe filename (never use original filename for storage)
    var safeFileName = $"{Guid.NewGuid()}{extension}";
    var uploadDir = config["Upload:Directory"]!;
    var filePath = Path.Combine(uploadDir, safeFileName);

    // 5. Verify the path is within the upload directory
    var fullPath = Path.GetFullPath(filePath);
    if (!fullPath.StartsWith(Path.GetFullPath(uploadDir)))
        return Results.BadRequest("Invalid file path");

    // 6. Save file
    await using var stream = new FileStream(fullPath, FileMode.Create);
    await file.CopyToAsync(stream);

    return Results.Ok(new { FileName = safeFileName });
}).RequireAuthorization()
  .DisableAntiforgery();

// Configure file size limit
builder.Services.Configure<FormOptions>(options =>
{
    options.MultipartBodyLengthLimit = 10 * 1024 * 1024;
});
```

---

## Rate Limiting

### Multiple Rate Limiting Policies

```csharp
builder.Services.AddRateLimiter(options =>
{
    // General API rate limit
    options.AddFixedWindowLimiter("api", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
    });

    // Strict rate limit for authentication endpoints
    options.AddSlidingWindowLimiter("auth", limiter =>
    {
        limiter.PermitLimit = 5;
        limiter.Window = TimeSpan.FromMinutes(5);
        limiter.SegmentsPerWindow = 5;
        limiter.QueueLimit = 0;
    });

    // Token bucket for download endpoints
    options.AddTokenBucketLimiter("downloads", limiter =>
    {
        limiter.TokenLimit = 10;
        limiter.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        limiter.TokensPerPeriod = 2;
        limiter.QueueLimit = 0;
    });

    // Per-user rate limiting using a partition key
    options.AddPolicy("per-user", context =>
    {
        var userId = context.User?.FindFirstValue(ClaimTypes.NameIdentifier) ?? "anonymous";
        return RateLimitPartition.GetFixedWindowLimiter(userId, _ =>
            new FixedWindowRateLimiterOptions
            {
                PermitLimit = 50,
                Window = TimeSpan.FromMinutes(1),
            });
    });

    options.OnRejected = async (context, ct) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;

        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            context.HttpContext.Response.Headers.RetryAfter = retryAfter.TotalSeconds.ToString("0");
        }

        await context.HttpContext.Response.WriteAsJsonAsync(
            new { error = "Rate limit exceeded" }, ct);
    };
});

app.UseRateLimiter();
```

---

## Azure Key Vault Integration

### Configuration

```csharp
// Program.cs
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUri = new Uri(builder.Configuration["KeyVault:Uri"]!);
    builder.Configuration.AddAzureKeyVault(keyVaultUri, new DefaultAzureCredential());
}
```

### Direct Secret Access

```csharp
public class SecretService(SecretClient secretClient)
{
    public async Task<string> GetSecretAsync(string name, CancellationToken ct = default)
    {
        var response = await secretClient.GetSecretAsync(name, cancellationToken: ct);
        return response.Value.Value;
    }
}

// Registration
builder.Services.AddSingleton(new SecretClient(
    new Uri(builder.Configuration["KeyVault:Uri"]!),
    new DefaultAzureCredential()));
builder.Services.AddSingleton<SecretService>();
```

---

## Health Check Security

Restrict health check endpoints to prevent information leakage:

```csharp
app.MapHealthChecks("/health", new HealthCheckOptions
{
    // Only show basic status publicly
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(new { status = report.Status.ToString() });
    },
});

app.MapHealthChecks("/health/detail", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
}).RequireAuthorization("AdminOnly"); // Detailed health only for admins
```
