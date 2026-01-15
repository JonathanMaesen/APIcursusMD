# Exam Foolproof Master Guide

## 1. The "Zero Thought" Project Setup

**Terminal Commands:**

```bash
# 1. Create the Solution
dotnet new sln -n ProjectAPI

# 2. Create the Projects
dotnet new webapi -n RestaurantsAPI
dotnet new classlib -n Restaurants.Shared

# 3. Add Projects to Solution
dotnet sln add RestaurantsAPI/RestaurantsAPI.csproj
dotnet sln add Restaurants.Shared/Restaurants.Shared.csproj

# 4. CRITICAL: Link API to Shared Library
dotnet add RestaurantsAPI/RestaurantsAPI.csproj reference Restaurants.Shared/Restaurants.Shared.csproj

# 5. Install NuGet Packages (Run in Solution Root)
# For the API Project (contains DbContext)
dotnet add RestaurantsAPI/RestaurantsAPI.csproj package Microsoft.EntityFrameworkCore.SqlServer
dotnet add RestaurantsAPI/RestaurantsAPI.csproj package Microsoft.EntityFrameworkCore.Tools
dotnet add RestaurantsAPI/RestaurantsAPI.csproj package Microsoft.EntityFrameworkCore.Design
```

---

## 2. The "Service Layer" Boilerplate

**Step A: The Interface (`Services/IRestaurantService.cs`)**

```csharp
using Restaurants.Shared; // Or wherever your models are

public interface IRestaurantService
{
    Task<List<Restaurant>> GetAllAsync();
    Task<Restaurant?> GetByIdAsync(int id);
    Task CreateAsync(Restaurant restaurant);
    Task UpdateAsync(Restaurant restaurant);
    Task DeleteAsync(int id);
}
```

**Step B: The Service Implementation (`Services/RestaurantService.cs`)**

```csharp
using Microsoft.EntityFrameworkCore;
using RestaurantsAPI.Data; // Your DbContext namespace
using Restaurants.Shared;

public class RestaurantService : IRestaurantService
{
    private readonly AppDbContext _context;

    public RestaurantService(AppDbContext context)
    {
        _context = context;
    }

    public async Task<List<Restaurant>> GetAllAsync()
    {
        return await _context.Restaurants.ToListAsync();
    }

    public async Task<Restaurant?> GetByIdAsync(int id)
    {
        return await _context.Restaurants.FindAsync(id);
    }

    public async Task CreateAsync(Restaurant restaurant)
    {
        _context.Restaurants.Add(restaurant);
        await _context.SaveChangesAsync();
    }

    public async Task UpdateAsync(Restaurant restaurant)
    {
        _context.Entry(restaurant).State = EntityState.Modified;
        await _context.SaveChangesAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var entity = await _context.Restaurants.FindAsync(id);
        if (entity != null)
        {
            _context.Restaurants.Remove(entity);
            await _context.SaveChangesAsync();
        }
    }
}
```

**Step C: The Registration (`Program.cs`)**

```csharp
// Add this BEFORE builder.Build()
builder.Services.AddScoped<IRestaurantService, RestaurantService>();
```

**Step D: The Controller Injection (`Controllers/RestaurantsController.cs`)**

```csharp
[ApiController]
[Route("api/[controller]")]
public class RestaurantsController : ControllerBase
{
    private readonly IRestaurantService _service;

    public RestaurantsController(IRestaurantService service)
    {
        _service = service;
    }

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        return Ok(await _service.GetAllAsync());
    }
    
    // ... implement other endpoints calling _service methods
}
```

---

## 3. The "Fluent API" Relationship Encyclopedia

**File:** `Data/AppDbContext.cs` -> `OnModelCreating` method.

**1-to-Many (Restaurant has many Reviews)**

```csharp
modelBuilder.Entity<Review>()
    .HasOne(r => r.Restaurant)
    .WithMany(r => r.Reviews)
    .HasForeignKey(r => r.RestaurantId)
    .OnDelete(DeleteBehavior.Cascade); // Optional: Deletes reviews if restaurant is deleted
```

**Many-to-Many (Restaurant has many Suppliers)**

```csharp
modelBuilder.Entity<Restaurant>()
    .HasMany(r => r.Suppliers)
    .WithMany(s => s.Restaurants)
    .UsingEntity(j => j.ToTable("RestaurantSuppliers")); // Creates the join table automatically
```

**1-to-1 (User has one Address)**

```csharp
modelBuilder.Entity<User>()
    .HasOne(u => u.Address)
    .WithOne(a => a.User)
    .HasForeignKey<Address>(a => a.UserId);
```

---

## 4. The "Panic Button" Migration List

**Create a Migration (Save your code changes)**
```bash
dotnet ef migrations add <NameOfMigration>
# Example: dotnet ef migrations add AddRestaurantTable
```

**Update Database (Apply changes to SQL)**
```bash
dotnet ef database update
```

**Undo Last Migration (If you messed up the code)**
*Condition: You have NOT run `database update` yet.*
```bash
dotnet ef migrations remove
```

**Rollback Database (If you ALREADY updated the DB)**
```bash
# 1. Revert DB to the previous migration
dotnet ef database update <NameOfPreviousMigration>

# 2. Remove the bad migration file
dotnet ef migrations remove
```
