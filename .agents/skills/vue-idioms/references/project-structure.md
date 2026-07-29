---
name: Vue Frontend Project Structure
description: Directory layout for Vue 3 frontend apps with Vite, Pinia, and vertical feature slices.
---

## Vue/React Frontend Layout

Use this structure for web frontend applications. The vertical slice principle applies — features are self-contained modules, not scattered across global folders.

```
  apps/
    frontend/                         # Frontend application source code
      src/
        assets/                       # Fonts, Images
        features/                     # Business features organized as vertical slices. Each feature is SELF-CONTAINED.
          task/                       # Task management
            components/               # Task Feature-specific components go HERE, DON'T Put feature components in top-level folders
              TaskForm.vue
              TaskListItem.vue
              TaskFilters.vue
              TaskInput.vue
              TaskInput.spec.ts       # Component unit tests
            store/
              task.store.ts           # Pinia store
              task.store.spec.ts      # Store unit tests
            api/
              task.api.ts             # interface TaskAPI
              task.api.backend.ts     # Production implementation
              task.api.mock.ts        # Test implementation
            services/
              task.service.ts         # Business logic
              task.service.spec.ts    # Logic unit tests
            types/                    # TS Interfaces for tasks (e.g. CreateTaskDTO interfaces)
            composables/              # Task Feature-specific hooks (e.g. useTaskFilters.ts)
            index.ts                  # Public exports. Export ONLY what's needed by `views/`
          order/
        composables/                  # Global reactive logic (useAuth, useTheme)
        components/                   # Shared Component (Buttons, Inputs) - Dumb UI, No Domain Logic. DON'T Put feature components HERE
          ui/                         # UI Components (Atoms & Molecules) Pure, reusable UI primitives. NO domain logic, NO feature knowledge.
            BaseButton.vue
            BaseButton.spec.ts        # Unit tests for button states
            types.ts                  # Shared UI types/interfaces
            index.ts                  # Barrel export for easy imports
          layout/                     # Layout Components (Organisms) Composite UI structures that combine multiple UI components. Still reusable, but more complex.
            AppHeader.vue             # Application header with nav, logo, user menu
            AppSidebar.vue            # Sidebar navigation structure
            ErrorBoundary.vue         # Error display wrapper
            EmptyState.vue            # Empty list placeholder
        layouts/                      # App shells (Sidebar, Navbar wrappers)
          MainLayout.vue              # Contains Navbar, Sidebar, Footer
          AuthLayout.vue              # Minimal layout for Login/Register
        views/                        # Route entry points (The "Glue")
          HomeView.vue                # Imports from features/analytics
          TaskView.vue                # Imports from features/task
        utils/                        # Pure, stateless helper functions. No domain knowledge, no Vue reactivity, (e.g. date-fns wrappers, math).
        router/                       # Route definitions
        plugins/                      # Library configs (Axios, I18n)
        App.vue                       # Root component (hosts <router-view>)
        main.ts                       # Entry point (bootstraps plugins & mounts app)
      ...
```

**Key frontend conventions:**
- `features/` for vertical slices — each feature exports only what `views/` needs via `index.ts`
- `components/ui/` for **shared** dumb UI (buttons, inputs) — NO domain logic, NO feature knowledge
- `components/layout/` for composite UI structures (headers, sidebars)
- `views/` are route entry points — they compose features, not implement them
- Feature components live **inside** the feature, NOT in top-level `components/`
- `composables/` at root for global reactive logic; feature-specific hooks inside the feature

> This structure applies equally to React (.tsx), Vue (.vue), and Svelte (.svelte). Replace component file extensions and state management (Redux/Zustand for React, Pinia for Vue) as needed.

---

### Test Organization

> For test naming conventions and pyramid ratios, see `@.agents/rules/testing-strategy.md`. This section covers file placement only.

**Unit Tests — Co-located (`.spec.ts` next to source)**
```
features/task/
  components/
    TaskForm.vue
    TaskForm.spec.ts           # ← co-located component test
  store/
    task.store.ts
    task.store.spec.ts         # ← co-located store test
  services/
    task.service.ts
    task.service.spec.ts       # ← co-located logic test
  api/
    task.api.ts                # interface (no test needed)
    task.api.mock.ts           # test double
```

**Integration Tests — Separate `tests/` directory**
```
tests/
  helpers/
    test-server.ts             # Shared setup/teardown
    factories.ts               # Test data factories
  integration/
    task-api.integration.spec.ts
```

**E2E Tests — Isolated**
```
tests/
  e2e/
    task-flow.e2e.spec.ts      # Full browser tests (Playwright)
```

> For monorepos: E2E tests go in `apps/e2e/`. For single apps: `tests/e2e/`.

---

### Related Principles
- Project Structure @.agents/rules/project-structure.md (core philosophy)
- Vue Idioms and Patterns @../SKILL.md (Composition API, Pinia, composables)
- TypeScript Idioms and Patterns @.agents/skills/typescript-idioms/SKILL.md (type system, async patterns)
