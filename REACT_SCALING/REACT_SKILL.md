# SKILL.md
## React Architecture at Scale
### Codex and Claude Skill Specification

## Purpose
This skill defines a production-grade architectural contract for React and Next.js applications that must remain maintainable under sustained domain growth, team scaling, feature proliferation, and increasing UI and state complexity.

This is not a beginner React guideline.

This is a structural operating model for keeping frontend systems evolvable after the point where naive co-location, route-heavy orchestration, and uncontrolled hook growth begin to degrade delivery speed.

The central thesis is simple:

React usually does not break at project start.

It breaks when the application grows, the domain surface expands, feature boundaries are absent, and the codebase keeps pretending it is still small.

The failure mode is not React itself.

The failure mode is boundary collapse.

---

## Core Problem Statement
In early-stage React codebases, the following decisions often appear harmless:

- API calls inside route components
- hooks stacked directly in `page.tsx`
- “temporary” global state for fast delivery
- components that fetch, mutate, render, validate, and navigate
- utility folders shared by everything without domain ownership

This works while the domain is small.

It degrades when:

- pages become orchestration layers and rendering layers simultaneously
- local UI concerns leak into global stores
- query ownership is unclear
- mutations trigger visual side effects from the wrong layer
- features consume each other’s internal files directly
- refactors require knowledge of unrelated modules
- teams cannot tell where behavior belongs

At scale, the front end needs explicit boundaries for:

- route entry
- behavior orchestration
- rendering
- external integration
- domain modeling
- shared infrastructure
- UI state locality
- server state ownership

---

## Architectural Position
A scalable React codebase should be organized by **responsibility first** and **domain second**, not by generic technical bucket alone.

The architecture should enforce the following separation:

- `page`: route entry only
- `capability`: feature orchestration and behavioral composition
- `components/view`: rendering only
- `service`: external integration only
- `queries/mutations`: server state adapters
- `store`: feature-scoped UI state
- `model`: domain contracts, schema, mapping, typing
- `core`: application foundation and infrastructure
- `shared`: reusable and domain-agnostic assets
- `features`: domain-centered business modules

This keeps the routing layer thin, the behavior centralized, the rendering deterministic, and the domain surface explicit.

---

## Non-Negotiable Principles

### 1. Route files must stay thin
A route file must not become a controller.

Its job is to resolve routing and delegate to a feature capability.

A route file should not accumulate:

- API access
- mutation flow
- query composition
- modal orchestration
- view state rules
- domain branching
- data transformation
- imperative business handlers

#### Correct pattern
```tsx
// app/(dashboard)/users/page.tsx
import { UsersPageCapability } from "@/features/users";

export default function Page() {
  return <UsersPageCapability />;
}
```

#### Why
This preserves route stability even when the feature grows. The route becomes a stable shell instead of a behavioral hotspot.

---

### 2. Capability is the orchestration boundary
A capability is the feature-level composition layer where behavior lives.

A capability coordinates:

- server reads
- server writes
- feature UI state
- command handlers
- state-to-props adaptation
- decision flow
- view composition

A capability should know the moving parts of the feature.

The view should not.

#### Example
```tsx
// features/users/capabilities/users-page.capability.tsx
"use client";

import { UsersPageView } from "../components/users-page.view";
import { useUsersQuery } from "../queries/use-users-query";
import { useCreateUserMutation } from "../queries/use-create-user-mutation";
import { useUsersUIStore } from "../store/users-ui.store";

export function UsersPageCapability() {
  const usersQuery = useUsersQuery();
  const createUserMutation = useCreateUserMutation();
  const ui = useUsersUIStore();

  async function handleCreateUser(input: { name: string; email: string }) {
    await createUserMutation.mutateAsync(input);
    ui.closeCreateModal();
  }

  return (
    <UsersPageView
      users={usersQuery.data ?? []}
      isLoading={usersQuery.isLoading}
      isCreateModalOpen={ui.isCreateModalOpen}
      onOpenCreateModal={ui.openCreateModal}
      onCloseCreateModal={ui.closeCreateModal}
      onCreateUser={handleCreateUser}
    />
  );
}
```

#### Why
This stops the main app from filling with dozens of hooks, handlers, and imperative side effects inside route files.

It also creates a stable point for tests, feature composition, and incremental extension.

---

### 3. View must be dumb in the right way
A view is not “dumb” because it is simplistic.

It is “dumb” because it does not own backend behavior, cache invalidation, integration concerns, or workflow orchestration.

A view should:

- receive prepared props
- render state
- emit events upward
- remain predictable under snapshot and interaction testing

A view should not:

- call service methods directly
- know query client details
- invalidate cache
- map transport payloads
- control routing policies
- open toasts from domain rules unless explicitly injected

#### Example
```tsx
// features/users/components/users-page.view.tsx
type UsersPageViewProps = {
  users: Array<{ id: string; name: string; email: string }>;
  isLoading: boolean;
  isCreateModalOpen: boolean;
  onOpenCreateModal: () => void;
  onCloseCreateModal: () => void;
  onCreateUser: (input: { name: string; email: string }) => Promise<void>;
};

export function UsersPageView({
  users,
  isLoading,
  onOpenCreateModal,
}: UsersPageViewProps) {
  return (
    <section>
      <h1>Usuários</h1>

      <button onClick={onOpenCreateModal}>
        Novo usuário
      </button>

      {isLoading ? (
        <p>Carregando...</p>
      ) : (
        <ul>
          {users.map((user) => (
            <li key={user.id}>
              {user.name} • {user.email}
            </li>
          ))}
        </ul>
      )}
    </section>
  );
}
```

#### Why
Deterministic views are easier to test, reuse, refactor, and reason about. They also reduce accidental coupling to infrastructure.

---

### 4. Service is an integration adapter, not a workflow owner
A service exists to communicate with external systems.

A service should:

- perform HTTP calls
- serialize requests
- return transport payloads or mapped payloads
- encapsulate endpoint access

A service should not:

- open modals
- navigate
- trigger toasts
- interpret visual flow
- coordinate cache invalidation
- know about UI components

#### Example
```ts
// features/users/services/users.service.ts
import { apiClient } from "@/core/http/api-client";

export const usersService = {
  async list() {
    const { data } = await apiClient.get("/users");
    return data;
  },

  async create(input: { name: string; email: string }) {
    const { data } = await apiClient.post("/users", input);
    return data;
  },
};
```

#### Why
This enforces single responsibility. The service transports data between frontend and backend. Nothing else.

---

### 5. Queries and mutations must live outside the view
Reading and writing remote data is not the responsibility of the render layer.

Queries and mutations should sit near the feature, not inside generic folders disconnected from domain meaning.

#### Query example
```ts
// features/users/queries/use-users-query.ts
"use client";

import { useQuery } from "@tanstack/react-query";
import { usersService } from "../services/users.service";

export function useUsersQuery() {
  return useQuery({
    queryKey: ["users"],
    queryFn: usersService.list,
  });
}
```

#### Mutation example
```ts
// features/users/queries/use-create-user-mutation.ts
"use client";

import { useMutation, useQueryClient } from "@tanstack/react-query";
import { usersService } from "../services/users.service";

export function useCreateUserMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: usersService.create,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["users"] });
    },
  });
}
```

#### Why
This isolates server-state contracts and keeps render units agnostic to fetching mechanics and cache policies.

---

### 6. State must be separated by nature, not convenience
A large portion of frontend chaos comes from mixing distinct state categories into a single abstraction.

Use state according to its nature.

#### Local state
Use for ephemeral component concerns:
- input value
- hover
- disclosure local to one component
- local uncontrolled UI details

#### Server state
Use for remote truth:
- API list results
- resource details
- cached backend entities
- optimistic and invalidation flows

This belongs to query tools such as React Query, not to ad hoc global stores.

#### Feature UI state
Use for visual coordination within a feature:
- modal open state
- selected tab in one feature
- temporary filters shared across sibling components
- drawers and overlays belonging to one domain area

#### Example
```ts
// features/users/store/users-ui.store.ts
import { create } from "zustand";

type UsersUIState = {
  isCreateModalOpen: boolean;
  openCreateModal: () => void;
  closeCreateModal: () => void;
};

export const useUsersUIStore = create<UsersUIState>((set) => ({
  isCreateModalOpen: false,
  openCreateModal: () => set({ isCreateModalOpen: true }),
  closeCreateModal: () => set({ isCreateModalOpen: false }),
}));
```

#### Why
This avoids the anti-pattern of one massive app-wide store pretending to own every state category.

---

### 7. Domain boundaries must be explicit
The most dangerous coupling in React codebases is silent coupling.

That happens when one feature starts importing internal implementation files from another feature.

Every feature should expose a public API.

#### Example
```ts
// features/users/index.ts
export { UsersPageCapability } from "./capabilities/users-page.capability";
export { useUsersQuery } from "./queries/use-users-query";
export type { User } from "./model/user.types";
```

#### Rule
Consumers outside the feature should import from:

```ts
@/features/users
```

Not from:

```ts
@/features/users/components/...
@/features/users/services/...
@/features/users/store/...
```

#### Why
A public API preserves the right to refactor internals without breaking consumers.

---

### 8. Organize by domain, not only by technical type
Folders like `components`, `hooks`, `services`, and `utils` at project root seem tidy early and degrade rapidly under scale.

Prefer a layered domain layout where features own their business logic and infrastructure remains outside.

#### Recommended structure
```txt
src/
  app/
    (dashboard)/
      users/
        page.tsx
        loading.tsx
        error.tsx
    layout.tsx
    providers.tsx

  core/
    http/
      api-client.ts
    config/
      env.ts
    providers/
      query-provider.tsx
    auth/
      session.ts

  shared/
    ui/
      button.tsx
      input.tsx
      modal.tsx
    lib/
      format-date.ts
      cn.ts
    hooks/
      use-disclosure.ts
    types/

  features/
    users/
      capabilities/
        users-page.capability.tsx
      components/
        users-page.view.tsx
        users-table.tsx
        user-form.tsx
      queries/
        use-users-query.ts
        use-create-user-mutation.ts
      services/
        users.service.ts
      model/
        user.types.ts
        user.schema.ts
        user.mapper.ts
      store/
        users-ui.store.ts
      index.ts

    auth/
      capabilities/
      components/
      services/
      queries/
      model/
      index.ts
```

#### Structural responsibility
- `app` resolves route and framework entry
- `core` owns cross-app infrastructure and foundation
- `shared` contains truly generic reusable artifacts
- `features` contains business modules and domain-specific behavior

---

## Layer Contracts

### `app/`
Framework boundary for route resolution and app composition.

Allowed:
- route definition
- layout definition
- route-level loading and error files
- feature delegation
- app-wide providers registration

Forbidden:
- domain orchestration
- transport logic
- feature-specific query composition
- feature state ownership

---

### `core/`
Foundation of the application.

Owns:
- HTTP client
- environment parsing
- auth session primitives
- framework-level providers
- telemetry and cross-cutting concerns
- shared infrastructure adapters

Forbidden:
- feature business rules
- feature-specific state
- domain-specific components

---

### `shared/`
Reusable artifacts with no feature ownership.

Owns:
- design system primitives
- stateless utility functions
- generic hooks without domain meaning
- shared types with no business ownership

Forbidden:
- domain rules
- feature-specific services
- implicit coupling to one business module

A common abuse of `shared` is dumping semi-random code there because no one knows ownership. Reject that pattern.

If an artifact belongs to one business area, it is not shared.

---

### `features/`
Primary domain boundary.

Owns:
- business UI
- orchestration
- feature-scoped state
- server-state adapters
- transport integration for that domain
- schemas and mapping logic
- internal implementation details

A feature should be internally cohesive and externally exposed through a stable public API.

---

## Capability Pattern Details

### Definition
A capability is a composition module that binds together the operational dependencies of a user-facing workflow.

### Responsibilities
A capability may:
- execute feature hooks
- bind remote and local state
- derive props for views
- create event handlers
- coordinate mutation lifecycle
- translate domain results into render-safe state
- connect child components into a screen-level flow

### A capability should not become:
- a giant service layer
- a giant business rules file with dozens of unrelated handlers
- a place for arbitrary helper dumping

### Extraction rule
When a capability grows, extract by responsibility:
- table behavior
- form behavior
- detail panel behavior
- filter orchestration
- command execution

This should produce smaller orchestrators or sub-capabilities, not route bloat.

---

## Model Layer Rules

The model layer is responsible for contracts, transformation, and invariants at the feature boundary.

Recommended contents:
- `*.types.ts` for explicit structural contracts
- `*.schema.ts` for runtime validation and parsing
- `*.mapper.ts` for translating backend payloads into frontend domain shapes

### Why model exists
Backend payloads are transport contracts, not necessarily UI contracts.

If the frontend consumes raw transport indiscriminately, the UI becomes coupled to backend shape volatility.

A model layer can:
- validate unsafe data
- normalize nullability
- map field names
- isolate transport changes
- define stable domain-facing types

---

## Public API Rule

Every feature must own a deliberate `index.ts`.

### Purpose
- expose only supported contracts
- hide internal file layout
- enforce stable imports
- reduce accidental dependency on implementation details

### Example
```ts
// features/users/index.ts
export { UsersPageCapability } from "./capabilities/users-page.capability";
export { useUsersQuery } from "./queries/use-users-query";
export type { User } from "./model/user.types";
```

### External import example
```ts
import { UsersPageCapability } from "@/features/users";
```

### Internal imports remain relative inside the feature
This helps preserve refactor safety and module locality.

---

## Documentation Standard

A good React architecture README should explain decisions, constraints, and boundaries.

It should not waste space narrating JSX details line by line.

Each feature should have a concise `README.md`.

### Example
```md
# Users Feature

## Responsibility
Gerenciar listagem e criação de usuários.

## Structure
- capabilities: orquestra o fluxo da tela
- components: renderização
- services: integração com API
- queries: leitura e escrita remota
- store: estado visual compartilhado

## Rules
- page.tsx apenas delega
- componentes não chamam API diretamente
- imports externos passam pelo index.ts da feature
```

### Documentation goals
The README should answer:
- what this feature owns
- what it explicitly does not own
- where behavior lives
- how external consumers should import it
- what invariants must not be violated

---

## Anti-Patterns to Reject

### 1. Fat route files
Symptoms:
- many hooks directly in `page.tsx`
- multiple mutation and query handlers
- modal state and transport logic in route file
- route file exceeding responsibility boundaries

Remedy:
- move orchestration into capability
- keep route as delegating shell

---

### 2. Components calling API directly
Symptoms:
- `useEffect` inside view calls service
- button click triggers HTTP call directly
- render component owns transport logic

Remedy:
- move integration to service and query/mutation layers
- inject behavior via props or capability

---

### 3. Over-globalized state
Symptoms:
- one giant Zustand or Redux store
- remote data duplicated into client store
- UI flags from unrelated domains living together
- every feature subscribing to broad slices

Remedy:
- keep server state in server-state tools
- keep feature UI state feature-scoped
- keep ephemeral state local when possible

---

### 4. Shared folder abuse
Symptoms:
- random business helpers in `shared`
- domain-specific hooks presented as generic
- shared becoming unowned dumping ground

Remedy:
- move domain-owned code back into the correct feature
- reserve `shared` for true cross-domain reuse

---

### 5. Cross-feature internal imports
Symptoms:
- feature A imports feature B service file
- feature A imports feature B store internals
- feature boundaries become implementation details

Remedy:
- expose public API through `index.ts`
- forbid external access to internal folders

---

### 6. Hooks as unstructured composition
Symptoms:
- pages with 10 to 20 hooks stacked
- no orchestration boundary
- no single place that describes feature workflow

Remedy:
- introduce capability
- use hooks inside feature orchestration, not directly at every route entry

---

## Advanced Operating Rules

### Rule 1
Every feature screen should have an obvious orchestration owner.

Usually this is a capability.

If a screen has no orchestration owner, complexity will spread into route files and child components.

### Rule 2
All side effects must have a responsible layer.
Examples:
- API request: service or query/mutation
- cache invalidation: mutation adapter
- modal open: feature UI store or local state
- navigation after success: capability or injected command boundary
- form validation: model schema or form boundary

### Rule 3
Never store remote truth redundantly unless there is a deliberate synchronization strategy.
If React Query owns the resource, avoid copying it into Zustand by default.

### Rule 4
Treat a feature as a mini-system.
It should have:
- input contracts
- state strategy
- integration boundary
- rendering boundary
- documentation
- public API

### Rule 5
Optimize for controlled refactorability, not shortest initial code.
The codebase must remain modifiable when the domain surface doubles.

---

## Suggested Conventions for Codex and Claude

When generating or refactoring React or Next.js code under this skill, follow these rules strictly.

### Generation rules
1. Create thin route files.
2. Prefer feature folders under `src/features`.
3. Create capability files for screen orchestration.
4. Keep visual components in `components`.
5. Put HTTP integration in `services`.
6. Put React Query hooks in `queries`.
7. Put schemas, mappers, and types in `model`.
8. Use feature-scoped stores for shared visual state only.
9. Expose public contracts through `index.ts`.
10. Never let components call backend integration directly unless the component itself is the integration boundary by explicit design.

### Refactor rules
1. When `page.tsx` has too many hooks, extract capability first.
2. When a component mixes rendering and fetch logic, split view from data adapter.
3. When backend payload is unstable or verbose, introduce a mapper and domain type.
4. When state is duplicated, classify by nature before consolidating.
5. When a feature is imported internally from many places, add or tighten its public API.

### Review rules
Flag code as structurally risky if:
- routes contain orchestration
- services control interface flow
- shared contains domain logic
- server data is mirrored into stores without a reason
- feature internals are imported externally
- views own queries or mutations
- no clear boundary exists between app, feature, and shared concerns

---

## Decision Matrix

### Where should this logic live

#### Route resolution
- `app/.../page.tsx`

#### Screen orchestration
- `features/<domain>/capabilities`

#### Stateless reusable UI primitives
- `shared/ui`

#### Feature-specific presentational pieces
- `features/<domain>/components`

#### Transport integration
- `features/<domain>/services`

#### Query and mutation hooks
- `features/<domain>/queries`

#### Domain parsing, types, mappers
- `features/<domain>/model`

#### Shared visual state inside one feature
- `features/<domain>/store`

#### App-wide infra
- `core`

---

## Heuristics for Complexity Growth

A frontend has probably crossed the boundary where naive structure fails when one or more of these are true:

- a route contains more than 3 to 5 behavioral hooks
- multiple modals and async flows coexist in one screen
- child components need the same state but no clear owner exists
- one feature depends on another feature’s internal files
- transport payload shapes are leaking into many components
- refactors require changes across unrelated folders
- onboarding requires tribal knowledge to locate behavior

At this point, boundary-oriented architecture is no longer optional.

---

## Example Enforcement Checklist

Before merging a feature, verify:

- Is `page.tsx` delegating instead of orchestrating
- Does a capability own screen behavior
- Do views receive prepared props instead of building backend logic
- Are queries and mutations outside the view
- Is remote state managed as server state rather than duplicated in store
- Is feature UI state scoped to the feature
- Are service files free from UI concerns
- Does the feature expose a stable `index.ts`
- Is documentation present and decision-oriented
- Is `shared` free from business leakage

---

## Minimal Implementation Blueprint

### Route
```tsx
import { UsersPageCapability } from "@/features/users";

export default function Page() {
  return <UsersPageCapability />;
}
```

### Public API
```ts
export { UsersPageCapability } from "./capabilities/users-page.capability";
export { useUsersQuery } from "./queries/use-users-query";
export type { User } from "./model/user.types";
```

### Capability
```tsx
"use client";

import { UsersPageView } from "../components/users-page.view";
import { useUsersQuery } from "../queries/use-users-query";
import { useCreateUserMutation } from "../queries/use-create-user-mutation";
import { useUsersUIStore } from "../store/users-ui.store";

export function UsersPageCapability() {
  const usersQuery = useUsersQuery();
  const createUserMutation = useCreateUserMutation();
  const ui = useUsersUIStore();

  async function handleCreateUser(input: { name: string; email: string }) {
    await createUserMutation.mutateAsync(input);
    ui.closeCreateModal();
  }

  return (
    <UsersPageView
      users={usersQuery.data ?? []}
      isLoading={usersQuery.isLoading}
      isCreateModalOpen={ui.isCreateModalOpen}
      onOpenCreateModal={ui.openCreateModal}
      onCloseCreateModal={ui.closeCreateModal}
      onCreateUser={handleCreateUser}
    />
  );
}
```

### Service
```ts
import { apiClient } from "@/core/http/api-client";

export const usersService = {
  async list() {
    const { data } = await apiClient.get("/users");
    return data;
  },

  async create(input: { name: string; email: string }) {
    const { data } = await apiClient.post("/users", input);
    return data;
  },
};
```

### Store
```ts
import { create } from "zustand";

type UsersUIState = {
  isCreateModalOpen: boolean;
  openCreateModal: () => void;
  closeCreateModal: () => void;
};

export const useUsersUIStore = create<UsersUIState>((set) => ({
  isCreateModalOpen: false,
  openCreateModal: () => set({ isCreateModalOpen: true }),
  closeCreateModal: () => set({ isCreateModalOpen: false }),
}));
```

---

## Final Directive
Scaling React well is not about adding abstraction for its own sake.

It is about introducing the right boundaries before complexity turns invisible coupling into operational drag.

The practical path is:

- thin page
- strong capability
- render-focused UI
- integration-only service
- state classified by nature
- explicit domain boundary
- public API per feature
- documentation focused on architectural decisions

Use this skill whenever the goal is not just to make React work, but to keep it evolvable after the system stops being small.
