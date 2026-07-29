# SQLx Patterns and Database Access

Database access patterns for Rust applications using `sqlx` (async, compile-time checked SQL).

> **Cross-reference:** For general database design principles (schema design, normalization, indexing), see `postgres-idioms` skill. This document covers Rust/SQLx-specific patterns only.

---

## Migration Organization

SQLx migrations live in a `migrations/` directory at the crate root:

```
crates/my-service/
  migrations/
    20260101000000_create_tasks.sql
    20260102000000_add_task_priority.sql
  src/
    ...
```

- **Naming:** `YYYYMMDDHHMMSS_description.sql` — timestamp ensures ordering
- **Run migrations at startup** (application code) or via CLI (`sqlx migrate run`)
- **Never modify** a deployed migration — create a new migration instead
- **Reversible migrations:** Create `.up.sql` and `.down.sql` pairs when rollback is needed

---

## Query Patterns

### Compile-Time Checked Queries (Recommended)

```rust
// ✅ Compile-time verified — SQL syntax and types checked against live DB schema
let task = sqlx::query_as!(Task,
    r#"SELECT id, title, priority, created_at FROM tasks WHERE id = $1"#,
    task_id
)
.fetch_optional(&pool)
.await?;
```

> **Requirement:** `sqlx` compile-time checking requires `DATABASE_URL` set at build time or an `.env` file. Use `cargo sqlx prepare` to generate offline query metadata for CI builds without a live database.

### Type Overrides for Custom Postgres Types

The most common `sqlx` compile-time hurdle: columns with types that don't auto-map (UUIDs, `chrono` datetimes, custom enums) require explicit type annotation in the SQL string using `as "col_name: Type"`.

```rust
// ❌ Compile error — sqlx cannot infer uuid::Uuid or DateTime<Utc> automatically
let task = sqlx::query_as!(Task,
    r#"SELECT id, title, created_at FROM tasks WHERE id = $1"#,
    task_id
)
.fetch_optional(&pool)
.await?;

// ✅ Type overrides fix the inference — note the r#"..."# raw strings
let task = sqlx::query_as!(Task,
    r#"
        SELECT
            id            as "id: uuid::Uuid",
            title,
            priority,
            created_at    as "created_at: chrono::DateTime<chrono::Utc>",
            status        as "status: TaskStatus"    -- custom enum
        FROM tasks
        WHERE id = $1
    "#,
    task_id
)
.fetch_optional(&pool)
.await?;
```

> **Rule:** Add type overrides for: `uuid::Uuid`, `chrono::DateTime<Utc>`, `chrono::NaiveDate`, custom `sqlx::Type` enums, and any `JSONB` column mapped to a typed struct. Standard primitives (`i32`, `i64`, `String`, `bool`) auto-map without overrides.

### Runtime Queries (When Needed)

```rust
// ✅ When dynamic queries are required (conditional filters, dynamic ORDER BY)
let tasks = sqlx::query_as::<_, Task>(
    "SELECT id, title, priority FROM tasks WHERE status = $1 ORDER BY created_at DESC"
)
.bind(status)
.fetch_all(&pool)
.await?;
```

### Choosing Between `fetch_one`, `fetch_optional`, `fetch_all`

| Method | When to Use | Returns |
|---|---|---|
| `fetch_one` | Exactly one row expected (error if 0 or 2+) | `T` |
| `fetch_optional` | Zero or one row expected | `Option<T>` |
| `fetch_all` | Multiple rows expected | `Vec<T>` |
| `fetch` (stream) | Large result sets — process row-by-row | `impl Stream<Item = T>` |

---

## Transaction Patterns

```rust
use sqlx::Acquire;

/// Create a task with tags in a single transaction.
pub async fn create_task_with_tags(
    pool: &PgPool,
    task: &NewTask,
    tags: &[String],
) -> Result<Task, StorageError> {
    let mut tx = pool.begin().await?;

    let task = sqlx::query_as!(Task,
        "INSERT INTO tasks (title, priority) VALUES ($1, $2) RETURNING *",
        task.title, task.priority
    )
    .fetch_one(&mut *tx)
    .await?;

    for tag in tags {
        sqlx::query!(
            "INSERT INTO task_tags (task_id, tag) VALUES ($1, $2)",
            task.id, tag
        )
        .execute(&mut *tx)
        .await?;
    }

    tx.commit().await?;
    Ok(task)
}
```

> **Rule:** Any operation that modifies multiple tables MUST use a transaction. Never rely on separate queries being "close enough."

---

## Connection Pool Configuration

```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(20)          // Match your workload, not your CPU count
    .min_connections(5)           // Keep warm connections ready
    .acquire_timeout(Duration::from_secs(5))  // Fail fast, don't queue forever
    .idle_timeout(Duration::from_secs(600))   // Release idle connections
    .max_lifetime(Duration::from_secs(1800))  // Force reconnect to pick up DNS changes
    .connect(&database_url)
    .await?;
```

### Pool Sizing Rule of Thumb

> **connections = (2 × CPU cores) + number_of_disks** (PostgreSQL recommendation). For cloud databases with connection limits, use PgBouncer or similar connection pooler.

---

## Repository Pattern with SQLx

Follow the trait-based repository pattern from `architectural-pattern.md`:

```rust
// repository.rs — Contract (no sqlx dependency)
pub trait TaskRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Task>, StorageError>;
    async fn create(&self, task: &NewTask) -> Result<Task, StorageError>;
    async fn list_by_user(&self, user_id: Uuid, limit: i64) -> Result<Vec<Task>, StorageError>;
}

// postgres.rs — Production implementation
pub struct PgTaskRepository {
    pool: PgPool,
}

impl PgTaskRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

impl TaskRepository for PgTaskRepository {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Task>, StorageError> {
        sqlx::query_as!(Task, "SELECT * FROM tasks WHERE id = $1", id)
            .fetch_optional(&self.pool)
            .await
            .map_err(StorageError::from)
    }

    // ...
}
```

---

## Testing with SQLx

### `#[sqlx::test]` — Per-Test Database

```rust
#[sqlx::test(migrations = "./migrations")]
async fn test_create_task(pool: PgPool) {
    let repo = PgTaskRepository::new(pool);
    let task = repo.create(&sample_new_task()).await.unwrap();
    assert_eq!(task.title, "Test Task");
}
```

> `#[sqlx::test]` creates a fresh database per test, runs migrations, and drops it after. Requires `DATABASE_URL` pointing to a test PostgreSQL instance.

### Testcontainers (No External DB Required)

```rust
use testcontainers::runners::AsyncRunner;
use testcontainers_modules::postgres::Postgres;

#[tokio::test]
async fn test_with_real_postgres() {
    let container = Postgres::default().start().await.unwrap();
    let pool = PgPoolOptions::new()
        .connect(&container.connection_string())
        .await
        .unwrap();
    sqlx::migrate!("./migrations").run(&pool).await.unwrap();

    let repo = PgTaskRepository::new(pool);
    // ... test against real PostgreSQL
}
```

---

## Offline Query Checking for CI

```bash
# Generate query metadata (requires DATABASE_URL for initial generation)
cargo sqlx prepare --workspace

# Check queries against saved metadata (no DATABASE_URL needed)
cargo sqlx prepare --workspace --check
```

> Add `cargo sqlx prepare --workspace --check` to your CI pipeline to verify queries without a live database.

---

## Anti-Patterns

- ❌ **Using `query()` when `query_as!()` works** — lose compile-time type checking
- ❌ **Skipping transactions for multi-table writes** — data inconsistency risk
- ❌ **Unbounded `fetch_all` without LIMIT** — OOM risk on large tables
- ❌ **Hardcoding connection strings** — use env vars via config module
- ❌ **Setting pool max_connections too high** — exhausts DB connection limits
- ❌ **Modifying deployed migrations** — creates schema drift between environments

---

### Related
- Rust Idioms @.agents/skills/rust-idioms/SKILL.md
- Architectural Patterns @.agents/rules/architectural-pattern.md
- Database Design Principles @.agents/rules/database-design-principles.md
- PostgreSQL Idioms @.agents/skills/postgres-idioms/SKILL.md
- Testability Patterns @.agents/skills/testability-patterns/SKILL.md
