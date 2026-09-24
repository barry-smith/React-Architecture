Yes — with one important refinement: **widgets are application-level composition/shell components, not a dumping ground for page-specific components.**

I would define the boundary like this:

| Layer           | What it owns                                                          | Example                                                     |
| --------------- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| `shared/ui`     | Generic, reusable UI with no business meaning                         | `Button`, `Dialog`, `Table`, `Tabs`                         |
| `entities/*/ui` | Small UI directly representing an entity                              | `ClientStatusBadge`, `RowsPerPage`, `WorkspaceTypeSelect`   |
| `features/*/ui` | UI that performs a specific capability/workflow                       | `RegisterForm`, `UpdateClientDialog`, `AssignResourceForm`  |
| `widgets/*`     | Larger application-level compositions that assemble entities/features | `ForecastGrid`, `ProjectSummary`, `ResourceAllocationPanel` |
| `routes/*`      | Route/page composition and routing concerns                           | `/projects/$projectId`, `/forecasts/$forecastId`            |

### The key distinction

A **widget isn't simply a "large component."**

It should represent a **meaningful piece of an application's interface that composes other architectural slices**.

For example:

```text
widgets/
└── forecast-grid/
    ├── ui/
    │   └── ForecastGrid.tsx
    └── index.ts
```

`ForecastGrid` might compose:

```text
Forecast entity
Resource entity
Update Forecast feature
Assign Resource feature
Change Plan feature
```

The widget is responsible for putting those things together into the application's forecast grid experience.

But it **doesn't own**:

* updating a forecast
* assigning a resource
* the Forecast domain model
* the API call
* the mutation
* the business rules

Those belong to the lower-level slices.

---

### What I would *not* put in `widgets`

Suppose you have:

```text
features/
└── update-client/
    └── ui/
        └── UpdateClientDialog.tsx
```

That's perfectly reasonable.

Even though `UpdateClientDialog` could be used on a client page, it's still **feature UI** because its purpose is:

> perform the Update Client capability.

I wouldn't move it into:

```text
widgets/
└── update-client-dialog/
```

just because it's a reasonably substantial component.

Likewise:

```text
features/
└── register/
    └── ui/
        └── RegisterForm.tsx
```

is feature UI.

It represents the **registration workflow**, not a generic application shell.

---

### What about page-specific components?

This is where I think your assumption is particularly useful.

With TanStack Router, I would **not create a generic `pages/` layer** and I wouldn't create widgets simply to avoid putting something in a route.

For example:

```text
routes/
└── _authenticated/
    └── clients/
        └── index.tsx
```

can compose:

```tsx
<ClientList />
<ClientFilters />
<ClientPagination />
```

If those are only meaningful to that particular route and don't constitute a substantial reusable application-level composition, they don't necessarily need to become widgets.

You can have route-local composition where appropriate.

The route can essentially be:

```tsx
export default function ClientsRoute() {
    return (
        <>
            <PageHeader />
            <ClientFilters />
            <ClientList />
        </>
    );
}
```

The important thing is not to create:

```text
widgets/
└── clients-page/
```

merely because the route contains several components.

That would make `widgets` become a disguised `pages` directory.

---

## I would therefore slightly tighten our definition

Instead of saying:

> "`widgets/` = substantial application-specific UI compositions"

I'd put this in the architecture document:

> **Widgets are application-level UI compositions that represent a substantial, meaningful part of the application's interface and compose entities and/or features into a cohesive experience. They are not generic UI components, individual feature components, or a replacement for pages/routes.**
>
> **Widgets should normally be used when a composition has its own meaningful application-level identity and may be reused across routes or represents a substantial reusable section of the application.**
>
> **Do not create a widget solely because a component is large, because a page contains multiple components, or because something needs somewhere to live. Route-specific UI can remain composed directly within the route when it has no meaningful widget-level boundary.**

And I'd add a very explicit rule:

> **A widget composes capabilities; it does not own capabilities.**

So:

```text
ForecastGrid
    ├── uses Forecast entity
    ├── uses Resource entity
    ├── uses Update Forecast feature
    └── uses Assign Resource feature
```

rather than:

```text
ForecastGrid
    ├── contains forecast mutation logic
    ├── contains resource assignment logic
    └── contains forecast business rules
```

That distinction is important enough that I'd include it in the final LLM context.

### One further nuance: "shell"

I **wouldn't define widgets as mostly/all shell components**, because "shell" can imply things like:

```text
AppShell
Sidebar
Header
Navigation
WorkspaceLayout
```

Those are often **app/layout concerns**, and may belong in `app/`, `shared/`, or route/layout composition depending on their responsibilities.

I'd use **"application-level composition"** rather than **"shell"**.

So your intuition is right that widgets should sit *above* feature-specific components, but I wouldn't limit them to visual shells. A `ForecastGrid` or `ProjectSummary` can absolutely be a widget even though it is specific to a particular application domain.
