In **.NET 8 (ASP.NET Core)**, routing is the mechanism that maps **incoming HTTP requests → to specific endpoints** (API actions, minimal API handlers, Razor pages, etc.).

Routing tells the framework:

* *Which code should run for a given URL?*
* *What parameters should be extracted from the URL?*
* *What HTTP method is allowed?*



# ✅ **What is Routing in .NET 8 API?**

Routing in .NET 8 determines how a request like:

```
GET /api/products/10?search=laptop
```

gets resolved to the correct handler, such as:

```csharp
app.MapGet("/api/products/{id}", (int id, string? search) => { ... });
```

Routing extracts:

* Path segments (`{id}`)
* Query parameters (`search=laptop`)
* Constraints (e.g., `{id:int}`)
* HTTP method (GET/POST/PUT/DELETE)
* Route groups & filters

Routing is processed via the **Endpoint Routing system**, introduced in ASP.NET Core 3.0 and improved in .NET 6–8.



# ✅ **Types of Routing in ASP.NET Core (.NET 8)**

There are **4 primary types** of routing you’ll use in API development:



# **1️⃣ Attribute Routing (Used in Controllers)**

Used with **Controller-based APIs**.

✔ Route is defined directly above the action.

### Controller Example

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id:int}")]
    public IActionResult GetProduct(int id) => Ok();
}
```

**Pros:**

* Very clear and readable
* Easy to manage versioning per-controller
* Best for large enterprise APIs

Routes are defined **directly on controller/actions**.

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    // GET /api/products/10
    [HttpGet("{id}")]
    public IActionResult GetProduct(int id)
    {
        return Ok($"Product id: {id}");
    }

    // POST /api/products
    [HttpPost]
    public IActionResult CreateProduct([FromBody] ProductDto product)
    {
        return Created($"/api/products/{product.Id}", product);
    }

    // PUT /api/products/10
    [HttpPut("{id}")]
    public IActionResult UpdateProduct(int id, [FromBody] ProductDto product)
    {
        return Ok($"Updated product {id}");
    }
}
```

**Notes:**

* More **explicit and clear** than conventional routing
* Routes are **directly tied to HTTP methods**

---


# **2️⃣ Conventional Routing (Older MVC style)**

Not very common in APIs, more used for MVC apps.

Defined centrally in Program.cs:

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "api/{controller}/{action}/{id?}");
```

Routes are *not* placed on actions (unless overridden).

**Pros:** centralized control
**Cons:** not recommended for APIs now


### **Controller Example**

```csharp
public class ProductsController : Controller
{
    // GET /Products/Get/10
    public IActionResult Get(int id)
    {
        return Content($"Get product id: {id}");
    }

    // POST /Products/Create
    [HttpPost]  // Optional, helps MVC know it's POST
    public IActionResult Create(ProductDto product)
    {
        return Created($"/Products/Get/{product.Id}", product);
    }

    // PUT /Products/Update/10
    [HttpPut]   // Optional, indicates PUT method
    public IActionResult Update(int id, ProductDto product)
    {
        return Ok($"Updated product {id}");
    }
}
```

**Notes:**

* URL format follows `{controller}/{action}/{id?}`
* `/Products/Get/10` → calls `Get(int id)`
* `/Products/Create` → POST with body → calls `Create(ProductDto product)`
* `/Products/Update/10` → PUT with body → calls `Update(int id, ProductDto product)`


# **3️⃣ Minimal API Routing (.NET 6 – .NET 8)**

Most common and modern approach.

No controllers. Routes defined directly in Program.cs.

### Example

```csharp
app.MapGet("/products/{id}", (int id) => Results.Ok());
app.MapPost("/products", (Product p) => Results.Created());
```

✔ Cleaner
✔ Faster
✔ Ideal for microservices

Everything is defined **inline in Program.cs**:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<ProductService>(); // business logic service

var app = builder.Build();

// GET /products/10
app.MapGet("/products/{id}", (int id, ProductService service) =>
{
    var product = service.GetById(id);
    return product != null ? Results.Ok(product) : Results.NotFound();
});

// POST /products
app.MapPost("/products", (ProductDto product, ProductService service) =>
{
    service.Add(product);
    return Results.Created($"/products/{product.Id}", product);
});

// PUT /products/10
app.MapPut("/products/{id}", (int id, ProductDto product, ProductService service) =>
{
    var updated = service.Update(id, product);
    return updated ? Results.Ok(product) : Results.NotFound();
});

app.Run();
```

**Notes:**

* No controllers needed
* Business logic lives in services
* Routes are **self-contained**



# **4️⃣ Route Groups (New in .NET 7/8)**

Allows grouping of routes under a common prefix and configuration.

### Example

```csharp
var group = app.MapGroup("/api/products");

group.MapGet("/", () => "all");
group.MapGet("/{id}", (int id) => id);
```

Supports:

* Authorization at group level
* Versioning
* Filters
* Middleware-like behavior



# 🎯 **Other Routing Features (Important)**

### ✔ **Route Constraints**

```csharp
app.MapGet("/emp/{id:int}", ...);
app.MapGet("/file/{name:minlength(3)}", ...);
```

### ✔ **Route Parameters**

* `{id}`
* `{id?}` optional
* `{*slug}` catch-all



# ⭐ **Summary Table**

| Routing Type             | Used In          | Best For                      | Example                       |
|  | - | -- | -- |
| **Attribute Routing**    | Controllers      | Large enterprise APIs         | `[HttpGet("items/{id}")]`     |
| **Conventional Routing** | MVC Apps         | Web apps, rarely for APIs     | `"api/{controller}/{action}"` |
| **Minimal APIs**         | Modern .NET APIs | Microservices, small services | `app.MapGet("/items")`        |
| **Route Groups**         | Minimal APIs     | Organizing routes             | `app.MapGroup("/api")`        |



