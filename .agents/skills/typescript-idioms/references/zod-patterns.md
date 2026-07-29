---
name: Zod Patterns and Best Practices
description: Common patterns for Zod runtime validation in TypeScript applications.
---

# Zod Patterns and Best Practices

Common patterns for `zod` runtime validation in TypeScript applications.

## Zod Version Compatibility

> **This file targets Zod 3.x (the default).** Always check the project's `package.json` before assuming v3 behavior.

| Version | Status | Notes |
|---|---|---|
| Zod 3.x | Stable default | All patterns in this file apply |
| Zod 4.x | Breaking changes | Released May 2025 — see migration notes below |

**Zod 4.x breaking changes (check `package.json` before writing schemas):**
- `z.object()` is now **strict by default** (equivalent to old `.strict()`) — use `z.looseObject()` for old passthrough behavior
- `z.string().email()` uses a stricter RFC 5321 validator — test against your actual data set
- `import { z } from 'zod'` still works (no path change for clean v4 installs)
- `z.interface()` is a new alternative to `z.object()` for open types — prefer `z.object()` unless you need it
- Error formatting API changed: `error.formErrors` → prefer `error.flatten()` (stable in both versions)

## Schema-First Design — Single Source of Truth
Define the Zod schema first, then infer the TypeScript type. Never duplicate.
```typescript
const CreateTaskSchema = z.object({
  title: z.string().min(1).max(200),
  priority: z.enum(['low', 'medium', 'high']),
});
type CreateTaskRequest = z.infer<typeof CreateTaskSchema>;
```

## Transform Pipelines
```typescript
const DateStringSchema = z.string().datetime().transform(s => new Date(s));
const PositiveIntSchema = z.string().transform(Number).pipe(z.number().int().positive());
```

## Refinements — Custom Validation
```typescript
const PasswordSchema = z.string()
  .min(8)
  .refine(pw => /[A-Z]/.test(pw), 'Must contain uppercase')
  .refine(pw => /[0-9]/.test(pw), 'Must contain digit');
```

## Discriminated Unions
```typescript
const EventSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('created'), title: z.string() }),
  z.object({ type: z.literal('completed'), completedAt: z.string().datetime() }),
]);
```

## Error Formatting
```typescript
const result = schema.safeParse(data);
if (!result.success) {
  const formatted = result.error.flatten();
  // { formErrors: string[], fieldErrors: { field: string[] } }
}
```

## Coercion Schemas
```typescript
// For form data / query params that arrive as strings
const QuerySchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});
```

## Schema Composition
```typescript
const BaseSchema = z.object({ id: z.string().uuid(), createdAt: z.string().datetime() });
const TaskSchema = BaseSchema.extend({ title: z.string(), status: z.enum(['pending', 'done']) });
const CreateSchema = TaskSchema.omit({ id: true, createdAt: true });
const UpdateSchema = CreateSchema.partial();
```

## Branded Types
```typescript
const UserId = z.string().uuid().brand<'UserId'>();
type UserId = z.infer<typeof UserId>;

const OrderId = z.string().uuid().brand<'OrderId'>();
type OrderId = z.infer<typeof OrderId>;

// Compile error: OrderId not assignable to UserId
```

## Preprocess — Transform Input Before Validation
```typescript
const TrimmedString = z.preprocess(
  (val) => typeof val === 'string' ? val.trim() : val,
  z.string().min(1)
);
```

## API Boundary Pattern
```typescript
// ❌ Raw .parse() — throws ZodError, loses control of error response shape
const body = CreateTaskSchema.parse(req.body);

// ✅ .safeParse() — full control, fully typed after narrowing
const result = CreateTaskSchema.safeParse(req.body);
if (!result.success) {
  // result.error is ZodError here — fully typed, structured error available
  return res.status(400).json({
    errors: result.error.flatten().fieldErrors,
    // { fieldErrors: { title: ['Required'], priority: ['Invalid enum value'] } }
  });
}
// result.data is CreateTaskRequest here — fully typed, no cast needed
const task = await taskService.create(result.data);
```

> **Rule:** Use `.parse()` only at startup for config/env (fail-fast is correct there).
> Use `.safeParse()` at all API boundaries where you need a custom error response.

## Environment Variable Validation
```typescript
const EnvSchema = z.object({
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});
export const env = EnvSchema.parse(process.env);
```

## Anti-Patterns
- ❌ Using `z.any()` or `z.unknown()` without subsequent refinement
- ❌ Duplicating TypeScript interfaces and Zod schemas — always infer from schema
- ❌ Using `as` type assertions instead of `.parse()` at boundaries
- ❌ Skipping `.safeParse()` in API handlers — raw `.parse()` throws, use safe parse for custom error responses
- ❌ Applying `z.coerce` on trusted internal data — coercion is for external boundaries only
- ❌ Using `z.object().passthrough()` without explicit reason — prefer strict schemas

### Related
- TypeScript Idioms @.agents/skills/typescript-idioms/SKILL.md
- Data Serialization Principles @.agents/rules/data-serialization-and-interchange-principles.md
- API Design Principles @.agents/rules/api-design-principles.md
