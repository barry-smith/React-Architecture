# Frontend Architecture Context

This document defines the architecture, directory structure, responsibilities, dependency rules, and naming conventions for the frontend application.

The goal is to preserve clear ownership boundaries and prevent code from being moved into the wrong architectural slice simply because it operates on the same domain concept.

The architecture is based around:

```text
routes
  ↓
widgets
  ↓
features
  ↓
entities
  ↓
shared
```

This is a conceptual dependency model rather than a strict hierarchy. Not every route must use a widget, and features/entities may use shared infrastructure directly.

The most important distinction is:

> **Entities describe what the application knows about. Features describe what the application does. Widgets compose substantial application UI. Routes compose screens and URLs. Shared contains genuinely generic functionality.**

---

# 1. Top-level structure

```text
apps/web/src/
├── routes/
├── widgets/
├── features/
├── entities/
└── shared/
```

Each top-level directory represents an architectural responsibility.

---

# 2. Routes

```text
routes/
├── __root.tsx
├── login.tsx
├── clients/
│   └── index.tsx
└── ...
```

Routes are responsible for:

* URL structure
* route parameters
* search parameters
* navigation
* route-level loading/error states
* route-level data requirements
* authentication/route guards
* composing widgets, features and entities into a screen

Routes are **not** the business/application layer.

Do not place API operations, domain models, mutation logic or substantial reusable UI directly into route files.

With TanStack Router/Start, route files effectively fulfil the role that a traditional `pages/` directory would fulfil.

Do not introduce a separate `pages/` directory unless there is a specific architectural reason.

---

# 3. Widgets

```text
widgets/
├── client-list/
│   └── client-list.tsx
├── forecast-grid/
│   └── forecast-grid.tsx
└── ...
```

A widget is a substantial, application-specific UI composition.

A widget commonly combines:

* entity queries
* entity UI
* feature UI
* feature hooks
* multiple domain concepts

For example:

```text
widgets/forecast-grid/
```

might use:

```text
entities/forecast
entities/resource
features/forecast/update
features/resource/assign
```

A widget is not itself a domain entity or feature.

Use a widget when the UI is substantial enough to represent a meaningful application-level composition.

Do not create widgets merely to wrap every small component.

---

# 4. Entities

Entities represent domain concepts that the application knows about.

Examples:

```text
entities/
├── client/
├── workspace/
├── auth/
├── forecast/
├── project/
└── resource/
```

An entity is primarily a **noun**:

```text
Client
Workspace
Forecast
Project
Resource
```

An entity can contain:

* domain/application types
* Zod schemas
* constants
* API access
* DTOs
* mappers
* standard queries
* entity-specific search state
* entity-specific query hooks
* small entity-specific UI

An entity does **not** automatically own every operation performed on that entity.

For example:

```text
Register Client
Update Client
Delete Client
```

are application capabilities and belong to features.

The fact that an operation creates, updates or deletes an entity does not make that operation part of the entity.

---

# 5. Entity structure

An entity can use responsibility-specific directories such as:

```text
entities/client/
├── api/
│   ├── client.api.ts
│   ├── client.dto.ts
│   └── client.mapper.ts
│
├── model/
│   ├── client.constants.ts
│   ├── client.query.ts
│   └── client.types.ts
│
├── queries/
│   └── use-client.query.ts
│
├── search/
│   ├── client-search.schema.ts
│   └── use-client-search-actions.ts
│
├── ui/
│   └── client-rows-per-page.tsx
│
└── index.ts
```

These directories are **not mandatory boilerplate**.

Only create a directory when the slice has a responsibility that belongs there.

For example:

```text
entities/workspace/
```

may not need `search/` if Workspace does not have entity-specific search state.

Likewise, an entity does not need `ui/` if it has no entity-specific UI.

---

# 6. Entity `api/`

The `api/` directory contains transport/API concerns owned by the entity.

Example:

```text
entities/client/api/
├── client.api.ts
├── client.dto.ts
└── client.mapper.ts
```

## `client.api.ts`

This is the API client for Client operations owned by the entity.

Example:

```ts
import type { ClientResponseDto } from "./client.dto";
import { apiClient } from "@/shared/api";
import { createQueryString } from "@/shared/lib";

export const clientApi = {
    all(params: ClientSearch) {
        return apiClient.get<ClientResponseDto>(
            `/client${createQueryString(params)}`
        );
    },

    create(formValues: CreateClientFormValues) {
        return apiClient.post<ClientItem>(
            "/client",
            formValues
        );
    },

    delete(clientId: DeleteClient) {
        return apiClient.delete(`/client/${clientId}`);
    },

    patch(formValues: UpdateClient) {
        return apiClient.patch(
            `/client/${formValues.id}`,
            formValues
        );
    }
};
```

### Responsibility

`client.api.ts` knows about:

* HTTP endpoints
* HTTP methods
* API client
* API request/response types
* query-string construction

It should not contain React code.

It should not contain UI behaviour.

It should not contain React Query configuration.

It should not contain feature orchestration.

---

# 7. Important API ownership rule

The application currently has some API methods on entities that accept types imported from features:

```ts
import type { CreateClientFormValues } from "@/features/create-client";
import type { DeleteClient } from "@/features/delete-client";
import type { UpdateClient } from "@/features/update-client";
```

This is an existing project pattern.

However, the preferred dependency direction is:

```text
feature
   ↓
entity
```

rather than:

```text
entity
   ↓
feature
```

Therefore, when modifying or extending this architecture, do not automatically propagate this dependency pattern.

A feature should normally be able to use an entity without the entity needing to know about the feature.

For example:

```text
features/create-client
        ↓
entities/client
```

is architecturally preferable to:

```text
entities/client
        ↓
features/create-client
```

If an existing API boundary makes this difficult, preserve existing behaviour unless there is a reason to refactor it. Do not perform broad architectural refactoring as part of an unrelated change.

---

# 8. Entity DTOs

Example:

```text
entities/client/api/client.dto.ts
```

```ts
export interface ClientResponseDto {
    items: Array<ClientItem>;
    total: number;
    page: number;
    pageSize: number;
    sortDirection: "asc" | "desc";
    sortBy: "createdAtUtc" | "displayName";
    search: string;
}
```

A DTO represents an API/transport contract.

A DTO does not have to be identical to an entity model.

For example:

```text
API representation
       ↓
ClientResponseDto
       ↓
Client application model
```

However, do not create artificial transformations merely to make DTOs different from models.

If the structures are already compatible, a mapper may simply return the value.

The distinction is about **ownership and boundaries**, not forcing unnecessary abstraction.

---

# 9. Entity mappers

Example:

```text
entities/client/api/client.mapper.ts
```

```ts
import type { PaginatedClients } from "../model/client.types";
import type { ClientResponseDto } from "./client.dto";

export function mapClientRequest(
    input: ClientResponseDto
): PaginatedClients {
    return input;
}
```

A mapper transforms transport data into an application/domain representation.

The mapper should be used when a transformation is needed.

Do not introduce mappings purely for ceremony.

Also ensure mapper names accurately describe their direction.

For example:

```text
mapClientResponse
mapClientRequest
mapClientToDto
mapDtoToClient
```

should reflect what the function actually does.

---

# 10. Entity model

The `model/` directory contains the entity's application/domain representation and entity-level query configuration.

Example:

```text
entities/client/model/
├── client.constants.ts
├── client.query.ts
└── client.types.ts
```

---

# 11. Entity constants

Example:

```text
client.constants.ts
```

```ts
export const clientTypeOptions = [
    "individual",
    "business"
] as const;

export const clientStatusOptions = [
    "active",
    "lead",
    "archived"
] as const;
```

Constants that define meaningful Client domain values belong with the Client entity.

This avoids scattering domain-specific options throughout components.

Do not move Client-specific constants into `shared/` merely because they are constants.

`shared/` is for generic functionality, not for all reusable values.

---

# 12. Entity types and schemas

Example:

```text
client.types.ts
```

```ts
export const clientItemSchema = z.object({
    id: z.string().uuid(),
    displayName: z.string().min(1),
    type: z.enum(clientTypeOptions),
    email: z.string().email().optional(),
    phone: z.string().optional(),
    status: z.enum(clientStatusOptions),
    createdAtUtc: z.string().datetime()
});

export type ClientItem =
    z.infer<typeof clientItemSchema>;
```

The entity model can use Zod as both:

* runtime validation
* TypeScript type generation

This is the preferred pattern where appropriate.

The entity model may also contain collection models:

```ts
export const paginatedClientSchema = z.object({
    items: z.array(clientItemSchema),
    total: z.number(),
    page: z.number(),
    pageSize: z.number(),
    sortBy: z.string().optional(),
    sortDirection: z.enum(["asc", "desc"]).optional(),
    type: z.array(z.enum(["individual", "business"])).optional(),
    status: z.array(z.enum(["active", "lead", "archived"])).optional(),
    search: z.string().optional()
});

export type PaginatedClients =
    z.infer<typeof paginatedClientSchema>;
```

The model represents how the frontend understands the entity.

---

# 13. Entity query definitions

Example:

```text
entities/client/model/client.query.ts
```

```ts
export const clientQuery = {
    root: ["clients"] as const,

    all: (params: ClientSearch) => ({
        queryKey: ["clients", params],
        queryFn: () => getClients(params)
    })
};
```

This defines the React Query/query configuration for retrieving Client data.

It is distinct from the React hook.

The distinction is:

```text
client.query.ts
    ↓
query definition/configuration

use-client.query.ts
    ↓
React integration
```

Basic entity retrieval belongs here.

For example:

```text
get Client
get Clients
```

should normally be entity queries rather than features called:

```text
features/client/get
features/client/get-all
```

---

# 14. Entity query hooks

Example:

```text
entities/client/queries/use-client.query.ts
```

```ts
export function useClientQuery() {
    const { rawSearch } = useClientSearchParams();

    return useQuery({
        ...clientQuery.all(rawSearch),
        placeholderData: keepPreviousData
    });
}
```

The hook is the React-facing adapter around the entity query.

It can combine:

* React Query
* entity query definitions
* entity search state
* React-specific behaviour

It should not contain unrelated application mutation logic.

---

# 15. Entity-specific search

An entity can own its collection search state.

Example:

```text
entities/client/search/
├── client-search.schema.ts
└── use-client-search-actions.ts
```

This is an important part of this application's architecture.

`search/` is a valid entity responsibility when filtering, sorting, pagination or URL search parameters are intrinsic to an entity collection.

For Client:

```text
ClientSearch
├── page
├── pageSize
├── sortBy
├── sortDir
├── search
├── type
└── status
```

This is not generic shared state.

It belongs to Client because it describes how Clients are searched and displayed.

---

# 16. Search schemas

Example:

```text
entities/client/search/client-search.schema.ts
```

```ts
export const clientSearchSchema = z.object({
    page: z.number().default(DEFAULT_PAGE).optional(),
    pageSize: z.number().default(DEFAULT_PAGE_SIZE).optional(),
    sortBy: z.string().default(DEFAULT_SORT_BY).optional(),
    sortDir: z.enum(["asc", "desc"])
        .default(DEFAULT_SORT_DIR)
        .optional(),
    search: z.string().optional(),
    type: z.array(
        z.enum(["individual", "business"])
    ).optional(),
    status: z.array(
        z.enum(["active", "lead", "archived"])
    ).optional()
});
```

This defines the shape and validation/defaults for Client search state.

It may be used in conjunction with route search parameters.

---

# 17. Entity search actions

Example:

```text
entities/client/search/use-client-search-actions.ts
```

This hook encapsulates interaction with Client search state.

It can own behaviour such as:

* debounced keyword search
* updating URL search parameters
* resetting pagination
* preserving existing search parameters
* navigation transitions

The important architectural distinction is:

```text
search schema
    ↓
defines search state

search actions hook
    ↓
changes search state

entity query
    ↓
uses search state to retrieve data
```

These are separate responsibilities.

Do not move Client-specific search behaviour into `shared/` merely because other entities might eventually have similar behaviour.

Only truly generic search infrastructure belongs in `shared/`.

---

# 18. Entity UI

Example:

```text
entities/client/ui/client-rows-per-page.tsx
```

Entity UI contains small UI components tightly coupled to the entity or its interaction model.

It does not have to literally render a Client.

For example:

```text
ClientRowsPerPage
ClientStatusBadge
ClientTypeLabel
ClientName
```

can belong under:

```text
entities/client/ui/
```

if they are tightly coupled to Client-specific representation or collection interaction.

The distinction is:

```text
small entity-specific UI
        ↓
entities/<entity>/ui

substantial application composition
        ↓
widgets/

application workflow/action UI
        ↓
features/<domain>/<action>/ui
```

---

# 19. Features

Features represent application capabilities/use cases.

A feature is primarily a **verb** or action.

Examples:

```text
Register
Create Client
Update Client
Delete Client
Assign Resource
Change Plan
Archive Project
```

The feature boundary is determined by the capability, not by the UI component.

A feature may contain:

* API operations
* request/response DTOs
* mappers
* input schemas
* feature-specific types
* use cases
* React Query mutations
* feature UI

---

# 20. Feature grouping

Features may be grouped by domain when there are multiple related capabilities.

For example:

```text
features/
├── client/
│   ├── create/
│   ├── update/
│   └── delete/
│
├── forecast/
│   ├── create/
│   ├── update/
│   └── compare/
│
└── resource/
    ├── assign/
    └── remove/
```

However, the actual existing application may also contain feature directories directly under `features/`:

```text
features/
├── register/
├── create-client/
├── update-client/
└── delete-client/
```

Do not restructure existing features simply to satisfy the theoretical grouped structure.

The important concept is:

```text
features/<domain>/<capability>
```

where grouping is useful.

The domain grouping does **not** mean the feature belongs inside the entity.

These are separate architectural boundaries:

```text
entities/client/

features/client/register/
```

The repeated `client` name is intentional.

---

# 21. Feature internal structure

A substantial feature can contain responsibility-specific directories:

```text
features/register/
├── api/
│   ├── register.dto.ts
│   └── register.mapper.ts
│
├── model/
│   ├── register.schema.ts
│   └── register.use-case.ts
│
├── hooks/
│   └── register.mutation.ts
│
├── ui/
│   └── register-form.tsx
│
└── index.ts
```

Not every feature needs every directory.

Do not create empty `api`, `model`, `hooks` or `ui` directories merely because they are part of the general convention.

The directories communicate responsibility.

---

# 22. Feature API DTOs

Example:

```text
features/register/api/register.dto.ts
```

```ts
export interface RegisterResponseDto {
    user: {
        id: string;
        email: string;
    };
}

export interface RegisterRequestDto {
    firstName: string;
    lastName: string;
    email: string;
    password: string;
}
```

DTOs in a feature describe the transport contract of that feature's operation.

This is different from an entity DTO.

For example:

```text
entities/client/api/client.dto.ts
```

describes Client retrieval/representation at the entity boundary.

Whereas:

```text
features/register/api/register.dto.ts
```

describes the Register operation.

A DTO belongs to the slice that owns the operation it represents.

---

# 23. Feature API mappers

Example:

```text
features/register/api/register.mapper.ts
```

```ts
export function mapRegisterRequest(
    input: RegisterInput
): RegisterRequestDto {
    return {
        firstName: input.firstName,
        lastName: input.lastName,
        email: input.email,
        password: input.password
    };
}
```

The mapper converts the feature's application input into the API request DTO.

The intended flow is:

```text
RegisterInput
     ↓
mapRegisterRequest()
     ↓
RegisterRequestDto
     ↓
API
```

This prevents API transport concerns from leaking into the form/model.

---

# 24. Feature model schemas

Example:

```text
features/register/model/register.schema.ts
```

```ts
export const registerSchema = z.object({
    firstName: z.string().nonempty("Required"),
    lastName: z.string().nonempty("Required"),
    businessName: z.string().nonempty("Required"),
    workspaceType: workspaceTypeSchema,
    email: z.string()
        .email("Invalid email address")
        .nonempty("Required"),
    password: z.string()
        .min(8, "Password must be at least 8 characters"),
    confirmPassword: z.string()
        .nonempty("Confirm your password")
})
.refine(
    data => data.password === data.confirmPassword,
    {
        message: "Passwords do not match",
        path: ["confirmPassword"]
    }
);

export type RegisterInput =
    z.infer<typeof registerSchema>;
```

This schema belongs to the feature because it validates **input to the Register use case**.

Do not put registration-specific validation into an entity schema.

For example:

```text
workspaceTypeSchema
```

may belong to:

```text
entities/workspace
```

because Workspace Type is a domain concept.

But:

```text
confirmPassword
```

belongs to:

```text
features/register
```

because it only exists as part of the registration workflow.

This is an important distinction.

---

# 25. Features may depend on entities

The Register feature imports:

```ts
import { workspaceTypeSchema } from "@/entities/workspace";
```

and:

```ts
import { authApi } from "@/entities/auth";
```

This is a good example of the intended relationship:

```text
features/register
       ↓
entities/workspace
entities/auth
```

The feature composes existing domain capabilities.

The Workspace entity owns the Workspace concept and its schema.

The Auth entity owns the authentication API boundary.

The Register feature orchestrates them for the specific registration workflow.

---

# 26. Feature use cases

Example:

```text
features/register/model/register.use-case.ts
```

```ts
import { mapRegisterRequest } from "../api/register.mapper";
import type { RegisterInput } from "./register.schema";
import { authApi } from "@/entities/auth";

export async function registerUseCase(
    input: RegisterInput
) {
    const mappedRequest = mapRegisterRequest(input);

    const result = await authApi.register(mappedRequest);

    return result;
}
```

A use case represents application behaviour.

It should answer:

> "What does the application do when this capability is executed?"

The use case can:

* validate/accept application input
* map application input into transport input
* call entity APIs
* coordinate multiple entities
* perform application-level orchestration
* return the result

It should not contain React component code.

It should not use React hooks.

It should not directly manipulate UI.

---

# 27. Important distinction: use case vs mutation hook

These are deliberately separate.

```text
register.use-case.ts
```

contains the application operation.

```text
register.mutation.ts
```

contains React Query integration.

The flow is:

```text
RegisterForm
      ↓
useRegisterMutation()
      ↓
registerUseCase()
      ↓
mapRegisterRequest()
      ↓
authApi.register()
      ↓
HTTP
```

This keeps React-specific concerns outside the use case.

---

# 28. Feature mutation hooks

Example:

```text
features/register/hooks/register.mutation.ts
```

```ts
export function useRegisterMutation() {
    const router = useRouter();

    return useMutation({
        mutationFn: registerUseCase,

        onSuccess: () => {
            router.navigate({ to: "/" });
        },

        onError: () => {
            toast.error(
                "There has been an unexpected error. Please try again."
            );
        }
    });
}
```

This layer owns React/application integration such as:

* React Query mutations
* navigation
* toast notifications
* mutation lifecycle
* cache invalidation
* UI-facing mutation state

The hook should delegate the actual application operation to the use case.

---

# 29. Feature UI

Example:

```text
features/register/ui/register-form.tsx
```

The Register form owns the UI for the Register capability.

It uses:

```text
registerSchema
RegisterInput
useRegisterMutation
```

and generic shared UI components.

The form is therefore responsible for:

* rendering fields
* wiring React Hook Form
* displaying validation errors
* submitting the feature
* displaying pending state

It should not know:

* the HTTP endpoint
* request DTO structure
* how authentication is implemented
* how API requests are made

Those responsibilities belong elsewhere.

---

# 30. Entity UI can be consumed by features

The Register form uses:

```ts
import { WorkSpaceTypeSelect } from "@/entities/workspace";
```

This is a good example of entity UI being reused by a feature.

The Workspace entity owns:

```text
Workspace
WorkspaceType
WorkspaceTypeSelect
workspaceTypeSchema
```

The Register feature uses those concepts to construct its workflow.

Therefore:

```text
features/register
       ↓
entities/workspace
```

is correct.

Do not move `WorkSpaceTypeSelect` into Register merely because Register happens to use it.

If the component represents Workspace-specific UI and can be used by multiple features, it belongs to the Workspace entity.

---

# 31. Feature UI vs Widget UI

A useful distinction is:

### Feature UI

UI directly implementing a specific capability:

```text
features/register/ui/register-form.tsx
features/client/update/ui/update-client-form.tsx
features/client/delete/ui/delete-client-dialog.tsx
```

### Widget UI

Larger application composition:

```text
widgets/client-list/
widgets/client-details/
widgets/forecast-grid/
```

A feature form answers:

> "How does the user perform this action?"

A widget answers:

> "How is this substantial part of the application presented and composed?"

---

# 32. Entity query vs feature use case

Do not equate HTTP verbs with architecture.

Bad rule:

```text
GET    = entity
POST   = feature
PUT    = feature
DELETE = feature
```

Better rule:

### Basic entity retrieval

```text
entities/client
    ↓
queries
```

Examples:

```text
useClientQuery()
useClientsQuery()
```

These answer:

> "Give me Client data."

### Application capability

```text
features/client/update
features/client/register
features/client/compare
```

These answer:

> "Perform this meaningful application operation."

A feature can technically be read-only.

For example:

```text
features/forecast/compare
```

could perform only GET requests while still being a feature because "Compare Forecasts" is an application capability.

Do not create trivial features such as:

```text
features/client/get
features/client/get-all
```

just to wrap basic entity queries.

---

# 33. Import boundaries

Import from slice public APIs wherever a public API exists.

Prefer:

```ts
import { Client } from "@/entities/client";
import { useClientQuery } from "@/entities/client";

import { RegisterForm } from "@/features/register";

import { ForecastGrid } from "@/widgets/forecast-grid";

import { Button } from "@workspace/ui/components/button";
```

Avoid deep imports from another slice:

```ts
import { ClientItem } from "@/entities/client/model/client.types";
import { registerUseCase } from "@/features/register/model/register.use-case";
```

unless there is a deliberate reason to access an internal implementation.

Each slice should expose its public API through:

```text
index.ts
```

---

# 34. Slice `index.ts`

For example:

```text
features/register/index.ts
```

might expose:

```ts
export { RegisterForm } from "./ui/register-form";
export { useRegisterMutation } from "./hooks/register.mutation";
export type { RegisterInput } from "./model/register.schema";
```

The purpose is to provide a stable public boundary.

Consumers should not need to know whether the feature internally uses:

```text
model/
hooks/
api/
ui/
```

The same principle applies to entities.

---

# 35. File naming conventions

Use filenames that communicate responsibility.

Examples:

```text
client.api.ts
client.dto.ts
client.mapper.ts
client.constants.ts
client.types.ts
client.query.ts

client-search.schema.ts
use-client-search-actions.ts
use-client.query.ts

register.dto.ts
register.mapper.ts
register.schema.ts
register.use-case.ts
register.mutation.ts
register-form.tsx
```

Suffixes are intentional.

### `.api.ts`

Transport/API interaction.

### `.dto.ts`

Transport/API data contract.

### `.mapper.ts`

Transformation between representations.

### `.schema.ts`

Runtime validation/schema definition.

### `.types.ts`

Domain/application TypeScript types.

### `.constants.ts`

Domain-specific constants/options.

### `.query.ts`

Query configuration/definition.

### `.use-case.ts`

Application operation/orchestration independent of React.

### `.mutation.ts`

React Query mutation integration.

### `use-*.ts`

React hooks.

### `.tsx`

UI components.

Avoid vague names such as:

```text
utils.ts
helpers.ts
service.ts
manager.ts
common.ts
misc.ts
```

unless the responsibility is genuinely generic.

---

# 36. Do not confuse folders with architectural layers

The following:

```text
entities/client/
├── api/
├── model/
├── queries/
├── search/
└── ui/
```

does not mean:

```text
API layer
Model layer
Query layer
Search layer
UI layer
```

are globally shared architectural layers.

They are **local responsibilities within the Client slice**.

The same responsibility directories may legitimately appear in different slices:

```text
entities/client/api/
features/client/register/api/

entities/client/model/
features/client/register/model/

entities/client/ui/
features/client/register/ui/
```

The parent slice determines ownership.

This is intentional.

---

# 37. Ownership beats subject matter

This is one of the most important rules.

Do not move code based only on the entity it operates on.

For example:

```text
Register Client
```

does not belong under:

```text
entities/client/register
```

just because it creates a Client.

It belongs under:

```text
features/client/register
```

because registration is an application capability.

Likewise:

```text
Update Forecast
```

belongs under:

```text
features/forecast/update
```

rather than:

```text
entities/forecast/update
```

The entity and feature are separate ownership boundaries.

---

# 38. Entity vs feature example

Consider:

```text
entities/client/
```

This answers:

```text
What is a Client?
How do I retrieve Clients?
How do I represent Clients?
How do I search/filter Clients?
How do I display small Client-specific UI?
```

Whereas:

```text
features/client/register/
```

answers:

```text
How does the application register a Client?
```

Therefore:

```text
entities/client
       ↑
       │ used by
       │
features/client/register
```

not:

```text
entities/client
└── register
```

---

# 39. Concrete Register flow

The Register feature demonstrates the intended separation particularly well.

```text
┌───────────────────────────────┐
│ register-form.tsx             │
│                               │
│ React Hook Form               │
│ UI                            │
└───────────────┬───────────────┘
                │
                ↓
┌───────────────────────────────┐
│ register.mutation.ts          │
│                               │
│ React Query                   │
│ navigation                    │
│ toast/error handling          │
└───────────────┬───────────────┘
                │
                ↓
┌───────────────────────────────┐
│ register.use-case.ts          │
│                               │
│ application orchestration     │
└───────────────┬───────────────┘
                │
        ┌───────┴────────┐
        ↓                ↓
┌──────────────┐  ┌───────────────┐
│ register     │  │ entities/auth │
│ mapper       │  │ authApi       │
└──────┬───────┘  └───────┬───────┘
       │                   │
       ↓                   ↓
RegisterRequestDto        HTTP
```

The important boundary is:

```text
UI
 ↓
React adapter
 ↓
Use case
 ↓
API/Entity
```

---

# 40. Concrete Client retrieval flow

```text
useClientQuery()
       ↓
clientQuery.all()
       ↓
getClients()
       ↓
clientApi.all()
       ↓
apiClient.get()
       ↓
HTTP
```

Search state participates in the flow:

```text
URL/search params
       ↓
useClientSearchParams()
       ↓
useClientQuery()
       ↓
clientQuery.all(rawSearch)
       ↓
clientApi.all(params)
```

This is an entity query rather than a feature.

---

# 41. Dependency rules

Preferred dependency direction:

```text
routes
  ↓
widgets
  ↓
features
  ↓
entities
  ↓
shared
```

But these are not rigid parent-child relationships.

In practice:

```text
routes
   ↓
widgets ─────→ features
   │              ↓
   └──────────→ entities
                  ↓
                shared

features ──────→ entities
features ──────→ shared
entities ──────→ shared
widgets ───────→ shared
routes ────────→ shared
```

Important rules:

1. Features may depend on entities.
2. Widgets may depend on entities and features.
3. Routes may compose widgets, features and entities.
4. Entities should not depend on features.
5. Shared should not depend on application-specific entities/features/widgets.
6. Do not move code between boundaries merely to eliminate a repeated concept/name.

---

# 42. Shared

`shared/` contains genuinely generic functionality.

Examples:

```text
shared/
├── api/
├── lib/
├── hooks/
└── ...
```

Examples of appropriate shared code:

```text
apiClient
generic URL/search utilities
generic date utilities
generic UI
generic hooks
generic infrastructure
```

Do not put domain-specific code into Shared simply because it is reused.

For example:

```text
clientStatusOptions
workspaceTypeSchema
forecastAggregationType
```

are domain-specific and should remain with their respective entity/domain.

---

# 43. Avoid premature abstraction

Do not create abstractions solely because two files look similar.

For example, if both:

```text
entities/client/
features/register/
```

contain:

```text
api/
model/
ui/
```

that does not mean these folders should be unified.

The duplication is architectural separation.

Likewise, do not create:

```text
shared/services/
shared/models/
shared/types/
```

as generic dumping grounds.

Move something into Shared only when it is genuinely domain-independent.

---

# 44. Do not over-engineer small slices

A small feature can be:

```text
features/delete-client/
├── model/
│   └── delete-client.use-case.ts
├── hooks/
│   └── delete-client.mutation.ts
└── index.ts
```

It does not necessarily need:

```text
api/
model/
hooks/
ui/
types/
schemas/
mappers/
```

Create files and directories based on actual responsibilities.

The architecture provides boundaries; it does not require boilerplate.

---

# 45. Do not flatten substantial slices

Conversely, do not turn a substantial feature into:

```text
features/register/
├── register.ts
├── registerApi.ts
├── registerTypes.ts
├── registerForm.tsx
└── registerHook.ts
```

when the feature has clear responsibilities.

Prefer:

```text
features/register/
├── api/
├── model/
├── hooks/
├── ui/
└── index.ts
```

The folder structure should make the responsibility of each file immediately apparent.

---

# 46. Decision rules for placing new code

When adding a new file, ask:

### 1. What concept does this belong to?

If it represents a domain concept:

```text
entities/
```

### 2. What capability does this implement?

If it performs an application operation:

```text
features/
```

### 3. Is it substantial application-specific UI?

```text
widgets/
```

### 4. Is it responsible for routing/navigation?

```text
routes/
```

### 5. Is it genuinely domain-independent?

```text
shared/
```

Then determine the responsibility inside that slice:

```text
API transport      → api/
DTO                → api/*.dto.ts
mapping            → api/*.mapper.ts
domain types       → model/*.types.ts
domain constants   → model/*.constants.ts
runtime schema     → model/*.schema.ts
query definition   → model/*.query.ts
React query hook   → queries/ or hooks/
search state       → search/
use-case           → model/*.use-case.ts
mutation hook      → hooks/*.mutation.ts
feature UI         → ui/
entity UI          → ui/
```

---

# 47. Most important instructions for an LLM

When modifying this codebase:

1. **Preserve the existing architectural boundaries.**
2. **Do not move code into an entity merely because it operates on that entity.**
3. **Do not move feature code into entities.**
4. **Features represent application capabilities/use cases.**
5. **Entities represent domain concepts and their standard data access/representation.**
6. **Basic entity queries belong to entities.**
7. **Meaningful application-level read operations may still be features.**
8. **Do not create trivial `get-*` features for ordinary entity retrieval.**
9. **Use `api/`, `model/`, `queries/`, `search/`, `hooks/`, and `ui/` as responsibility folders where needed.**
10. **Do not assume every slice requires every folder.**
11. **Repeated folders such as `api/`, `model/`, and `ui/` across entities and features are intentional.**
12. **The parent slice establishes ownership.**
13. **Use `.api.ts`, `.dto.ts`, `.mapper.ts`, `.schema.ts`, `.types.ts`, `.constants.ts`, `.query.ts`, `.use-case.ts`, and `.mutation.ts` to communicate responsibility.**
14. **Use React hooks for React/React Query integration; keep use cases independent of React.**
15. **Use entity schemas for entity/domain concepts and feature schemas for feature-specific input/workflow validation.**
16. **A feature may depend on one or more entities.**
17. **An entity should not depend on a feature.**
18. **Prefer imports through a slice's `index.ts` public API.**
19. **Do not introduce global barrels such as `entities/index.ts` or `features/index.ts` merely for convenience.**
20. **Do not introduce broad architectural refactors while implementing an unrelated feature.**
21. **Avoid generic dumping grounds such as `utils`, `helpers`, `services`, or `common` when a more precise ownership boundary exists.**
22. **Do not create abstractions solely to eliminate small amounts of duplication if doing so weakens architectural ownership.**
23. **Do not infer that a directory name repeated in two slices represents the same layer. `entities/client/model` and `features/client/register/model` are separate ownership boundaries.**
24. **When adding code, follow the closest existing pattern in the same architectural slice.**
25. **Prefer minimal, typed, focused changes over broad restructuring.**

---

# 48. Canonical mental model

The architecture can be reduced to the following:

```text
                    APPLICATION
                         │
          ┌──────────────┴──────────────┐
          ↓                             ↓
       WIDGETS                       FEATURES
          │                             │
          │                    "What does the app do?"
          │                             │
          │                             ↓
          └──────────────────────→ ENTITIES
                                       │
                              "What does the app know?"
                                       │
                                       ↓
                                    SHARED
```

Or, even more simply:

```text
Entity  = noun
Feature = verb
Widget  = substantial UI composition
Route   = screen/URL composition
Shared  = generic infrastructure
```

The architecture should be understood through **ownership and responsibility**, not merely through folder names.
