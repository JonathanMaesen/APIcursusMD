# Exam Implementation Recipes

This document consolidates essential CLI commands, code patterns, and setup checklists for the API Development exam. It is designed for quick reference and copy-pasting.

---

## I. The "Emergency" CLI List

Use these commands to set up your project, add dependencies, and manage the database.

### 1. Project Setup
```bash
# Create a new Web API project
dotnet new webapi -n MyApiProject

# Add the PostgreSQL Entity Framework Core provider
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL

# Install the EF Core tools globally (if not already installed)
dotnet tool install --global dotnet-ef
```

### 2. Database Migrations
```bash
# Create a migration (captures changes in your C# models)
dotnet ef migrations add <MigrationName>
# Example: dotnet ef migrations add InitialCreate

# Apply the migration to the database
dotnet ef database update

# Remove the last migration (ONLY if not yet applied to DB)
dotnet ef migrations remove

# Generate a SQL script for production deployment
dotnet ef migrations script --output deploy.sql
```

### 3. Docker (PostgreSQL)
```bash
# Start a PostgreSQL container
docker run --name postgres-db -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=password -p 5432:5432 -d postgres
```

---

## II. Recipe: The Concurrency Pattern (Bank Account Example)

This recipe implements Optimistic Concurrency Control to prevent lost updates (e.g., two users withdrawing money simultaneously).

### 1. The Model (`BankAccount.cs`)
The `[Timestamp]` attribute creates a `byte[]` column that the database automatically updates on every write.

```csharp
using System.ComponentModel.DataAnnotations;

public class BankAccount
{
    public int Id { get; set; }
    public string AccountHolder { get; set; }
    public decimal Balance { get; set; }

    // The Concurrency Token
    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

### 2. The Controller (`BankAccountsController.cs`)
The `PUT` endpoint must catch `DbUpdateConcurrencyException` and return `409 Conflict`.

```csharp
[HttpPut("{id}")]
public async Task<IActionResult> UpdateBalance(int id, BankAccount updatedAccount)
{
    if (id != updatedAccount.Id)
    {
        return BadRequest();
    }

    // 1. Attach the entity to the context
    // This tells EF Core to track this object as "Modified"
    _context.Entry(updatedAccount).State = EntityState.Modified;

    try
    {
        // 2. Attempt to save changes
        // EF Core generates SQL: UPDATE ... WHERE Id = @Id AND RowVersion = @RowVersion
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException)
    {
        // 3. Handle the conflict
        if (!BankAccountExists(id))
        {
            return NotFound();
        }
        else
        {
            // The RowVersion in the DB didn't match the one sent by the client.
            // Someone else modified the record in the meantime.
            return Conflict(new { message = "The record you attempted to edit was modified by another user after you got the original value. The edit operation was canceled." });
        }
    }

    return NoContent();
}

private bool BankAccountExists(int id)
{
    return _context.BankAccounts.Any(e => e.Id == id);
}
```

---

## III. Recipe: The "Happy Path" Setup

**Goal:** Go from a blank folder to a running API with a database in < 5 minutes.

1.  **Create Project:**
    *   `dotnet new webapi -n MyExamApi`
    *   `cd MyExamApi`

2.  **Add Packages:**
    *   `dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL`
    *   `dotnet add package Microsoft.EntityFrameworkCore.Design`

3.  **Create Model:**
    *   Create `Models/Item.cs` with `Id` and `Name`.

4.  **Create DbContext:**
    *   Create `Data/AppDbContext.cs` inheriting from `DbContext`.
    *   Add `public DbSet<Item> Items { get; set; }`.
    *   Add constructor accepting `DbContextOptions<AppDbContext>`.

5.  **Configure Connection:**
    *   **`appsettings.json`**: Add `"ConnectionStrings": { "DefaultConnection": "Host=localhost;Port=5432;Database=MyDb;Username=admin;Password=password;" }`
    *   **`Program.cs`**: Add `builder.Services.AddDbContext<AppDbContext>(options => options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));`

6.  **Start Database:**
    *   Run the Docker command (see Section I).

7.  **Run Migrations:**
    *   `dotnet ef migrations add InitialCreate`
    *   `dotnet ef database update`

8.  **Run API:**
    *   `dotnet run`

---

## IV. Common Exam Pitfalls

*   **Forgot `UseNpgsql`:** Did you register the DbContext in `Program.cs`? If you see an error about "No database provider," this is why.
*   **Forgot `[Timestamp]`:** Without this attribute (or the Fluent API equivalent), EF Core won't check for concurrency conflicts.
*   **Forgot `try-catch`:** The `DbUpdateConcurrencyException` **must** be caught. If not, the API will return a generic `500 Internal Server Error` instead of the correct `409 Conflict`.
*   **Forgot to set `OriginalValue`:** In some advanced scenarios (manual tracking), you must explicitly set `entry.OriginalValues["RowVersion"]`. However, if you are passing the whole entity to `Update` or setting `State = Modified` (as in the recipe above), EF Core uses the `RowVersion` property from the object you passed in. **Crucial:** The client *must* send the `RowVersion` back in the PUT request!
*   **Docker Port Conflict:** If `docker run` fails, check if another container is already using port 5432 (`docker ps`). Stop it with `docker stop <container_id>`.
