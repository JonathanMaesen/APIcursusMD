# Chapter 2: Getting Started with ASP.NET Core Web APIs

This chapter is your starting point for building ASP.NET Core Web APIs. We'll cover environment setup, project creation, core development patterns, and the essential concept of Dependency Injection (DI).

---

## I. Setting Up the Development Environment

First, let's get your machine ready and create the initial project.

### 1. Technical Requirements

*   **.NET SDK:** You need the .NET 8 SDK (or later). Verify your installation with the CLI command:
    ```bash
    dotnet --version
    ```
*   **IDE/Editor:**
    *   **Visual Studio:** Install with the "ASP.NET and web development" workload.
    *   **Visual Studio Code:** Install the "C# Dev Kit" extension.

### 2. Creating Your First Web API Project

The .NET Command-Line Interface (CLI) is the quickest way to start.

```bash
# Create a new web API project in a folder named "MyFirstApi"
dotnet new webapi -n MyFirstApi

# Navigate into the new project directory
cd MyFirstApi
```

#### Project Structure
Modern .NET API projects have a minimal structure:

*   `Program.cs`: The single entry point for the application. It handles both service registration and middleware pipeline configuration.
*   `appsettings.json`: For configuration settings.
*   `Properties/launchSettings.json`: Defines profiles for local development (URLs, environment variables).
*   `Controllers/`: A directory for your API controller classes (in a controller-based project).

### 3. Building, Running, and Hot Reload

*   **Build:** Compile your project to check for errors.
    ```bash
    dotnet build
    ```
*   **Run:** Build and run the application using the Kestrel web server.
    ```bash
    dotnet run
    ```
*   **Hot Reload:** For better productivity, run your app with `watch`. It automatically applies most code changes without a full restart.
    ```bash
    dotnet watch run
    ```

---

## II. Development and Debugging Tools

### 1. Swagger UI (OpenAPI)

The default project template includes Swagger, which provides interactive API documentation.

> **What it does:** It generates a web page that lists all your API endpoints, their parameters, and expected responses. You can even test your API directly from this page.

It is automatically enabled in `Program.cs` for the development environment:

```csharp
// Program.cs (simplified)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(); // This middleware enables the UI
}

app.Run();
```

### 2. Debugging
To find and fix issues, you'll need to debug.

1.  **Set Breakpoints:** Click in the margin next to a line of code in your editor to set a breakpoint.
2.  **Start Debugging:** Press `F5` in Visual Studio or use the "Run and Debug" view in VS Code.
3.  **Send a Request:** Use your browser or an API tool to send a request to the endpoint you want to debug.
4.  **Inspect:** When the breakpoint is hit, you can inspect variables, examine the call stack, and step through your code line-by-line.

---

## III. The MVC Pattern (Controller-Based APIs)

The Model-View-Controller (MVC) pattern is a robust way to organize your API by separating concerns. In a Web API, the "View" is typically the data returned, often as a JSON object.

### Implementing a Basic Controller

Controllers handle incoming HTTP requests. They use attributes like `[HttpGet]` and `[HttpPost]` to map methods to HTTP verbs.

> **Example:** A simple `ProductsController`.
>
> ```csharp
> // In Controllers/ProductsController.cs
> using Microsoft.AspNetCore.Mvc;
>
> [ApiController]
> [Route("api/[controller]")] // Sets the base route to "api/products"
> public class ProductsController : ControllerBase
> {
>     // GET: api/products
>     [HttpGet]
>     public IActionResult GetAllProducts()
>     {
>         // In a real app, you'd get this from a database
>         var products = new[] { new { Id = 1, Name = "Keyboard" } };
>         return Ok(products); // Returns 200 OK with the data
>     }
>
>     // GET: api/products/1
>     [HttpGet("{id}")]
>     public IActionResult GetProductById(int id)
>     {
>         // Find product...
>         if (product == null)
>         {
>             return NotFound(); // Returns 404 Not Found
>         }
>         return Ok(product);
>     }
>
>     // POST: api/products
>     [HttpPost]
>     public IActionResult CreateProduct([FromBody] Product newProduct)
>     {
>         // Save product...
>         // Returns a 201 Created status with a link to the new resource
>         return CreatedAtAction(nameof(GetProductById), new { id = newProduct.Id }, newProduct);
>     }
> }
> ```

---

## IV. Dependency Injection (DI)

DI is a core principle in ASP.NET Core. Instead of creating dependencies inside a class, you "inject" them from an external source (the DI container). This leads to more modular, testable, and maintainable code.

### 1. Principles of DI

*   **Inversion of Control (IoC):** The framework manages the creation and lifetime of your services.
*   **Interface-Based Programming:** Depend on abstractions (interfaces), not concrete implementations. This decouples your components.

### 2. Registering Services
You must register your services in `Program.cs`. Each service is registered with a specific **lifetime**.

```csharp
// In Program.cs
var builder = WebApplication.CreateBuilder(args);

// Registering services with different lifetimes
builder.Services.AddSingleton<ISomeSingletonService, SomeSingletonService>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddTransient<IEmailSender, EmailSender>();
```

*   **Singleton:** **One instance** for the entire application lifetime. Good for configuration or logging services.
*   **Scoped:** **One instance per HTTP request.** Good for database contexts or repositories.
*   **Transient:** **A new instance every time it's requested.** Good for lightweight, stateless services.

### 3. Consuming Services (Constructor Injection)

The most common way to receive a dependency is through the constructor.

```csharp
// In Controllers/ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _productRepository;

    // The DI container "injects" an instance of IProductRepository here
    public ProductsController(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }

    [HttpGet]
    public IActionResult Get()
    {
        var products = _productRepository.GetAll();
        return Ok(products);
    }
}
```

---

## V. Introduction to Minimal APIs

Minimal APIs are a lightweight alternative to controller-based APIs, perfect for microservices or simple APIs where the overhead of MVC isn't needed.

| Feature        | Controller-Based API (MVC)          | Minimal API                               |
|----------------|-------------------------------------|-------------------------------------------|
| **Structure**  | Dedicated `Controller` classes      | Logic defined directly in `Program.cs`    |
| **Routing**    | Uses attributes (`[HttpGet]`, etc.) | Uses `app.MapGet()`, `app.MapPost()`, etc. |
| **Boilerplate**| More setup and files                | Less code, minimal setup                  |

### Creating a Minimal API Endpoint

Endpoints are mapped directly in `Program.cs`. Dependencies are injected by adding them as parameters to the lambda expression.

> **Example:** A minimal API defined in `Program.cs`.
>
> ```csharp
> // In Program.cs
> var builder = WebApplication.CreateBuilder(args);
> builder.Services.AddScoped<IProductRepository, ProductRepository>(); // Register service
>
> var app = builder.Build();
>
> // Map a GET endpoint
> app.MapGet("/products", (IProductRepository repo) => {
>     var products = repo.GetAll();
>     return Results.Ok(products);
> });
>
> // Map a GET endpoint with a route parameter
> app.MapGet("/products/{id}", (int id, IProductRepository repo) => {
>     var product = repo.GetById(id);
>     return product is not null ? Results.Ok(product) : Results.NotFound();
> });
>
> app.Run();
> ```
