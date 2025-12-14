# Chapter 5: Data Access with Entity Framework Core

This chapter introduces data access in ASP.NET Core using **Entity Framework (EF) Core**, the standard Object-Relational Mapper (ORM) for .NET. We'll cover setup, basic CRUD operations, and mapping your C# models to a database.

---

## I. Why Use an Object-Relational Mapper (ORM)?

An ORM like EF Core acts as a bridge between your object-oriented C# code and a relational database (like SQL Server, PostgreSQL, or SQLite).

*   **Abstraction:** Lets you write database queries using C# and LINQ instead of raw SQL.
*   **Productivity:** Automates the tedious task of mapping database rows to C# objects.
*   **Type Safety:** Your queries are checked by the C# compiler, reducing runtime errors.
*   **Portability:** Makes it easier to switch between different database systems.

---

## II. Setting Up EF Core

The `DbContext` is the heart of EF Core. It represents a session with the database and allows you to query and save data.

### 1. The Entity Model (POCO)
An entity is a simple C# class (a "Plain Old C# Object" or POCO) that represents a table in your database.

> **Example:** A `Product` entity.
>
> ```csharp
> // In Models/Product.cs
> public class Product
> {
>     // Convention: A property named "Id" or "ProductId" becomes the primary key.
>     public int Id { get; set; }
>     public string Name { get; set; }
>     public decimal Price { get; set; }
>     public bool IsAvailable { get; set; }
> }
> ```

### 2. The `DbContext`
Create a class that inherits from `DbContext`. This class will contain `DbSet<T>` properties for each entity you want to manage.

> **Example:** `ApplicationDbContext`.
>
> ```csharp
> // In Data/ApplicationDbContext.cs
> using Microsoft.EntityFrameworkCore;
>
> public class ApplicationDbContext : DbContext
> {
>     public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
>         : base(options)
>     {
>     }
>
>     // A DbSet represents a table of Products.
>     public DbSet<Product> Products { get; set; }
> }
> ```

### 3. Registering the `DbContext`
Register your `DbContext` in the Dependency Injection (DI) container in `Program.cs`. This is also where you configure the database provider and connection string.

> **Example:** Registering with SQL Server.
>
> ```csharp
> // In Program.cs
> var builder = WebApplication.CreateBuilder(args);
>
> // 1. Get the connection string from appsettings.json
> var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
>
> // 2. Register the DbContext with a Scoped lifetime
> builder.Services.AddDbContext<ApplicationDbContext>(options =>
>     options.UseSqlServer(connectionString));
>
> // ...
> ```

### 4. Database Migrations (Code-First)
Migrations allow you to create and update your database schema based on your C# models.

**Step 1: Add a Migration**
This command creates a C# file that describes the changes needed to align the database with your models.

```bash
dotnet ef migrations add InitialCreate
```

**Step 2: Update the Database**
This command applies the pending migration to the database, creating tables and columns as needed.

```bash
dotnet ef database update
```

---

## III. Implementing CRUD Operations

Once set up, you can inject your `DbContext` into controllers or services to perform Create, Read, Update, and Delete (CRUD) operations.

> **Example:** A `ProductsRepository` using the `DbContext`.
>
> ```csharp
> public class ProductsRepository
> {
>     private readonly ApplicationDbContext _context;
>
>     public ProductsRepository(ApplicationDbContext context)
>     {
>         _context = context;
>     }
>
>     // CREATE
>     public async Task<Product> CreateAsync(Product newProduct)
>     {
>         _context.Products.Add(newProduct);
>         await _context.SaveChangesAsync(); // Persists changes to the database
>         return newProduct;
>     }
>
>     // READ (All)
>     public async Task<List<Product>> GetAllAsync()
>     {
>         return await _context.Products.ToListAsync();
>     }
>
>     // READ (By ID)
>     public async Task<Product?> GetByIdAsync(int id)
>     {
>         // FindAsync is optimized for finding by primary key
>         return await _context.Products.FindAsync(id);
>     }
>
>     // UPDATE
>     public async Task UpdateAsync(Product productToUpdate)
>     {
>         // EF Core's change tracker automatically detects modifications
>         _context.Products.Update(productToUpdate);
>         await _context.SaveChangesAsync();
>     }
>
>     // DELETE
>     public async Task DeleteAsync(int id)
>     {
>         var product = await GetByIdAsync(id);
>         if (product != null)
>         {
>             _context.Products.Remove(product);
>             await _context.SaveChangesAsync();
>         }
>     }
> }
> ```

---

## IV. Basic LINQ Querying Techniques

Use LINQ (Language Integrated Query) to write powerful queries that EF Core translates into efficient SQL.

### 1. Filtering (`Where`)
Translates to a SQL `WHERE` clause, ensuring filtering happens on the database server.

```csharp
// Get all available products cheaper than $50
var affordableProducts = await _context.Products
    .Where(p => p.IsAvailable && p.Price < 50)
    .ToListAsync();
```

### 2. Projection (`Select`)
Shapes the result into a different object (like a DTO), retrieving only the columns you need. This is a critical performance optimization.

```csharp
// Get only the names and prices of all products
var productDtos = await _context.Products
    .Select(p => new ProductDto { Name = p.Name, Price = p.Price })
    .ToListAsync();
```

### 3. Sorting and Paging
Essential for building APIs that return large datasets.

```csharp
public async Task<List<Product>> GetProductsPageAsync(int pageNumber, int pageSize)
{
    return await _context.Products
        .OrderBy(p => p.Name) // Sort by name
        .Skip((pageNumber - 1) * pageSize) // Skip previous pages
        .Take(pageSize) // Get the number of items for the current page
        .ToListAsync();
}
```

---

## V. Configuring the Model Mapping

While EF Core conventions are smart, you often need to customize the mapping between your models and the database.

### 1. Data Annotations
Attributes placed directly on your entity properties. Quick and easy for simple cases.

| Annotation          | Purpose                               |
|---------------------|---------------------------------------|
| `[Table("MyTable")]`| Maps the entity to a specific table.  |
| `[Column("DB_Name")]`| Maps a property to a specific column. |
| `[Key]`             | Marks a property as the primary key.  |
| `[Required]`        | Makes the column non-nullable.        |
| `[MaxLength(100)]`  | Sets the max length for a string.     |

> **Example:**
> ```csharp
> [Table("InventoryItems")]
> public class Product
> {
>     [Key]
>     public int ProductId { get; set; }
>
>     [Required]
>     [MaxLength(200)]
>     [Column("ItemName")]
>     public string Name { get; set; }
> }
> ```

### 2. Fluent API
A more powerful, code-based approach configured in your `DbContext`. It keeps your entity classes clean and centralizes mapping logic.

> **Example:** Overriding `OnModelCreating` in your `DbContext`.
>
> ```csharp
> protected override void OnModelCreating(ModelBuilder modelBuilder)
> {
>     modelBuilder.Entity<Product>(entity =>
>     {
>         // Map entity to the "InventoryItems" table
>         entity.ToTable("InventoryItems");
>
>         // Set ProductId as the primary key
>         entity.HasKey(p => p.ProductId);
>
>         // Configure the "Name" property
>         entity.Property(p => p.Name)
>             .HasColumnName("ItemName")
>             .IsRequired()
>             .HasMaxLength(200);
>     });
> }
> ```
> **Best Practice:** For larger models, move each entity's configuration into its own class that implements `IEntityTypeConfiguration<TEntity>`.
