# Authentication and Authorization in .NET

## Complete JWT Flow

### Configuration
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true, ValidateAudience = true, ValidateLifetime = true, ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });
```

### Refresh Token Rotation
```csharp
public async Task<TokenResponse> RefreshAsync(string refreshToken, CancellationToken ct)
{
    var stored = await _db.RefreshTokens.FirstOrDefaultAsync(t => t.Token == refreshToken && !t.IsRevoked, ct);
    if (stored is null || stored.ExpiresAt < DateTime.UtcNow) throw new UnauthorizedException("Invalid refresh token");

    // Revoke old token (rotation)
    stored.IsRevoked = true;
    stored.RevokedAt = DateTime.UtcNow;

    // Issue new tokens
    var user = await _userManager.FindByIdAsync(stored.UserId);
    var roles = await _userManager.GetRolesAsync(user!);
    var accessToken = _tokenService.GenerateAccessToken(user!, roles);
    var newRefreshToken = _tokenService.GenerateRefreshToken();

    _db.RefreshTokens.Add(new RefreshToken { Token = newRefreshToken, UserId = user!.Id, ExpiresAt = DateTime.UtcNow.AddDays(7) });
    await _db.SaveChangesAsync(ct);

    return new TokenResponse(accessToken, newRefreshToken);
}
```

## API Key Authentication

```csharp
public class ApiKeyAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-API-Key", out var apiKey))
            return Task.FromResult(AuthenticateResult.NoResult());

        var validKey = Context.RequestServices.GetRequiredService<IConfiguration>()["ApiKey"];
        if (apiKey != validKey)
            return Task.FromResult(AuthenticateResult.Fail("Invalid API key"));

        var claims = new[] { new Claim(ClaimTypes.Name, "ApiClient") };
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        return Task.FromResult(AuthenticateResult.Success(new AuthenticationTicket(new ClaimsPrincipal(identity), Scheme.Name)));
    }
}
```

## Multi-Scheme Authentication

```csharp
builder.Services.AddAuthentication()
    .AddJwtBearer("Bearer", options => { /* JWT config */ })
    .AddScheme<AuthenticationSchemeOptions, ApiKeyAuthHandler>("ApiKey", null);

builder.Services.AddAuthorization(options =>
{
    options.DefaultPolicy = new AuthorizationPolicyBuilder()
        .AddAuthenticationSchemes("Bearer", "ApiKey")
        .RequireAuthenticatedUser()
        .Build();
});
```

## Identity Setup

```csharp
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.Password.RequiredLength = 12;
    options.Password.RequireUppercase = true;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.User.RequireUniqueEmail = true;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();
```
