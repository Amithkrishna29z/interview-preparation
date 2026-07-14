# Microsoft SQL Server — A Study Guide for .NET Developers (Coming From Java/MySQL)

## Overview

**Microsoft SQL Server** (often just "SQL Server" or "MSSQL") is Microsoft's flagship **relational database**. In the .NET world it is the *default, first-class* database — the same way MySQL or PostgreSQL is the go-to in the Java/open-source world. If you already know SQL (from your MySQL/PostgreSQL notes), you already know **90% of SQL Server**. What's left is: the **T-SQL dialect** (Microsoft's flavor of SQL), the **tooling** (SSMS, `sqlcmd`, Azure Data Studio), and — most importantly for a .NET job — **how a .NET app talks to it** (connection strings, ADO.NET, `SqlClient`, and EF Core's SQL Server provider).

> **This guide does NOT re-teach SQL.** Joins, indexes, normalization, transactions, and ACID are language-agnostic — reuse your existing SQL notes ([`MySQL_Interview_Questions.md`](../Java/MySQL_Interview_Questions.md), [`PostgreSQL_Interview_Questions.md`](../Java/PostgreSQL_Interview_Questions.md), [`SQL_Advanced_Window_Functions.md`](../Java/SQL_Advanced_Window_Functions.md), [`Database_Concepts_Interview_Questions.md`](../Java/Database_Concepts_Interview_Questions.md)). This guide covers **only what is SQL Server-specific and .NET-specific.**

This guide assumes **SQL Server 2019/2022** (or **Azure SQL Database**) and **.NET 8+**. It connects everything back to what you already know from MySQL/PostgreSQL and pairs with the [**Entity Framework Core Guide**](Entity_Framework_Core_Guide.md) — EF Core sits *on top* of what you learn here.

---

## Table of Contents

- [Overview](#overview)
- [MySQL/PostgreSQL → SQL Server Mapping](#mysqlpostgresql--sql-server-mapping)
- [1. Editions, Setup, and Tooling](#1-editions-setup-and-tooling)
- [2. T-SQL: How Microsoft's SQL Dialect Differs](#2-t-sql-how-microsofts-sql-dialect-differs)
- [3. Data Types You'll Actually Use](#3-data-types-youll-actually-use)
- [4. Identity, Sequences, and Keys](#4-identity-sequences-and-keys)
- [5. Connecting From .NET: Connection Strings](#5-connecting-from-net-connection-strings)
- [6. ADO.NET: The Raw Data Layer (Microsoft.Data.SqlClient)](#6-adonet-the-raw-data-layer-microsoftdatasqlclient)
- [7. Parameterized Queries & SQL Injection](#7-parameterized-queries--sql-injection)
- [8. Stored Procedures From .NET](#8-stored-procedures-from-net)
- [9. EF Core With the SQL Server Provider](#9-ef-core-with-the-sql-server-provider)
- [10. Transactions & Isolation Levels](#10-transactions--isolation-levels)
- [11. Indexes & Execution Plans](#11-indexes--execution-plans)
- [12. Connection Pooling & Common Pitfalls](#12-connection-pooling--common-pitfalls)
- [13. Running SQL Server in Docker (Great for Dev)](#13-running-sql-server-in-docker-great-for-dev)
- [Common Interview Questions](#common-interview-questions)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## MySQL/PostgreSQL → SQL Server Mapping

| MySQL / PostgreSQL | SQL Server (T-SQL) | Notes |
|---|---|---|
| `LIMIT 10` | `TOP 10` or `OFFSET/FETCH` | `SELECT TOP 10 * FROM t` |
| `LIMIT 10 OFFSET 20` | `OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY` | Needs an `ORDER BY` |
| `AUTO_INCREMENT` / `SERIAL` | `IDENTITY(1,1)` | Auto-generated PK |
| `` `backticks` `` (MySQL) | `[square brackets]` | Quoting identifiers |
| `NOW()` / `CURRENT_TIMESTAMP` | `GETDATE()` / `SYSDATETIME()` | Current time |
| `CONCAT(a,b)` or `a \|\| b` | `CONCAT(a,b)` or `a + b` | `+` concatenates strings |
| `IFNULL(x, y)` / `COALESCE` | `ISNULL(x, y)` / `COALESCE` | `ISNULL` is SQL Server-only |
| `LENGTH()` | `LEN()` | String length |
| `VARCHAR` (UTF-8) | `NVARCHAR` (Unicode) / `VARCHAR` | Use **`NVARCHAR`** for Unicode |
| `TEXT` | `NVARCHAR(MAX)` | Large text |
| `BOOLEAN` / `TINYINT(1)` | `BIT` (0/1) | No true boolean type |
| `DOUBLE` | `FLOAT` | |
| `SHOW TABLES;` | `SELECT * FROM sys.tables;` | Metadata via `sys.*` / `INFORMATION_SCHEMA` |
| `DESCRIBE t;` | `sp_help 't';` or `sp_columns` | Table structure |
| Semicolon optional | Semicolon optional (but `;` before CTEs) | `WITH` must follow a `;` |
| `--` and `/* */` comments | Same | |
| `mysql` / `psql` CLI | `sqlcmd` CLI | |
| MySQL Workbench / pgAdmin | **SSMS** / Azure Data Studio | GUI clients |
| Stored proc `DELIMITER //` | `CREATE PROCEDURE ... AS BEGIN ... END` | No delimiter dance |
| Auto-commit per statement | Auto-commit per statement | Same default |
| Schema = database (MySQL) | Server → **Database** → **Schema** (`dbo`) | Extra `schema` layer; default schema is `dbo` |
| JDBC + `mysql-connector` | ADO.NET + `Microsoft.Data.SqlClient` | The driver |
| `PreparedStatement` | `SqlCommand` + `SqlParameter` | Parameterized queries |

---

## 1. Editions, Setup, and Tooling

**Think of it like the JDK's editions**, but for a database. You pick an edition based on scale and budget:

- **Express** — free, capped (10 GB/DB). Perfect for learning and small apps. (Like the free tier.)
- **Developer** — free, *full* Enterprise features, but **licensed for dev/test only** (not production). **Use this to study** — you get every feature at zero cost.
- **Standard / Enterprise** — paid, production. Enterprise adds advanced HA, partitioning, etc.
- **Azure SQL Database** — the fully-managed cloud version (PaaS). You don't manage the server; you just get a database + connection string. Extremely common in modern .NET shops.
- **LocalDB** — a lightweight, on-demand SQL Server that installs with Visual Studio. Zero-config, file-based, great for local dev. Connection string: `Server=(localdb)\\MSSQLLocalDB;...`.

**Tooling you'll actually use:**

| Tool | What it is | Java analog |
|---|---|---|
| **SSMS** (SQL Server Management Studio) | The heavyweight Windows GUI — queries, admin, everything | MySQL Workbench / DBeaver |
| **Azure Data Studio** | Lightweight, cross-platform (VS Code-based) query tool | DBeaver |
| **`sqlcmd`** | Command-line client | `mysql` / `psql` CLI |
| **`dotnet ef`** | EF Core migrations CLI | Flyway / Liquibase CLI |

```sql
-- Quick sanity check in SSMS or Azure Data Studio:
SELECT @@VERSION;              -- prints the SQL Server version string
SELECT DB_NAME();              -- current database name
SELECT name FROM sys.databases; -- list all databases on the server
```

---

## 2. T-SQL: How Microsoft's SQL Dialect Differs

**T-SQL (Transact-SQL)** is Microsoft's extension of standard SQL — the equivalent of MySQL's dialect or PostgreSQL's PL/pgSQL. The `SELECT/INSERT/UPDATE/DELETE`/join grammar is identical to what you know. Here are the differences that trip people up:

```sql
-- 1) TOP instead of LIMIT
SELECT TOP 5 * FROM Employees ORDER BY Salary DESC;   -- top 5 earners

-- 2) Paging: OFFSET/FETCH (needs ORDER BY)
SELECT * FROM Employees
ORDER BY Id
OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;               -- page 3, 10 per page

-- 3) Variables use @ and DECLARE
DECLARE @minSalary INT = 50000;                        -- declare + assign
SELECT * FROM Employees WHERE Salary > @minSalary;

-- 4) String concatenation with +
SELECT FirstName + ' ' + LastName AS FullName FROM Employees;

-- 5) IIF and CASE
SELECT Name, IIF(Salary > 100000, 'Senior', 'Junior') AS Tier FROM Employees;

-- 6) ISNULL (SQL Server-specific) vs COALESCE (standard)
SELECT ISNULL(MiddleName, 'N/A') FROM Employees;       -- replace NULL

-- 7) Identifiers with spaces/keywords use [brackets]
SELECT [Order], [Group Name] FROM [User Data];

-- 8) Batch separator: GO (SSMS/sqlcmd only — NOT valid T-SQL, it's a tool directive)
CREATE TABLE Foo (Id INT);
GO   -- sends the batch to the server; do not put this in app code
```

**Control-of-flow** (for stored procs/scripts) — note there's no `THEN`/`ENDIF`, you use `BEGIN...END`:

```sql
IF EXISTS (SELECT 1 FROM Employees WHERE Id = 1)
BEGIN
    PRINT 'Employee exists';        -- PRINT is like console output
END
ELSE
BEGIN
    PRINT 'Not found';
END

-- Loops:
DECLARE @i INT = 0;
WHILE @i < 5
BEGIN
    PRINT @i;
    SET @i = @i + 1;                -- SET assigns to a variable
END
```

---

## 3. Data Types You'll Actually Use

**Think of it like Java's primitives + boxed types**, but you must pick precise column types. The one rule to burn in: **use `NVARCHAR` for text** (Unicode) unless you have a specific reason not to.

| Category | SQL Server type | Use for | Notes / gotcha |
|---|---|---|---|
| Whole numbers | `INT`, `BIGINT`, `SMALLINT`, `TINYINT` | IDs, counts | `TINYINT` is 0–255 (unsigned byte) |
| Boolean | `BIT` | true/false | Stores `0`/`1`; maps to C# `bool` |
| Decimal (exact) | `DECIMAL(p,s)` / `NUMERIC` | **money**, precise values | Use for currency, not `FLOAT` |
| Money | `MONEY` | currency | Prefer `DECIMAL(19,4)` in practice |
| Float (approx) | `FLOAT`, `REAL` | scientific | Never use for money (rounding) |
| Unicode text | `NVARCHAR(n)`, `NVARCHAR(MAX)` | **all text** | `N` = Unicode; maps to C# `string` |
| ASCII text | `VARCHAR(n)` | ASCII-only | Half the storage, but no Unicode |
| Date/time | `DATE`, `TIME`, `DATETIME2`, `DATETIMEOFFSET` | timestamps | **Prefer `DATETIME2`** over legacy `DATETIME` |
| GUID | `UNIQUEIDENTIFIER` | distributed IDs | Maps to C# `Guid`; `NEWID()` generates one |
| Binary | `VARBINARY(MAX)` | files/blobs | Maps to C# `byte[]` |

```sql
CREATE TABLE Products (
    Id          INT IDENTITY(1,1) PRIMARY KEY,   -- auto-increment PK
    Sku         UNIQUEIDENTIFIER DEFAULT NEWID(), -- auto GUID
    Name        NVARCHAR(200)  NOT NULL,          -- Unicode text
    Price       DECIMAL(19,4)  NOT NULL,          -- exact money
    IsActive    BIT            NOT NULL DEFAULT 1, -- boolean
    CreatedAt   DATETIME2      NOT NULL DEFAULT SYSDATETIME()
);
```

**C# ↔ SQL Server type mapping** (what EF Core / ADO.NET maps automatically):

| C# | SQL Server |
|---|---|
| `int` / `long` | `INT` / `BIGINT` |
| `bool` | `BIT` |
| `decimal` | `DECIMAL` |
| `double` | `FLOAT` |
| `string` | `NVARCHAR` |
| `DateTime` | `DATETIME2` |
| `DateTimeOffset` | `DATETIMEOFFSET` |
| `Guid` | `UNIQUEIDENTIFIER` |
| `byte[]` | `VARBINARY` |

---

## 4. Identity, Sequences, and Keys

Instead of MySQL's `AUTO_INCREMENT` or Postgres's `SERIAL`, SQL Server uses **`IDENTITY(seed, increment)`**:

```sql
CREATE TABLE Orders (
    Id INT IDENTITY(1,1) PRIMARY KEY   -- start at 1, step by 1
);

INSERT INTO Orders DEFAULT VALUES;
SELECT SCOPE_IDENTITY();               -- the ID just generated (this session/scope)
-- ⚠️ Use SCOPE_IDENTITY(), NOT @@IDENTITY: @@IDENTITY leaks IDs from triggers.
```

**Sequences** (like an Oracle/Postgres sequence — a standalone number generator, decoupled from a table):

```sql
CREATE SEQUENCE OrderNumberSeq START WITH 1000 INCREMENT BY 1;
SELECT NEXT VALUE FOR OrderNumberSeq;   -- 1000, then 1001, ...
```

**GUID primary keys** are common in distributed .NET systems (no central counter needed):

```sql
CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID()
    -- NEWSEQUENTIALID() generates *sequential* GUIDs → far less index fragmentation
    -- than random NEWID() when used as a clustered PK.
);
```

> **Interview nugget:** A random GUID (`NEWID()`) as a **clustered** primary key fragments the index badly because rows insert in random order. Use `NEWSEQUENTIALID()`, or make the GUID a non-clustered key, or generate sequential GUIDs in C# (`Guid.NewGuid()` is random — libraries exist for sequential).

---

## 5. Connecting From .NET: Connection Strings

The connection string is the single most common thing you'll configure — and misconfigure. It lives in **`appsettings.json`** (the .NET analog of `application.properties`):

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=ShopDb;User Id=sa;Password=Your_password123;TrustServerCertificate=True;"
  }
}
```

**Anatomy of a connection string:**

| Key | Meaning | Java/JDBC analog |
|---|---|---|
| `Server` (or `Data Source`) | host\instance, e.g. `localhost`, `.\SQLEXPRESS`, `(localdb)\MSSQLLocalDB` | JDBC host:port |
| `Database` (or `Initial Catalog`) | the database name | JDBC db name |
| `User Id` + `Password` | **SQL authentication** | username/password |
| `Integrated Security=true` | **Windows authentication** (no password) | — (Windows-only) |
| `TrustServerCertificate=True` | skip TLS cert validation (dev only!) | — |
| `Encrypt=True` | encrypt the connection (default in newer drivers) | `useSSL=true` |
| `MultipleActiveResultSets=True` | allow multiple open readers on one connection | — |

**Two authentication modes** (know both for interviews):

- **SQL Authentication** — username + password stored in SQL Server (e.g. the `sa` account). Portable, works everywhere, used in Docker/Linux/Azure.
- **Windows / Integrated Authentication** — uses your Windows/AD identity, no password in the string (`Integrated Security=true`). Common in Windows-only corporate environments; more secure (no stored password).

> **Security:** Never hardcode passwords or commit them. Use **User Secrets** in dev (`dotnet user-secrets set`), and **environment variables / Azure Key Vault** in prod — see the [Deployment & Ops guide](DotNet_Deployment_And_Ops.md).

---

## 6. ADO.NET: The Raw Data Layer (Microsoft.Data.SqlClient)

**ADO.NET is the JDBC of .NET** — the low-level API every higher-level tool (EF Core, Dapper) is built on. You rarely write it by hand day-to-day, but interviewers love it because it shows you understand what EF Core hides.

The modern package is **`Microsoft.Data.SqlClient`** (NOT the old `System.Data.SqlClient` — that's deprecated).

```bash
dotnet add package Microsoft.Data.SqlClient
```

The **core objects** (memorize these four — they mirror JDBC):

| ADO.NET | JDBC | Role |
|---|---|---|
| `SqlConnection` | `Connection` | the open pipe to the DB |
| `SqlCommand` | `Statement` / `PreparedStatement` | a SQL command to run |
| `SqlDataReader` | `ResultSet` | forward-only, streaming row reader |
| `SqlParameter` | `?` bind parameter | a safe, typed parameter |

```csharp
using Microsoft.Data.SqlClient;                        // the SQL Server ADO.NET driver

string connString = config.GetConnectionString("Default"); // from appsettings.json

// 'using' = try-with-resources in Java: auto-closes/disposes the connection.
using var conn = new SqlConnection(connString);
await conn.OpenAsync();                                 // open the physical connection

// Parameterized query — NEVER string-concatenate user input (see §7).
using var cmd = new SqlCommand(
    "SELECT Id, Name, Price FROM Products WHERE Price > @min", conn);
cmd.Parameters.AddWithValue("@min", 100m);             // bind @min safely

using var reader = await cmd.ExecuteReaderAsync();      // like executeQuery()
while (await reader.ReadAsync())                        // advance one row at a time
{
    int id       = reader.GetInt32(0);                 // column 0 → int
    string name  = reader.GetString(1);                // column 1 → string
    decimal price = reader.GetDecimal(2);              // column 2 → decimal
    Console.WriteLine($"{id}: {name} = {price:C}");
}
```

**Three ways to execute a command** (know when to use each):

```csharp
// 1) ExecuteReader   → rows back (SELECT)
using var r = await cmd.ExecuteReaderAsync();

// 2) ExecuteNonQuery → no rows; returns # rows affected (INSERT/UPDATE/DELETE)
int rowsAffected = await cmd.ExecuteNonQueryAsync();

// 3) ExecuteScalar   → a single value (e.g. SELECT COUNT(*) or SCOPE_IDENTITY())
object? count = await cmd.ExecuteScalarAsync();
```

> **Where Dapper fits:** In real jobs you'll often see **Dapper**, a "micro-ORM" that's a thin, fast wrapper over ADO.NET (`conn.Query<Product>("SELECT ...")`). It maps rows to objects for you but you still write the SQL — the middle ground between raw ADO.NET and full EF Core.

---

## 7. Parameterized Queries & SQL Injection

This is the **#1 security question** you'll get. The rule is identical to Java's `PreparedStatement`: **never concatenate user input into SQL.** Use parameters, always.

```csharp
// ❌ VULNERABLE — string concatenation. A user entering
//    name = "'; DROP TABLE Products; --" wrecks you.
var bad = new SqlCommand(
    $"SELECT * FROM Products WHERE Name = '{userInput}'", conn);

// ✅ SAFE — the value is sent separately from the SQL text; the DB never
//    treats it as code. This is parameterization / prepared statements.
var good = new SqlCommand(
    "SELECT * FROM Products WHERE Name = @name", conn);
good.Parameters.Add("@name", SqlDbType.NVarChar, 200).Value = userInput;
// (Prefer the typed .Add(...) over AddWithValue for large tables — it avoids
//  parameter-type-inference surprises that can hurt query plans.)
```

**Why it's safe:** the SQL text and the data travel on separate channels. The `@name` value is *always* data, never executable SQL — so injection is structurally impossible, not just filtered.

> **EF Core does this automatically.** Every LINQ query is parameterized under the hood. The only time you can reintroduce injection in EF Core is with `FromSqlRaw($"...{userInput}")` string interpolation — use `FromSqlInterpolated` or explicit `SqlParameter`s instead.

---

## 8. Stored Procedures From .NET

**Think of a stored procedure like a method defined and stored *in the database*** — precompiled SQL you call by name. Common in enterprise .NET shops.

```sql
-- Define it once in SQL Server:
CREATE PROCEDURE GetProductsByMinPrice
    @MinPrice DECIMAL(19,4)          -- parameter, like a method argument
AS
BEGIN
    SET NOCOUNT ON;                  -- suppress "N rows affected" chatter
    SELECT Id, Name, Price FROM Products WHERE Price >= @MinPrice;
END
```

Calling it from ADO.NET — the only change is `CommandType.StoredProcedure`:

```csharp
using var cmd = new SqlCommand("GetProductsByMinPrice", conn);
cmd.CommandType = CommandType.StoredProcedure;         // it's a proc, not raw SQL
cmd.Parameters.AddWithValue("@MinPrice", 50m);
using var reader = await cmd.ExecuteReaderAsync();
// ... read rows as usual
```

Calling it from **EF Core**:

```csharp
// For an entity-shaped result set:
var products = await db.Products
    .FromSqlRaw("EXEC GetProductsByMinPrice @MinPrice = {0}", 50m)
    .ToListAsync();

// For a non-query proc (INSERT/UPDATE):
await db.Database.ExecuteSqlRawAsync("EXEC ArchiveOldOrders @Days = {0}", 30);
```

> **Interview angle — procs vs LINQ/EF:** Procs can be faster (precompiled plan), centralize logic in the DB, and let DBAs tune them independently. Downsides: logic lives outside source control/your app, harder to test, and couples you to SQL Server. Modern .NET favors EF Core/LINQ but you *will* meet proc-heavy legacy codebases.

---

## 9. EF Core With the SQL Server Provider

This is where it all comes together for the job. The [**EF Core guide**](Entity_Framework_Core_Guide.md) covers the ORM in depth; here's specifically the **SQL Server wiring**:

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer   # the SQL Server provider
dotnet add package Microsoft.EntityFrameworkCore.Design      # for migrations
```

```csharp
// Program.cs — register the DbContext against SQL Server
builder.Services.AddDbContext<ShopDbContext>(options =>
    options.UseSqlServer(                                    // ← the provider swap
        builder.Configuration.GetConnectionString("Default")));
```

That `UseSqlServer(...)` line is the *only* SQL Server-specific part — swap it for `UseNpgsql` (PostgreSQL) or `UseSqlite` and the rest of your EF Core code is identical. That provider-abstraction is EF Core's big selling point.

**Migrations against SQL Server** (the Flyway/Liquibase analog):

```bash
dotnet ef migrations add InitialCreate   # generate a C# migration from your model
dotnet ef database update                # apply pending migrations to SQL Server
dotnet ef migrations remove              # undo the last (unapplied) migration
```

**SQL Server-specific EF Core knobs you should recognize:**

```csharp
protected override void OnModelCreating(ModelBuilder mb)
{
    mb.Entity<Product>()
      .Property(p => p.Price)
      .HasColumnType("decimal(19,4)");        // control the exact SQL type

    mb.Entity<Product>()
      .Property(p => p.RowVersion)
      .IsRowVersion();                         // SQL Server ROWVERSION for
                                               // optimistic concurrency (see §10)
}
```

---

## 10. Transactions & Isolation Levels

ACID and isolation-level *theory* is in your [Database Concepts notes](../Java/Database_Concepts_Interview_Questions.md) — reuse it. Here's what's **SQL Server-specific**:

- SQL Server's **default isolation level is `READ COMMITTED`** (same as Postgres/Oracle; MySQL InnoDB defaults to `REPEATABLE READ`).
- SQL Server uses **locking** by default and can suffer **blocking/deadlocks**. It also offers **`READ COMMITTED SNAPSHOT` (RCSI)** and `SNAPSHOT` isolation, which use row-versioning (like Postgres MVCC) to avoid readers blocking writers. Enabling RCSI is a common performance fix.

```sql
-- Explicit T-SQL transaction:
BEGIN TRANSACTION;
    UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;
    UPDATE Accounts SET Balance = Balance + 100 WHERE Id = 2;
    -- if anything failed you'd ROLLBACK; otherwise:
COMMIT TRANSACTION;

SET TRANSACTION ISOLATION LEVEL SNAPSHOT;   -- change isolation for the session
```

**Transactions in EF Core** — `SaveChanges()` is already atomic (all changes in one transaction). For spanning multiple `SaveChanges` calls:

```csharp
using var tx = await db.Database.BeginTransactionAsync();
try
{
    db.Accounts.Add(new Account { ... });
    await db.SaveChangesAsync();
    // ... more work ...
    await db.SaveChangesAsync();
    await tx.CommitAsync();                  // commit all-or-nothing
}
catch
{
    await tx.RollbackAsync();                // undo everything
    throw;
}
```

**Optimistic concurrency** — SQL Server's `ROWVERSION`/`TIMESTAMP` column auto-increments on every update. EF Core checks it on `SaveChanges` and throws `DbUpdateConcurrencyException` if another user changed the row first (the analog of JPA's `@Version`).

---

## 11. Indexes & Execution Plans

Index *fundamentals* transfer from your SQL notes. The **SQL Server-specific** things to know:

- **Clustered index** — *physically orders* the table's rows and **IS the table** (the leaf level holds the actual data). A table has **at most one**. By default, the **primary key becomes the clustered index**. Choosing the right clustered key (narrow, ever-increasing, like an `IDENTITY int`) matters a lot.
- **Non-clustered index** — a separate structure with pointers back to the row (up to ~999 per table). The analog of a normal secondary index.
- **Covering index / `INCLUDE`** — a non-clustered index that stashes extra columns so a query is answered from the index alone (no lookup back to the table).

```sql
-- Non-clustered index that "covers" a common query:
CREATE NONCLUSTERED INDEX IX_Products_Price
    ON Products (Price)          -- key column (for WHERE/ORDER BY)
    INCLUDE (Name);              -- extra columns carried along (for SELECT)

-- Look at how SQL Server will run a query:
SET STATISTICS IO ON;            -- shows logical reads (I/O cost)
-- In SSMS: Ctrl+M for the actual execution plan, or:
SET SHOWPLAN_ALL ON;
```

> **Interview nugget:** In an execution plan, an **Index Seek** (jump straight to matching rows) is good; a **Clustered Index Scan / Table Scan** (read everything) on a large table usually means a missing or unusable index — often caused by a non-SARGable predicate like `WHERE YEAR(CreatedAt) = 2024` (wrap the column in a function and the index can't be used).

---

## 12. Connection Pooling & Common Pitfalls

**Connection pooling is on by default** and is critical — opening a physical SQL Server connection is expensive (~ms). `SqlConnection` pools under the hood: `Open()` grabs a pooled connection, `Dispose()`/`Close()` **returns it to the pool** (it doesn't truly close). Pools are keyed by the exact connection string.

**Pitfalls interviewers probe:**

- **Not disposing connections** → pool exhaustion → `timeout expired` errors under load. Always use `using`. (Same lesson as leaking JDBC connections.)
- **Slightly different connection strings** → multiple separate pools. Keep it consistent.
- **`DbContext` is not thread-safe** and should be **scoped** (one per request), never a singleton — see the [EF Core guide §15](Entity_Framework_Core_Guide.md).
- **`AddWithValue` type inference** → can produce bad query plans (e.g. inferring `NVARCHAR` against a `VARCHAR` column forces a scan). Use typed `.Add(name, SqlDbType...)` on hot paths.
- **N+1 queries** in EF Core → use `.Include()` / projection; covered in the EF Core guide.

---

## 13. Running SQL Server in Docker (Great for Dev)

You don't need Windows or a full install — run SQL Server in a container (reuse your [Docker notes](../Java/Docker_Concepts_Study_Guide.md)):

```bash
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=Your_password123" \
   -p 1433:1433 --name mssql -d \
   mcr.microsoft.com/mssql/server:2022-latest
```

- `1433` is SQL Server's default port (the analog of MySQL's 3306 / Postgres's 5432).
- Connect with `Server=localhost,1433;User Id=sa;Password=Your_password123;TrustServerCertificate=True;`.
- Great for CI pipelines and local dev — spin up, run integration tests, tear down.

```yaml
# docker-compose.yml snippet
services:
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "Your_password123"
    ports: ["1433:1433"]
```

---

## Common Interview Questions

1. **How does SQL Server differ from MySQL/PostgreSQL?**
   Same core SQL; differences are the **T-SQL dialect** (`TOP` vs `LIMIT`, `IDENTITY` vs `AUTO_INCREMENT`, `NVARCHAR`, `ISNULL`, `[brackets]`), tooling (SSMS), Windows-first heritage, and it's the default DB in the .NET ecosystem.

2. **What's `NVARCHAR` vs `VARCHAR`?**
   `NVARCHAR` stores **Unicode** (2 bytes/char), `VARCHAR` stores non-Unicode (1 byte/char). Use `NVARCHAR` for any text that might contain international characters (which maps cleanly to C#'s `string`).

3. **How do you connect a .NET app to SQL Server?**
   A **connection string** in `appsettings.json`, the **`Microsoft.Data.SqlClient`** driver (directly via ADO.NET, or under EF Core via `UseSqlServer(...)`). Mention SQL vs Windows authentication.

4. **How do you prevent SQL injection?**
   **Parameterized queries** — `SqlParameter` / `@params` in ADO.NET; EF Core/LINQ parameterizes automatically. Never concatenate user input into SQL. The value travels separately from the SQL text so it can never be executed as code.

5. **`SCOPE_IDENTITY()` vs `@@IDENTITY`?**
   Both return the last generated identity, but `@@IDENTITY` also picks up IDs from **triggers** (wrong table!). Always use `SCOPE_IDENTITY()`.

6. **Clustered vs non-clustered index?**
   Clustered physically orders the rows and *is* the table (one per table, usually the PK). Non-clustered is a separate lookup structure with pointers (many allowed). `INCLUDE` columns create covering indexes.

7. **What is connection pooling and why does it matter?**
   Physical connections are expensive; ADO.NET reuses them from a pool keyed by connection string. `Dispose()` returns the connection to the pool. Forgetting to dispose leaks connections and exhausts the pool.

8. **Stored procedure vs a LINQ/EF query — when to use which?**
   Procs: precompiled plan, DB-side logic, DBA-tunable, but live outside your app/source control and couple you to SQL Server. EF/LINQ: testable, in-app, provider-portable. Modern .NET leans EF; legacy leans procs.

9. **What isolation level does SQL Server use by default, and what's RCSI?**
   Default is `READ COMMITTED` (lock-based). **Read Committed Snapshot Isolation** switches reads to row-versioning (MVCC-style) so readers don't block writers — a common fix for blocking.

10. **How does EF Core handle optimistic concurrency in SQL Server?**
    A `ROWVERSION` column auto-changes on each update; EF includes it in the `WHERE` of `UPDATE` and throws `DbUpdateConcurrencyException` if 0 rows matched (someone else changed it first). The analog of JPA's `@Version`.

---

## Quick Reference Cheat Sheet

```
DIALECT TRANSLATION (MySQL/PG → T-SQL):
  LIMIT 10                 → TOP 10  /  OFFSET..FETCH
  AUTO_INCREMENT/SERIAL    → IDENTITY(1,1)
  NOW()                    → GETDATE() / SYSDATETIME()
  IFNULL/COALESCE          → ISNULL / COALESCE
  LENGTH()                 → LEN()
  `backticks`              → [brackets]
  BOOLEAN                  → BIT (0/1)
  VARCHAR (unicode)        → NVARCHAR
  TEXT                     → NVARCHAR(MAX)
  last insert id           → SCOPE_IDENTITY()   (NOT @@IDENTITY)

CONNECT FROM .NET:
  Driver:   Microsoft.Data.SqlClient   (NOT System.Data.SqlClient)
  EF Core:  options.UseSqlServer(connString)
  ConnStr:  "Server=localhost;Database=Db;User Id=sa;Password=...;TrustServerCertificate=True;"
  Auth:     SQL auth (user/pwd)  |  Integrated Security=true (Windows/AD)

ADO.NET CORE OBJECTS (= JDBC):
  SqlConnection   = Connection      (using => auto-close, returns to pool)
  SqlCommand      = PreparedStatement
  SqlParameter    = ? bind param    (@name)
  SqlDataReader   = ResultSet       (forward-only)
  Execute:  ExecuteReader (rows) | ExecuteNonQuery (count) | ExecuteScalar (1 value)

DATA TYPES (C# → SQL):
  int/long→INT/BIGINT   bool→BIT   decimal→DECIMAL(19,4)   double→FLOAT
  string→NVARCHAR   DateTime→DATETIME2   Guid→UNIQUEIDENTIFIER   byte[]→VARBINARY

MIGRATIONS (EF Core):
  dotnet ef migrations add <Name>
  dotnet ef database update
  dotnet ef migrations remove

INDEXES:
  Clustered = physically orders rows, IS the table, 1 per table (usually PK)
  Non-clustered = separate lookup structure, many allowed; INCLUDE = covering
  Index Seek (good)  vs  Table/Index Scan (often missing index / non-SARGable)

DOCKER:
  docker run -e ACCEPT_EULA=Y -e MSSQL_SA_PASSWORD=... -p 1433:1433 \
    mcr.microsoft.com/mssql/server:2022-latest   (port 1433)

GOLDEN RULES:
  1. Reuse your SQL notes — only the dialect/tooling/driver is new
  2. NVARCHAR for text, DECIMAL for money, DATETIME2 for time
  3. ALWAYS parameterize — SqlParameter / EF LINQ (never concat)
  4. Use Microsoft.Data.SqlClient + UseSqlServer(); dispose connections (using)
  5. EF Core is provider-swappable — UseSqlServer is the only MSSQL-specific line
```

---

*Pairs with the [Entity Framework Core Guide](Entity_Framework_Core_Guide.md) (the ORM on top of this) and reuses your Java SQL notes for the language-agnostic fundamentals. Part of the [.NET Roadmap](00_START_HERE_DotNet_Roadmap.md).*

*Last Updated: 2026-07-14*
