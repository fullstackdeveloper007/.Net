Sure! Let’s rewrite the same **JWT login + token validation + Serilog logging** example using **controller-based APIs** instead of Minimal APIs. This is a **classic approach** in .NET 8.

---

# **1️⃣ Install required packages**

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
```

---

# **2️⃣ Program.cs**

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using Serilog;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// ===== Serilog Configuration =====
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateLogger();

builder.Host.UseSerilog();

// ===== JWT Configuration =====
var jwtSecretKey = "super_secret_key_123!"; // store securely
var issuer = "my-api";
var audience = "my-api";

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = issuer,
            ValidAudience = audience,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtSecretKey))
        };

        options.Events = new JwtBearerEvents
        {
            OnTokenValidated = context =>
            {
                var userId = context.Principal?.FindFirst("sub")?.Value;
                var email = context.Principal?.FindFirst(System.Security.Claims.ClaimTypes.Email)?.Value;
                Log.Information("JWT validated for UserID: {UserId}, Email: {Email}", userId, email);
                return Task.CompletedTask;
            },
            OnAuthenticationFailed = context =>
            {
                Log.Warning("JWT authentication failed: {Message}", context.Exception.Message);
                return Task.CompletedTask;
            }
        };
    });

// ===== Authorization =====
builder.Services.AddAuthorization();

// ===== Controllers =====
builder.Services.AddControllers(config =>
{
    // Apply global authorize policy
    var policy = new Microsoft.AspNetCore.Authorization.AuthorizationPolicyBuilder()
                     .RequireAuthenticatedUser()
                     .Build();
    config.Filters.Add(new Microsoft.AspNetCore.Mvc.Authorization.AuthorizeFilter(policy));
});

var app = builder.Build();

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

---

# **3️⃣ AuthController.cs (Login API)**

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.IdentityModel.Tokens;
using Serilog;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly string jwtSecretKey = "super_secret_key_123!";
    private readonly string issuer = "my-api";
    private readonly string audience = "my-api";

    [HttpPost("login")]
    [AllowAnonymous] // Only login API is anonymous
    public IActionResult Login([FromBody] LoginDto login)
    {
        // ===== Simple login check =====
        if (login.Username != "admin" || login.Password != "password")
            return Unauthorized();

        // ===== Create JWT =====
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, "1"),
            new Claim(ClaimTypes.Email, "admin@example.com"),
            new Claim(ClaimTypes.Name, "AdminUser"),
            new Claim(ClaimTypes.Role, "Admin")
        };

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtSecretKey));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: issuer,
            audience: audience,
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: creds
        );

        var jwt = new JwtSecurityTokenHandler().WriteToken(token);

        Log.Information("User {Username} logged in successfully", login.Username);

        return Ok(new { token = jwt });
    }
}

public record LoginDto(string Username, string Password);
```

---

# **4️⃣ ProductsController.cs (Protected API)**

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProducts()
    {
        var products = new[]
        {
            new { Id = 1, Name = "Laptop" },
            new { Id = 2, Name = "Phone" }
        };

        return Ok(products);
    }
}
```

* **No `[AllowAnonymous]` here** → global authorization policy ensures JWT is required.

---

# **✅ How it Works**

1. **Login (`/api/auth/login`)**

   * Anonymous access
   * Checks username/password
   * Generates JWT
   * Logs login via Serilog

2. **Protected APIs (`/api/products`)**

   * Require JWT (`[Authorize]`)
   * JWT automatically validated by middleware
   * Logs user info after successful validation

---

# **Key Notes**

* **Global `[Authorize]`** applied in `Program.cs` → only login API is exempt via `[AllowAnonymous]`.
* **JWT validation events** (`OnTokenValidated`, `OnAuthenticationFailed`) → central logging for all requests.
* **Serilog** handles console (or file) logging for both login and token validation events.

---

If you want, I can also add **automatic logging of failed login attempts** and **store them in DB or file** using Serilog, which is a common requirement for secure APIs.

Do you want me to do that?
