
If we're **grouping features by domain**, I would also keep the **internal responsibility directories** inside each individual feature. Otherwise we end up with a structure that groups by domain but then dumps all the implementation concerns into the use-case folder.

I'd structure it like this:

```text
src/
├── entities/
│   ├── forecast/
│   │   ├── api/
│   │   ├── model/
│   │   ├── queries/
│   │   ├── ui/
│   │   └── index.ts
│   ├── project/
│   │   ├── api/
│   │   ├── model/
│   │   ├── queries/
│   │   ├── ui/
│   │   └── index.ts
│   └── resource/
│       ├── api/
│       ├── model/
│       ├── queries/
│       ├── ui/
│       └── index.ts
│
├── features/
│   ├── forecast/
│   │   ├── create/
│   │   │   ├── api/
│   │   │   ├── model/
│   │   │   ├── ui/
│   │   │   ├── hooks/
│   │   │   └── index.ts
│   │   │
│   │   ├── update/
│   │   │   ├── api/
│   │   │   ├── model/
│   │   │   ├── ui/
│   │   │   ├── hooks/
│   │   │   └── index.ts
│   │   │
│   │   └── compare/
│   │       ├── api/
│   │       ├── model/
│   │       ├── ui/
│   │       ├── hooks/
│   │       └── index.ts
│   │
│   ├── project/
│   │   ├── create/
│   │   │   ├── api/
│   │   │   ├── model/
│   │   │   ├── ui/
│   │   │   ├── hooks/
│   │   │   └── index.ts
│   │   └── update/
│   │       ├── api/
│   │       ├── model/
│   │       ├── ui/
│   │       ├── hooks/
│   │       └── index.ts
│   │
│   └── resource/
│       └── assign/
│           ├── api/
│           ├── model/
│           ├── ui/
│           ├── hooks/
│           └── index.ts
│
├── widgets/
│   ├── forecast-grid/
│   ├── project-summary/
│   └── resource-allocation/
│
├── routes/
└── shared/
```

### Why I think this is the better model

There are actually **two different kinds of grouping happening**:

```text
features/
└── forecast/        ← domain grouping
    └── update/      ← specific capability
        ├── api/     ← implementation responsibility
        ├── model/
        ├── hooks/
        └── ui/
```

Each level answers a different question:

| Level       | Question it answers                                     |
| ----------- | ------------------------------------------------------- |
| `features/` | Is this application behaviour?                          |
| `forecast/` | What domain does the behaviour concern?                 |
| `update/`   | What capability/use case is this?                       |
| `api/`      | How does it communicate externally?                     |
| `model/`    | What state/types/business logic belong to this feature? |
| `hooks/`    | How does React consume the feature?                     |
| `ui/`       | What UI implements/exposes the feature?                 |

That is considerably clearer than:

```text
features/
└── forecast/
    └── update/
        ├── update-forecast.ts
        ├── update-forecast-api.ts
        ├── update-forecast-hook.ts
        ├── update-forecast-dialog.tsx
        └── update-forecast-types.ts
```

The latter isn't *wrong*, but once a feature becomes substantial, the folder starts becoming a dumping ground.

### And importantly, this reinforces the entity/feature distinction

We can legitimately have:

```text
entities/
└── forecast/
    ├── api/
    ├── model/
    ├── queries/
    └── ui/

features/
└── forecast/
    └── update/
        ├── api/
        ├── model/
        ├── hooks/
        └── ui/
```

The repeated `api`, `model`, `ui` etc. **isn't duplication of architecture**. They're local implementation boundaries.

The parent tells us **who owns the code**:

```text
entities/forecast/
```

owns the Forecast domain concept.

```text
features/forecast/update/
```

owns the Update Forecast capability.

So even if:

```text
features/forecast/update/api/
```

calls the same backend endpoint that is conceptually related to Forecast, that code belongs to the **update feature**, not automatically to `entities/forecast/api`.

That's the point we were getting at earlier with:

> **Ownership beats subject matter.**

### One small refinement

I wouldn't mandate that **every** feature must contain all of:

```text
api/
hooks/
model/
ui/
```

Those should be available **when needed**, rather than empty folders created by convention.

For example, a simple feature might be:

```text
features/
└── project/
    └── archive/
        ├── api/
        ├── hooks/
        └── index.ts
```

Whereas a substantial feature might be:

```text
features/
└── forecast/
    └── update/
        ├── api/
        ├── hooks/
        ├── model/
        ├── ui/
        └── index.ts
```

That gives us a useful rule:

> **A feature is a self-contained slice of application behaviour. Its internal directories are determined by the responsibilities that feature actually has.**

And yes, I would explicitly put this into the eventual LLM context document. The previous architecture context we recovered does support the idea that **both entities and features can have their own `api`, `model`, `ui`, etc. directories**, rather than treating those directories as belonging exclusively to entities.
