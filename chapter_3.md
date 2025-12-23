# Chapter 3: ASP.NET Core Fundamentals - Routing & Configuration

This chapter introduces two fundamental concepts in ASP.NET Core: **Routing**, which maps incoming HTTP requests to your code, and **Configuration**, which manages application settings across different environments.

---

## I. Routing in ASP.NET Core

Routing is the process of matching an incoming request's URL to an endpoint in your application. For Web APIs, **Attribute Routing** is the preferred method.

### 1. Attribute Routing (Controller-Based APIs)

Attribute routing uses C# attributes to define the routes directly on your controllers and action methods.

*   **`[ApiController]`**: This attribute enables several API-specific conventions, including attribute routing.
*   **`[Route]`**: Defines the URL path. It can be placed on the controller or on individual action methods.
    *   The token `[controller]` is automatically replaced with the controller's name (e.g., `ProductsController` becomes `products`).

> **Example:** A controller with basic routing.
>
> ```csharp
> // The base route for all actions in this controller will be "api/products"
> [ApiController]
> [Route("api/[controller]")]
> public class ProductsController : ControllerBase
> {
>     // Handles GET requests to "api/products"
>     [HttpGet]
>     public IActionResult GetAll() { /* ... */ }
>
>     // Handles GET requests to "api/products/5"
>     // The {id} is a route parameter
>     [HttpGet("{id}")]
>     public IActionResult GetById(int id) { /* ... */ }
>
>     // Handles POST requests to "api/products"
>     [HttpPost]
>     public IActionResult Create([FromBody] Product product) { /* ... */ }
> }
> ```

### 2. Route Constraints

Route constraints enforce a specific data type or format on a route parameter. If the constraint is not met, the route will not match.

> **Syntax:** `{parameter:constraint}`

| Constraint      | Description                               | Example                       |
|-----------------|-------------------------------------------|-------------------------------|
| **`:int`**      | Matches a 32-bit integer.                 | `[HttpGet("{id:int}")]`       |
| **`:guid`**     | Matches a GUID.                           | `[HttpGet("{userId:guid}")]`  |
| **`:alpha`**    | Matches only alphabetical characters.     | `[HttpGet("{name:alpha}")]`   |
| **`:min(value)`** | Matches a value greater than or equal to `value`. | `[HttpGet("{id:min(1)}")]`    |
| **`:required`** | Ensures the parameter must be present.    | `[HttpGet("find/{term:required}")]` |

> **Example:** Using a route constraint.
>
> ```csharp
> // This endpoint will only match if 'id' is a valid integer.
> // A request to "api/products/abc" will result in a 404 Not Found.
> [HttpGet("{id:int}")]
> public IActionResult GetProductById(int id)
> {
>     // ... logic
> }
> ```

### 3. Binding Source Attributes

Model binding maps data from an HTTP request to your action method's parameters. Binding source attributes tell the framework where to find the data.

| Attribute | Binds From... | Default For... |
| :--- | :--- | :--- |
| `[FromRoute]` | The URL path segment. | Simple types (`int`, `string`) matching a route parameter. |
| `[FromQuery]` | The URL query string. | Simple types not found in the route. |
| `[FromBody]` | The request body (usually JSON/XML). | Complex types (`Product`, `UserDto`). |
| `[FromHeader]` | An HTTP request header. | Must be specified explicitly. |
| `[FromServices]` | The Dependency Injection container. | Must be specified explicitly. |

Example: An action method using multiple binding sources.

```csharp
// Request: PUT /api/products/123?notify=true
// Headers: "X-Api-Version": "2.0"
// Body: { "name": "New Name", "price": 150 }
[HttpPut("{id}")]
public IActionResult UpdateProduct(
    [FromRoute] int id,
    [FromBody] ProductUpdateDto productDto,
    [FromQuery] bool notify,
    [FromHeader(Name = "X-Api-Version")] string apiVersion)
{
    // id = 123
    // productDto = { Name = "New Name", Price = 150 }
    // notify = true
    // apiVersion = "2.0"
    // ...
}
```

---

## II. Configuration

ASP.NET Core has a flexible, hierarchical configuration system that allows you to manage settings for different environments.

### 1. `appsettings.json`

This JSON file is the primary place to store configuration data. You can have environment-specific versions like `appsettings.Development.json` that override the base settings.

> **Example:** `appsettings.json`
>
> ```json
> {
>   "Logging": {
>     "LogLevel": {
>       "Default": "Information"
>     }
>   },
>   "SmtpSettings": {
>     "Server": "smtp.example.com",
>     "Port": 587,
>     "SenderEmail": "noreply@example.com"
>   }
> }
> ```

### 2. The Options Pattern

The Options Pattern provides a strongly-typed way to access configuration settings, which is safer and cleaner than using string keys.

**Step 1: Create a Settings Class**
Create a C# class that matches the structure of your configuration section.

```csharp
public class SmtpSettings
{
    public string Server { get; set; }
    public int Port { get; set; }
    public string SenderEmail { get; set; }
}
```

**Step 2: Register the Options in `Program.cs`**
Bind the configuration section to your class.

```csharp
// In Program.cs
var builder = WebApplication.CreateBuilder(args);

// Binds the "SmtpSettings" section from appsettings.json to the SmtpSettings class
builder.Services.Configure<SmtpSettings>(
    builder.Configuration.GetSection("SmtpSettings")
);
```

**Step 3: Inject and Use the Options**
Use dependency injection to get an `IOptions<T>` instance in your services or controllers.

```csharp
public class EmailService
{
    private readonly SmtpSettings _smtpSettings;

    // Inject IOptions<SmtpSettings>
    public EmailService(IOptions<SmtpSettings> smtpOptions)
    {
        _smtpSettings = smtpOptions.Value; // The .Value property gives you the instance
    }

    public void SendEmail()
    {
        Console.WriteLine($"Sending email via {_smtpSettings.Server}:{_smtpSettings.Port}");
        // ... use _smtpSettings.SenderEmail, etc.
    }
}
```

### 3. Configuration Hierarchy
ASP.NET Core loads configuration from multiple providers in a specific order. Each subsequent provider overrides the previous one.

1.  `appsettings.json`
2.  `appsettings.{Environment}.json` (e.g., `appsettings.Development.json`)
3.  User Secrets (for development)
4.  **Environment Variables** (very common for production/Docker/Azure)
5.  Command-line Arguments

---

## III. Environments

Environments allow your app to behave differently depending on where it's running (e.g., `Development`, `Staging`, `Production`).

### 1. `launchSettings.json`
This file (in the `Properties` folder) is used for **local development only**. It defines profiles for launching your app and lets you set the `ASPNETCORE_ENVIRONMENT` variable.

> **Example:** `launchSettings.json`
>
> ```json
> {
>   "profiles": {
>     "http": {
>       "commandName": "Project",
>       "dotnetRunMessages": true,
>       "launchBrowser": true,
>       "applicationUrl": "http://localhost:5100",
>       "environmentVariables": {
>         "ASPNETCORE_ENVIRONMENT": "Development" // Sets the environment
>       }
>     }
>   }
> }
> ```

### 2. Checking the Environment in Code
Inject the `IWebHostEnvironment` service to check the current environment and execute conditional logic.

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // The 'env' variable gives you access to the current environment
    if (env.IsDevelopment())
    {
        // Enable developer-friendly features like Swagger UI and detailed error pages
        app.UseDeveloperExceptionPage();
        app.UseSwagger();
        app.UseSwaggerUI();
    }
    else
    {
        // Use more robust, production-ready middleware
        app.UseExceptionHandler("/error");
        app.UseHsts();
    }

    // ... other middleware
}
```
