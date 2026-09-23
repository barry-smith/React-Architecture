Yes. **That is actually an important part of the architecture**, because the directory structure tells you *where code belongs*, while the filename conventions tell you *what role that particular file plays*.

I would include this explicitly in the LLM context. The `.use-case`, `.dto`, `.mapper`, etc. conventions make the architecture much more discoverable.

I'd model the overall approach like this:

```text
src/
├── app/
├── pages/
├── widgets/
├── features/
├── entities/
└── shared/
```

with **files inside those boundaries named according to their responsibility**.

## 1. The filename suffix is part of the architecture

Rather than:

```text
forecastService.ts
forecastHelper.ts
forecastUtils.ts
forecastStuff.ts
```

we make the responsibility explicit:

```text
forecast.dto.ts
forecast.mapper.ts
update-forecast.use-case.ts
forecast.query.ts
forecast.schema.ts
forecast.types.ts
```

So an LLM should be able to infer something from a filename without opening the file.

For example:

```text
update-forecast.use-case.ts
```

should immediately communicate:

> This implements the Update Forecast application use case.

Whereas:

```text
forecast.mapper.ts
```

communicates:

> This transforms Forecast data between representations.

That's valuable both for humans and LLMs.

---

# 2. Entities

An entity folder might look something like:

```text
entities/
└── forecast/
    ├── model/
    │   ├── forecast.types.ts
    │   ├── forecast.dto.ts
    │   └── forecast.mapper.ts
    │
    ├── api/
    │   └── forecast.api.ts
    │
    ├── queries/
    │   ├── forecast.query.ts
    │   └── use-forecast.query.ts
    │
    ├── ui/
    │   ├── forecast-status.tsx
    │   └── forecast-name.tsx
    │
    └── index.ts
```

The exact subdivisions don't need to be rigid, but the **responsibility-based naming** should be.

### `.dto.ts`

A DTO describes the shape crossing a boundary, normally the API boundary.

For example:

```ts
export type ForecastDto = {
  id: string
  name: string
  plan_id: string
}
```

The DTO isn't necessarily the same thing as the application's domain model.

That's why:

```text
forecast.dto.ts
```

and:

```text
forecast.ts
```

shouldn't automatically be treated as interchangeable.

---

# 3. Mappers

A mapper translates between representations.

For example:

```text
API DTO
   ↓
mapper
   ↓
Domain/application model
```

Something like:

```ts
export function mapForecast(dto: ForecastDto): Forecast {
  return {
    id: dto.id,
    name: dto.name,
    planId: dto.plan_id,
  }
}
```

This keeps API concerns such as:

```text
snake_case
API-specific naming
nullability
response wrappers
```

from leaking throughout the application.

So:

```text
forecast.dto.ts
forecast.mapper.ts
forecast.ts
```

have deliberately different responsibilities.

---

# 4. `.use-case.ts`

This is particularly important for the **features** directory.

For example:

```text
features/
└── update-forecast/
    ├── update-forecast.use-case.ts
    ├── update-forecast.dto.ts
    ├── update-forecast.mapper.ts
    ├── update-forecast.api.ts
    ├── use-update-forecast.ts
    ├── UpdateForecastDialog.tsx
    └── index.ts
```

The naming tells us:

```text
update-forecast.use-case.ts
```

= the application-level operation.

```text
update-forecast.api.ts
```

= communication with the backend.

```text
update-forecast.dto.ts
```

= boundary data shape.

```text
update-forecast.mapper.ts
```

= conversion between representations.

```text
use-update-forecast.ts
```

= React integration.

```text
UpdateForecastDialog.tsx
```

= UI.

That is considerably more informative than simply having:

```text
service.ts
hook.ts
types.ts
component.tsx
```

---

# 5. The React hook isn't the use case

This distinction is worth documenting explicitly.

You might have:

```text
update-forecast.use-case.ts
```

and:

```text
use-update-forecast.ts
```

They are related but aren't the same thing.

Conceptually:

```text
                   React
                     │
                     ▼
          use-update-forecast.ts
                     │
                     ▼
        update-forecast.use-case.ts
                     │
                     ▼
           update-forecast.api.ts
                     │
                     ▼
                   HTTP
```

The hook is the **React adapter**.

The use case is the **application behaviour**.

This means the use case shouldn't inherently need React.

For example, you don't want:

```ts
// update-forecast.use-case.ts

const mutation = useMutation(...)
```

if we're trying to keep the application logic independent of React.

Instead:

```ts
// update-forecast.use-case.ts

export async function updateForecast(
  input: UpdateForecastInput
): Promise<Forecast> {
  ...
}
```

and:

```ts
// use-update-forecast.ts

export function useUpdateForecast() {
  return useMutation({
    mutationFn: updateForecast,
  })
}
```

This separation becomes especially useful when testing.

---

# 6. Queries deserve similar naming

Following our previous discussion, an entity might have:

```text
entities/
└── forecast/
    ├── forecast.dto.ts
    ├── forecast.mapper.ts
    ├── forecast.api.ts
    ├── forecast.query.ts
    └── use-forecast.ts
```

Or, if we want the React Query distinction to be explicit:

```text
forecast.query.ts
use-forecast.query.ts
```

The important thing isn't necessarily which exact convention we choose.

The important thing is that the convention communicates:

> This is a query, not an application use case.

So we don't end up with:

```text
features/get-forecast/
```

unless retrieving the forecast genuinely represents an application capability.

---

# 7. Schemas

Another useful suffix is:

```text
.schema.ts
```

This should normally represent validation/parsing schemas.

For example:

```ts
const ForecastSchema = z.object({
  id: z.string(),
  name: z.string(),
})
```

You could have:

```text
forecast.dto.ts
forecast.schema.ts
forecast.mapper.ts
```

where:

* `.dto` = TypeScript representation of a boundary object
* `.schema` = runtime validation
* `.mapper` = transformation
* `.types` = application/domain types

That distinction prevents one giant `forecast.ts` from accumulating everything.

---

# 8. Types

I'd be slightly cautious about `.types.ts`.

It's useful, but it can easily become a dumping ground.

For example:

```text
forecast.types.ts
```

should contain genuinely Forecast-related types.

Not:

```ts
export type SomethingUsedByForecastAndResourceAndProject = ...
```

simply because Forecast happens to use it.

If the type belongs to a specific concept, keep it there.

If it is genuinely shared, move it to an appropriate shared/domain location.

---

# 9. API files

`.api.ts` should describe the **transport boundary**, rather than becoming a generic service.

For example:

```text
forecast.api.ts
```

might contain:

```ts
getForecast()
getForecasts()
```

while:

```text
update-forecast.api.ts
```

might contain:

```ts
updateForecast()
```

The API layer should primarily concern itself with:

* HTTP
* URLs
* request/response DTOs
* authentication/headers where appropriate
* transport-specific concerns

It shouldn't become the place where all the application's business logic lives.

---

# 10. Widgets

This naming approach becomes particularly useful with widgets.

For example:

```text
widgets/
└── forecast-grid/
    ├── forecast-grid.tsx
    ├── forecast-grid.model.ts
    ├── forecast-grid.mapper.ts
    ├── forecast-grid.types.ts
    └── index.ts
```

Although I wouldn't automatically create all of those.

The principle should be:

> **Only create a file when there is a meaningful responsibility to isolate.**

Don't create:

```text
forecast-grid/
├── forecast-grid.types.ts
├── forecast-grid.constants.ts
├── forecast-grid.utils.ts
├── forecast-grid.helpers.ts
└── forecast-grid.helpers-2.ts
```

just to satisfy an architectural pattern.

The architecture should make code easier to understand, not force fragmentation.

---

# 11. Pages

Pages should generally be relatively thin.

For example:

```text
pages/
└── forecast/
    ├── ForecastPage.tsx
    └── index.ts
```

The page composes:

```text
widgets
features
entities
shared UI
```

rather than implementing the application's behaviour itself.

So we don't want:

```text
ForecastPage.tsx
```

becoming a 1,500-line component containing:

* API calls
* mutations
* transformation logic
* grid configuration
* modal state
* business rules
* routing
* etc.

The page should largely answer:

> "What does this route consist of?"

---

# 12. `index.ts` is an architectural boundary

I'd include this in the LLM instructions too.

For example:

```text
features/update-forecast/
├── index.ts
├── update-forecast.use-case.ts
├── update-forecast.api.ts
└── use-update-forecast.ts
```

`index.ts` defines the **public API of the feature**.

Consumers should ideally do:

```ts
import { useUpdateForecast } from '@/features/update-forecast'
```

rather than:

```ts
import { useUpdateForecast }
  from '@/features/update-forecast/use-update-forecast'
```

This gives us freedom to reorganise internals later.

---

# 13. Naming should describe responsibility, not implementation

This is probably the most important naming rule.

Prefer:

```text
update-forecast.use-case.ts
forecast.mapper.ts
forecast.dto.ts
forecast.schema.ts
forecast.api.ts
forecast.query.ts
```

over:

```text
forecastService.ts
forecastHelper.ts
forecastUtils.ts
forecastManager.ts
forecastHandler.ts
```

The latter names tell you almost nothing.

A `Service` could mean practically anything.

A `Mapper` has a well-defined responsibility.

A `DTO` has a well-defined responsibility.

A `UseCase` has a well-defined responsibility.

---

# 14. One thing I'd change from the previous description

I wouldn't make the LLM think that **every file must use a suffix**.

For example:

```text
UpdateForecastDialog.tsx
ForecastGrid.tsx
ForecastPage.tsx
```

are perfectly clear.

Likewise:

```text
useForecast.ts
```

is already idiomatic React naming.

The suffix convention is most valuable where **multiple roles could otherwise be confused**.

So:

```text
update-forecast.use-case.ts
```

is useful because it distinguishes the application operation from:

```text
update-forecast.api.ts
use-update-forecast.ts
```

That's the real benefit.

---

# 15. The complete mental model

Putting everything together, we get something like:

```text
src/
│
├── app/
│
├── pages/
│   └── forecast/
│       └── ForecastPage.tsx
│
├── widgets/
│   └── forecast-grid/
│       ├── ForecastGrid.tsx
│       └── index.ts
│
├── features/
│   └── update-forecast/
│       ├── update-forecast.use-case.ts
│       ├── update-forecast.api.ts
│       ├── update-forecast.dto.ts
│       ├── update-forecast.mapper.ts
│       ├── use-update-forecast.ts
│       ├── UpdateForecastDialog.tsx
│       └── index.ts
│
├── entities/
│   └── forecast/
│       ├── forecast.types.ts
│       ├── forecast.dto.ts
│       ├── forecast.mapper.ts
│       ├── forecast.api.ts
│       ├── forecast.query.ts
│       ├── use-forecast.ts
│       ├── ForecastStatus.tsx
│       └── index.ts
│
└── shared/
    ├── ui/
    ├── hooks/
    ├── lib/
    └── ...
```

And the LLM should be able to interpret this almost like a sentence:

> **Forecast is an entity. Update Forecast is a feature/use case. Forecast Grid is a widget that composes them. Forecast Page is the route-level composition. DTOs represent external boundaries, mappers translate between representations, API files handle transport, query files retrieve entity state, use-case files implement application behaviour, and React hooks adapt that behaviour/state to React.**

That, IMO, is much closer to the **actual architecture context we should give an LLM** than just giving it a directory tree.

One further thing I'd add to the eventual document is a **complete end-to-end example** of a single operation — e.g. `Update Forecast` — showing exactly how `page → widget → feature → use-case → API → DTO → mapper → entity/query → React Query` fits together. That would make the conventions considerably harder for an LLM to misunderstand.


Yes — I agree. **With TanStack Router, I wouldn't necessarily have a separate `pages/` directory at all.** My previous example was carrying over a more traditional React architecture.

For your TanStack Start/TanStack Router setup, I'd distinguish **routing from page composition** rather than introduce a `pages` layer.

A more appropriate structure would be something like:

```text
src/
├── routes/
│   ├── __root.tsx
│   ├── _authenticated/
│   │   ├── route.tsx
│   │   ├── projects/
│   │   │   ├── index.tsx
│   │   │   └── $projectId/
│   │   │       └── index.tsx
│   │   └── forecasts/
│   │       └── $forecastId/
│   │           └── index.tsx
│   │
│   └── login.tsx
│
├── widgets/
├── features/
├── entities/
└── shared/
```

The **route file effectively becomes the page**.

### So what does a route do?

A route should primarily be responsible for **routing concerns and composition**:

```tsx
export const Route = createFileRoute('/forecasts/$forecastId/')({
  component: ForecastRoute,
})
```

and then:

```tsx
function ForecastRoute() {
  const { forecastId } = Route.useParams()

  return (
    <>
      <ForecastHeader forecastId={forecastId} />
      <ForecastGrid forecastId={forecastId} />
    </>
  )
}
```

It shouldn't become the place where all the Forecast business logic lives.

So the architecture becomes:

```text
Route
  │
  ├── Widget
  │     │
  │     ├── Entity
  │     └── Feature
  │
  ├── Widget
  │
  └── Feature
```

rather than:

```text
Page
  ↓
Widget
```

---

## This also makes the four layers cleaner

I'd now describe them as:

### `routes/` — Where am I?

Routing concerns:

* URL structure
* route parameters
* search parameters
* loaders
* route-level data dependencies
* authentication/route guards
* layout composition
* route error/pending states

### `widgets/` — What substantial UI am I looking at?

Examples:

```text
ForecastGrid
ProjectSummary
ResourceAllocationPanel
ForecastHeader
```

Widgets can compose entities and features.

### `features/` — What can I do?

Examples:

```text
update-forecast
assign-resource
create-project
change-forecast-plan
```

### `entities/` — What things does the application know about?

Examples:

```text
forecast
project
resource
```

### `shared/` — What is genuinely generic?

Examples:

```text
Button
Dialog
useDebounce
formatCurrency
```

That gives us:

```text
                         ROUTE
                           │
             ┌─────────────┴─────────────┐
             │                           │
          WIDGET                       FEATURE
             │                           │
       ┌─────┴─────┐                     │
       │           │                     │
    ENTITY      FEATURE                ENTITY
```

---

## There's an interesting TanStack Router consequence

The route itself can also own **route-specific data requirements**.

For example, suppose:

```text
/forecasts/$forecastId
```

needs a Forecast.

You don't necessarily need to put:

```ts
const { data } = useForecast(forecastId)
```

inside a page component simply because "pages fetch data."

With TanStack Router, the route can participate in the data-loading lifecycle.

That means we need to be careful in our eventual architecture document about saying:

> "Queries always live inside entities."

I'd refine that to:

> **The entity owns the query definition and data-access abstraction; routes/widgets/features decide when that data is required and consume it through the appropriate mechanism.**

That keeps ownership separate from orchestration.

---

## And I'd probably remove `pages/` from our LLM context entirely

For **your** architecture, I'd now start the canonical tree as:

```text
src/
├── routes/
├── widgets/
├── features/
├── entities/
└── shared/
```

Potentially with:

```text
src/
├── routes/
├── widgets/
├── features/
├── entities/
├── shared/
└── app/
```

if there are genuinely application-level concerns that don't belong in the routes themselves.

The important thing is that **`routes` isn't another business layer**. It's the application's navigation/composition boundary.

And I think that's a better fit for the architecture you've been describing than introducing a conventional `pages` layer just because many React architecture examples have one.

Yes — **both are worth making explicit**, and the first one is especially important given what your LLM is doing.

There are actually two separate architectural rules here:

1. **Entities and features are independent ownership boundaries. They may have similar internal structure and even similarly named files.**
2. **Imports should expose the architectural boundary, not hide it.**

I would add both to the LLM context.

---

# 1. Entities and features can absolutely have the same directory names

This is an important point.

Suppose we have:

```text id="9d7f4n"
entities/
└── forecast/
    ├── api/
    ├── model/
    └── queries/

features/
└── update-forecast/
    ├── api/
    ├── model/
    └── ui/
```

There's nothing wrong with both having:

```text
api/
model/
ui/
```

because those directories describe the **internal responsibility of the code**, not the architectural ownership.

The mistake your LLM is making is presumably something like:

> "Forecast is an entity, and Update Forecast operates on Forecast, therefore Update Forecast code should live under Forecast."

That's exactly what we **don't** want.

### Ownership beats subject matter

`update-forecast` is *about* a Forecast.

But it is **owned by the Update Forecast use case**.

So:

```text id="1z49kw"
entities/forecast/
```

owns:

> What is a Forecast?

while:

```text id="7k0n0k"
features/update-forecast/
```

owns:

> How does the application update a Forecast?

Those are different responsibilities.

---

# 2. This is analogous to nouns vs verbs

I think this is actually a useful rule to give an LLM.

### Entities are primarily nouns

```text id="pyrk0j"
Forecast
Project
Resource
Plan
```

### Features are primarily verbs/actions

```text id="xax1b3"
Create Forecast
Update Forecast
Assign Resource
Change Plan
Compare Forecast
```

Therefore:

```text id="q4x6wp"
entities/forecast
```

doesn't mean:

> "Everything concerning Forecast goes here."

It means:

> "Code whose primary responsibility is the Forecast concept goes here."

And:

```text id="r5y2pp"
features/update-forecast
```

means:

> "Code whose primary responsibility is updating a Forecast goes here."

This is probably the single most important thing to tell your LLM to stop it collapsing features into entities.

---

# 3. Features may import entities

The dependency is normally:

```text id="a8f25m"
feature
   ↓
entity
```

For example:

```ts id="lqf1jv"
import { Forecast } from '@/entities/forecast'
```

That's completely fine.

But that doesn't mean the feature **belongs to** the entity.

Think:

```text id="e6x5lq"
entities/forecast
       ▲
       │
       │ uses
       │
features/update-forecast
```

not:

```text id="v89x8s"
entities/forecast
    └── update-forecast
```

The former is a dependency.

The latter is an ownership hierarchy.

Those are different things.

---

# 4. And yes — I would absolutely add another layer to imports

I think your instinct here is good.

Instead of:

```ts id="6d6k6r"
import { Forecast } from '@/entities'
```

I'd strongly prefer:

```ts id="br1sjr"
import { Forecast } from '@/entities/forecast'
```

And:

```ts id="x3x5bx"
import { useUpdateForecast } from '@/features/update-forecast'
```

This makes the architectural boundary visible **at the import site**.

You immediately know:

```text id="w9m4z3"
@/entities/forecast
```

= Forecast entity

and:

```text id="j3v7kl"
@/features/update-forecast
```

= Update Forecast feature.

---

# 5. I wouldn't have a global `entities` barrel

This is an important distinction.

I'd avoid:

```text id="w8xj91"
entities/
├── index.ts
├── forecast/
├── project/
└── resource/
```

where:

```ts id="f0c3kv"
import {
  Forecast,
  Project,
  Resource,
} from '@/entities'
```

Why?

Because now the import tells you very little about ownership.

Compare:

```ts id="n0zqqq"
import { Forecast } from '@/entities'
```

with:

```ts id="rx0x5n"
import { Forecast } from '@/entities/forecast'
```

The second is much more expressive.

I'd use the same principle for features:

```ts id="k1q7l9"
import { useUpdateForecast }
  from '@/features/update-forecast'
```

rather than:

```ts id="i4f1yq"
import { useUpdateForecast }
  from '@/features'
```

---

# 6. The import path becomes part of the architecture

This is actually quite powerful.

Consider:

```ts id="2yk0bw"
import { Forecast } from '@/entities/forecast'
import { Resource } from '@/entities/resource'

import { useUpdateForecast }
  from '@/features/update-forecast'

import { AssignResourceDialog }
  from '@/features/assign-resource'

import { ForecastGrid }
  from '@/widgets/forecast-grid'

import { Button }
  from '@/shared/ui/button'
```

You can almost read the architecture directly from the imports:

```text id="7g1g8p"
entities → domain concepts

features → capabilities

widgets → composed application UI

shared → generic infrastructure
```

That's excellent context for an LLM.

---

# 7. It also prevents the "everything becomes an entity" problem

Suppose the LLM sees:

```text id="7sp6cz"
features/update-forecast/
```

and decides:

> "This operates on Forecast, therefore I'll move it to entities/forecast."

But then the imports are explicitly:

```ts id="zqyscz"
import { useUpdateForecast }
  from '@/features/update-forecast'
```

The architecture itself is telling the LLM:

> **This is a feature. Don't move it.**

And if the feature needs the entity:

```ts id="7u9j72"
import { Forecast }
  from '@/entities/forecast'
```

the dependency is explicit.

---

# 8. Same internal directory names are therefore fine

You could quite legitimately have:

```text id="w5om2d"
entities/
└── forecast/
    ├── api/
    ├── model/
    ├── ui/
    └── index.ts

features/
└── update-forecast/
    ├── api/
    ├── model/
    ├── ui/
    └── index.ts
```

and imports:

```ts id="k1wh40"
import { Forecast } from '@/entities/forecast'

import {
  UpdateForecastDialog,
  useUpdateForecast,
} from '@/features/update-forecast'
```

The repeated `api`, `model`, and `ui` names aren't confusing because the **architectural context comes first**.

It's similar to namespaces:

```text id="3j5v6e"
entities/forecast/api
features/update-forecast/api
```

They are two different APIs because they have different owners.

---

# 9. I'd make the public API rule explicit too

Each entity/feature should ideally have an `index.ts`:

```text id="r2i6xe"
entities/
└── forecast/
    ├── index.ts
    ├── forecast.types.ts
    ├── forecast.mapper.ts
    └── ...

features/
└── update-forecast/
    ├── index.ts
    ├── update-forecast.use-case.ts
    └── ...
```

Then:

```ts id="9k6dps"
import { Forecast } from '@/entities/forecast'
```

and:

```ts id="0j7c7x"
import { useUpdateForecast } from '@/features/update-forecast'
```

The `index.ts` becomes the **public contract of that architectural slice**.

Internal files remain internal.

---

# 10. I'd add explicit "do not do this" rules

For the LLM context, I'd actually be quite blunt:

> **Do not move a feature into an entity merely because the feature operates on that entity.**
>
> `features/update-forecast` should remain a feature even though it depends heavily on `entities/forecast`.
>
> An entity represents a domain concept. A feature represents a capability/use case.
>
> Features may import entities. Entities should not import features.
>
> Similar directory names such as `api`, `model`, `ui`, and `queries` may legitimately appear inside both entities and features. The parent architectural boundary determines ownership.

And:

> **Do not create a global barrel import for all entities or all features unless there is a specific architectural reason to do so.**
>
> Prefer imports that identify the individual architectural slice:
>
> `@/entities/forecast`
>
> `@/features/update-forecast`
>
> `@/widgets/forecast-grid`
>
> `@/shared/ui`

---

# 11. One subtle point: I wouldn't necessarily call this "another layer"

I'd describe it as **a scoped public import boundary** rather than an additional architectural layer.

So:

```text id="t1a7zq"
@/
├── entities/
│   ├── forecast/
│   │   └── index.ts
│   └── project/
│       └── index.ts
│
├── features/
│   ├── update-forecast/
│   │   └── index.ts
│   └── assign-resource/
│       └── index.ts
│
└── widgets/
    └── forecast-grid/
        └── index.ts
```

The import path corresponds directly to the **architectural slice**.

That's better than having a flat alias such as:

```text id="6f4y9b"
@/entries
```

or:

```text id="h9v7xc"
@/features
```

because those imports lose the ownership information.

---

## So I'd now add these rules to our architecture document

The core structure is becoming:

```text id="2r7jgd"
routes/
    ↓
widgets/
    ↓
features/ ──────→ entities/
    ↓                 ↓
    └──────────────→ shared/
```

with the important caveat that this is **dependency direction, not necessarily a strict hierarchy**.

And each slice has its own public API:

```text id="7o4d4q"
@/entities/forecast
@/entities/project

@/features/update-forecast
@/features/assign-resource

@/widgets/forecast-grid
```

while **similar internal directory names are completely legitimate**:

```text id="6h1s3u"
entities/forecast/model
features/update-forecast/model
```

because `model` describes the *kind of responsibility*, while `forecast` / `update-forecast` describes the *owner*.

I think this is an important refinement because it changes the architecture from merely being a **folder convention** into something enforceable through the code's import graph. An LLM can then see not only *where a file lives*, but also *which architectural slice it belongs to and which direction dependencies are supposed to flow*.
