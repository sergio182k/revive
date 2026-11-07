# Tech Steering

> Persistent technical ground truth for this project. Every design.md and generated code must
> comply with this file. If a spec needs to deviate, it must call out the deviation explicitly
> and justify it — never deviate silently.

## 1. Stack Summary

| Layer | Choice | Version / Notes |
|---|---|---|
| Framework | Angular | v22 (latest stable, "signal-first" release) |
| Language | TypeScript | Strict mode ON |
| Reactivity | Signals | Primary state primitive — see §3 |
| Change detection | Zoneless + OnPush | Zoneless is default in v22; do not (re)add zone.js |
| Components | Standalone | `standalone: true` is the default; never generate NgModules |
| Forms | Signal Forms | Stable as of v22 — use instead of classic Reactive Forms for new work |
| Async data | `resource` / `httpResource` | Stable as of v22 — preferred over manual RxJS HTTP calls |
| Templates | Built-in control flow | `@if / @for / @switch / @let` — never `*ngIf` / `*ngFor` |
| Backend | Firebase | Firestore, Auth, Cloud Functions (2nd gen), Storage, Hosting |
| Firebase client SDK | `@angular/fire` (AngularFire) | Wraps Firebase JS SDK with Angular DI + signal-friendly APIs |
| Mobile/UI shell | Ionic Framework | `@ionic/angular/standalone` submodule only — see §8 |
| Native runtime | Capacitor | iOS + Android wrapper around the same web codebase |
| Build tool | Angular CLI (esbuild/Vite dev server) | `ng build` / `ng serve` |
| Testing | Vitest | Default test runner in v22; Karma is legacy — don't generate Karma config |
| Styling | Mobile-first CSS (see §9) — Tailwind or plain SCSS on top of Ionic's CSS variables, pick one and record it here |
| Design tokens | `docs/steering/design-system.md` | Colors, typography, spacing — do not hardcode values already defined there |
| Package manager | TODO: npm / pnpm |

## 2. Non-Negotiable Conventions

1. **Standalone only.** Every component/directive/pipe is standalone. No `NgModule`, ever, including for the root app (`bootstrapApplication`).
2. **Signals for state, not `BehaviorSubject`.** Component and service state is modeled with `signal()`, derived state with `computed()`, side effects with `effect()`. RxJS is still fine for genuine streams (websockets, multi-source event composition) but is not the default for local/component state.
3. **`inject()` over constructor injection** in new code, including in `signal()`/`computed()` initializers where constructor injection isn't possible anyway.
4. **`input()` / `output()` / `model()` signal APIs**, not `@Input()` / `@Output()` decorators, for standalone components.
5. **OnPush is implicit** (v22 default) — never manually set `ChangeDetectionStrategy.Default`/`Eager` unless interoperating with a legacy non-signal library that requires it, and if so, comment why.
6. **No Zone.js.** Provide `provideZonelessChangeDetection()` in `app.config.ts`. If a third-party lib needs zone.js, flag it in design.md rather than silently re-adding the polyfill.
7. **Data fetching via `resource()` / `httpResource()`**, reactive to signal inputs, instead of manual `subscribe()` + manual loading/error state tracking.
8. **Signal Forms** for anything beyond a trivial 1-2 field form. Reactive Forms (`FormGroup`/`FormControl`) only when interoperating with a library that requires it.
9. **Strict TypeScript.** `strict: true`, `noImplicitAny`, no `any` in new code without a `// TODO(reason)` comment.
10. **Lazy-loaded routes** via `loadComponent()` for every feature route — no eager-loaded feature modules.

## 3. State Management Pattern

- **Local UI state** → `signal()` inside the component.
- **Derived state** → `computed()`.
- **Cross-component / feature state** → a `providedIn: 'root'` (or route-scoped) injectable service exposing `readonly` signals; only the service mutates its own signals.
- **Remote/async state (Firestore docs, HTTP)** → `resource()` / `httpResource()`, wrapped in a service if reused across components.
- Avoid NgRx/Akita/etc. unless product.md's scale explicitly justifies it — signals + services are the default until proven insufficient.

## 4. Firebase Integration Rules

- **Auth**: `@angular/fire/auth`. Route guards implemented as functional guards (`CanActivateFn`) reading an injected auth signal/service, not class-based guards.
- **Firestore**: `@angular/fire/firestore`. Prefer the signal-based/`rxResource`-wrapped read APIs over manual `onSnapshot` subscriptions where possible. Model documents with typed TypeScript interfaces colocated with the feature (see structure.md).
- **Cloud Functions**: 2nd-gen, TypeScript, one function per file under `functions/src/`. Callable functions (`onCall`) preferred over raw HTTPS endpoints for anything invoked from the Angular app, so App Check + Auth context come for free.
- **Security rules**: `firestore.rules` / `storage.rules` are part of the deliverable for any feature that reads/writes new collections — a feature spec is not "done" without corresponding rules and, ideally, rules unit tests (`@firebase/rules-unit-testing`).
- **App Check**: enabled in production; note in design.md if a feature needs to bypass it during local dev (emulator).
- **Emulator Suite**: local development and tests run against the Firebase Local Emulator Suite (Auth, Firestore, Functions, Storage) — never against production Firebase during dev/test.
- **Environment config**: Firebase config lives in `src/environments/`, never hardcoded inline in services.

## 5. Testing

- **Unit tests**: Vitest, colocated `*.spec.ts` next to the file under test.
- **Component tests**: Angular Testing Library or Angular's `TestBed` with standalone component `imports`, targeting behavior not implementation.
- **Firestore rules tests**: `@firebase/rules-unit-testing` against the emulator.
- **E2E**: TODO — pick Playwright or Cypress and record here.

## 6. Tooling & CI Assumptions

- `ng generate` schematics are used for new components/services to keep boilerplate consistent (standalone by default in v22, so no extra flags needed).
- Linting: ESLint with `@angular-eslint`.
- Formatting: Prettier.
- CI runs: lint → unit tests (Vitest) → build → (optionally) deploy to a Firebase preview channel per PR.

## 8. Ionic Integration Rules

- **Import path**: always `@ionic/angular/standalone`, never bare `@ionic/angular` (the latter pulls in lazy-loaded/NgModule code and defeats treeshaking).
- **Bootstrap**: `provideIonicAngular({ mode: 'md' })` (or `'ios'`/`'md'` per-platform) in `app.config.ts`, plus `{ provide: RouteReuseStrategy, useClass: IonicRouteStrategy }`. `provideIonicAngular()` must be called even with no custom config.
- **Per-component imports**: every Ionic element used in a template (`IonButton`, `IonContent`, etc.) is imported individually into that standalone component's `imports: [...]`. Don't import a catch-all `IonicModule`.
- **Icons**: use `addIcons()` from `ionicons` (called once per component that needs icons, typically in the constructor or a field initializer) rather than the global icon registration pattern.
- **Known caveat — zoneless + Ionic internals**: Angular app code (your components/services) is fully zoneless per this document, but as of mid-2026 some Ionic web-component internals still rely on their own lifecycle rather than Angular signals end-to-end. This is transparent for typical usage (forms, navigation, overlays) — if you hit a case where an `ion-*` component doesn't visually update after a signal change, call `ChangeDetectorRef.markForCheck()` as a bridge and note it with a comment; don't reintroduce Zone.js to "fix" it globally.
- **Native features**: access device APIs (camera, geolocation, filesystem, push notifications, etc.) only through official Capacitor plugins (`@capacitor/*`), wrapped in an injectable service under `core/native/` — never call `window.cordova` or platform APIs directly from components.
- **Platform checks**: use Capacitor's `Capacitor.isNativePlatform()` / `Capacitor.getPlatform()` to branch web-vs-native behavior (e.g. skip a native-only plugin call when running in a browser tab), centralized in a `PlatformService`, not scattered `if` checks.
- **Firebase on native**: prefer the Firebase JS SDK (via AngularFire) for Firestore/Auth/Storage on both web and native through Capacitor's webview — it works out of the box. Only reach for native Firebase plugins (`@capacitor-firebase/*`) if you need a feature the JS SDK can't do in a webview (e.g. native push notifications via FCM, native Google Sign-In UI) — record that decision in the relevant feature's `design.md`.

## 9. Mobile-First Styling Rules

- **Author for the smallest viewport first.** Base styles (no media query) target a ~360–390px wide phone screen. Add complexity upward with `min-width` media queries for tablet/desktop — never the reverse (`max-width`-only, desktop-first).
- **Use Ionic's CSS custom properties** (`--ion-color-*`, `--ion-font-family`, safe-area variables like `env(safe-area-inset-*)`) for theming instead of hardcoded colors/fonts, so theming stays consistent across `ion-*` and custom components.
- **Respect safe areas** on every full-bleed screen (notches, home indicators) via Ionic's built-in safe-area CSS variables — don't hardcode top/bottom padding.
- **Touch targets** minimum 44×44px (iOS HIG) / 48×48dp (Material) for any tappable element, including custom (non-Ionic) buttons.
- **No hover-only interactions.** Anything that works on `:hover` must also have a tap/click equivalent, since most target devices have no pointer.
- **Layout primitives**: prefer CSS Grid/Flexbox with `rem`/`%`/viewport units over fixed pixel widths, so layouts adapt across phone/tablet/desktop without component-level breakpoints.
- **Test matrix**: at minimum, verify each screen at a small phone width (~360px), a large phone width (~430px), and desktop — via `ionic serve` in browser plus at least one real/simulated device per platform (iOS + Android) before marking a feature done.

## 10. Explicitly Deprecated / Forbidden in This Project

- `NgModule`-based architecture
- `*ngIf`, `*ngFor`, `*ngSwitch` structural directives
- `@Input()`/`@Output()` decorators on new standalone components
- Manual Zone.js re-introduction
- Class-based route guards/resolvers
- `HttpClient` manual `subscribe()` chains where `httpResource()` covers the case
- Bare `@ionic/angular` imports (must use `@ionic/angular/standalone`)
- Desktop-first CSS (`max-width`-only media queries as the primary layout strategy)
- Direct Cordova/native API access bypassing Capacitor plugins
