# Chapter 6: Data Access in EF Core - Entity Relationships

This chapter provides a deep dive into modeling and managing the various types of relationships between entities in a relational database using Entity Framework (EF) Core.

---

## I. One-to-Many Relationships

This is the most common relationship, where one entity (the "principal") can be related to many instances of another entity (the "dependent").
*   **Example:** One `Department` has many `Employees`.

### 1. Configuration by Convention
EF Core can automatically detect this relationship if your models follow a naming convention.

*   **Principal Entity (`Department`):** Contains a collection of the dependent entities.
*   **Dependent Entity (`Employee`):** Contains a **foreign key** property (`DepartmentId`) and a **navigation property** back to the principal (`Department`).

> **Example Models:**
>
> ```csharp
> // Principal (the "one" side)
> public class Department
> {
>     public int Id { get; set; }
>     public string Name { get; set; }
>
>     // Navigation property: a collection of related employees
>     public ICollection<Employee> Employees { get; set; }
> }
>
> // Dependent (the "many" side)
> public class Employee
> {
>     public int Id { get; set; }
>     public string Name { get; set; }
>
>     // Foreign Key property
>     public int DepartmentId { get; set; }
>
>     // Navigation property: a reference back to the single department
>     public Department Department { get; set; }
> }
> ```

### 2. Required vs. Optional Relationships
The nullability of the foreign key property determines if the relationship is required.

*   **Required:** `int DepartmentId` (non-nullable). An `Employee` *must* have a `Department`. This is the default for non-nullable types.
*   **Optional:** `int? DepartmentId` (nullable). An `Employee` *can* exist without a `Department`.

### 3. Cascade Delete Behavior
This defines what happens to dependent entities when their principal is deleted.

| Behavior      | Description                                                              | Default For...         |
|---------------|--------------------------------------------------------------------------|------------------------|
| **`Cascade`** | Deleting the `Department` also deletes all its `Employees`.              | Required relationships |
| **`Restrict`**| Prevents deleting the `Department` if it still has `Employees`.          | (Manual setting)       |
| **`SetNull`** | Deleting the `Department` sets `DepartmentId` to `NULL` for its `Employees`. | Optional relationships |

> **Fluent API Configuration:**
> ```csharp
> modelBuilder.Entity<Department>()
>     .HasMany(d => d.Employees)
>     .WithOne(e => e.Department)
>     .OnDelete(DeleteBehavior.SetNull); // Explicitly set behavior
> ```

### 4. Eager Loading with `Include`
To avoid performance issues (the "N+1 problem"), use `Include()` to load related data in a single query.

```csharp
// This query uses a JOIN to fetch the department and all its employees at once.
var department = await _context.Departments
    .Include(d => d.Employees) // Eagerly load the Employees collection
    .FirstOrDefaultAsync(d => d.Id == departmentId);
```

---

## II. One-to-One Relationships

Ensures that an instance of one entity is related to at most one instance of another.
*   **Example:** One `Employee` has one `Address`.

This relationship is ambiguous and **almost always requires Fluent API configuration.**

> **Example Models & Configuration:**
>
> ```csharp
> public class Employee
> {
>     public int Id { get; set; }
>     public string Name { get; set; }
>     public Address Address { get; set; } // Navigation to dependent
> }
>
> public class Address
> {
>     public int Id { get; set; }
>     public string Street { get; set; }
>
>     public int EmployeeId { get; set; } // Foreign Key
>     public Employee Employee { get; set; } // Navigation to principal
> }
>
> // In your DbContext's OnModelCreating:
> modelBuilder.Entity<Employee>()
>     .HasOne(e => e.Address) // Employee has one Address
>     .WithOne(a => a.Employee) // Address has one Employee
>     .HasForeignKey<Address>(a => a.EmployeeId); // The FK is in the Address table
> ```

---

## III. Many-to-Many Relationships

Allows flexible associations where entities on both sides can be related to multiple partners.
*   **Example:** A `Book` can have many `Authors`, and an `Author` can have many `Books`.

This requires a **join table** in the database.

### 1. Implicit Join Table (EF Core 5+)
This is the simplest approach when the relationship itself has no extra data.

> **Example Models & Configuration:**
>
> ```csharp
> public class Book
> {
>     public int Id { get; set; }
>     public string Title { get; set; }
>     public ICollection<Author> Authors { get; set; } // Collection navigation
> }
>
> public class Author
> {
>     public int Id { get; set; }
>     public string Name { get; set; }
>     public ICollection<Book> Books { get; set; } // Collection navigation
> }
>
> // In OnModelCreating, EF Core understands this convention automatically.
> // You can be explicit if you want:
> modelBuilder.Entity<Book>()
>     .HasMany(b => b.Authors)
>     .WithMany(a => a.Books);
> ```
> EF Core will create a `BookAuthor` join table behind the scenes.

### 2. Explicit Join Table (with Payload)
Use this when the relationship itself needs to store data (e.g., `DateJoined`, `Role`).

1.  **Create a Join Entity:** Model the join table as its own entity.
2.  **Configure Two One-to-Many Relationships:** Link both principal entities to the join entity.
3.  **Define a Composite Key:** The join entity's primary key must be a combination of the two foreign keys.

> **Example:**
> ```csharp
> // Join entity with a "payload" property
> public class BookAuthor
> {
>     public int BookId { get; set; }
>     public Book Book { get; set; }
>
>     public int AuthorId { get; set; }
>     public Author Author { get; set; }
>
>     public string Role { get; set; } // The extra data (payload)
> }
>
> // In OnModelCreating:
> // 1. Define the composite key
> modelBuilder.Entity<BookAuthor>()
>     .HasKey(ba => new { ba.BookId, ba.AuthorId });
>
> // 2. Configure the first one-to-many relationship
> modelBuilder.Entity<BookAuthor>()
>     .HasOne(ba => ba.Book)
>     .WithMany(b => b.BookAuthors) // Requires ICollection<BookAuthor> on Book
>     .HasForeignKey(ba => ba.BookId);
>
> // 3. Configure the second one-to-many relationship
> modelBuilder.Entity<BookAuthor>()
>     .HasOne(ba => ba.Author)
>     .WithMany(a => a.BookAuthors) // Requires ICollection<BookAuthor> on Author
>     .HasForeignKey(ba => ba.AuthorId);
> ```

---

## IV. Data Loading Strategies (Performance is Key)

| Strategy          | How it Works                                                              | When to Use                                                                 | Pro / Con                                                              |
|-------------------|---------------------------------------------------------------------------|-----------------------------------------------------------------------------|------------------------------------------------------------------------|
| **Eager Loading** | Loads related data in the same query using `.Include()`.                  | **Default choice for Web APIs.** When you know you'll always need the data. | **Pro:** Efficient (one query).<br/>**Con:** Can over-fetch data if not needed. |
| **Explicit Loading**| Loads related data on demand *after* the initial query.                   | When you only need the related data conditionally.                          | **Pro:** More control.<br/>**Con:** Requires an extra database round trip. |
| **Lazy Loading**  | Automatically loads related data the first time you access the property.  | **AVOID IN WEB APIS.** Can be useful in desktop apps.                       | **Pro:** Simple to write.<br/>**Con:** Causes the **N+1 query problem**, which is a major performance killer. |

> **Eager Loading is your best friend in API development.**
>
> ```csharp
> // GOOD: One efficient query
> var books = await _context.Books.Include(b => b.Authors).ToListAsync();
>
> // BAD (if Authors were lazy-loaded):
> // This would execute 1 query for the books, then N more queries for each book's authors.
> var books = await _context.Books.ToListAsync();
> foreach (var book in books) {
>     var authorNames = book.Authors.Select(a => a.Name); // <-- N+1 problem here!
> }
> ```
