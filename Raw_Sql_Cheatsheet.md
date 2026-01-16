# Raw SQL Cheatsheet

This cheatsheet provides a quick reference for common SQL commands and concepts, tailored for developers working with relational databases. It focuses on practical examples and standard SQL syntax.

---

## I. Data Query Language (DQL)

DQL is used to retrieve data from the database. The primary command is `SELECT`.

### 1. Basic Selection

Retrieve specific columns or all columns from a table.

*   **Syntax:** `SELECT column1, column2 FROM table_name;`
*   **Select All:** Use `*` to select all columns.

**Example:** Get the names and prices of all products.

```sql
SELECT Name, Price FROM Products;
```

### 2. Filtering Data (`WHERE`)

Filter rows based on specific conditions.

*   **Syntax:** `SELECT * FROM table_name WHERE condition;`
*   **Operators:** `=`, `<>`, `>`, `<`, `>=`, `<=`, `AND`, `OR`, `NOT`.

**Example:** Find all products with a price greater than 100.

```sql
SELECT * FROM Products WHERE Price > 100;
```

### 3. Pattern Matching (`LIKE`)

Search for a specified pattern in a column.

*   **Wildcards:**
    *   `%`: Represents zero or more characters.
    *   `_`: Represents a single character.

**Example:** Find customers whose names start with "J".

```sql
SELECT * FROM Customers WHERE Name LIKE 'J%';
```

### 4. Sorting Data (`ORDER BY`)

Sort the result set in ascending or descending order.

*   **Syntax:** `SELECT * FROM table_name ORDER BY column1 [ASC|DESC];`
*   **Default:** Ascending (`ASC`).

**Example:** List employees sorted by salary (highest to lowest).

```sql
SELECT * FROM Employees ORDER BY Salary DESC;
```

### 5. Limiting Results

Limit the number of rows returned.

*   **SQL Server:** `SELECT TOP number * FROM table_name;`
*   **PostgreSQL/MySQL:** `SELECT * FROM table_name LIMIT number;`

**Example (PostgreSQL):** Get the top 5 most expensive products.

```sql
SELECT * FROM Products ORDER BY Price DESC LIMIT 5;
```

---

## II. Data Manipulation Language (DML)

DML is used to modify data in the database.

### 1. Inserting Data (`INSERT`)

Add new rows to a table.

*   **Syntax:** `INSERT INTO table_name (column1, column2) VALUES (value1, value2);`

**Example:** Add a new customer.

```sql
INSERT INTO Customers (Name, Email) VALUES ('Alice Smith', 'alice@example.com');
```

### 2. Updating Data (`UPDATE`)

Modify existing data in a table.

*   **Syntax:** `UPDATE table_name SET column1 = value1 WHERE condition;`
*   **Warning:** Always use a `WHERE` clause, or you will update *all* rows.

**Example:** Update the price of product ID 10.

```sql
UPDATE Products SET Price = 19.99 WHERE ProductId = 10;
```

### 3. Deleting Data (`DELETE`)

Remove rows from a table.

*   **Syntax:** `DELETE FROM table_name WHERE condition;`
*   **Warning:** Always use a `WHERE` clause, or you will delete *all* rows.

**Example:** Remove a customer with ID 5.

```sql
DELETE FROM Customers WHERE CustomerId = 5;
```

---

## III. Joins

Joins are used to combine rows from two or more tables based on a related column.

### 1. Inner Join

Returns records that have matching values in both tables.

**Example:** Get orders and the customers who placed them.

```sql
SELECT Orders.OrderId, Customers.Name
FROM Orders
INNER JOIN Customers ON Orders.CustomerId = Customers.CustomerId;
```

### 2. Left (Outer) Join

Returns all records from the left table, and the matched records from the right table. If there is no match, the result is NULL on the right side.

**Example:** List all customers and their orders (if any).

```sql
SELECT Customers.Name, Orders.OrderId
FROM Customers
LEFT JOIN Orders ON Customers.CustomerId = Orders.CustomerId;
```

---

## IV. Aggregation

Perform calculations on a set of values and return a single value.

### 1. Common Functions

*   `COUNT()`: Returns the number of rows.
*   `SUM()`: Returns the total sum of a numeric column.
*   `AVG()`: Returns the average value.
*   `MIN()` / `MAX()`: Returns the smallest/largest value.

**Example:** Calculate the total revenue from all orders.

```sql
SELECT SUM(TotalAmount) FROM Orders;
```

### 2. Grouping (`GROUP BY`)

Group rows that have the same values into summary rows.

**Example:** Count the number of products in each category.

```sql
SELECT CategoryId, COUNT(*) as ProductCount
FROM Products
GROUP BY CategoryId;
```

### 3. Filtering Groups (`HAVING`)

Filter groups created by `GROUP BY`.

**Example:** Find categories with more than 10 products.

```sql
SELECT CategoryId, COUNT(*) as ProductCount
FROM Products
GROUP BY CategoryId
HAVING COUNT(*) > 10;
```

---

## V. Data Definition Language (DDL)

DDL is used to define and manage database structure.

### 1. Creating a Table (`CREATE TABLE`)

**Example:** Create a `Users` table.

```sql
CREATE TABLE Users (
    UserId INT PRIMARY KEY,
    Username VARCHAR(50) NOT NULL,
    Email VARCHAR(100) UNIQUE,
    CreatedAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2. Modifying a Table (`ALTER TABLE`)

**Example:** Add a `PhoneNumber` column to `Users`.

```sql
ALTER TABLE Users ADD PhoneNumber VARCHAR(20);
```

### 3. Deleting a Table (`DROP TABLE`)

**Example:** Delete the `Users` table completely.

```sql
DROP TABLE Users;
```
