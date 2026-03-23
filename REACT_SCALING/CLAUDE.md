# CLAUDE.md
## React Frontend Architecture Contract
### Boundary-Oriented Scaling for Next.js and React

## Mission
This document defines the architectural contract Claude must follow when generating, reviewing, refactoring, or extending React and Next.js code in this repository.

The goal is not to make React work only at the beginning.

The goal is to keep the frontend evolvable after feature count, state surface, domain complexity, and team interaction all increase.

React usually does not fail at project start.

It fails when the application grows and the codebase keeps behaving as if it were still small.

The core failure mode is not React.

It is missing boundaries.

---

## Primary Architectural Thesis
When a React application scales well, three things are usually clear:

- component responsibility
- state ownership
- domain boundary

This repository must preserve those three concerns explicitly.

The preferred structure is:

- `page`: route entry only
- `capability`: feature orchestration
- `view/components`: rendering
- `service`: external integration
- `queries/mutations`: server state adapters
- `store`: feature UI state
- `model`: contracts, schemas, mappers
- `core`: foundation and infrastructure
- `shared`: truly reusable primitives
- `features`: domain modules

Claude must optimize for clarity of ownership, bounded change, and safe refactorability.

---

## Hard Rules

### 1. Keep route files thin
Files such as `app/**/page.tsx` must only resolve route entry and delegate screen behavior to a feature capability.

A route file must not become:
- controller
- orchestration layer
- query composition layer
- mutation coordination layer
- modal manager
- data mapping layer

### Good
```tsx
import { UsersPageCapability } from "@/features/users";

export default function Page() {
  return <UsersPageCapability />;
}
```

### Bad
- stacking many hooks in `page.tsx`
- calling APIs directly in the route
- mutating cache in the route
- combining UI rendering and behavioral orchestration in the route

---

### 2. Use capability as the screen orchestration layer
A capability is the behavioral composition boundary of a feature screen.

A capability may:
- call feature hooks
- combine query and mutation state
- read feature UI store
- define handlers
- transform raw state into render-ready props
- connect child components

A capability should not:
- become a dumping ground for unrelated helpers
- replace the service layer
- absorb all domain logic without extraction strategy

### Example
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

---

### 3. Views render, they do not orchestrate
UI components must receive prepared state and callbacks.

Views must not own:
- HTTP calls
- query client invalidation
- navigation rules
- backend payload mapping
- domain workflow coordination

Views should:
- render props
- emit events
- stay deterministic
- remain testable in isolation

### Example
```tsx
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

---

### 4. Services only integrate with external systems
A service is an integration adapter.

A service may:
- call backend endpoints
- serialize input
- normalize transport output
- encapsulate API access

A service must not:
- open modal
- trigger toast
- navigate
- control render behavior
- manage UI state

### Example
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

---

### 5. Queries and mutations stay outside views
Server-state adapters belong to the feature, not to the view.

Use dedicated query and mutation files.

### Query
```ts
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

### Mutation
```ts
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

Claude must prefer server state tools for remote truth instead of copying remote data into client stores.

---

### 6. Separate state by nature
State must be classified before it is stored.

#### Local state
Use for ephemeral component-only concerns:
- input state
- hover state
- single-component disclosure
- temporary interaction data

#### Server state
Use for backend-owned truth:
- lists
- details
- cached resources
- invalidation and mutation lifecycle

#### Feature UI state
Use for UI coordination inside one feature:
- modal visibility
- shared filters for one domain screen
- active local tab in a feature
- drawer visibility inside one business area

### Example
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

Claude must not introduce a large app-wide store unless there is a strong cross-domain reason.

---

### 7. Domain boundaries must be explicit
A feature must be treated as a bounded module.

Other parts of the app must not depend on its internal file structure.

Each feature should expose a public API through `index.ts`.

### Example
```ts
export { UsersPageCapability } from "./capabilities/users-page.capability";
export { useUsersQuery } from "./queries/use-users-query";
export type { User } from "./model/user.types";
```

### Allowed external import
```ts
import { UsersPageCapability } from "@/features/users";
```

### Forbidden external import
```ts
import { useUsersUIStore } from "@/features/users/store/users-ui.store";
import { usersService } from "@/features/users/services/users.service";
```

Claude must preserve feature encapsulation.

---

## Preferred Project Structure
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

---

## Ownership Model

### `app`
Framework entry and route resolution.

Owns:
- route entry
- layout composition
- loading and error boundaries
- provider registration

Does not own:
- feature behavior
- query orchestration
- business state
- integration details

### `core`
Application foundation.

Owns:
- HTTP client
- environment config
- auth session primitives
- global providers
- cross-cutting infrastructure

Does not own:
- feature business behavior
- domain-specific UI

### `shared`
Reusable assets with no business ownership.

Owns:
- design system primitives
- generic utility helpers
- generic hooks
- reusable stateless components

Does not own:
- domain behavior
- feature-specific services
- business workflows

If something belongs clearly to one feature, it must not go to `shared`.

### `features`
Primary business boundary.

Owns:
- feature orchestration
- feature-specific UI
- domain model
- remote integration for that domain
- server-state adapters
- feature-scoped UI state

---

## Model Layer Guidance
The model layer exists to stabilize domain contracts against backend shape volatility.

Recommended use:
- `*.types.ts` for static contracts
- `*.schema.ts` for runtime validation
- `*.mapper.ts` for transport-to-domain transformation

Claude should introduce model files when:
- backend payloads are noisy
- naming differs from UI language
- nullability is inconsistent
- transport contracts should not leak into views
- parsing or normalization is required

Do not let raw API shapes silently spread across the feature unless the shape is intentionally trivial and stable.

---

## Readme Standard per Feature
Every feature should have a short `README.md` that explains architectural decisions.

Recommended structure:

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

The README should explain ownership and rules, not narrate JSX.

---

## Anti-Patterns Claude Must Reject

### Fat pages
Too many hooks, handlers, side effects, queries, and UI state in route files.

### API calls inside render components
Views directly fetching or mutating backend data.

### Global store abuse
Remote truth duplicated in Zustand or Redux without clear synchronization design.

### Shared folder dumping
Placing domain-specific code into `shared` because ownership was ignored.

### Cross-feature internal imports
Importing another feature’s service, store, or component internals directly.

### Hook pileup without orchestration boundary
Route files or large components using many hooks without a capability layer.

---

## Refactor Protocol

When Claude encounters structural degradation, refactor in this order:

### Case 1: `page.tsx` is overloaded
1. Extract a capability
2. Move hooks and handlers into the capability
3. Leave the page as a delegating shell

### Case 2: View mixes rendering and data logic
1. Split render component from orchestration
2. Move remote interaction into query, mutation, or capability
3. Pass prepared props into the view

### Case 3: Remote data copied into store
1. Identify whether it is actually server state
2. Move ownership to React Query or equivalent
3. Keep only UI coordination in store

### Case 4: Backend shape leaks through many files
1. Add model contracts
2. Introduce mapper
3. Normalize at the feature boundary

### Case 5: Feature internals are imported externally
1. Add or tighten `index.ts`
2. Replace direct imports with public API imports
3. Preserve internals as private implementation

---

## Review Checklist Claude Must Apply
Before finalizing generated or refactored code, validate:

- Is the route file thin
- Is there a clear orchestration owner
- Are views render-focused
- Are services free from UI concerns
- Are queries and mutations outside the view
- Is state separated by nature
- Is remote data not duplicated unnecessarily
- Is feature UI state scoped locally
- Is the feature boundary explicit
- Is there a public API through `index.ts`
- Is shared truly shared
- Is documentation decision-oriented

If several answers are no, the architecture is already degrading.

---

## Generation Directives for Claude

When generating code in this repository, Claude must:

1. Prefer feature-first organization under `src/features`
2. Create thin route files under `app`
3. Introduce a capability for screen-level orchestration
4. Keep views focused on rendering
5. Place backend integration in `services`
6. Place server-state hooks in `queries`
7. Use `model` for types, schemas, and mappers
8. Use feature-local store only for shared visual state
9. Expose feature contracts via `index.ts`
10. Avoid introducing abstractions that do not improve boundary clarity

Claude must optimize for systems that can evolve, not only for concise initial implementation.

---

## Minimal Blueprint

### Route
```tsx
import { UsersPageCapability } from "@/features/users";

export default function Page() {
  return <UsersPageCapability />;
}
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

### Query
```ts
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

### Mutation
```ts
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

### Public API
```ts
export { UsersPageCapability } from "./capabilities/users-page.capability";
export { useUsersQuery } from "./queries/use-users-query";
export type { User } from "./model/user.types";
```

---

## Final Rule
Scaling React well is not about adding abstraction everywhere.

It is about introducing the correct boundaries before growth turns invisible coupling into delivery drag.

Claude must favor:

- thin pages
- strong capabilities
- render-only views
- integration-only services
- state by nature
- explicit feature ownership
- stable public APIs
- concise architectural documentation

When in doubt, prefer boundary clarity over short-term convenience.
