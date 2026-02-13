# Authentication & Authorization in ASP.NET Core

Advanced patterns for authentication and authorization in ASP.NET Core MVC applications.

## ASP.NET Core Identity Setup

### Full Identity Configuration

```csharp
// Program.cs
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    // Password policy
    options.Password.RequiredLength = 12;
    options.Password.RequireDigit = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredUniqueChars = 4;

    // Lockout
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;

    // User
    options.User.RequireUniqueEmail = true;

    // Sign-in
    options.SignIn.RequireConfirmedEmail = true;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();
```

### Custom ApplicationUser

```csharp
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public bool IsActive { get; set; } = true;
    public string? RefreshToken { get; set; }
    public DateTime? RefreshTokenExpiryTime { get; set; }
}
```

## JWT Authentication with Refresh Tokens

### Token Service

```csharp
public class TokenService : ITokenService
{
    private readonly JwtSettings _jwtSettings;
    private readonly UserManager<ApplicationUser> _userManager;

    public TokenService(
        IOptions<JwtSettings> jwtSettings,
        UserManager<ApplicationUser> userManager)
    {
        _jwtSettings = jwtSettings.Value;
        _userManager = userManager;
    }

    public async Task<TokenResponse> GenerateTokensAsync(ApplicationUser user)
    {
        var roles = await _userManager.GetRolesAsync(user);
        var claims = await _userManager.GetClaimsAsync(user);

        var tokenClaims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id),
            new(ClaimTypes.Email, user.Email!),
            new(ClaimTypes.Name, $"{user.FirstName} {user.LastName}"),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64)
        };

        tokenClaims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));
        tokenClaims.AddRange(claims);

        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_jwtSettings.Key));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var accessToken = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: tokenClaims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.AccessTokenExpiryMinutes),
            signingCredentials: credentials);

        var refreshToken = GenerateRefreshToken();

        // Store refresh token in database
        user.RefreshToken = refreshToken;
        user.RefreshTokenExpiryTime =
            DateTime.UtcNow.AddDays(_jwtSettings.RefreshTokenExpiryDays);
        await _userManager.UpdateAsync(user);

        return new TokenResponse
        {
            AccessToken = new JwtSecurityTokenHandler().WriteToken(accessToken),
            RefreshToken = refreshToken,
            ExpiresAt = accessToken.ValidTo
        };
    }

    public async Task<TokenResponse> RefreshTokenAsync(
        string accessToken, string refreshToken)
    {
        var principal = GetPrincipalFromExpiredToken(accessToken);
        var userId = principal.FindFirstValue(ClaimTypes.NameIdentifier)
            ?? throw new SecurityException("Invalid token");

        var user = await _userManager.FindByIdAsync(userId)
            ?? throw new SecurityException("User not found");

        if (user.RefreshToken != refreshToken ||
            user.RefreshTokenExpiryTime <= DateTime.UtcNow)
        {
            throw new SecurityException("Invalid or expired refresh token");
        }

        return await GenerateTokensAsync(user);
    }

    private static string GenerateRefreshToken()
    {
        var randomBytes = new byte[64];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomBytes);
        return Convert.ToBase64String(randomBytes);
    }

    private ClaimsPrincipal GetPrincipalFromExpiredToken(string token)
    {
        var parameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = _jwtSettings.Issuer,
            ValidAudience = _jwtSettings.Audience,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(_jwtSettings.Key)),
            ValidateLifetime = false // Allow expired tokens for refresh
        };

        var principal = new JwtSecurityTokenHandler()
            .ValidateToken(token, parameters, out var securityToken);

        if (securityToken is not JwtSecurityToken jwt ||
            !jwt.Header.Alg.Equals(
                SecurityAlgorithms.HmacSha256,
                StringComparison.InvariantCultureIgnoreCase))
        {
            throw new SecurityException("Invalid token algorithm");
        }

        return principal;
    }
}
```

### Auth Controller

```csharp
[ApiController]
[Route("api/[controller]")]
[AllowAnonymous]
public class AuthController : ControllerBase
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly ITokenService _tokenService;

    [HttpPost("login")]
    [EnableRateLimiting("auth")]
    public async Task<IActionResult> Login(LoginRequest request)
    {
        var user = await _userManager.FindByEmailAsync(request.Email);
        if (user is null || !user.IsActive)
            return Unauthorized(new { Message = "Invalid credentials" });

        if (await _userManager.IsLockedOutAsync(user))
            return StatusCode(429, new { Message = "Account locked. Try again later." });

        if (!await _userManager.CheckPasswordAsync(user, request.Password))
        {
            await _userManager.AccessFailedAsync(user);
            return Unauthorized(new { Message = "Invalid credentials" });
        }

        await _userManager.ResetAccessFailedCountAsync(user);
        var tokens = await _tokenService.GenerateTokensAsync(user);
        return Ok(tokens);
    }

    [HttpPost("refresh")]
    public async Task<IActionResult> Refresh(RefreshTokenRequest request)
    {
        try
        {
            var tokens = await _tokenService.RefreshTokenAsync(
                request.AccessToken, request.RefreshToken);
            return Ok(tokens);
        }
        catch (SecurityException)
        {
            return Unauthorized(new { Message = "Invalid token" });
        }
    }

    [HttpPost("revoke")]
    [Authorize]
    public async Task<IActionResult> Revoke()
    {
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        var user = await _userManager.FindByIdAsync(userId!);
        if (user is null) return NotFound();

        user.RefreshToken = null;
        user.RefreshTokenExpiryTime = null;
        await _userManager.UpdateAsync(user);

        return NoContent();
    }
}
```

## Advanced Authorization Patterns

### Permission-Based Authorization

```csharp
// Define permissions as constants
public static class Permissions
{
    public static class Products
    {
        public const string View = "Permissions.Products.View";
        public const string Create = "Permissions.Products.Create";
        public const string Edit = "Permissions.Products.Edit";
        public const string Delete = "Permissions.Products.Delete";
    }

    public static class Orders
    {
        public const string View = "Permissions.Orders.View";
        public const string Create = "Permissions.Orders.Create";
        public const string Process = "Permissions.Orders.Process";
    }
}

// Custom policy provider for dynamic permission policies
public class PermissionPolicyProvider : IAuthorizationPolicyProvider
{
    private readonly DefaultAuthorizationPolicyProvider _fallback;

    public PermissionPolicyProvider(IOptions<AuthorizationOptions> options)
        => _fallback = new DefaultAuthorizationPolicyProvider(options);

    public Task<AuthorizationPolicy?> GetPolicyAsync(string policyName)
    {
        if (policyName.StartsWith("Permissions."))
        {
            var policy = new AuthorizationPolicyBuilder()
                .AddRequirements(new PermissionRequirement(policyName))
                .Build();
            return Task.FromResult<AuthorizationPolicy?>(policy);
        }
        return _fallback.GetPolicyAsync(policyName);
    }

    public Task<AuthorizationPolicy> GetDefaultPolicyAsync()
        => _fallback.GetDefaultPolicyAsync();

    public Task<AuthorizationPolicy?> GetFallbackPolicyAsync()
        => _fallback.GetFallbackPolicyAsync();
}

// Permission requirement and handler
public record PermissionRequirement(string Permission) : IAuthorizationRequirement;

public class PermissionHandler : AuthorizationHandler<PermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PermissionRequirement requirement)
    {
        if (context.User.HasClaim("Permission", requirement.Permission))
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}

// Usage
[Authorize(Policy = Permissions.Products.Create)]
[HttpPost]
public async Task<IActionResult> Create(CreateProductCommand command) { }
```

### Multi-Tenant Authorization

```csharp
// Tenant-aware authorization
public class TenantRequirement : IAuthorizationRequirement { }

public class TenantHandler : AuthorizationHandler<TenantRequirement>
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly ITenantService _tenantService;

    public TenantHandler(
        IHttpContextAccessor httpContextAccessor,
        ITenantService tenantService)
    {
        _httpContextAccessor = httpContextAccessor;
        _tenantService = tenantService;
    }

    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        TenantRequirement requirement)
    {
        var tenantId = _httpContextAccessor.HttpContext?
            .Request.Headers["X-Tenant-Id"].FirstOrDefault();

        if (string.IsNullOrEmpty(tenantId))
            return; // Fail — no tenant header

        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        if (userId is null) return;

        var hasAccess = await _tenantService
            .UserBelongsToTenantAsync(userId, tenantId);

        if (hasAccess)
            context.Succeed(requirement);
    }
}
```

### Claims Transformation

Enrich the user's claims after authentication:

```csharp
public class CustomClaimsTransformation : IClaimsTransformation
{
    private readonly IPermissionService _permissionService;

    public CustomClaimsTransformation(IPermissionService permissionService)
        => _permissionService = permissionService;

    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        var identity = (ClaimsIdentity)principal.Identity!;
        var userId = principal.FindFirstValue(ClaimTypes.NameIdentifier);

        if (userId is null) return principal;

        // Load permissions from database and add as claims
        var permissions = await _permissionService.GetUserPermissionsAsync(userId);
        foreach (var permission in permissions)
        {
            if (!identity.HasClaim("Permission", permission))
                identity.AddClaim(new Claim("Permission", permission));
        }

        return principal;
    }
}

// Register
services.AddScoped<IClaimsTransformation, CustomClaimsTransformation>();
```

## External Authentication Providers

```csharp
builder.Services.AddAuthentication()
    .AddGoogle(options =>
    {
        options.ClientId = builder.Configuration["Auth:Google:ClientId"]!;
        options.ClientSecret = builder.Configuration["Auth:Google:ClientSecret"]!;
        options.Scope.Add("profile");
        options.Scope.Add("email");
    })
    .AddMicrosoftAccount(options =>
    {
        options.ClientId = builder.Configuration["Auth:Microsoft:ClientId"]!;
        options.ClientSecret = builder.Configuration["Auth:Microsoft:ClientSecret"]!;
    });
```

## Current User Service

Provide authenticated user context throughout the application:

```csharp
public interface ICurrentUserService
{
    string? UserId { get; }
    string? Email { get; }
    bool IsAuthenticated { get; }
    bool IsInRole(string role);
    bool HasPermission(string permission);
}

public class CurrentUserService : ICurrentUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;
    private ClaimsPrincipal? User => _httpContextAccessor.HttpContext?.User;

    public CurrentUserService(IHttpContextAccessor httpContextAccessor)
        => _httpContextAccessor = httpContextAccessor;

    public string? UserId => User?.FindFirstValue(ClaimTypes.NameIdentifier);
    public string? Email => User?.FindFirstValue(ClaimTypes.Email);
    public bool IsAuthenticated => User?.Identity?.IsAuthenticated ?? false;
    public bool IsInRole(string role) => User?.IsInRole(role) ?? false;
    public bool HasPermission(string permission)
        => User?.HasClaim("Permission", permission) ?? false;
}
```
