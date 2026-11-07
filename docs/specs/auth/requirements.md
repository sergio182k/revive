# Requirements — Authentication

> Derived from: `Fisioterapia_inteligente.png` (Welcome), `Bienvenido.png` (Login),
> `Crear_cuenta.png` (Signup), `Recuperar.png` (Forgot Password), and decisions confirmed
> 2026-07-04 (Acceso Rápido = biometric; role selector added to signup).
> References `docs/steering/product.md`, `tech.md`, `design-system.md`.

## 1. Scope

Email/password authentication via Firebase Auth, account creation with an explicit Patient /
Physiotherapist role, password recovery, and biometric quick-access to an already-authenticated
session. Out of scope for this spec: social sign-in, admin-driven therapist invitations, MFA.

## 2. Personas

| Persona | Role value | Notes |
|---|---|---|
| Patient | `patient` | Self-registers, tracks own recovery |
| Physiotherapist | `physiotherapist` | Self-registers, manages assigned patients (later feature) |

## 3. User Stories & Acceptance Criteria

### 3.1 Welcome / Landing (`Fisioterapia_inteligente.png`)

**Story:** As any user, I want a landing screen that lets me sign in or quickly resume my session.

- WHEN the app launches and no active Firebase session exists THE SYSTEM SHALL display the Welcome screen with "Ingresar" and "Acceso Rápido" actions.
- WHEN the app launches and an active Firebase session exists AND the device does not support/has not enabled biometrics THE SYSTEM SHALL skip the Welcome screen and navigate directly to the authenticated home route.
- WHEN the user taps "Ingresar" THE SYSTEM SHALL navigate to the Login screen.
- WHEN the user taps "Acceso Rápido" AND a biometric-eligible session exists (see §3.5) THE SYSTEM SHALL start the biometric unlock flow.
- WHEN the user taps "Acceso Rápido" AND no biometric-eligible session exists THE SYSTEM SHALL navigate to the Login screen instead (graceful fallback, no dead-end).

### 3.2 Login (`Bienvenido.png`)

**Story:** As a registered user, I want to sign in with my email and password.

- WHEN the user submits the login form with a syntactically valid email and non-empty password THE SYSTEM SHALL call Firebase `signInWithEmailAndPassword`.
- WHEN authentication succeeds THE SYSTEM SHALL persist the session, offer to enable biometric quick-access if not already set up on this device, and navigate to the authenticated home route.
- WHEN authentication fails due to invalid credentials THE SYSTEM SHALL display an inline error message near the password field without revealing whether the email exists (avoid user enumeration — generic "Correo o contraseña incorrectos").
- WHEN the email field is empty or malformed on submit THE SYSTEM SHALL show a field-level validation error and SHALL NOT call Firebase.
- WHEN the user taps "¿Olvidaste tu contraseña?" THE SYSTEM SHALL navigate to the Forgot Password screen.
- WHEN the user taps "Regístrate" THE SYSTEM SHALL navigate to the Signup screen.
- WHEN the user taps the back chevron THE SYSTEM SHALL return to the Welcome screen.

### 3.3 Signup (`Crear_cuenta.png` + role selector)

**Story:** As a new user, I want to create an account and identify myself as a patient or physiotherapist.

- WHEN the Signup screen is shown THE SYSTEM SHALL display fields: display name, email, password, and a required role selector (Patient / Physiotherapist), defaulting to no pre-selected role.
- WHEN the user submits the form without selecting a role THE SYSTEM SHALL show a validation error on the role selector and SHALL NOT submit.
- WHEN the user submits with a password shorter than 8 characters THE SYSTEM SHALL show a field-level validation error and SHALL NOT call Firebase.
- WHEN all fields are valid THE SYSTEM SHALL call Firebase `createUserWithEmailAndPassword`, then set the display name via `updateProfile`, then create a corresponding Firestore `users/{uid}` document containing `{ displayName, email, role, createdAt }`.
- WHEN the email is already registered THE SYSTEM SHALL show an inline error ("Este correo ya está registrado") on the email field.
- WHEN account + profile document creation both succeed THE SYSTEM SHALL navigate to the authenticated home route appropriate to the selected role.
- WHEN account creation succeeds but the Firestore profile write fails THE SYSTEM SHALL retry the profile write before allowing the user to proceed, rather than leaving an auth user with no role/profile (see design.md §4 for the mechanism).
- WHEN the user taps "Inicia sesión" THE SYSTEM SHALL navigate to the Login screen.

### 3.4 Forgot Password (`Recuperar.png`)

**Story:** As a user who forgot my password, I want to receive reset instructions by email.

- WHEN the user submits a syntactically valid email THE SYSTEM SHALL call Firebase `sendPasswordResetEmail`.
- WHEN the call succeeds THE SYSTEM SHALL show a confirmation state (e.g. "Revisa tu correo") regardless of whether the email is registered (avoid user enumeration).
- WHEN the email field is empty or malformed on submit THE SYSTEM SHALL show a field-level validation error and SHALL NOT call Firebase.
- WHEN the user taps "Volver a iniciar sesión" THE SYSTEM SHALL navigate to the Login screen.

### 3.5 Biometric Quick Access ("Acceso Rápido")

**Story:** As a returning user on a trusted device, I want to unlock my existing session with Face ID / Touch ID instead of retyping my password.

- WHEN a user successfully logs in via email/password on a device with biometric hardware THE SYSTEM SHALL prompt once, asking whether to enable "Acceso Rápido" for this device.
- WHEN the user accepts THE SYSTEM SHALL register the device's biometry via the Capacitor biometric plugin and mark this device as biometric-eligible in local secure storage. **No new Firebase Auth credential is created** — biometrics gate the existing, already-persisted Firebase session (see design.md §3 for why).
- WHEN the user taps "Acceso Rápido" on a biometric-eligible device THE SYSTEM SHALL invoke the native biometric prompt.
- WHEN the biometric check succeeds AND a persisted Firebase session is still valid THE SYSTEM SHALL restore the app to the authenticated home route without re-entering credentials.
- WHEN the biometric check succeeds but the persisted Firebase session has expired/been revoked THE SYSTEM SHALL fall back to the Login screen with a message explaining re-authentication is required.
- WHEN the biometric check fails or is cancelled THE SYSTEM SHALL remain on the Welcome screen and allow retry or fallback to "Ingresar".
- WHEN the user signs out explicitly THE SYSTEM SHALL clear the persisted session and disable "Acceso Rápido" for that device until the user logs in again and re-opts in.

### 3.6 Session & Route Guarding (cross-cutting)

- WHEN any route under the authenticated area is activated AND no valid session exists THE SYSTEM SHALL redirect to the Welcome screen.
- WHEN a route is restricted to a specific role (e.g. a future therapist-only dashboard) AND the current user's role does not match THE SYSTEM SHALL redirect to that user's own role-appropriate home route.

## 4. Non-Functional Requirements

- All four screens follow the mobile-first rules in `tech.md` §9 (safe areas, 44–48px touch targets, no hover-only interaction).
- All forms use Signal Forms (`tech.md` §2.8), not Reactive Forms.
- No plaintext password is logged or persisted outside Firebase Auth's own handling.
- Firestore security rules must allow a user to read/write only their own `users/{uid}` document, and must make `role` immutable after creation (see design.md §5).
- Error states must not disclose whether a given email is registered (applies to both login and password reset).

## 5. Open Items Requiring Design-System Follow-Up

- Error/success color tokens are not defined yet in `design-system.md` (flagged there) — needed before building the validation-error UI in this feature.
- Whether the flat dark "Iniciar sesión"/"Crear cuenta"/"Enviar instrucciones" button styling is a disabled state (teal once valid) or an intentional secondary style is unresolved — assumed here to be **disabled-until-valid**, matching standard form UX and the "Disabled State" button variant shown in the design system PDF's Buttons page. Flag if that's wrong.
