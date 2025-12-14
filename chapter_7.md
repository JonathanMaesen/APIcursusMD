# Chapter 7: Advanced EF Core Techniques & Best Practices

This chapter delves into advanced topics for optimizing data access performance, ensuring data integrity, and leveraging specialized querying features within Entity Framework (EF) Core.

---

## I. Performance and Resource Optimization

Optimizing the interaction between your API and the database is crucial for high throughput and responsiveness.

### 1. `DbContext` Pooling
In a typical web app, a new `DbContext` instance is created for every request, which has some overhead. **`DbContext` pooling** reuses `DbContext` instances instead of creating new ones, which can significantly improve performance in high-traffic scenarios.

> **How to implement it:** In `Program.cs`, simply replace `AddDbContext` with `AddDbContextPool`.
>
> ```csharp
> // In Program.cs
> builder.Services.AddDbContextPool<ApplicationDbContext>(options =>
>     options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"))
> );
> ```
> EF Core automatically resets the state of the pooled context before reusing it.

### 2. Tracking vs. No-Tracking Queries
EF Core's **change tracker** is what allows you to modify an entity and have `SaveChangesAsync()` automatically know what to update. However, this tracking has a performance cost.

*   **Tracking Queries (Default):** Use when you intend to **update** the data you retrieve.
*   **No-Tracking Queries:** Use for **read-only** operations (like in a `GET` endpoint). They are faster and use less memory.

> **How to use no-tracking:**
>
> ```csharp
> // Per-query basis: Use AsNoTracking()
> // Ideal for GET endpoints.
> [HttpGet]
> public async Task<IActionResult> GetProducts()
> {
>     var products = await _context.Products.AsNoTracking().ToListAsync();
>     return Ok(products);
> }
> ```
>
> You can also configure it globally in your `DbContext` if your application is mostly read-heavy.

### 3. `IQueryable` vs. `IEnumerable` (Server vs. Client Evaluation)
This distinction is critical for performance.

*   **`IQueryable`:** Represents a query that has **not yet executed**. LINQ methods like `Where()`, `OrderBy()`, and `Select()` on an `IQueryable` build up a SQL query. The filtering and sorting happen **on the database server**.
*   **`IEnumerable`:** Represents an **in-memory collection**. As soon as you convert to an `IEnumerable` (e.g., by calling `.ToList()` or `.AsEnumerable()`), the query executes and **all data is pulled into your application's memory**. Subsequent filtering happens on the client.

> **The Golden Rule:** Defer calling `.ToList()` or `.AsEnumerable()` for as long as possible so that the database does the heavy lifting.
>
> ```csharp
> // GOOD: Filtering happens on the database server. Only matching products are sent over the network.
> var cheapProducts = await _context.Products
>     .Where(p => p.Price < 10) // This is IQueryable
>     .ToListAsync(); // Query executes here
>
> // BAD: All products are loaded into memory, then filtered. Very inefficient!
> var allProducts = await _context.Products.ToListAsync(); // Pulls the ENTIRE table into memory
> var cheapProducts = allProducts.Where(p => p.Price < 10); // Filtering happens in your app
> ```

---

## II. Advanced Data Manipulation

### 4. Raw SQL Queries
For complex scenarios like stored procedures or highly optimized queries, EF Core allows you to drop down to raw SQL.

*   **`FromSql`:** The safe, recommended method for querying entities. It automatically parameterizes inputs to prevent SQL injection.
*   **`ExecuteSql`:** For executing non-query commands like `UPDATE`, `DELETE`, or calling stored procedures that don't return data.

> **Example:**
> ```csharp
> // Safely query entities using an interpolated string
> var recentProducts = await _context.Products
>     .FromSql($"SELECT * FROM Products WHERE CreationDate > {oneMonthAgo}")
>     .ToListAsync();
>
> // Safely execute a command
> await _context.Database.ExecuteSqlAsync(
>     $"UPDATE Products SET IsActive = 0 WHERE LastStockDate < {cutoffDate}"
> );
> ```

### 5. Bulk Operations (EF Core 7+)
These are highly efficient methods for updating or deleting many rows at once because they bypass the change tracker and execute a single SQL statement.

*   **`ExecuteUpdateAsync`:** Updates multiple rows based on a `Where` clause.
*   **`ExecuteDeleteAsync`:** Deletes multiple rows based on a `Where` clause.

> **Example:**
> ```csharp
> // Increase the price of all books by 10% in a single database command
> await _context.Books
>     .Where(b => b.Category == "Fiction")
>     .ExecuteUpdateAsync(setters =>
>         setters.SetProperty(b => b.Price, b => b.Price * 1.1m)
>     );
>
> // Delete all logs older than a year in a single database command
> await _context.Logs
>     .Where(l => l.Timestamp < DateTime.UtcNow.AddYears(-1))
>     .ExecuteDeleteAsync();
> ```
> **Warning:** Because these methods bypass the change tracker, any entities you have already loaded in memory will become stale.

---

## III. Concurrency Management

### 6. Optimistic Concurrency
This pattern prevents users from accidentally overwriting each other's changes.

**The Problem:**
1.  User A loads a product to edit.
2.  User B loads the same product, changes its price, and saves.
3.  User A (unaware of User B's change) saves their own changes, overwriting User B's update.

**The Solution (Optimistic Concurrency):**
You add a special "concurrency token" to your entity. When EF Core saves an entity, it checks if this token has changed in the database since it was first read. If it has, it means someone else modified it, and EF Core throws a `DbUpdateConcurrencyException`.

> **Implementation with `[Timestamp]` (SQL Server):**
>
> ```csharp
> public class Product
> {
>     public int Id { get; set; }
>     public string Name { get; set; }
>
>     [Timestamp] // This attribute tells EF Core to use this for concurrency checks
>     public byte[] RowVersion { get; set; }
> }
> ```
>
> **Handling the Exception:**
> In your controller, you must `try-catch` this exception and typically return an `HTTP 409 Conflict` status code, allowing the client to handle the conflict.
>
> ```csharp
> try
> {
>     await _context.SaveChangesAsync();
> }
> catch (DbUpdateConcurrencyException)
> {
>     // The record was modified by someone else.
>     return Conflict("This record has been modified by another user. Please reload and try again.");
> }
> ```

---

## IV. Other Frameworks and Approaches

### 7. Reverse Engineering (Database-First)
If you have an existing database, you can use EF Core to generate your `DbContext` and entity models from it.

> **Use the .NET CLI:**
> ```bash
> dotnet ef dbcontext scaffold "Your-Connection-String" Microsoft.EntityFrameworkCore.SqlServer -o Models
> ```
> This command will create C# classes for all the tables in your database and place them in the `Models` folder.

### 8. Dapper: The Micro-ORM
While EF Core is a full-featured ORM, **Dapper** is a "micro-ORM" known for its raw performance.
*   **What it is:** A simple library that excels at mapping the results of raw SQL queries to C# objects.
*   **What it isn't:** It does not have a change tracker, LINQ-to-SQL translation, or migrations.
*   **When to use it:** It's an excellent choice for performance-critical, read-heavy queries. Many projects use **both EF Core (for writes) and Dapper (for complex reads)** to get the best of both worlds.
