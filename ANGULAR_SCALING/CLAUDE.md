# CLAUDE_ANGULAR.md
## Angular Frontend Architecture Contract
### Boundary-Oriented Scaling for Claude

## Mission
This document defines the architectural contract Claude must follow when generating, reviewing, refactoring, or extending Angular code in this repository.

The goal is not only to make Angular work in the first phase of the project.

The goal is to keep the frontend evolvable after the application grows in number of features, async flows, domain rules, contributors, and integration points.

Angular usually does not fail at the beginning of the project.

It fails when the system changes complexity level and the codebase still behaves as if it were small.

The core failure mode is not Angular itself.

It is missing boundaries.

---

## Primary Thesis
A scalable Angular frontend keeps three concerns explicit:

- component responsibility
- state ownership
- domain boundary

Claude must preserve these boundaries across all generated and refactored code.

The preferred macro organization is:

- `core`: app-wide infrastructure
- `shared`: truly reusable cross-domain primitives
- `features`: business capabilities

Inside each feature, the preferred internal separation is:

- `pages` or `containers`: route-facing composition
- `components`: rendering-focused pieces
- `facade`: orchestration and use-case coordination
- `data-access` or `services`: external integration
- `models`: DTOs, domain models, view models, mappers, adapters
- `state`: optional feature-local state
- `guards`: route access rules when feature-owned
- `resolvers`: optional route prefetch logic when justified

Claude must optimize for clarity of ownership, bounded change, and safe refactorability.

---

## Hard Rules

### 1. Organize by feature, not only by technical bucket
Claude must prefer feature-first architecture.

#### Preferred
```txt
src/
  app/
    core/
    shared/
    features/
      auth/
      customers/
      billing/
```

#### Not preferred as the primary system shape
```txt
src/
  app/
    components/
    services/
    models/
    pages/
    guards/
```

Global technical folders may look clean early, but they scale poorly because ownership becomes ambiguous and cross-feature coupling grows silently.

---

### 2. Keep route-facing pages thin
A page or container component must not become a god component.

A page may:
- bind route params
- connect facade state to template
- trigger top-level use-case actions
- compose feature UI

A page must not:
- call `HttpClient` directly
- perform raw DTO mapping
- own large orchestration flows
- accumulate unrelated business rules
- become the place where every subscription, state flag, and mutation lives

Claude must keep page-level composition thin and delegate behavior to a facade or other explicit orchestration layer.

---

### 3. Use facade as the orchestration boundary
A facade is the feature-level API that coordinates workflow.

A facade may:
- expose observables, signals, computed state, or selectors
- coordinate one or more data-access services
- manage feature loading, error, empty, and success states
- expose methods such as `load`, `submit`, `refresh`, `select`, `reset`
- aggregate data into render-friendly state

A facade must not:
- become a global dumping ground
- coordinate unrelated domains
- replace model mapping discipline
- hide all state with no explicit contract

#### Example
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

Claude must ensure every screen with meaningful workflow has a clear orchestration owner.

---

### 4. Components render, they do not integrate
Presentational components should receive state and emit events.

They may:
- accept `@Input` values
- expose `@Output` events
- render loading, error, empty, or success states
- manage minimal local UI interaction when necessary

They must not:
- call APIs directly
- know transport endpoint details
- map backend DTOs inline
- decide routing policy
- own large business workflows

#### Example
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

Claude must preserve a clean line between rendering and orchestration.

---

### 5. Data-access services only integrate with external systems
A data-access service is for transport and integration.

It may:
- call `HttpClient`
- serialize params and payloads
- encapsulate endpoint paths
- return DTOs or transport-level contracts
- wrap HTTP-specific behavior

It must not:
- manipulate modal state
- decide navigation
- emit UI notifications directly as business policy
- mix unrelated feature rules
- become a multi-domain god service

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

Claude must keep services transport-oriented.

---

### 6. Adapt DTOs before UI consumption
The backend contract is not automatically the correct frontend contract.

Claude must introduce a mapping layer when:
- API names differ from frontend names
- nullability is noisy
- payloads are too verbose
- UI only needs a subset
- multiple backends feed one screen
- the API contract is unstable or unfriendly to rendering

Use these layers intentionally:
- DTO: transport contract
- domain model: normalized internal contract
- view model: render-oriented shape
- mapper or adapter: transformation boundary

#### Example
```ts
export type CustomerDto = {
  customer_id: string;
  full_name: string | null;
  email_address: string | null;
  active_flag: 0 | 1;
};

export type CustomerVm = {
  id: string;
  displayName: string;
  email: string;
  isActive: boolean;
};

export function mapCustomerDtoToVm(dto: CustomerDto): CustomerVm {
  return {
    id: dto.customer_id,
    displayName: dto.full_name ?? "Sem nome",
    email: dto.email_address ?? "",
    isActive: dto.active_flag === 1,
  };
}
```

Claude must not let raw DTOs leak across templates unless the transport contract is intentionally trivial and stable.

---

### 7. Separate state by nature
Claude must classify state before choosing where it lives.

#### Local component state
Use for:
- input interaction
- local toggles
- isolated disclosure
- temporary UI-only behavior

#### Feature state
Use for:
- shared workflow state inside one feature
- filters shared by sibling components
- selection state inside one use case
- multi-step feature flow

#### Remote state
Use for:
- backend-owned resources
- cached data
- lists, details, server results
- refresh and revalidation flows

Claude must not mirror remote truth into arbitrary local stores without a deliberate synchronization reason.

---

### 8. RxJS must remain legible
RxJS is essential in Angular but frequently abused.

Claude should use RxJS for:
- async composition
- cancellation
- stream coordination
- dependent requests
- event-driven flow control
- time-based operations

Claude must avoid:
- giant unreadable pipelines
- business rules hidden across random operators
- heavy side effects inside `tap`
- using reactive ceremony where simple imperative code is clearer

A reactive flow should be explainable by another engineer without decoding a puzzle.

---

### 9. `shared` must stay small and clean
`shared` is not a fallback location for unowned code.

Allowed in `shared`:
- UI primitives
- generic pipes
- generic utils
- generic directives
- low-level reusable types
- reusable layout primitives with no business ownership

Forbidden in `shared`:
- auth-specific rules
- billing-specific mappers
- customer-specific validators
- one-feature facades or services pretending to be generic

If an artifact belongs clearly to one feature, Claude must keep it in that feature.

---

### 10. Keep feature internals private
Features should expose clear contracts and hide internal layout when possible.

A feature may expose a public surface via `index.ts` or explicit route integration points.

Other features must not casually import:
- internal facades
- internal mappers
- internal data-access services
- internal state utilities
- internal helper files

Claude must preserve feature encapsulation to protect future refactors.

---

## Preferred Project Structure
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
          components/
          facade/
          data-access/
          models/
          pages/
        create-customer/
        customer-details/

      billing/
        issue-invoice/
        invoice-list/
```

Claude should preserve this shape or move code toward it when refactoring.

---

## Ownership Model

### `core`
Owns:
- global config
- interceptors
- root guards
- layout shell
- app providers
- cross-cutting infrastructure

Does not own:
- feature-specific business logic
- domain-specific UI
- business mappers tied to one feature

### `shared`
Owns:
- reusable UI atoms and primitives
- generic helpers
- common directives and pipes
- business-agnostic types

Does not own:
- domain workflows
- feature-specific service logic
- specific business mapping

### `features`
Owns:
- business use cases
- orchestration
- feature UI
- data-access for that domain
- local models and mappers
- local state
- local guards when justified

---

## Feature Anatomy

### `pages/`
Top-level route-facing components.

Responsibilities:
- connect route to feature
- consume facade state
- compose feature UI
- delegate rendering

### `components/`
Render-focused pieces.

Responsibilities:
- display state
- emit user events
- stay as UI-oriented as possible

### `facade/`
Use-case orchestration.

Responsibilities:
- own workflow entrypoints
- aggregate state
- combine integration and mapping
- expose stable view state

### `data-access/`
Transport and integration.

Responsibilities:
- endpoint communication
- transport contract usage
- HTTP encapsulation

### `models/`
Contracts and transformation.

Responsibilities:
- DTO types
- domain models
- view models
- mapping and adapters
- normalization helpers

### `state/`
Optional local feature state boundary.

Responsibilities:
- store feature-local shared state
- expose explicit state API
- avoid becoming hidden orchestration without contract

### `guards/`
Feature-owned route restrictions only when appropriate.

Responsibilities:
- feature access rules
- navigation gating
- local policy checks

Claude must keep each folder honest to its intended role.

---

## Angular-Specific Guidance

### Standalone Components
Claude should prefer standalone components in modern Angular codebases when that matches the project baseline.

Benefits:
- local dependency declaration
- less module ceremony
- stronger feature locality
- simpler feature composition

But standalone components are not architecture by themselves.

Claude must not confuse standalone with proper responsibility boundaries.

---

### Dependency Injection
Use DI to:
- express explicit dependencies
- isolate infrastructure
- improve testability
- swap implementations when needed

Do not use DI to hide oversized dependency graphs or poor ownership.

---

### Guards
Use guards for:
- navigation access control
- route policy gates
- auth checks tied to routing

Do not use guards as a place for business workflow orchestration.

---

### Interceptors
Use interceptors for:
- auth headers
- correlation IDs
- generic request logging
- global error shaping
- broad retry rules when justified

Do not place feature-specific business rules inside interceptors.

---

### Signals, RxJS, and Store Choice
Claude may use signals, RxJS, or a store library depending on the complexity.

#### Prefer signals for:
- feature-local state
- computed render state
- simple reactive coordination
- low-ceremony screen state

#### Prefer RxJS for:
- async streams
- request composition
- cancellation
- sequential or dependent remote flows
- time-based logic

#### Prefer a store library when:
- state is large and shared broadly
- effect modeling is necessary
- debugging and explicit governance matter
- the system complexity justifies it

Claude must not force one abstraction on every problem.

---

## Anti-Patterns Claude Must Reject

### Fat page components
Symptoms:
- too many subscriptions
- HTTP inside page
- mapping inside page
- large workflow logic in page

### Giant components
Symptoms:
- rendering, transport, validation, and business rules combined
- hard-to-test template and class
- many local flags with unclear ownership

### God services
Symptoms:
- one service used by many unrelated features
- transport, rules, navigation, cache, and UI concerns mixed together

### Shared dumping ground
Symptoms:
- business-specific helpers inside `shared`
- domain code pretending to be generic

### Raw DTO leakage
Symptoms:
- templates know backend naming quirks
- null checks repeated across components
- transport shape treated as UI shape

### RxJS over-ceremony
Symptoms:
- unreadable operator chains
- side effects hidden in streams
- difficult reasoning about flow ownership

### Cross-feature internal imports
Symptoms:
- direct imports from another feature’s internals
- refactors break unrelated areas
- hidden coupling accumulates

Claude must flag and avoid these patterns.

---

## Refactor Protocol

### Case 1: overloaded page component
1. extract workflow into facade
2. move transport concerns into data-access
3. leave page responsible for composition and delegation

### Case 2: component mixes rendering and integration
1. split page or container from presentational child
2. move API and orchestration out of the presentational component
3. pass view-ready inputs and event outputs

### Case 3: DTO leaks into multiple templates
1. define DTO type
2. define domain model or view model
3. introduce mapper
4. consume adapted model in UI

### Case 4: service knows too much
1. isolate transport in data-access
2. move workflow coordination into facade
3. move mapping into models
4. reduce cross-domain coupling

### Case 5: shared contains business code
1. identify real owner feature
2. move code back into that feature
3. leave only true primitives in `shared`

Claude should refactor toward clearer boundaries instead of layering more convenience logic on top of structural problems.

---

## Review Checklist Claude Must Apply
Before finalizing Angular code, validate:

- Is the project organized by feature instead of only by technical type
- Is there a clear orchestration owner
- Is the page thin
- Are render components free from transport logic
- Are services transport-oriented only
- Are DTOs adapted before UI consumption
- Is state separated by nature
- Is shared free from business leakage
- Are guards and interceptors in the right layer
- Are reactive flows readable
- Are feature boundaries explicit and protected

If several answers are no, the structure is already degrading.

---

## Generation Directives for Claude

When generating Angular code in this repository, Claude must:

1. prefer feature-first structure
2. keep route-facing components thin
3. introduce a facade for screen or use-case orchestration
4. keep presentational components render-focused
5. place HTTP integration in `data-access` or feature-local services
6. adapt DTOs before UI use when needed
7. use `models` for contracts and mappers
8. keep shared minimal and truly generic
9. use signals or RxJS with clarity
10. optimize for evolvable architecture, not just short initial code

---

## Minimal Blueprint

### Page
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

### Data Access
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

### Presentational Component
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

- what the feature owns
- how the structure is divided
- who orchestrates the flow
- what state strategy is used
- what contributors must not violate

### Example
```md
# List Customers Feature

## Responsibility
Carregar, filtrar e exibir clientes.

## Structure
- pages: composição da rota
- facade: orquestra o caso de uso
- data-access: conversa com a API
- models: DTOs, view models e mapeamento
- components: renderização

## Rules
- componente não chama HttpClient diretamente
- DTO não vaza para a UI
- page delega para facade
- compartilhamento externo deve passar por contrato explícito
```

Documentation should explain decisions and boundaries, not trivial template details.

---

## Final Rule
Angular scales well when the application stops being organized only by framework artifact and starts being organized by ownership, responsibility, and bounded change.

Claude must favor:

- feature-first structure
- thin pages
- strong facades
- honest components
- transport-only data-access
- mapped models instead of raw DTO leakage
- state by nature
- explicit feature boundaries
- concise architectural documentation

When in doubt, prefer boundary clarity over short-term convenience.
