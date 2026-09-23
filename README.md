I think **`widgets` was part of the architecture**, and it actually fills an important gap in the model I described above.

The architecture was closer to:

```text
src/
├── app/
├── pages/
├── widgets/
├── features/
├── entities/
└── shared/
```

The key distinction was roughly:

```text
entities  → domain things
features  → things the user/application can do
widgets   → substantial pieces of UI assembled from entities/features
pages     → route/screen composition
shared    → genuinely generic infrastructure/UI
```

### The important difference between `features` and `widgets`

A **feature** represents a capability:

```text
features/
└── update-forecast/
```

It might provide:

```text
UpdateForecastButton
UpdateForecastDialog
useUpdateForecast
```

A **widget** represents a larger, reusable piece of application UI that **composes multiple things**.

For example:

```text
widgets/
└── forecast-grid/
    ├── ForecastGrid.tsx
    ├── ...
    └── index.ts
```

The Forecast Grid might consume:

```text
entities/
├── forecast
├── project
└── resource

features/
├── update-forecast
├── assign-resource
└── change-forecast-plan
```

So conceptually:

```text
                    Forecast Page
                         │
                         ▼
                  Forecast Grid
                     WIDGET
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      Forecast        Features       Resources
       Entity        / Use Cases       Entity
```

That gives us a much better architectural hierarchy.

---

## Widgets aren't just "big components"

This distinction matters.

You don't want:

```text
widgets/
├── Button
├── Input
├── Dialog
└── ForecastGrid
```

The first three are shared UI components.

A widget is something more substantial and **application-specific**.

For example:

```text
widgets/
├── forecast-grid/
├── project-summary/
├── resource-allocation-panel/
└── forecast-overview/
```

These might combine:

* entity data
* feature UI
* shared components
* local presentation state
* layout
* orchestration

but shouldn't generally contain the underlying business operation themselves.

---

## This gives us four useful levels

I think this is the model we were getting toward:

### 1. Entity — "What is it?"

```text
entities/forecast
entities/project
entities/resource
```

Owns the application's understanding of domain concepts.

---

### 2. Feature — "What can I do?"

```text
features/update-forecast
features/assign-resource
features/create-project
features/change-forecast-plan
```

Owns a specific application capability/use case.

---

### 3. Widget — "What substantial thing do I see/use?"

```text
widgets/forecast-grid
widgets/project-summary
widgets/resource-allocation
```

Composes entities and features into meaningful pieces of application UI.

---

### 4. Page — "What screen/route am I on?"

```text
pages/forecast
pages/project
pages/dashboard
```

Composes widgets and perhaps individual features/entities into a route-level experience.

---

So the flow becomes:

```text
┌─────────────────────────────────────┐
│                PAGE                 │
│                                     │
│   ┌─────────────────────────────┐   │
│   │           WIDGET            │   │
│   │                             │   │
│   │   ┌─────────┐ ┌──────────┐  │   │
│   │   │ ENTITY  │ │ FEATURE  │  │   │
│   │   └─────────┘ └──────────┘  │   │
│   │                             │   │
│   └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

And `shared` sits underneath all of them as generic infrastructure.

This is also why **widgets are particularly useful for your AG Grid-heavy application**. Something like `ForecastGrid` is too application-specific to be `shared`, but it isn't really an entity or a use case either. `widgets` gives it a natural home.

I think we should add **widgets** to the LLM context and then refine the boundaries between all four layers before we turn this into the final context document.
