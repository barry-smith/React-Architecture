# React-Architecture

The key idea is:

> **Entities describe what the application knows about. Features describe what the application does.**

That distinction gives us a useful foundation for everything else.

# Frontend Architecture Context

## 1. Architectural goals

This React application uses a **feature-oriented architecture** designed to keep business capabilities cohesive and prevent the codebase from degenerating into large global folders such as:

```text
components/
hooks/
services/
utils/
types/
```

Those folders tend to organise code according to *technical implementation details* rather than according to *why the code exists*.

Instead, the architecture primarily organises code around:

1. **Entities** — important concepts in the application's domain.
2. **Features** — discrete capabilities/use cases the user or application can perform.
3. **Shared infrastructure** — genuinely reusable code that has no particular domain ownership.
4. **Application/page composition** — code responsible for bringing features and entities together into screens.

The architecture should make it possible to answer:

> "Why does this code exist?"

by looking at where it lives.

For example:

```text
entities/forecast
```

should contain code because the application has a **Forecast** concept.

Whereas:

```text
features/update-forecast
```

exists because the application allows a user to **update a forecast**.

These are deliberately different concepts.

---

# 2. The high-level structure

A representative structure is:

```text
src/
│
├── app/
│
├── entities/
│   ├── project/
│   ├── forecast/
│   ├── resource/
│   └── ...
│
├── features/
│   ├── create-project/
│   ├── update-project/
│   ├── update-forecast/
│   ├── assign-resource/
│   └── ...
│
├── shared/
│   ├── ui/
│   ├── lib/
│   ├── hooks/
│   └── ...
│
└── pages/
```

The exact names aren't important. The **ownership model** is.

A useful mental model is:

```text
                    Application
                         │
                         ▼
                       Pages
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
          Features                Entities
             │                       │
             └───────────┬───────────┘
                         ▼
                       Shared
```

This isn't necessarily a rigid runtime dependency diagram. It is primarily a way of establishing **ownership and dependency direction**.

---

# 3. Entities

An **entity** represents a meaningful concept in the domain.

Examples:

```text
Project
Forecast
Resource
Plan
User
Customer
Task
```

An entity generally contains things that are fundamentally about that concept.

For example:

```text
entities/
└── forecast/
    ├── api/
    ├── model/
    ├── queries/
    └── ui/
```

The important question is:

> "Would this code still make sense if we removed all of the application's individual use cases?"

If yes, it is potentially entity-level code.

For example:

```ts
type Forecast = {
  id: string
  name: string
  startDate: string
  endDate: string
  status: ForecastStatus
}
```

belongs naturally to the Forecast entity.

Likewise:

```ts
const forecastQueryKey = (id: string) =>
  ['forecast', id] as const
```

could belong to Forecast because it describes how the application identifies Forecast data.

---

# 4. Entities aren't database tables

This distinction is important.

An entity isn't simply:

> "A TypeScript representation of a backend database table."

The frontend entity represents a **concept that the frontend needs to understand**.

The backend may have:

```text
Forecast
ForecastVersion
ForecastResource
ForecastResourceAllocation
ForecastRevision
```

while the frontend may conceptualise these as:

```text
Forecast
```

with nested or derived data.

Conversely, a backend object may not deserve its own frontend entity if the frontend never treats it as an independent concept.

The frontend architecture should therefore be driven by the application's **domain model**, not mechanically copied from the API.

---

# 5. What belongs inside an entity?

Potentially:

### Types

```ts
Forecast
ForecastStatus
ForecastId
```

### API/data access

```ts
getForecast()
getForecasts()
```

### Queries

```ts
useForecast()
useForecasts()
```

### Entity-specific transformations

```ts
toForecastViewModel()
```

### Entity-specific UI

For example:

```text
entities/forecast/ui/ForecastStatusBadge
entities/forecast/ui/ForecastName
```

provided that the UI genuinely represents the Forecast itself.

The important distinction is that an entity shouldn't become a dumping ground for every piece of functionality involving that entity.

---

# 6. Features

Features represent **things the application can do**.

This is probably the most important concept in the architecture.

For example:

```text
features/
├── create-project
├── edit-project
├── delete-project
├── create-forecast
├── update-forecast
├── assign-resource
└── change-forecast-plan
```

These aren't entities.

They're **use cases**.

A feature answers:

> "What action is being performed?"

rather than:

> "What thing are we talking about?"

---

# 7. Feature = use case

A feature should generally correspond to a meaningful application capability.

For example:

```text
features/update-forecast
```

might contain:

```text
update-forecast/
├── api/
├── model/
├── ui/
└── index.ts
```

The feature could expose something like:

```ts
useUpdateForecast()
```

and internally:

```ts
mutationFn: updateForecast
```

The important point is that the mutation isn't just:

> "An HTTP PUT."

It represents:

> "The application can update a Forecast."

That gives the code a meaningful boundary.

---

# 8. Why not just put everything under `forecast`?

You might initially think:

```text
entities/
└── forecast/
    ├── create.ts
    ├── update.ts
    ├── delete.ts
    ├── assign-resource.ts
    └── ...
```

This looks attractive because everything concerning Forecast is together.

But it eventually creates a problem.

The Forecast entity becomes responsible for:

* displaying forecasts
* creating forecasts
* editing forecasts
* deleting forecasts
* importing forecasts
* exporting forecasts
* assigning resources
* changing plans
* comparing forecasts
* approving forecasts

The entity stops representing the concept and starts representing **everything the application does with that concept**.

That's exactly what the feature boundary is intended to prevent.

Instead:

```text
entities/
└── forecast/

features/
├── create-forecast/
├── update-forecast/
├── delete-forecast/
├── assign-resource/
├── compare-forecasts/
└── approve-forecast/
```

Now the distinction is explicit.

---

# 9. Features can use entities

A feature will frequently depend on one or more entities.

For example:

```text
features/update-forecast
```

might use:

```text
entities/forecast
entities/project
entities/resource
```

because updating a forecast might involve all three.

Conceptually:

```text
             update-forecast
              /      |      \
             ▼       ▼       ▼
        Forecast   Project   Resource
```

The reverse dependency should generally be avoided.

An entity shouldn't need to know about a feature.

For example, this is undesirable:

```text
entities/forecast
    ↓
features/update-forecast
```

because then the Forecast entity is coupled to one particular operation.

The entity should remain useful independently.

---

# 10. Features should represent cohesive behaviour

Not every function deserves a feature.

For example:

```ts
formatForecastDate()
```

is not a feature.

Neither is:

```ts
calculateForecastVariance()
```

necessarily a feature.

Nor:

```text
ForecastTable
```

by itself.

A feature should represent **meaningful behaviour**.

A useful test is:

> Could I describe this as something a user or application can do?

For example:

> "Update the forecast."

Yes.

> "Assign a resource to a project."

Yes.

> "Change the forecast plan."

Yes.

> "Format a date."

No.

---

# 11. Features aren't necessarily screens

This is another important distinction.

A feature isn't synonymous with:

```text
pages/
```

or:

```text
components/
```

A feature can provide UI, but its boundary is determined by **behaviour**, not by visual composition.

For example:

```text
features/update-forecast/
    ui/
        UpdateForecastButton.tsx
        UpdateForecastDialog.tsx
    model/
        useUpdateForecast.ts
    api/
        updateForecast.ts
```

The UI is merely one way of exposing the use case.

The feature boundary remains:

> Update Forecast.

---

# 12. A feature may contain multiple UI components

There's no requirement that a feature be one component.

For example:

```text
features/assign-resource/
├── ui/
│   ├── AssignResourceButton.tsx
│   ├── AssignResourceDialog.tsx
│   ├── ResourceSelector.tsx
│   └── AssignedResourceList.tsx
│
├── model/
│   └── useAssignResource.ts
│
└── api/
    └── assignResource.ts
```

These components belong together because they implement one capability.

That is much more meaningful than putting them into:

```text
components/buttons
components/dialogs
components/selects
```

where the architectural relationship between them disappears.

---

# 13. React Query

React Query is primarily an **application data-access mechanism**, not a domain concept.

A useful separation is:

```text
Entity
    ↓
data/query definition

Feature
    ↓
mutation/use case
```

For example, fetching a Forecast:

```text
entities/forecast
    └── queries/
        └── useForecast.ts
```

makes sense because retrieving a Forecast is fundamentally about obtaining Forecast data.

Whereas:

```text
features/update-forecast
    └── model/
        └── useUpdateForecast.ts
```

represents an operation.

Conceptually:

```text
GET Forecast
    → entity

UPDATE Forecast
    → feature/use case
```

This isn't an absolute law, but it is a useful default.

---

# 14. Queries vs mutations

A useful rule is:

### Queries

Queries tend to belong with the entity because they answer:

> "What is this thing?"

For example:

```ts
useForecast(id)
useForecasts(projectId)
```

### Mutations

Mutations frequently belong with features because they answer:

> "What are we doing?"

For example:

```ts
useCreateForecast()
useUpdateForecast()
useDeleteForecast()
```

So:

```text
entities/forecast
    └── queries/
        ├── useForecast
        └── useForecasts

features/
├── create-forecast
│   └── useCreateForecast
│
├── update-forecast
│   └── useUpdateForecast
│
└── delete-forecast
    └── useDeleteForecast
```

This makes the application's capabilities discoverable.

---

# 15. But don't blindly follow the rule

Architecture shouldn't become ceremony.

If a tiny application has:

```ts
useUpdateForecast()
```

and there is no meaningful feature boundary around it, splitting it across six directories may make the code harder to understand.

The architecture should optimise for **cohesion and discoverability**, not maximum directory count.

The question is always:

> Does this boundary make the code easier to reason about?

---

# 16. Shared

`shared/` is for code that genuinely has **no domain ownership**.

Examples:

```text
shared/
├── ui/
│   ├── Button
│   ├── Dialog
│   └── Input
│
├── lib/
│   ├── date
│   ├── currency
│   └── formatting
│
└── hooks/
    └── useDebounce
```

The important word is **genuinely**.

`shared` should not mean:

> "I don't know where this goes."

Nor:

> "Several things currently import this."

---

# 17. Shared is not a dumping ground

A common failure mode is:

```text
shared/
├── utils.ts
├── helpers.ts
├── api.ts
├── hooks.ts
└── constants.ts
```

Eventually everything ends up there.

Instead, ask:

> "Who owns this?"

If the answer is Forecast:

```text
entities/forecast
```

If the answer is Update Forecast:

```text
features/update-forecast
```

If it genuinely has no domain owner:

```text
shared
```

---

# 18. Reuse doesn't automatically mean shared

This is particularly important.

Imagine:

```ts
ForecastStatusBadge
```

is used in five places.

That doesn't necessarily mean it belongs in:

```text
shared/ui
```

It may still belong to:

```text
entities/forecast/ui
```

because it is specifically a Forecast UI concept.

**Reuse is not the same thing as ownership.**

A component can be reused extensively while remaining owned by an entity.

---

# 19. Pages / application composition

Pages are where the application starts assembling things.

For example:

```tsx
<ForecastPage>
  <ForecastHeader />
  <ForecastGrid />
  <UpdateForecastButton />
</ForecastPage>
```

The page shouldn't contain all the implementation details of:

* updating forecasts
* loading forecasts
* assigning resources
* calculating forecast state

Instead, it composes the appropriate entities and features.

Conceptually:

```text
Page
 ├── Forecast entity
 ├── Update Forecast feature
 ├── Assign Resource feature
 └── Shared UI
```

The page is therefore mostly **composition**.

---

# 20. A concrete Forecast example

Suppose we have:

```text
Forecast
Project
Resource
```

and the user can:

1. View a Forecast
2. Edit Forecast values
3. Assign Resources
4. Change the Forecast Plan
5. Compare against a baseline

We could model that as:

```text
entities/
├── forecast/
├── project/
└── resource/

features/
├── update-forecast/
├── assign-resource/
├── change-forecast-plan/
└── compare-forecast/
```

The Forecast entity might contain:

```ts
type Forecast = {
  id: string
  name: string
  planId: string
  status: ForecastStatus
}
```

The update feature might contain:

```ts
type UpdateForecastInput = {
  forecastId: string
  changes: ForecastChanges
}
```

The feature then owns the operation:

```ts
useUpdateForecast()
```

The entity owns the concept:

```ts
Forecast
```

That's the core distinction.

---

# 21. Where the Forecast Grid fits

This becomes particularly useful with something like the AG Grid `ForecastGrid` we were discussing.

The grid itself isn't necessarily an entity.

Nor is it necessarily a feature.

It depends on what it represents.

If it's primarily a reusable representation of Forecast data:

```text
entities/forecast/ui/ForecastGrid
```

may be appropriate.

If the grid implements a specific use case such as:

> "Edit the project forecast"

then it may belong to:

```text
features/update-forecast/ui/
```

The distinction is:

```text
ForecastGrid
    = representation of Forecast data

EditableForecastGrid
    = UI for the Update Forecast use case
```

This is exactly the kind of decision the architecture is intended to make clearer.

---

# 22. Derived data

Not everything returned by the backend needs to become an entity.

For example, suppose the API returns:

```ts
{
  forecast: ...,
  baseline: ...,
  deltas: ...
}
```

and the frontend needs to display:

```text
Forecast | Baseline | Variance
```

The derived representation might belong to the feature consuming it rather than becoming:

```text
entities/delta
```

simply because there happens to be a `deltas` property.

Again:

> Domain concepts determine entities, not API response shapes.

---

# 23. Dependency direction

A good default is:

```text
app/pages
     │
     ▼
 features
     │
     ▼
 entities
     │
     ▼
 shared
```

But there is some nuance.

A feature can use multiple entities:

```text
                 feature
               /    |    \
              ▼     ▼     ▼
          Project Forecast Resource
               \    |    /
                 shared
```

An entity can use shared infrastructure.

But:

```text
entity → feature
```

should generally be avoided.

And unrelated features shouldn't reach deeply into each other's internals.

Prefer:

```ts
import { useUpdateForecast } from '@/features/update-forecast'
```

over:

```ts
import { useUpdateForecast } from '@/features/update-forecast/model/internal/useUpdateForecast'
```

This is where public `index.ts` boundaries become useful.

---

# 24. Public APIs

Each entity/feature should ideally expose a deliberate public surface.

For example:

```text
features/update-forecast/
├── api/
├── model/
├── ui/
└── index.ts
```

Then:

```ts
export { UpdateForecastDialog } from './ui/UpdateForecastDialog'
export { useUpdateForecast } from './model/useUpdateForecast'
```

Consumers don't need to know the internal structure.

This allows us to refactor:

```text
model/
```

without changing every consumer.

---

# 25. Avoid deep imports

Prefer:

```ts
import { useUpdateForecast } from '@/features/update-forecast'
```

rather than:

```ts
import { useUpdateForecast }
  from '@/features/update-forecast/model/hooks/useUpdateForecast'
```

The former communicates:

> "I depend on the Update Forecast capability."

The latter communicates:

> "I depend on this particular implementation file."

That's an important architectural distinction.

---

# 26. The most important classification rules

When adding new code, an LLM should ask these questions **in this order**.

### Question 1 — What does this code represent?

Is it:

```text
domain concept → entity
user/application capability → feature
generic infrastructure → shared
screen composition → page/app
```

### Question 2 — Who owns it?

Don't ask:

> "Where is this type of file normally stored?"

Ask:

> "Which concept or capability owns this code?"

### Question 3 — Is this actually reusable?

Don't move something to `shared` merely because it has two consumers.

### Question 4 — Is this a use case?

If it represents an action such as:

```text
Create
Update
Delete
Assign
Import
Export
Approve
Compare
Publish
```

consider a feature.

### Question 5 — Does the dependency direction make sense?

A lower-level concept shouldn't depend on a higher-level use case.

---

# 27. What we are deliberately avoiding

The architecture is intended to avoid this:

```text
components/
├── ForecastGrid
├── ResourceSelector
├── ForecastDialog
└── ProjectHeader

hooks/
├── useForecast
├── useUpdateForecast
├── useResource
└── useProject

services/
├── forecastService
├── resourceService
└── projectService

types/
├── forecast
├── resource
└── project

utils/
├── forecast
└── project
```

The problem isn't that any individual folder is inherently bad.

The problem is that **the relationships between the code become implicit**.

You have to search the whole project to discover:

> What code constitutes "Update Forecast"?

The feature architecture makes that explicit:

```text
features/
└── update-forecast/
    ├── api/
    ├── model/
    ├── ui/
    └── index.ts
```

Now the answer is immediately visible.

---

# 28. The philosophy in one sentence

The architecture should follow this principle:

> **Organise code according to the concepts and capabilities of the application, rather than according to the technical type of the files.**

Therefore:

```text
Forecast
```

is an entity.

```text
Update Forecast
```

is a feature.

```text
Button
```

is shared UI.

```text
ForecastStatusBadge
```

is probably Forecast-owned UI.

```text
useUpdateForecast
```

belongs to the Update Forecast capability.

```text
useDebounce
```

is shared infrastructure.

```text
ForecastPage
```

is application/page composition.

---

# 29. LLM-specific rules

If this is being supplied to another LLM alongside the codebase, I'd explicitly give it these rules:

> **When modifying the frontend, do not default to creating generic `components`, `hooks`, `services`, `utils`, or `types` directories.**
>
> First determine the domain concept or application capability that owns the code.
>
> If the code represents a domain concept, place it under the appropriate entity.
>
> If it represents a user/application capability or use case, place it under the appropriate feature.
>
> Only place code under `shared` when it has no meaningful domain or feature ownership.
>
> Keep entities independent of specific features.
>
> Features may compose and depend on entities.
>
> Prefer public feature/entity APIs over deep imports.
>
> Do not create an entity merely because an API returns an object with that name.
>
> Do not create a feature merely because a component happens to perform an action.
>
> A feature should represent a cohesive capability/use case.
>
> Reuse does not automatically imply that code belongs in `shared`.
>
> Before creating a new abstraction, check whether an existing entity or feature already owns the concept.
>
> Prefer cohesive local code over premature extraction.
>
> When uncertain about ownership, explain the alternatives and choose the location that best represents the business/domain responsibility of the code.

---

## 30. The deeper idea

The architecture isn't really about folders.

The folders are simply a **physical manifestation of boundaries**.

The important boundaries are:

```text
             WHAT THINGS EXIST?
                    │
                    ▼
                 ENTITIES
                    │
                    │
             WHAT CAN WE DO?
                    │
                    ▼
                 FEATURES
                    │
                    │
             HOW DO WE COMPOSE IT?
                    │
                    ▼
               APPLICATION
```

That gives the codebase a vocabulary.

Instead of saying:

> "I need to add a hook and a service for this."

we want the developer/LLM to think:

> "I'm implementing the `Update Forecast` use case."

That shift is the real architectural benefit.

---

I think this is a good **v1 of the context**, but I would *not* treat it as the final version yet. The next useful step would be to make it much more concrete against your actual React project — particularly **the entity directory, feature/use-case directory, React Query, API clients, AG Grid/ForecastGrid, routing/pages, and the boundaries between feature UI and entity UI**. That's where we can turn this from a generic architecture description into something an LLM could actually use to make correct changes to the codebase.
