---

# Serde Patterns and Best Practices

Common patterns for `serde` serialization/deserialization in Rust applications.

---

## Container Attributes

### `rename_all` — Consistent Field Naming

Apply at the struct/enum level for consistent casing across all fields:

```rust
// ✅ API responses use camelCase (JavaScript convention)
#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct TaskResponse {
    pub task_id: Uuid,           // Serializes as "taskId"
    pub created_at: DateTime<Utc>, // Serializes as "createdAt"
    pub is_completed: bool,      // Serializes as "isCompleted"
}

// ✅ Config files use kebab-case (TOML/YAML convention)
#[derive(Deserialize)]
#[serde(rename_all = "kebab-case")]
pub struct AppConfig {
    pub database_url: String,     // Reads "database-url"
    pub max_connections: u32,     // Reads "max-connections"
}
```

### `deny_unknown_fields` — Strict Input Validation

```rust
// ✅ Use for internal config / CLI input — catches typos
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
pub struct DatabaseConfig {
    pub host: String,
    pub port: u16,
    pub name: String,
}

// ❌ Do NOT use for external API responses — APIs add fields over time
// #[serde(deny_unknown_fields)]  // Will break when API adds new fields
pub struct ExternalApiResponse { ... }
```

> **Rule:** Use `deny_unknown_fields` for inputs you control (config files, internal messages). Never for external API responses.

---

## Field Attributes

### `skip_serializing_if` — Omit Optional Fields

```rust
#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
pub struct TaskResponse {
    pub id: Uuid,
    pub title: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub description: Option<String>,  // Omitted from JSON when None
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub tags: Vec<String>,            // Omitted when empty
}
```

### `default` — Backward-Compatible Deserialization

```rust
#[derive(Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CreateTaskRequest {
    pub title: String,
    #[serde(default)]                    // Defaults to 0
    pub priority: u8,
    #[serde(default = "default_status")]  // Custom default
    pub status: TaskStatus,
}

fn default_status() -> TaskStatus {
    TaskStatus::Pending
}
```

### `with` — Custom Serialization

```rust
use chrono::{DateTime, Utc};

#[derive(Serialize, Deserialize)]
pub struct Event {
    pub name: String,
    #[serde(with = "chrono::serde::ts_seconds")]
    pub timestamp: DateTime<Utc>,  // Serialized as Unix timestamp
}
```

---

## Enum Representations

Serde supports multiple enum serialization strategies. Choose based on API requirements:

```rust
// Default: externally tagged (most common for Rust APIs)
// JSON: {"Completed": {"at": "2026-01-01"}}
#[derive(Serialize, Deserialize)]
pub enum TaskStatus {
    Pending,
    InProgress,
    Completed { at: DateTime<Utc> },
}

// Internally tagged: cleaner for APIs
// JSON: {"type": "completed", "at": "2026-01-01"}
#[derive(Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "camelCase")]
pub enum TaskEvent {
    Created { title: String },
    Completed { at: DateTime<Utc> },
}

// Untagged: when the variant is inferred from shape
// JSON: {"title": "my task"} or {"at": "2026-01-01"}
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
pub enum Input {
    Create(CreateRequest),
    Update(UpdateRequest),
}
```

> **Default recommendation:** Use `#[serde(tag = "type", rename_all = "camelCase")]` (internally tagged) for API enums — it produces the cleanest JSON and is easiest for clients to parse.

---

## Request / Response Type Pattern

The full request/response type pattern (struct definitions, `#[validate]` attributes, `From<Domain> for Response` conversion, and `.into()` in handlers) is documented in `axum-idioms/SKILL.md` §Response Types and Domain Conversion.

**Key serde attributes for the pattern:**
- Request structs: `#[serde(rename_all = "camelCase")]` + `#[serde(default)]` on optional fields
- Response structs: `#[serde(rename_all = "camelCase")]` + `#[serde(skip_serializing_if = "Option::is_none")]`
- Never expose domain models directly to the API layer

> See `@.agents/skills/axum-idioms/SKILL.md` §Response Types and Domain Conversion for the complete, authoritative pattern.

---

## Anti-Patterns

- ❌ **Using `serde_json::Value` for typed data** — always deserialize into typed structs for compile-time safety
- ❌ **Applying `deny_unknown_fields` on external API responses** — breaks when API evolves
- ❌ **Mixing `flatten` with `deny_unknown_fields`** — produces unpredictable behavior
- ❌ **Renaming fields individually** when `rename_all` covers the entire struct
- ❌ **Using `String` for structured data** — parse into domain types at the boundary
- ❌ **Skipping `#[serde(default)]` on optional fields** in requests — forces clients to send every field

---

### Related
- Rust Idioms @.agents/skills/rust-idioms/SKILL.md
- Axum Idioms @.agents/skills/axum-idioms/SKILL.md
- Data Serialization Principles @.agents/rules/data-serialization-and-interchange-principles.md
- API Design Principles @.agents/rules/api-design-principles.md
