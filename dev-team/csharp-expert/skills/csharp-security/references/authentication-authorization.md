# Authentication and Authorization in ASP.NET Core

Comprehensive reference covering JWT implementation, custom authorization handlers, API key authentication, multi-scheme auth, OAuth2/OIDC, account lockout, and two-factor authentication.

---

## Complete JWT Setup

### Token Validation Configuration

```csharp
// Program.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

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
        ClockSkew = TimeSpan.FromSeconds(30),
        NameClaimType = ClaimTypes.NameIdentifier,
        RoleClaimType = ClaimTypes.Role,
    };

    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            if (context.Exception is SecurityTokenExpiredException)
            {
                context.Response.Headers["Token-Expired"] = "true";
            }
            return Task.CompletedTask;
        },
    };
});
```

### Refresh Token Storage

```csharp
public class RefreshToken
{
    public int Id { get; set; }
    public required string Token { get; set; }
    public required string UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; }
    public string? ReplacedByToken { get; set; }
    public bool IsRevoked { get; set; }
    public bool IsExpired => DateTime.UtcNow >= ExpiresAt;
    public bool IsActive => !IsRevoked && !IsExpired;
}
```

### Token Refresh Endpoint

```csharp
app.MapPost("/auth/refresh", async (
    RefreshRequest request,
    TokenService tokenService,
    AppDbContext db,
    CancellationToken ct) =>
{
    var storedToken = await db.RefreshTokens
        .Include(t => t.User)
        .FirstOrDefaultAsync(t => t.Token == request.RefreshToken, ct);

    if (storedToken is null || !storedToken.IsActive)
        return Results.Unauthorized();

    // Rotate: revoke old token, create new pair
    storedToken.IsRevoked = true;

    var newAccessToken = tokenService.GenerateAccessToken(
        storedToken.User, await GetRolesAsync(storedToken.User));
    var newRefreshToken = tokenService.GenerateRefreshToken();

    storedToken.ReplacedByToken = newRefreshToken;

    db.RefreshTokens.Add(new RefreshToken
    {
        Token = newRefreshToken,
        UserId = storedToken.UserId,
        ExpiresAt = DateTime.UtcNow.AddDays(7),
        CreatedAt = DateTime.UtcNow,
    });

    await db.SaveChangesAsync(ct);

    return Results.Ok(new
    {
        AccessToken = newAccessToken,
        RefreshToken = newRefreshToken,
    });
});
```

---

## API Key Authentication

### Custom Authentication Handler

```csharp
public class ApiKeyAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder,
    AppDbContext db)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-API-Key", out var apiKeyHeader))
            return AuthenticateResult.NoResult();

        var apiKey = apiKeyHeader.ToString();
        var keyHash = SHA256.HashData(Encoding.UTF8.GetBytes(apiKey));
        var keyHashHex = Convert.ToHexString(keyHash).ToLowerInvariant();

        var record = await db.ApiKeys
            .FirstOrDefaultAsync(k =>
                k.KeyHash == keyHashHex &&
                k.IsActive &&
                (k.ExpiresAt == null || k.ExpiresAt > DateTime.UtcNow));

        if (record is null)
            return AuthenticateResult.Fail("Invalid API key");

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, record.OwnerId),
            new Claim("api_key_id", record.Id.ToString()),
        };

        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);

        return AuthenticateResult.Success(ticket);
    }
}
```

### Multi-Scheme Authentication (JWT + API Key)

```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, options => { /* JWT config */ })
    .AddScheme<AuthenticationSchemeOptions, ApiKeyAuthHandler>("ApiKey", null);

builder.Services.AddAuthorizationBuilder()
    .SetDefaultPolicy(new AuthorizationPolicyBuilder()
        .AddAuthenticationSchemes(JwtBearerDefaults.AuthenticationScheme, "ApiKey")
        .RequireAuthenticatedUser()
        .Build());
```

---

## Account Lockout and Brute Force Protection

### ASP.NET Core Identity Lockout

```csharp
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    // Password policy
    options.Password.RequiredLength = 12;
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;

    // Lockout policy
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;

    // User settings
    options.User.RequireUniqueEmail = true;
    options.SignIn.RequireConfirmedEmail = true;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();
```

### Login with Lockout Check

```csharp
app.MapPost("/auth/login", async (
    LoginRequest request,
    SignInManager<ApplicationUser> signInManager,
    UserManager<ApplicationUser> userManager,
    TokenService tokenService) =>
{
    var user = await userManager.FindByEmailAsync(request.Email);
    if (user is null)
        return Results.Unauthorized();

    if (await userManager.IsLockedOutAsync(user))
        return Results.Problem("Account is locked. Try again later.", statusCode: 423);

    var result = await signInManager.CheckPasswordSignInAsync(
        user, request.Password, lockoutOnFailure: true);

    if (!result.Succeeded)
    {
        if (result.IsLockedOut)
            return Results.Problem("Account locked due to too many failed attempts.", statusCode: 423);
        return Results.Unauthorized();
    }

    var roles = await userManager.GetRolesAsync(user);
    var accessToken = tokenService.GenerateAccessToken(user, roles);
    var refreshToken = tokenService.GenerateRefreshToken();

    return Results.Ok(new { AccessToken = accessToken, RefreshToken = refreshToken });
});
```

---

## OAuth2/OIDC with Microsoft Entra ID

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.Authority = $"https://login.microsoftonline.com/{builder.Configuration["AzureAd:TenantId"]}/v2.0";
    options.ClientId = builder.Configuration["AzureAd:ClientId"];
    options.ClientSecret = builder.Configuration["AzureAd:ClientSecret"];
    options.ResponseType = "code";
    options.SaveTokens = true;
    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.Scope.Add("email");
    options.MapInboundClaims = false;
    options.TokenValidationParameters.NameClaimType = "preferred_username";
    options.TokenValidationParameters.RoleClaimType = "roles";
});
```

---

## Two-Factor Authentication

### Setup

```csharp
app.MapPost("/auth/2fa/setup", async (
    UserManager<ApplicationUser> userManager,
    ClaimsPrincipal user) =>
{
    var appUser = await userManager.GetUserAsync(user);
    if (appUser is null) return Results.Unauthorized();

    var key = await userManager.GetAuthenticatorKeyAsync(appUser);
    if (string.IsNullOrEmpty(key))
    {
        await userManager.ResetAuthenticatorKeyAsync(appUser);
        key = await userManager.GetAuthenticatorKeyAsync(appUser);
    }

    var uri = $"otpauth://totp/MyApp:{appUser.Email}?secret={key}&issuer=MyApp&digits=6";

    return Results.Ok(new { SharedKey = key, AuthenticatorUri = uri });
}).RequireAuthorization();

app.MapPost("/auth/2fa/verify", async (
    TwoFactorVerifyRequest request,
    UserManager<ApplicationUser> userManager,
    ClaimsPrincipal user) =>
{
    var appUser = await userManager.GetUserAsync(user);
    if (appUser is null) return Results.Unauthorized();

    var isValid = await userManager.VerifyTwoFactorTokenAsync(
        appUser,
        userManager.Options.Tokens.AuthenticatorTokenProvider,
        request.Code);

    if (!isValid) return Results.BadRequest("Invalid verification code");

    await userManager.SetTwoFactorEnabledAsync(appUser, true);
    var recoveryCodes = await userManager.GenerateNewTwoFactorRecoveryCodesAsync(appUser, 10);

    return Results.Ok(new { RecoveryCodes = recoveryCodes });
}).RequireAuthorization();
```
