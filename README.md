**In the architecture we're describing, features will usually represent commands/mutations, while entity queries represent reading state.** But I wouldn't make "feature = mutation" an absolute rule.

The distinction I'd use is:

> **Entities own facts about domain objects. Features own meaningful application actions.**

That means a query can absolutely be a use case, but it doesn't automatically need to become a `feature`.

### Simple example

Suppose we have:

```text
entities/
└── forecast/
    ├── model/
    │   └── Forecast.ts
    └── queries/
        ├── useForecast.ts
        └── useForecasts.ts
```

These are straightforward:

```ts
useForecast(forecastId)
useForecasts(projectId)
```

They're basically asking:

> "Give me Forecast data."

There's little application behaviour involved. They're **entity queries**.

Then:

```text
features/
├── create-forecast/
├── update-forecast/
├── delete-forecast/
└── assign-resource/
```

These represent actions:

> Create a Forecast.

> Update a Forecast.

> Delete a Forecast.

> Assign a Resource.

These are naturally mutation-oriented.

---

## But consider a more interesting query

Imagine the application has:

> **Compare a forecast against its baseline**

That's technically a read.

There may be no mutation at all:

```text
GET /forecasts/{id}/comparison
```

But this isn't simply:

> "Get Forecast."

It's an actual **application capability**.

The user is asking the system to perform a meaningful operation:

```text
Compare Forecast
       ↓
Forecast + Baseline
       ↓
Calculate differences
       ↓
Return comparison
```

I'd be quite comfortable modelling that as:

```text
features/
└── compare-forecast/
    ├── api/
    ├── model/
    ├── ui/
    └── index.ts
```

even though it ultimately performs a GET.

That's because **the feature boundary comes from the use case, not HTTP verb**.

---

# A useful distinction

I'd therefore separate these concepts:

| Type                                               | Example                   | Likely location                         |
| -------------------------------------------------- | ------------------------- | --------------------------------------- |
| Retrieve an entity                                 | `useForecast(id)`         | `entities/forecast`                     |
| List an entity                                     | `useForecasts(projectId)` | `entities/forecast`                     |
| Create entity                                      | `createForecast()`        | `features/create-forecast`              |
| Update entity                                      | `updateForecast()`        | `features/update-forecast`              |
| Delete entity                                      | `deleteForecast()`        | `features/delete-forecast`              |
| Perform meaningful domain operation                | `compareForecast()`       | `features/compare-forecast`             |
| Retrieve a complex application-specific projection | `useForecastComparison()` | potentially `features/compare-forecast` |
| Generic UI/data utility                            | `useDebounce()`           | `shared`                                |

So the important distinction isn't:

```text
GET → entity
POST → feature
```

It's:

```text
simple entity retrieval → entity

application capability → feature
```

---

## This actually gives us a nice rule for the architecture

I'd revise the earlier wording slightly.

Instead of saying:

> "Queries belong in entities and mutations belong in features."

I'd say:

> **Basic entity queries belong with their entity. Application-level use cases belong in features, regardless of whether they read or write data.**

That is much more robust.

### For example

This:

```text
entities/forecast/queries/useForecast.ts
```

is sensible because the query is fundamentally about retrieving a Forecast.

Whereas this:

```text
features/get-forecast-comparison/
```

could be sensible because "Forecast Comparison" is an application capability rather than simply another property of Forecast.

---

# And I think there's another useful consequence

We probably **shouldn't call every feature a "use case" in the strict sense**.

There are two related concepts:

### Entity operation

```text
Get Forecast
Get Project
Get Resources
```

These are essentially data access.

### Application use case

```text
Create Forecast
Update Forecast
Assign Resource
Compare Forecast
Approve Forecast
Publish Forecast
```

These describe something the application *does*.

That's why your observation that **"there probably wouldn't ever be get use cases"** is mostly correct for the kind of application architecture we're designing.

You don't need:

```text
features/
└── get-forecast/
```

just because somebody can navigate to a Forecast page.

You'd normally have:

```text
entities/
└── forecast/
    └── queries/
        └── useForecast.ts
```

and the page consumes it.

---

## Where this gets particularly useful in your project

Think about your ForecastGrid.

The page might do:

```text
Forecast page
    │
    ├── useForecast(...)
    │
    ├── useForecastResources(...)
    │
    ├── Update Forecast feature
    │
    ├── Assign Resource feature
    │
    └── Change Forecast Plan feature
```

The reads establish **the state you're looking at**.

The features establish **the things you can do with that state**.

That's a very clean conceptual separation:

```text
             "What do we know?"
                     │
                     ▼
                 ENTITIES
                     │
              queries / state
                     │
                     ▼
             "What can we do?"
                     │
                     ▼
                 FEATURES
                     │
             commands / actions
                     │
                     ▼
                  UI
```

And importantly, **features can consume entity queries**.

For example, `assign-resource` might need the current Forecast and available Resources, but it owns the *assignment operation*, not the underlying Resource/Forecast retrieval.

I think this is actually a stronger formulation of the architecture than the one I gave you initially, and I'd use **"entity queries vs application use cases"** as one of the core rules in the eventual LLM context.
