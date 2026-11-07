# Tasks — Authentication

> Ordered implementation checklist derived from `design.md`. Check off as completed.

## 0. Prerequisites

- [ ] Firebase project created; Firestore, Auth (Email/Password provider), Emulator Suite enabled
- [ ] `@angular/fire`, `firebase` installed; `core/firebase/` providers wired in `app.config.ts`
- [ ] `@aparajita/capacitor-biometric-auth` and `@capacitor/preferences` installed
- [ ] `design-system.md` gaps (error/success colors) resolved or provisionally assigned

## 1. Data Layer

- [ ] `features/auth/models/user-profile.model.ts` — `UserProfile`, `UserRole` types
- [ ] `features/auth/data/user-profile.service.ts` — `create()`, `get(uid)` against `users/{uid}`, with retry-on-write per design.md §4
- [ ] `features/auth/data/auth.service.ts` — `login()`, `signup()`, `sendPasswordReset()`, `signOut()`, `user` signal, `isAuthenticated` computed, Firebase error mapping (design.md §8)
- [ ] `features/auth/data/biometric.service.ts` — `isAvailable()`, `enroll()`, `verify()`, wraps Capacitor plugin + `Preferences` flag
- [ ] Unit tests (Vitest) for all of the above, mocked against the Firebase Emulator

## 2. Security Rules

- [ ] `firestore.rules` — rules from design.md §5
- [ ] Rules unit tests: own-doc allowed, cross-user denied, role-change-on-update denied, invalid-role-on-create denied

## 3. Routing & Guards

- [ ] `features/auth/auth.routes.ts` — `welcome`, `login`, `signup`, `forgot-password`, each via `loadComponent()`
- [ ] `features/auth/guards/auth.guard.ts` — functional guard, redirects to `welcome` if unauthenticated
- [ ] `features/auth/guards/role.guard.ts` — functional guard, checks `route.data['requiredRole']`
- [ ] Wire `auth.guard.ts` onto the authenticated area's parent route in `app.routes.ts`

## 4. UI — Welcome Page

- [ ] `pages/welcome/welcome.page.ts` — standalone, imports from `@ionic/angular/standalone`
- [ ] Logo/title, tagline, "Ingresar" button, "Acceso Rápido" button (conditionally shown per requirements.md §3.1/§3.5), footer tagline
- [ ] On init: check `isAuthenticated` + `biometric.isAvailable()` to decide whether to auto-skip or show quick-access
- [ ] Mobile-first styles using tokens from `design-system.md` (bg `#08090D`, primary gradient `#4BA7B4`→`#3A8490`)
- [ ] Component test: renders both buttons; "Acceso Rápido" hidden when not eligible

## 5. UI — Login Page

- [ ] `pages/login/login.page.ts` — Signal Forms model: `email`, `password`
- [ ] Back button, "Bienvenido" heading, subtitle, email input, password input, forgot-password link, submit button, signup link
- [ ] Field-level validation errors (empty/malformed email, empty password)
- [ ] Submit → `AuthService.login()`; on success, prompt biometric opt-in if first time on device; on failure, generic error per design.md §8
- [ ] Component test: validation blocks submit; error message shown on failed login; success navigates

## 6. UI — Signup Page

- [ ] `pages/signup/signup.page.ts` — Signal Forms model: `displayName`, `email`, `password`, `role`
- [ ] Back button, heading, subtitle, name/email/password inputs, **role selector (new — not in original mockup)**, submit button, login link
- [ ] Validation: all fields required, password ≥ 8 chars, role required
- [ ] Submit → sequence from design.md §4 (create Auth user → updateProfile → Firestore write → retry-on-failure)
- [ ] Component test: role required to submit; retries Firestore write on simulated failure without re-creating Auth user

## 7. UI — Forgot Password Page

- [ ] `pages/forgot-password/forgot-password.page.ts` — Signal Forms model: `email`
- [ ] Back button, heading, subtitle, email input (with leading icon per screenshot), submit button, back-to-login link
- [ ] Submit → `AuthService.sendPasswordReset()`; always show confirmation state regardless of whether email exists
- [ ] Component test: shows confirmation on submit; validation blocks empty/malformed email

## 8. Biometric Enrollment Flow

- [ ] Post-login prompt component/modal: "¿Activar Acceso Rápido en este dispositivo?"
- [ ] Accept → `biometric.service.ts.enroll()`; Decline → no-op, don't ask again this session
- [ ] Sign-out clears the biometric-enabled flag (requirements.md §3.5)

## 9. Cloud Function Safety Net

- [ ] `functions/src/onUserCreate.ts` — Auth `onCreate` trigger; logs if no matching `users/{uid}` doc exists after a delay (design.md §4 step 5)

## 10. Definition of Done Check

- [ ] All items in `structure.md`'s per-feature Definition of Done checklist pass for this feature
- [ ] Manually tested: signup (both roles) → logout → login → enable biometric → relaunch → "Acceso Rápido" unlock, on both an iOS and Android simulator/device
