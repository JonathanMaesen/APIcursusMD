# Chapter 4: ASP.NET Core Fundamentals - Logging & Middleware

This chapter covers two fundamental pillars of any production-ready ASP.NET Core application: **Logging** for observability and diagnostics, and **Middleware** for building a flexible request processing pipeline.

---

## I. Comprehensive Logging in ASP.NET Core

Logging is vital for monitoring application health, diagnosing issues, and understanding runtime behavior. ASP.NET Core provides a robust, built-in logging system.

### 1. Log Levels and Configuration

Log messages are classified by severity, allowing you to filter out noise.

| Level         | Purpose                                                              |
|---------------|----------------------------------------------------------------------|
| `Trace`       | Highly detailed diagnostic information.                              |
| `Debug`       | Information useful during development and debugging.                 |
| `Information` | Tracks the general flow of the application (e.g., "Request started"). |
| `Warning`     | An unusual event that might indicate a problem.                      |
| `Error`       | The current operation failed due to an error or exception.           |
| `Critical`    | A failure requiring immediate attention (e.g., app crash).           |

You configure logging rules in `appsettings.json`.

> **Example:** `appsettings.json`
>
> ```json
> {
>   "Logging": {
>     "LogLevel": {
>       "Default": "Information", // Default level for all logs
>       "Microsoft.AspNetCore": "Warning" // Reduces noise from the framework itself
>     }
>   }
> }
> ```

### 2. How to Log: `ILogger<T>` and Structured Logging

Inject the `ILogger<T>` interface into your services or controllers to start logging.

**Crucially, always use message templates (structured logging).** This allows logging providers to treat your log data as searchable key-value pairs, which is essential for modern log analysis tools.

> **Example:** Injecting and using `ILogger` in a controller.
>
> ```csharp
> [ApiController]
> [Route("api/[controller]")]
> public class ProductsController : ControllerBase
> {
>     private readonly ILogger<ProductsController> _logger;
>
>     public ProductsController(ILogger<ProductsController> logger)
>     {
>         _logger = logger;
>     }
>
>     [HttpGet("{id}")]
>     public IActionResult GetProduct(int id)
>     {
>         _logger.LogInformation("Attempting to retrieve product with ID {ProductId}", id);
>
>         try
>         {
>             // ... logic to find the product
>             if (product == null)
>             {
>                 _logger.LogWarning("Product with ID {ProductId} was not found", id);
>                 return NotFound();
>             }
>
>             return Ok(product);
>         }
>         catch (Exception ex)
>         {
>             // Log the exception with details
>             _logger.LogError(ex, "An error occurred while retrieving product {ProductId}", id);
>             return StatusCode(500, "An internal server error occurred.");
>         }
>     }
> }
> ```
>
> **DO THIS:**
> ```csharp
> // Structured: Parameters are captured as distinct properties
> _logger.LogInformation("Product {ProductId} updated by user {UserName}", product.Id, user.Name);
> ```
>
> **NOT THIS:**
> ```csharp
> // Unstructured: Just a flat string, hard to query
> _logger.LogInformation($"Product {product.Id} updated by user {user.Name}");
> ```

### 3. Third-Party Providers (Serilog)

For production applications, third-party libraries like **Serilog** are highly recommended. They provide powerful features and "sinks" to send logs to various destinations (files, databases, cloud services like Azure Application Insights, etc.).

### 4. What to Log (and What Not To)

*   **DO Log:**
    *   Request start/end times.
    *   Errors and exceptions (with stack traces).
    *   Key business events (e.g., "Order created," "Payment processed").
    *   Calls to external services.
*   **DO NOT Log:**
    *   **Personal Identifiable Information (PII):** Passwords, credit card numbers, social security numbers.
    *   **Sensitive Data:** Raw API keys, connection strings, authentication tokens.

---

## II. The Middleware Pipeline

Middleware components form a processing pipeline that handles every incoming HTTP request and outgoing response. Each component can act on the request and either pass it to the next component or short-circuit the flow.

> **Execution Flow:** Requests travel "down" the pipeline, and responses travel back "up" in the reverse order.
>
> ```
> Request --> [MW1] --> [MW2] --> [MW3] --> Endpoint
> Response <-- [MW1] <-- [MW2] <-- [MW3] <--
> ```

The pipeline is configured in `Program.cs` using the `app` object. **Order is critical.**

### Common Built-in Middleware

Here is a typical middleware configuration in `Program.cs`:

```csharp
var app = builder.Build();

// 1. Exception Handler should be first to catch errors from later middleware.
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error"); // Redirects to a generic error page
    app.UseHsts(); // Enforces HTTPS
}

// 2. Redirect HTTP to HTTPS.
app.UseHttpsRedirection();

// 3. Serve static files (CSS, JS, images).
app.UseStaticFiles();

// 4. Routing determines which endpoint to hit.
app.UseRouting();

// 5. CORS (Cross-Origin Resource Sharing) must be after routing.
app.UseCors("MyCorsPolicy");

// 6. Authentication identifies the user (e.g., from a JWT).
app.UseAuthentication();

// 7. Authorization checks if the identified user has permission.
app.UseAuthorization();

// 8. Map endpoints (Controllers or Minimal APIs).
app.MapControllers();

app.Run();
```

### Creating Custom Middleware

You can create your own middleware to handle cross-cutting concerns like performance monitoring or custom request logging.

The cleanest way is to create a dedicated class.

> **Example:** A middleware to log request processing time.
>
> **1. The Middleware Class**
>
> ```csharp
> using System.Diagnostics;
>
> public class PerformanceLoggingMiddleware
> {
>     private readonly RequestDelegate _next;
>     private readonly ILogger<PerformanceLoggingMiddleware> _logger;
>
>     public PerformanceLoggingMiddleware(RequestDelegate next, ILogger<PerformanceLoggingMiddleware> logger)
>     {
>         _next = next;
>         _logger = logger;
>     }
>
>     public async Task InvokeAsync(HttpContext context)
>     {
>         var stopwatch = Stopwatch.StartNew();
>
>         // Call the next middleware in the pipeline
>         await _next(context);
>
>         stopwatch.Stop();
>         var elapsedMs = stopwatch.ElapsedMilliseconds;
>
>         _logger.LogInformation(
>             "Request [{Method}] {Path} processed in {ElapsedMilliseconds}ms",
>             context.Request.Method,
>             context.Request.Path,
>             elapsedMs);
>     }
> }
> ```
>
> **2. The Extension Method (for clean registration)**
>
> ```csharp
> public static class PerformanceLoggingMiddlewareExtensions
> {
>     public static IApplicationBuilder UsePerformanceLogging(this IApplicationBuilder builder)
>     {
>         return builder.UseMiddleware<PerformanceLoggingMiddleware>();
>     }
> }
> ```
>
> **3. Register it in `Program.cs`**
>
> ```csharp
> // In Program.cs
> var app = builder.Build();
>
> // ... other middleware
>
> // Add our custom middleware to the pipeline
> app.UsePerformanceLogging();
>
> app.UseRouting();
> // ...
> ```
