# Secure Coding Practices in .NET

## Data Protection API

```csharp
public class SecureDataService(IDataProtectionProvider provider)
{
    private readonly IDataProtector _protector = provider.CreateProtector("SecureData.v1");
    public string Protect(string plainText) => _protector.Protect(plainText);
    public string Unprotect(string cipherText) => _protector.Unprotect(cipherText);
}

// Time-limited protection
var timeLimited = _protector.ToTimeLimitedDataProtector();
var token = timeLimited.Protect(data, lifetime: TimeSpan.FromHours(1));
```

## Anti-Forgery Tokens

```csharp
// MVC forms — automatic via Tag Helpers
<form asp-action="Create"><input asp-for="Name" /><button type="submit">Create</button></form>

// Global validation for all POST/PUT/DELETE
builder.Services.AddControllersWithViews(options => options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute()));

// SPA pattern — send token via header
builder.Services.AddAntiforgery(options => options.HeaderName = "X-XSRF-TOKEN");
```

## SQL Injection Prevention

```csharp
// SAFE — EF Core parameterizes automatically
var products = await db.Products.Where(p => p.Name == searchTerm).ToListAsync();

// SAFE — interpolated SQL is parameterized
var products = await db.Products.FromSqlInterpolated($"SELECT * FROM Products WHERE Name = {searchTerm}").ToListAsync();

// DANGEROUS — never concatenate user input
// var products = await db.Products.FromSqlRaw($"SELECT * FROM Products WHERE Name = '{searchTerm}'").ToListAsync();
```

## XSS Prevention

```csharp
// Razor auto-encodes by default: @Model.UserInput is safe
// DANGER: @Html.Raw(Model.UserInput) bypasses encoding

// Manual encoding when needed
using Microsoft.Extensions.WebEncoders;
var safe = HtmlEncoder.Default.Encode(userInput);
```

## File Upload Security

```csharp
app.MapPost("/upload", async (IFormFile file) =>
{
    // 1. Validate size
    if (file.Length > 10 * 1024 * 1024) return Results.BadRequest("File too large (max 10MB)");

    // 2. Validate content type
    var allowed = new[] { "image/jpeg", "image/png", "application/pdf" };
    if (!allowed.Contains(file.ContentType)) return Results.BadRequest("Invalid file type");

    // 3. Generate safe filename (never use user-provided filename in path)
    var safeFileName = $"{Guid.NewGuid()}{Path.GetExtension(file.FileName)}";

    // 4. Store outside web root
    var path = Path.Combine("/secure-uploads", safeFileName);
    using var stream = File.Create(path);
    await file.CopyToAsync(stream);

    return Results.Ok(new { fileName = safeFileName });
}).DisableAntiforgery(); // Only if using JWT (not cookies)
```

## Azure Key Vault Integration

```csharp
// Program.cs
if (!builder.Environment.IsDevelopment())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
        new DefaultAzureCredential());
}
```

## Dependency Scanning

```bash
# Check for vulnerable packages
dotnet list package --vulnerable --include-transitive

# Update vulnerable packages
dotnet outdated --upgrade
```
