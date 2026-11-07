# Structure Steering

> Defines where code lives and how it's named, so generated specs and code land in predictable
> places instead of the agent re-deciding project layout on every feature.

## 1. Repository Layout

```
/
├── src/
│   ├── app/
│   │   ├── app.config.ts          # providers: zoneless CD, router, Firebase, etc.
│   │   ├── app.routes.ts          # top-level lazy routes
│   │   ├── app.component.ts       # standalone root shell component
│   │   │
│   │   ├── core/                  # singleton, app-wide concerns — imported once
│   │   │   ├── auth/              # auth service, functional guards
│   │   │   ├── firebase/          # firebase provider setup, converters
│   │   │   ├── native/            # Capacitor plugin wrappers (Camera, Geolocation, PlatformService, etc.)
│   │   │   └── interceptors/      # functional HTTP interceptors
│   │   │
│   │   ├── shared/                # reusable, presentation-only, no feature knowledge
│   │   │   ├── ui/                # dumb/standalone UI components (buttons, cards, etc.)
│   │   │   ├── pipes/
│   │   │   └── directives/
│   │   │
│   │   └── features/
│   │       └── <feature-name>/
│   │           ├── <feature-name>.routes.ts
│   │           ├── models/            # TS interfaces mirroring Firestore documents
│   │           ├── data/              # services wrapping resource()/httpResource()/Firestore
│   │           ├── components/
│   │           │   └── <component-name>/
│   │           │       ├── <component-name>.component.ts
│   │           │       ├── <component-name>.component.html
│   │           │       ├── <component-name>.component.scss
│   │           │       └── <component-name>.component.spec.ts
│   │           └── pages/             # route-level components (loadComponent targets)
│   │
│   ├── environments/
│   │   ├── environment.ts
│   │   └── environment.prod.ts
│   │
│   └── styles/                    # global styles/tokens
│       ├── theme.scss             # Ionic CSS variable overrides (--ion-color-*, etc.)
│       └── global.scss            # mobile-first base styles, imported once in angular.json
│
├── ios/                            # Capacitor-generated native iOS project (do not hand-edit generated files)
├── android/                        # Capacitor-generated native Android project (do not hand-edit generated files)
├── capacitor.config.ts             # Capacitor app id, native plugin config
│
├── functions/                     # Firebase Cloud Functions (separate TS project)
│   └── src/
│       └── <function-name>.ts     # one exported function per file
│
├── firestore.rules
├── storage.rules
├── firebase.json
├── .firebaserc
│
└── docs/
    └── specs/
        └── <feature-name>/
            ├── requirements.md     # generated per feature, references steering/product.md
            ├── design.md           # references steering/tech.md + steering/structure.md
            └── tasks.md            # implementation checklist derived from design.md
```

## 2. Spec-Driven Development Workflow

This project follows SDD: steering docs (this folder) are always-loaded context; specs are
per-feature and generated on demand.

1. **Steering** (`docs/steering/product.md`, `tech.md`, `structure.md`) — loaded first, always. Rarely changes.
2. **Requirements** (`docs/specs/<feature>/requirements.md`) — user stories + acceptance criteria (EARS-style: "WHEN ... THE SYSTEM SHALL ...") for one feature, derived from `product.md`.
3. **Design** (`docs/specs/<feature>/design.md`) — technical approach for that feature: data model (Firestore collections/fields), components, services, security rules changes — must comply with `tech.md` and place files per this document.
4. **Tasks** (`docs/specs/<feature>/tasks.md`) — an ordered, checkbox implementation plan generated from `design.md`, granular enough to execute task-by-task.

Rule of thumb: **never write code before requirements.md and design.md exist for that feature**, unless the change is a trivial bugfix.

## 3. Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Component file | `kebab-case.component.ts` | `task-list.component.ts` |
| Component class | `PascalCase` + `Component` | `TaskListComponent` |
| Component selector | `app-kebab-case` | `app-task-list` |
| Service file | `kebab-case.service.ts` | `task.service.ts` |
| Signal-based store service | `kebab-case.store.ts` | `task.store.ts` |
| Firestore model interface | `PascalCase`, singular | `Task`, `UserProfile` |
| Route file | `kebab-case.routes.ts` | `tasks.routes.ts` |
| Cloud Function file/export | `camelCase`, verb-first | `onTaskCreated.ts` |
| Test file | mirrors source + `.spec.ts` | `task.service.spec.ts` |

## 4. Component Authoring Template

Every generated component should follow this shape (illustrative, not literal boilerplate to paste):

- `standalone: true` (implicit in v22, but `imports: [...]` still explicit)
- Inputs via `input()` / `input.required()`, outputs via `output()`, two-way via `model()`
- Dependencies via `inject()` at class-field initialization
- Local state via `signal()`, derived via `computed()`
- Template uses `@if` / `@for (item of items(); track item.id)` / `@switch`
- No lifecycle-based data fetching (`ngOnInit` + subscribe) — prefer `resource()` tied to a signal input

## 5. Where Firebase-Specific Code Lives

- Firestore/Auth/Storage **wiring** (SDK init, providers) → `core/firebase/`
- Firestore **converters & typed collection references** → colocated in the feature's `models/` + `data/`
- **Security rules** → repo root (`firestore.rules`, `storage.rules`), updated in the same PR as the feature that needs new access patterns
- **Cloud Functions** → `functions/src/`, one concern per file, never mixed into the Angular `src/app` tree

## 6. Definition of Done (per feature)

A feature is done when:
- [ ] `requirements.md`, `design.md`, `tasks.md` exist under `docs/specs/<feature>/`
- [ ] Code follows the layout above and the conventions in `tech.md`
- [ ] Firestore/Storage rules updated + rules tests pass (if new data access introduced)
- [ ] Unit tests (Vitest) pass for new services/components
- [ ] Feature route is lazy-loaded via `loadComponent()`
- [ ] Ionic components imported from `@ionic/angular/standalone` only, individually per component
- [ ] Styles are mobile-first (base = phone, `min-width` media queries for larger screens) and respect safe-area insets
- [ ] Touch targets meet minimum size; no hover-only interaction
- [ ] Manually checked in `ionic serve` at a small phone width, plus at least one native simulator/device (iOS or Android)
