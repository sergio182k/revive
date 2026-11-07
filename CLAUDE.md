# ReVive — Project Instructions for Claude

Fisioterapia inteligente: an Angular + Ionic + Capacitor mobile-first app connecting patients and
physiotherapists to manage recovery, backed by Firebase.

## Before doing anything else

**Read all four files in `docs/steering/` before starting any task in this repo, every session:**

- `docs/steering/product.md` — product vision, users, scope, non-goals
- `docs/steering/tech.md` — required stack, Angular/Ionic/Firebase conventions, forbidden patterns
- `docs/steering/structure.md` — folder layout, naming conventions, SDD workflow, Definition of Done
- `docs/steering/design-system.md` — color/typography/spacing tokens; do not invent new ones

These are persistent ground truth. If a request conflicts with something in these files, say so
explicitly instead of silently overriding it.

## Spec-Driven Development workflow

This project follows SDD. Feature specs live in `docs/specs/<feature>/`:

1. `requirements.md` — user stories + EARS-style acceptance criteria
2. `design.md` — technical design (data model, components, security rules) complying with `tech.md`
3. `tasks.md` — ordered implementation checklist derived from `design.md`

**Do not write feature code before `requirements.md` and `design.md` exist for that feature**,
unless it's a trivial bugfix. If asked to build a feature with no spec yet, write the spec first
and confirm it before implementing.

Existing specs: `docs/specs/auth/` (Firebase email/password + role-based signup + biometric
quick-access).

## Non-negotiables (see `tech.md` for full detail)

- Standalone components only — no NgModules
- Signals for state, not `BehaviorSubject`; Signal Forms, not Reactive Forms
- Zoneless + OnPush (Angular 22 defaults) — never reintroduce Zone.js
- Ionic imports from `@ionic/angular/standalone` only
- Mobile-first CSS — base styles target a phone, `min-width` media queries scale up
- Firestore security rules are part of the deliverable for any feature touching new collections

## Useful commands

```bash
ng serve              # dev server
ionic serve           # dev server via Ionic CLI (same app)
npx cap run ios       # run on iOS simulator
npx cap run android   # run on Android emulator
firebase emulators:start   # local Firebase (Auth, Firestore, Functions, Storage)
```
