# Database Migrations: Flyway & Liquibase in Spring Boot

> **How to use this guide (junior dev):** The single most-asked interview question in this space is **"Why not just use `ddl-auto=update` in production?"** — start with [Section 1](#1-the-problem-ddl-auto-in-production) and be able to answer it cold. After that, interviewers expect you to know how Flyway file naming works and why you must never edit an applied migration.

---

## Table of Contents

1. [The Problem: ddl-auto in Production](#1-the-problem-ddl-auto-in-production)
2. [What Schema Migrations Are](#2-what-schema-migrations-are)
3. [Flyway: The Common Default](#3-flyway-the-common-default)
   - [3.1 Setup](#31-setup)
   - [3.2 File Naming & Location](#32-file-naming--location)
   - [3.3 The Schema History Table](#33-the-schema-history-table)
   - [3.4 Versioned vs Repeatable Migrations](#34-versioned-vs-repeatable-migrations)
4. [Liquibase: The Alternative](#4-liquibase-the-alternative)
5. [Flyway vs Liquibase Comparison](#5-flyway-vs-liquibase-comparison)
6. [How It Fits with JPA/Hibernate](#6-how-it-fits-with-jpahibernate)
7. [Best Practices](#7-best-practices)
8. [Common Mistakes & Pitfalls](#8-common-mistakes--pitfalls)
9. [Common Interview Questions](#9-common-interview-questions)
10. [Quick Reference Cheat Sheet](#10-quick-reference-cheat-sheet)

---

## 1. The Problem: ddl-auto in Production

Spring Boot lets you set `spring.jpa.hibernate.ddl-auto` to control how Hibernate manages the schema. The tempting options are:

| Value | What Hibernate Does |
|-------|---------------------|
| `create` | Drops and recreates all tables on startup — **data loss every restart** |
| `create-drop` | Same as `create`, drops tables on shutdown — only for in-memory test DBs |
| `update` | Adds missing columns/tables, but **never drops or renames** anything |
| `validate` | Checks that entities match the DB schema; crashes fast if they don't |
| `none` | Does nothing to the schema |

**Why `update` is dangerous in production:**

1. **Data loss risk** — you cannot undo a column drop or a bad rename; `update` will silently add the new column but the old data is gone if you renamed it in your entity.
2. **No version history** — no record of what changed, when, or who approved it.
3. **No code review** — schema changes happen implicitly when you deploy, not when someone reviews a SQL file.
4. **Environment drift** — dev, staging, and prod can silently diverge over time.

**Production-correct pattern:**

```yaml
# application.yml
spring:
  jpa:
    hibernate:
      ddl-auto: validate   # Hibernate checks entities match DB — crashes early if they don't
  flyway:
    enabled: true          # Flyway owns the schema; Hibernate just validates it
```

---

## 2. What Schema Migrations Are

A schema migration is a **versioned, ordered SQL script** that is:
- Checked into source control alongside application code
- Reviewed and approved like any other code change
- Run **automatically on app startup** in the correct order
- Tracked in a **schema history table** so each migration runs exactly once

Think of it like version control for your database structure: just as Git tracks code history, a migration tool tracks schema history. Every environment (dev, staging, prod) applies the same scripts in the same order, guaranteeing they are identical.

---

## 3. Flyway: The Common Default

### 3.1 Setup

Add the dependency — that is all Spring Boot needs to auto-configure Flyway:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <!-- version managed by Spring Boot BOM -->
</dependency>

<!-- For MySQL, also add: -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
</dependency>
```

Spring Boot auto-detects Flyway on the classpath and runs all pending migrations before the application context finishes loading.

### 3.2 File Naming & Location

Default location: `src/main/resources/db/migration/`

**Versioned migration naming:** `V{version}__{description}.sql`

```
V1__create_users_table.sql
V2__add_email_column.sql
V3__create_orders_table.sql
V4__add_index_on_users_email.sql
```

Rules:
- `V` prefix (uppercase)
- Version number (integers, or `1.1`, `1.2` for sub-versions)
- **Double underscore** `__` separator (not single)
- Description in snake_case (becomes the `description` in history)
- `.sql` extension

**Example migrations:**

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    username   VARCHAR(50)  NOT NULL UNIQUE,
    created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

```sql
-- V2__add_email_column.sql
ALTER TABLE users
    ADD COLUMN email VARCHAR(255) NOT NULL;

-- Add a unique index while we're here
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

### 3.3 The Schema History Table

On first run Flyway creates a `flyway_schema_history` table. Each applied migration gets one row:

| installed_rank | version | description | type | script | checksum | success |
|---|---|---|---|---|---|---|
| 1 | 1 | create users table | SQL | V1__create_users_table.sql | -12345678 | true |
| 2 | 2 | add email column | SQL | V2__add_email_column.sql | 98765432 | true |

Key point: Flyway stores a **checksum** of each applied script. If you edit a script that has already been applied, Flyway will detect the checksum mismatch on the next startup and **refuse to start the application**. This is intentional — it prevents silent divergence.

### 3.4 Versioned vs Repeatable Migrations

| Type | Prefix | When It Runs | Use Case |
|------|--------|--------------|----------|
| Versioned | `V1__`, `V2__` | Once, in version order | DDL changes, data migrations |
| Repeatable | `R__` | Every time its checksum changes | Views, stored procedures, seed data |

```sql
-- R__seed_reference_data.sql  (re-runs whenever the file changes)
TRUNCATE TABLE countries;
INSERT INTO countries(code, name) VALUES ('US', 'United States'), ('GB', 'United Kingdom');
```

---

## 4. Liquibase: The Alternative

Liquibase uses a **changelog** file that contains **changesets** — discrete units of change, each with an `id` and `author`.

```xml
<!-- db/changelog/db.changelog-master.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog" ...>

    <changeSet id="1" author="amith">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="username" type="VARCHAR(50)">
                <constraints nullable="false" unique="true"/>
            </column>
        </createTable>
    </changeSet>

    <changeSet id="2" author="amith">
        <addColumn tableName="users">
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
        </addColumn>
    </changeSet>

</databaseChangeLog>
```

Changelogs can be written in XML, YAML, JSON, or plain SQL. Liquibase tracks applied changesets in its own `databasechangelog` table and supports **rollback** with `<rollback>` blocks — a key advantage over Flyway.

---

## 5. Flyway vs Liquibase Comparison

| Feature | Flyway | Liquibase |
|---------|--------|-----------|
| **Format** | Plain SQL (primary) | XML / YAML / JSON / SQL |
| **Learning curve** | Low — just write SQL | Medium — learn changelog syntax |
| **DB-agnostic abstractions** | No — SQL is DB-specific | Yes — `<createTable>` works on any DB |
| **Rollback support** | No built-in rollback | Yes — `<rollback>` blocks |
| **History table** | `flyway_schema_history` | `databasechangelog` |
| **Spring Boot auto-config** | Yes | Yes |
| **Best for** | Simple apps, SQL-first teams | Multi-DB support, formal rollback needed |
| **Industry prevalence** | Very common default | Common in enterprise / multi-DB |

**Rule of thumb:** Start with Flyway. Move to Liquibase if you need rollback support or must support multiple database vendors.

---

## 6. How It Fits with JPA/Hibernate

The correct division of responsibility:

```
Flyway/Liquibase  →  owns and evolves the schema (creates tables, adds columns)
Hibernate         →  maps Java entities to the existing schema (reads and writes data)
```

Set `ddl-auto=validate` so Hibernate verifies that your `@Entity` classes match the migrated schema on startup. If a column is missing or a type does not match, the app fails fast with a clear error — better than a runtime `SQLException` during a user request.

```yaml
# application-prod.yml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
    locations: classpath:db/migration
```

```yaml
# application-test.yml  (integration tests with H2)
spring:
  jpa:
    hibernate:
      ddl-auto: none        # Flyway still creates the schema
  flyway:
    enabled: true
    locations: classpath:db/migration
```

---

## 7. Best Practices

1. **Never edit an applied migration.** Flyway's checksum check will fail. Create a new migration instead.
2. **One logical change per migration.** `V5__add_phone_column.sql` not `V5__various_changes.sql` — easier to review and revert.
3. **Keep migrations in version control.** They are code. They go through pull requests.
4. **Test migrations against a real DB.** Use Testcontainers in CI to run migrations against a real PostgreSQL/MySQL container.
5. **Baseline an existing database.** If you add Flyway to an app with an existing schema, use `flyway baseline` to mark the current state as version 1 so Flyway does not try to re-run old scripts.
6. **Use `validate` in production, `none` or `validate` in tests.** Never `create` or `update` outside local dev.

---

## 8. Common Mistakes & Pitfalls

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Editing an applied migration | Flyway checksum mismatch; app won't start | Create a new migration `V{n+1}__fix_xyz.sql` |
| Using `ddl-auto=update` in prod | Silent schema drift, irreversible data loss | Switch to `ddl-auto=validate` + Flyway |
| Out-of-order version numbers | Flyway ignores them (or fails with `outOfOrder=false`) | Coordinate with teammates; use timestamps as versions if needed |
| Putting raw data inserts in `V__` migrations | Not idempotent; hard to update later | Use `R__` repeatable migrations for reference/seed data |
| Forgetting double underscore | Flyway cannot parse the filename; migration silently skipped | Always use `V1__` not `V1_` |
| Running `flyway clean` in production | Drops every table — complete data loss | Disable `flyway.cleanDisabled=true` in prod config |

---

## 9. Common Interview Questions

**Q: Why should you not use `ddl-auto=update` in production?**
`update` lets Hibernate modify the schema at runtime without any audit trail, code review, or rollback path. It can silently drop data when you rename a field, and it causes environments to drift. Production databases need versioned, reviewable SQL scripts run by a migration tool.

**Q: What happens if you edit a Flyway migration that has already been applied?**
Flyway stores a checksum of each applied script in `flyway_schema_history`. On the next startup it recalculates the checksum, detects the mismatch, and throws `FlywayException`, preventing the application from starting. The fix is always to create a new migration, never to edit the old one.

**Q: What is the difference between a versioned and a repeatable Flyway migration?**
Versioned migrations (`V1__`) run exactly once in order and are never re-run. Repeatable migrations (`R__`) run on every startup where their checksum has changed — useful for views, stored procedures, or seed data that needs to stay in sync with the file.

**Q: When would you choose Liquibase over Flyway?**
Liquibase is preferred when you need built-in rollback support, must support multiple database vendors from one codebase (its XML/YAML abstractions are DB-agnostic), or work in an enterprise environment that requires formal change management with rollback scripts.

**Q: How do Flyway and JPA/Hibernate work together?**
Flyway owns the schema — it creates and alters tables. Hibernate is set to `ddl-auto=validate` so it only checks that the entity mappings match what Flyway created. This gives you migration-controlled schema evolution with early startup validation.

---

## 10. Quick Reference Cheat Sheet

```
FLYWAY FILE NAMING
  V{version}__{description}.sql   — versioned, runs once
  R__{description}.sql            — repeatable, runs when file changes
  Location: src/main/resources/db/migration/

FLYWAY HISTORY TABLE: flyway_schema_history
  Columns: version, description, script, checksum, success

DDL-AUTO VALUES (HIBERNATE)
  create       — drops + recreates on startup (dev/test only)
  create-drop  — drops on shutdown (in-memory test only)
  update       — adds missing schema, NEVER in production
  validate     — checks entities match DB, PRODUCTION DEFAULT
  none         — does nothing

PRODUCTION PATTERN
  spring.jpa.hibernate.ddl-auto=validate
  spring.flyway.enabled=true

FLYWAY COMMANDS (CLI / Maven plugin)
  flyway migrate    — apply pending migrations
  flyway info       — show migration status
  flyway validate   — verify checksums
  flyway baseline   — mark existing DB as baseline
  flyway repair     — fix checksum mismatches in dev

GOLDEN RULES
  1. Never edit an applied migration
  2. One change per migration file
  3. Migrations live in version control
  4. validate in prod, never update
```

---

*Last Updated: 2026-06-18*
