# SKILL.md
## Angular Frontend Architecture at Scale
### Skill Specification for Codex, Claude, and Similar Coding Agents

## Purpose
This skill defines a production-grade architectural contract for Angular applications that must continue to evolve under real business growth, increasing domain complexity, more contributors, and a wider set of asynchronous flows.

Angular does not become unmaintainable because it has modules, services, dependency injection, guards, interceptors, or RxJS.

It becomes unmaintainable when those tools are used without boundaries.

A frontend Angular codebase usually does not fail at the beginning.

It fails when growth changes the complexity level of the system and the architecture still behaves as if the application were small.

The main failure mode is not rendering.

It is uncontrolled responsibility spread.

---

## Core Thesis
A scalable Angular frontend needs explicit boundaries for:

- feature ownership
- orchestration flow
- UI rendering
- access to external systems
- domain contracts
- state ownership
- shared infrastructure
- public feature APIs

The central operating model of this skill is:

- `core` owns application-wide infrastructure
- `shared` owns truly cross-domain reusable primitives
- `features` own business capabilities
- each feature separates rendering, orchestration, data access, and contracts
- DTOs do not leak freely into the UI
- containers do not become giant workflow sinks
- services do not become global dumping grounds
- RxJS is used for composition, not obfuscation

---

## Primary Problem Statement
Angular applications tend to degrade when the following happens:

- new features are inserted into old modules without structural criteria
- components render UI, call HTTP, map DTOs, handle loading, and decide navigation at the same time
- services become storage for every rule no one knew where to place
- `shared` turns into a dumping ground for business code disguised as generic reuse
- feature boundaries disappear and internal implementation details become public dependencies
- the team avoids refactors because every change can reverberate across unrelated screens

At that point, the system may still compile and run.

It is already becoming expensive to change.

Architecture is what keeps change localized.

---

## Architectural Position
The recommended Angular structure is **feature-first**, with internal separation by responsibility inside each feature.

Do not optimize the repository around global technical buckets only.

Global folders such as `components`, `services`, `models`, and `utils` may look organized initially, but they degrade fast because the real unit of change is not “file type”.

The real unit of change is business capability.

A healthier structure is:

- `core`: infrastructure and cross-app foundation
- `shared`: truly reusable design and technical primitives
- `features/<domain>/<use-case>`: vertically organized functional units

Inside each feature, responsibilities are separated into:
- `components`: render-only or mostly presentational UI pieces
- `containers` or `pages`: screen composition and binding
- `facade`: orchestration and use-case coordination
- `data-access` or `services`: API communication and integration
- `models`: contracts, DTOs, view models, adapters, mappers, schemas
- `state`: optional feature store or signal-based feature state
- `guards`: feature access and route restrictions when truly local
- `resolvers`: optional pre-fetch boundaries when justified

---

## Recommended Macro Structure
```txt
src/
  app/
    core/
      config/
      guards/
      interceptors/
      layout/
      providers/

    shared/
      ui/
      utils/
      types/

    features/
      auth/
        login/
          components/
          facade/
          data-access/
          models/
          pages/
        reset-password/

      customers/
        list-customers/
        create-customer/
        customer-details/

      billing/
        issue-invoice/
        invoice-list/
```

This structure is valid because it preserves three high-value boundaries:

- infrastructure is not mixed with domain logic
- reusable assets are not confused with business-specific code
- each feature owns what changes with it

---

## Flow Contract
A healthy Angular feature should follow a predictable action flow:

1. user interacts with UI
2. container or page dispatches the intent
3. facade or application layer orchestrates the use case
4. data-access service talks to the API or external resource
5. result is adapted into a UI-friendly model
6. components render the resulting state

### Mental model
```txt
User -> Container/Page -> Facade -> Data Access -> Model Adapter -> Presentational Components
```

This prevents the classic anti-pattern where one component:
- subscribes manually
- manages loading and error
- maps DTOs
- applies business rules
- triggers side effects
- decides navigation
- updates local and shared state

That is how a component becomes a maintenance hotspot.

---

## Non-Negotiable Principles

### 1. Organize by feature, not by technical type alone
A frontend does not evolve by “all services changing together”.

It evolves by business capability.

Therefore:
- put business code under `features`
- isolate each capability or use case
- keep infrastructure in `core`
- keep only true primitives in `shared`

#### Reject this as the main architecture
```txt
src/
  app/
    components/
    services/
    models/
    guards/
    pages/
```

This may work early, but it scales poorly because ownership becomes ambiguous and cross-domain coupling grows quietly.

---

### 2. Separate orchestration from rendering
Angular components should not all have the same responsibility.

Use a split such as:
- container or page component for orchestration
- presentational components for rendering
- facade for use-case coordination

#### Why
This keeps templates simpler, reduces behavioral duplication, and prevents presentational UI from accumulating backend and workflow knowledge.

#### Good responsibility split
- page or container: bind state to the template, dispatch user intent
- facade: coordinate workflow, combine streams or signals, expose view state
- presentational component: render inputs, emit outputs

---

### 3. Use facade as the feature orchestration boundary
A facade is the application-facing API of a feature or use case.

A facade may:
- expose observables, signals, or computed state
- coordinate multiple services
- centralize loading and error state
- manage state transitions for the use case
- provide methods such as `load`, `submit`, `refresh`, `select`, `reset`

A facade must not become:
- a global god object
- a dumping ground for unrelated flows
- a replacement for proper model mapping
- a hidden store with no explicit contract

#### Example
```ts
@Injectable()
export class LoginFacade {
  private readonly state = signal<{
    status: "idle" | "loading" | "success" | "error";
    errorMessage: string | null;
  }>({
    status: "idle",
    errorMessage: null,
  });

  readonly vm = computed(() => this.state());

  constructor(private readonly authDataAccess: AuthDataAccess) {}

  async login(input: LoginFormValue): Promise<void> {
    this.state.update((current) => ({
      ...current,
      status: "loading",
      errorMessage: null,
    }));

    try {
      await this.authDataAccess.login(input);
      this.state.set({ status: "success", errorMessage: null });
    } catch (error) {
      this.state.set({
        status: "error",
        errorMessage: "Falha ao autenticar",
      });
    }
  }
}
```

#### Why
A facade hides workflow complexity from the page and from UI components.

The feature gains one obvious orchestration owner.

---

### 4. Data-access services should only integrate with external systems
A data-access service is responsible for transport and integration.

It may:
- call `HttpClient`
- serialize query params
- submit payloads
- return DTOs
- encapsulate endpoint details

It must not:
- manipulate UI state
- trigger modal behavior
- decide routing
- format view labels
- define visual policy
- contain broad unrelated domain logic

#### Example
```ts
@Injectable()
export class CustomersDataAccess {
  constructor(private readonly http: HttpClient) {}

  list(): Observable<CustomerDto[]> {
    return this.http.get<CustomerDto[]>("/api/customers");
  }

  create(payload: CreateCustomerDto): Observable<CustomerDto> {
    return this.http.post<CustomerDto>("/api/customers", payload);
  }
}
```

#### Why
The service boundary should be transport-oriented, not UI-oriented.

---

### 5. DTOs must not leak unchecked into the UI
The API contract is not automatically the best UI contract.

Introduce explicit mapping when:
- backend names are inconsistent with frontend language
- nullability is noisy
- payload is verbose
- UI needs only a subset
- backend shape is unstable or tenant-specific

Use:
- DTOs for transport
- domain models for internal feature contracts
- view models for render-ready state
- adapters or mappers for transformation

#### Example
```ts
export type CustomerDto = {
  customer_id: string;
  full_name: string | null;
  email_address: string | null;
  active_flag: 0 | 1;
};

export type Customer = {
  id: string;
  name: string;
  email: string;
  isActive: boolean;
};

export function mapCustomerDtoToModel(dto: CustomerDto): Customer {
  return {
    id: dto.customer_id,
    name: dto.full_name ?? "Sem nome",
    email: dto.email_address ?? "",
    isActive: dto.active_flag === 1,
  };
}
```

#### Why
This preserves independence between backend evolution and UI readability.

---

### 6. Shared must remain small and defended
A scalable Angular codebase treats `shared` as a strict boundary.

Valid content for `shared`:
- design system primitives
- generic UI atoms or layout helpers
- small utility functions
- stable injection tokens
- pure pipes with cross-domain value
- generic types with no business ownership

Invalid content for `shared`:
- feature-specific facades
- customer-specific validators
- billing-specific mappers
- login workflow rules
- fake “generic” services that only one feature uses

A large `shared` folder is often a symptom of architectural avoidance.

---

### 7. State must be separated by nature
Do not choose state management based on trend or ideology.

Classify state first.

#### Local component state
Use for:
- form interaction
- local toggles
- ephemeral UI-only concerns
- isolated display state

#### Feature state
Use for:
- workflow state shared across multiple components in one feature
- selected entity inside a feature
- multi-step feature flows
- local filters or local pagination scoped to the feature

#### Remote state
Use for:
- backend-owned resources
- lists, details, queries, submitted results
- cacheable API responses
- polling and refresh flows

#### Recommended rule
- small isolated feature: service or signal store may be enough
- moderate feature with many consumers: facade plus feature-local state
- complex multi-source app: stronger store solution may be justified

Do not introduce a global store because opening a modal feels important.

---

### 8. RxJS is a composition tool, not a complexity badge
RxJS is powerful and essential in Angular, but it is frequently abused.

Use RxJS to:
- compose async events
- cancel stale flows
- derive reactive state
- sequence dependent async actions
- coordinate UI and remote streams cleanly

Do not use RxJS to:
- hide business rules in unreadable pipelines
- create giant operator chains with unclear ownership
- scatter side effects inside random `tap`
- replace simple code with reactive ceremony

#### Bad smell
A chain with multiple nested `switchMap`, `mergeMap`, `combineLatest`, `shareReplay`, and side effects, where no one can quickly explain the business flow.

#### Preferred outcome
A reactive flow another engineer can read without decoding a ritual.

---

### 9. Components must be explicit about role
Every Angular component should have a clear category.

#### Presentational component
- receives inputs
- emits outputs
- does minimal state handling
- does not know API details
- is easy to preview and test

#### Container or page component
- binds facade state to template
- handles route params if needed
- dispatches user actions
- composes presentational children

Avoid components that are simultaneously:
- route handler
- data loader
- mapper
- validator
- navigation decider
- notification manager
- presentational widget

That is structural overload.

---

### 10. Keep feature internals private
A feature should have an explicit public surface.

Expose a small API through an `index.ts` when needed.

#### Example
```ts
export * from "./pages/login-page.component";
export * from "./facade/login.facade";
export * from "./models/login.models";
```

Other features should not depend on internal folders casually.

#### Forbidden coupling
- customer feature imports billing internal mapper
- auth feature imports customer internal service
- dashboard imports internal implementation details from multiple features

The more internal files become public dependencies, the less safe refactoring becomes.

---

## Feature Anatomy

A scalable Angular feature commonly contains the following parts.

### `pages/`
Top-level page or route-facing containers.

Responsibilities:
- bind route params
- connect facade to the template
- orchestrate layout composition
- delegate rendering to subcomponents

### `components/`
Render-focused building blocks of the feature.

Responsibilities:
- accept `@Input` or signal inputs
- emit `@Output` or callback events
- remain UI-focused

### `facade/`
Application layer for use-case orchestration.

Responsibilities:
- load data
- submit actions
- expose state
- coordinate between services and models
- aggregate workflow transitions

### `data-access/`
External integration.

Responsibilities:
- HTTP
- WebSocket or external gateway if applicable
- endpoint configuration
- transport contracts

### `models/`
Contracts and transformation.

Responsibilities:
- DTOs
- domain models
- view models
- adapters
- mapping utilities
- optional validation schemas

### `state/`
Optional feature-local store.

Responsibilities:
- maintain feature state not owned by one single component
- expose selectors, signals, or state API
- avoid becoming a second hidden facade

### `guards/`
Feature route restrictions only when truly feature-owned.

Responsibilities:
- protect feature navigation
- depend on explicit auth or policy contracts

Do not use guards as a business logic landfill.

---

## Example Structure by Feature
```txt
src/app/features/customers/list-customers/
  components/
    customer-list.component.ts
    customer-list-filters.component.ts
    customer-list-table.component.ts

  facade/
    list-customers.facade.ts

  data-access/
    customers.data-access.ts

  models/
    customer.dto.ts
    customer.model.ts
    customer.vm.ts
    customer.mapper.ts

  pages/
    list-customers-page.component.ts
```

This is healthy because:
- one use case owns its dependencies
- flow is easy to trace
- state and transport are explicit
- refactor cost is localized

---

## Angular-Specific Guidance

### Standalone Components
Prefer standalone components for modern Angular composition when consistent with the project baseline.

Benefits:
- reduced module ceremony
- easier local dependency declaration
- better feature locality
- simpler migration paths

But do not mistake standalone components for architecture.

Standalone does not replace boundary design.

---

### Dependency Injection
Use DI for:
- explicit dependency contracts
- replacing implementations
- isolating infrastructure
- testing

Do not use DI to hide oversized service graphs or circular responsibility.

A DI container can distribute good architecture.

It can also distribute chaos efficiently.

---

### Guards and Interceptors
Keep them in the correct layer.

#### Guards
Use for:
- navigation restrictions
- route policy checks
- access gates

Do not put business workflow orchestration into guards.

#### Interceptors
Use for:
- auth headers
- request correlation
- generic error translation
- logging or tracing
- retry policy when globally justified

Do not place feature-specific business behavior in interceptors.

---

### Signals vs RxJS vs Store
This skill is compatible with Angular signals, RxJS, or a store library.

Choose based on responsibility and system complexity.

#### Signals fit well for:
- local and feature state
- computed render state
- simple reactive orchestration
- low-ceremony UI coordination

#### RxJS fits well for:
- transport streams
- complex async workflows
- cancellation
- event composition
- time-based flows

#### Store libraries fit well for:
- large cross-component state
- explicit effect modeling
- debugging and replay
- stronger governance across a large app

Do not force a single abstraction on every problem.

---

## Design Patterns Recommended

### Smart and Dumb Components
Use when:
- one component should orchestrate
- others should only render

Benefit:
- UI becomes predictable and reusable
- business flow leaves the template

---

### Facade Pattern
Use when:
- the screen depends on multiple data sources
- state transitions must be coordinated
- the page should not know store and service details
- multiple child components need one stable API

Benefit:
- screen complexity is centralized without leaking internals to the UI

---

### Container and Presentational Pattern
Use when:
- the UI should be isolated from orchestration
- large screens need composition discipline

Benefit:
- testability improves
- responsibilities become explicit

---

### Adapter Pattern
Use when:
- DTOs differ from frontend needs
- backend naming is inconsistent
- multiple APIs feed one feature

Benefit:
- UI is protected from transport volatility

---

### Strategy Pattern
Use when:
- rules vary by tenant, product type, role, or environment
- branching logic is growing inside one service

Benefit:
- behaviors become composable instead of hard-coded in large condition chains

---

### Observer Pattern
Naturally present via RxJS and signals.

Use with intent.

Do not model every simple event as a highly abstract reactive lattice.

---

### State Modeling
Even without NgRx or another store, explicitly model states such as:
- idle
- loading
- success
- error
- empty

This reduces UI ambiguity and edge-case bugs.

---

## Anti-Patterns to Reject

### 1. Giant components
Symptoms:
- HTTP inside component
- DTO mapping inside component
- too many subscriptions
- rule handling mixed with template logic

Remedy:
- split into page or container, facade, and presentational pieces

---

### 2. God services
Symptoms:
- one service used across many unrelated features
- HTTP, validation, navigation, toasts, caching, and business rules together
- increasing fear of changing service methods

Remedy:
- split into feature-local data-access, facade, and model mapping layers

---

### 3. Shared dumping ground
Symptoms:
- business validators in `shared`
- domain-specific mappers in `shared`
- “generic” helpers that only one feature uses

Remedy:
- move ownership back into the correct feature

---

### 4. Raw DTO leakage
Symptoms:
- UI fields named after backend transport quirks
- null checks duplicated everywhere
- components knowing API naming conventions

Remedy:
- adapt DTO to model or view model at the boundary

---

### 5. RxJS over-ceremony
Symptoms:
- unreadable streams
- side effects hidden in operators
- hard-to-debug async behavior

Remedy:
- simplify pipelines
- move business decisions into named methods
- keep reactive graphs legible

---

### 6. Global technical folders as primary structure
Symptoms:
- low ownership clarity
- cross-domain coupling
- ever-growing central services and models

Remedy:
- move to feature-first structure with local responsibility slices

---

## Review Checklist for Agents
Before generating, refactoring, or approving Angular code, verify:

- Is the code organized by feature rather than only by technical bucket
- Is there a clear orchestration owner for the use case
- Are components explicit about being presentational or container-like
- Are data-access services transport-oriented only
- Are DTOs adapted before reaching the UI
- Is state classified by nature
- Is `shared` free from domain leakage
- Are guards and interceptors used only for correct cross-cutting concerns
- Are RxJS flows legible and bounded
- Is the feature easy to trace from user action to rendered state
- Can the feature evolve without forcing changes across unrelated modules

If several answers are no, the structure is already degrading.

---

## Refactor Protocol

### Case 1: giant component
1. Extract API logic into data-access
2. Extract workflow into facade
3. Keep component focused on binding and rendering
4. Split large template into presentational children

### Case 2: service knows too much
1. separate transport concerns from orchestration
2. move use-case coordination into facade
3. move contract transformation into models or mapper

### Case 3: backend shape leaks into the UI
1. define DTO
2. define model or view model
3. add mapper
4. consume adapted shape in components

### Case 4: multiple components coordinate the same feature flow
1. introduce facade or feature state owner
2. expose stable view state
3. reduce cross-component implicit coupling

### Case 5: `shared` contains business code
1. identify true owner feature
2. relocate code into feature boundary
3. leave only truly reusable primitives in `shared`

---

## Example Operational Blueprint

### Page component
```ts
@Component({
  selector: "app-list-customers-page",
  standalone: true,
  template: `
    <app-customer-list
      [customers]="vm().customers"
      [isLoading]="vm().status === 'loading'"
      [errorMessage]="vm().errorMessage"
      (refresh)="refresh()"
    />
  `,
  imports: [CustomerListComponent],
  providers: [ListCustomersFacade, CustomersDataAccess],
})
export class ListCustomersPageComponent {
  readonly vm = this.facade.vm;

  constructor(private readonly facade: ListCustomersFacade) {
    this.facade.load();
  }

  refresh(): void {
    this.facade.load();
  }
}
```

### Facade
```ts
@Injectable()
export class ListCustomersFacade {
  private readonly state = signal<{
    status: "idle" | "loading" | "success" | "error";
    customers: CustomerVm[];
    errorMessage: string | null;
  }>({
    status: "idle",
    customers: [],
    errorMessage: null,
  });

  readonly vm = computed(() => this.state());

  constructor(private readonly customersDataAccess: CustomersDataAccess) {}

  load(): void {
    this.state.update((current) => ({
      ...current,
      status: "loading",
      errorMessage: null,
    }));

    this.customersDataAccess
      .list()
      .pipe(map((dtos) => dtos.map(mapCustomerDtoToVm)))
      .subscribe({
        next: (customers) => {
          this.state.set({
            status: "success",
            customers,
            errorMessage: null,
          });
        },
        error: () => {
          this.state.set({
            status: "error",
            customers: [],
            errorMessage: "Falha ao carregar clientes",
          });
        },
      });
  }
}
```

### Data access
```ts
@Injectable()
export class CustomersDataAccess {
  constructor(private readonly http: HttpClient) {}

  list(): Observable<CustomerDto[]> {
    return this.http.get<CustomerDto[]>("/api/customers");
  }
}
```

### Mapper
```ts
export type CustomerVm = {
  id: string;
  displayName: string;
  email: string;
};

export function mapCustomerDtoToVm(dto: CustomerDto): CustomerVm {
  return {
    id: dto.customer_id,
    displayName: dto.full_name ?? "Sem nome",
    email: dto.email_address ?? "",
  };
}
```

### Presentational component
```ts
@Component({
  selector: "app-customer-list",
  standalone: true,
  template: `
    <section>
      <h1>Clientes</h1>

      <button type="button" (click)="refresh.emit()">Atualizar</button>

      @if (isLoading) {
        <p>Carregando...</p>
      } @else if (errorMessage) {
        <p>{{ errorMessage }}</p>
      } @else {
        <ul>
          @for (customer of customers; track customer.id) {
            <li>{{ customer.displayName }} - {{ customer.email }}</li>
          }
        </ul>
      }
    </section>
  `,
})
export class CustomerListComponent {
  @Input({ required: true }) customers: CustomerVm[] = [];
  @Input({ required: true }) isLoading = false;
  @Input() errorMessage: string | null = null;

  @Output() refresh = new EventEmitter<void>();
}
```

---

## Documentation Standard
Each feature should have a concise `README.md` describing:

- responsibility of the feature
- internal structure
- orchestration owner
- state strategy
- data-access boundaries
- rules that contributors must not violate

### Example
```md
# List Customers Feature

## Responsibility
Carregar, filtrar e exibir clientes.

## Structure
- pages: composição da rota
- facade: orquestra o fluxo do caso de uso
- data-access: conversa com a API
- models: DTOs, view models e mapeamento
- components: renderização

## Rules
- componente não chama HttpClient diretamente
- DTO não vaza para a UI
- page delega para facade
- qualquer compartilhamento externo deve passar por contrato explícito
```

Good documentation explains decisions and boundaries, not obvious template details.

---

## Final Directive
Angular scales well when the application stops being organized merely by framework artifact and starts being organized by ownership, responsibility, and bounded change.

The practical defaults of this skill are:

- structure by feature
- isolate orchestration in facade or application layer
- keep components honest about their role
- keep services focused on integration
- adapt transport contracts before UI consumption
- use RxJS and signals with legibility
- keep `shared` small
- model state intentionally
- protect feature boundaries
- document architecture, not trivia

Do not add abstraction for its own sake.

Add boundaries where complexity would otherwise spread.
