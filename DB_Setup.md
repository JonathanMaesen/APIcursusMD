# Chapter 8: Essential EF Core CLI Commands

This chapter serves as a practical reference for the .NET Core CLI tools used to manage database migrations and updates. These commands are the bridge between your C# code and the actual database schema.

---

## Prerequisites

Before running any commands, ensure the tools are installed globally on your machine:

```bash
dotnet tool install --global dotnet-ef
```

> **Verify installation:** Run `dotnet ef --version` to check if it is successfully installed.

---

## I. Managing Migrations

Migrations are version control for your database schema. They record the changes made to your C# data models so they can be applied to the database.

### 1. Adding a Migration (`migrations add`)

Use this command whenever you change your entity models (e.g., adding a property, creating a new class) and need to capture those changes.

*   **Syntax:** `dotnet ef migrations add <MigrationName>`
*   **Best Practice:** Give your migration a descriptive name indicating what changed (e.g., `AddProductTable`, `UpdateCustomerEmail`).

**Example:** You added a `Price` property to your `Product` class.

```bash
dotnet ef migrations add AddPriceToProduct
```

> **Output:** Creates two files in your `Migrations` folder:
> *   `<Timestamp>_AddPriceToProduct.cs`: Contains the instructions to apply (`Up`) and revert (`Down`) the changes.
> *   `<Timestamp>_AddPriceToProduct.Designer.cs`: Metadata for EF Core.

### 2. Removing the Last Migration (`migrations remove`)

If you made a mistake in your model before applying the migration to the database, use this to delete the last created migration file.

*   **Syntax:** `dotnet ef migrations remove`
*   **Warning:** Do not use this if you have already applied the migration to the database (using `database update`). If you have, you must rollback the database first (see Section II).

**Example:** You realized you misspelled "Price" as "Prce".

```bash
# 1. Remove the bad migration file
dotnet ef migrations remove

# 2. Fix the typo in your C# class

# 3. Create the migration again
dotnet ef migrations add AddPriceToProduct
```

---

## II. Updating the Database

Once a migration file exists, you must apply it to the actual database.

### 1. Applying Changes (`database update`)

This command looks for any migrations in your project that haven't been applied to the database yet and runs them.

*   **Syntax:** `dotnet ef database update`
*   **Targeting a Specific Migration:** You can travel back or forward in time by specifying a migration name.

**Example 1: Apply all pending migrations (Standard)**

```bash
dotnet ef database update
```

**Example 2: Rollback to a previous state**

If you need to undo changes in the database, target the migration *before* the one you want to remove.

```bash
# Reverts the database schema to the state of "InitialCreate"
dotnet ef database update InitialCreate
```

---

## III. Generating SQL Scripts

In production environments, it is often unsafe to run `dotnet ef database update`. Instead, you generate a SQL script to hand to a DBA or run as part of a deployment pipeline.

### 1. Scripting Migrations (`migrations script`)

Generates the SQL required to go from one migration to another.

*   **Syntax:** `dotnet ef migrations script <FromMigration> <ToMigration>`
*   **Defaults:** If no arguments are provided, it scripts all migrations.
*   **Idempotency:** You can add the `-i` or `--idempotent` flag. This checks if a migration has already been applied before trying to run it, which is much safer for production.

**Example: Generate a script for all migrations and save it to a file.**

```bash
dotnet ef migrations script --output deploy.sql
```

**Example: Generate an idempotent script (safe for running multiple times).**

```bash
dotnet ef migrations script --idempotent --output safe_deploy.sql
```

---

## IV. Scaffolding (Database-First)

If you are starting with an existing database and need to generate C# models from it.

### 1. Reverse Engineering (`dbcontext scaffold`)

*   **Syntax:** `dotnet ef dbcontext scaffold "<ConnectionString>" <Provider> -o <OutputFolder>`

**Example: Create models from a SQL Server database into a Models folder.**

```bash
dotnet ef dbcontext scaffold "Server=.;Database=MyDb;Trusted_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer -o Models
```

---

## V. Troubleshooting & Common Flags

These flags can be added to almost any command to help debug issues.

| Flag | Description |
| :--- | :--- |
| `--verbose` | Shows detailed output, including the raw SQL being executed. Essential for debugging errors. |
| `--project <Path>` | Specifies which project file to use. Useful if your DbContext is in a separate class library. |
| `--startup-project <Path>` | Specifies the executable project (usually your API) that contains the connection string configuration. |

**Example:** Running a migration when your DbContext is in a separate library (`DataLayer`) but your config is in the API (`MyApi`).

```bash
dotnet ef database update --project DataLayer --startup-project MyApi
```
