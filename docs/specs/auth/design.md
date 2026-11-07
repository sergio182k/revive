# Design — Authentication

> Complies with `docs/steering/tech.md` and `docs/steering/structure.md`. Implements
> `requirements.md` in this folder.

## 1. Route & File Layout

Per `structure.md`, this feature lives under `src/app/features/auth/`:

```
features/auth/
├── auth.routes.ts
├── models/
│   └── user-profile.model.ts        # UserProfile interface
├── data/
│   ├── auth.service.ts              # Firebase Auth wrapper, session signals
│   ├── user-profile.service.ts      # Firestore users/{uid} read/write
│   └── biometric.service.ts         # Capacitor biometric wrapper
├── guards/
│   ├── auth.guard.ts                # CanActivateFn — require session
│   └── role.guard.ts                # CanActivateFn — require specific role
└── pages/
    ├── welcome/
    │   └── welcome.page.ts (+html/scss/spec)
    ├── login/
    │   └── login.page.ts
    ├── signup/
    │   └── signup.page.ts
    └── forgot-password/
        └── forgot-password.page.ts
```

`core/firebase/` (already defined in `structure.md`) hosts the app-wide Firebase provider setup
(`provideFirebaseApp`, `provideAuth`, `provideFirestore`) that this feature depends on but does not
own.

## 2. Data Model

```ts
// features/auth/models/user-profile.model.ts
export type UserRole = 'patient' | 'physiotherapist';

export interface UserProfile {
  uid: string;               // == Firebase Auth uid, also the Firestore doc id
  displayName: string;
  email: string;
  role: UserRole;
  createdAt: Timestamp;       // Firestore server timestamp
  biometricEnabledDeviceIds?: string[]; // optional, for multi-device biometric tracking (see §3)
}
```

Firestore path: `users/{uid}` — one document per user, doc id == Auth uid (not auto-generated),
so security rules can match `request.auth.uid == resource.id` directly.

## 3. Biometric ("Acceso Rápido") Architecture — key decision

**Firebase Auth has no biometric method.** "Acceso Rápido" therefore does *not* create a second
credential. Instead:

1. Firebase's JS SDK (via AngularFire) already persists the signed-in session (refresh token) in
   the webview's IndexedDB across app restarts — this is default behavior, nothing extra to build.
2. On first successful email/password login on a given device, prompt the user to enable
   biometrics. If accepted, `biometric.service.ts` calls the Capacitor biometric plugin
   (`@aparajita/capacitor-biometric-auth` — reason: actively maintained, supports both iOS
   Face ID/Touch ID and Android BiometricPrompt through one API) to register that the OS-level
   biometric check is available, and stores a local flag (`Preferences` API from
   `@capacitor/preferences`, **not** `localStorage` per `tech.md`'s zoneless/artifact
   constraints and generally because it's the Capacitor-native, cross-platform-correct store) —
   e.g. `revive.biometricEnabled = true`.
3. On subsequent app opens, `AuthService` checks: is there a persisted Firebase session AND is
   `revive.biometricEnabled` true? If both, show "Acceso Rápido" on the Welcome screen instead of
   auto-navigating past it — this gives the user an explicit, deliberate unlock action rather than
   silently reusing the session (better UX for a shared-device / privacy-sensitive health app).
4. Tapping "Acceso Rápido" calls `biometric.service.ts`'s `verify()`, which triggers the native
   Face ID/Touch ID/BiometricPrompt UI. On success, the app simply proceeds — the already-valid
   Firebase session is used as-is; no Firebase call is needed. On failure/cancel, stay put.
5. If the persisted Firebase session itself has expired (Firebase's refresh token was revoked,
   e.g. password changed elsewhere), the first authenticated read will fail with an
   auth error; `AuthService` catches this, clears the biometric flag, and redirects to Login.

This means biometrics are a **local device gate on an existing session**, not a Firebase-level
auth method — no Cloud Function or custom Firebase flow is required for this piece.

## 4. Signup Flow — Auth + Firestore Consistency

`createUserWithEmailAndPassword` and the Firestore profile write are two separate calls that can't
be wrapped in one atomic transaction (Auth isn't part of Firestore). To satisfy requirement 3.3
("don't leave an auth user with no profile"):

1. Call `createUserWithEmailAndPassword`.
2. Call `updateProfile` (sets `displayName` on the Auth user record too, so it's available even if
   the Firestore read hasn't completed yet).
3. Write `users/{uid}` in Firestore.
4. If step 3 fails, retry up to 2 times with backoff before surfacing an error. If it still fails,
   keep the user on the signup screen with an error, *not* signed out — the Auth account exists,
   so simply retrying the Firestore write (without recreating the Auth user) resolves it.
5. A Cloud Function `onUserCreate` (Auth trigger, `functions/src/onUserCreate.ts`) acts as a
   safety net: if no `users/{uid}` doc exists N minutes after Auth user creation, nothing
   destructive happens automatically (no auto-delete), but it logs for manual/ops follow-up. This
   is a monitoring backstop, not the primary consistency mechanism (step 4 is).

## 5. Firestore Security Rules (excerpt)

```
match /users/{userId} {
  allow read: if request.auth != null && request.auth.uid == userId;
  allow create: if request.auth != null
                && request.auth.uid == userId
                && request.resource.data.role in ['patient', 'physiotherapist']
                && request.resource.data.uid == userId;
  allow update: if request.auth != null
                && request.auth.uid == userId
                // role is immutable after creation:
                && request.resource.data.role == resource.data.role;
}
```

A rules unit test (`@firebase/rules-unit-testing`) must cover: own-doc read/write allowed,
cross-user read/write denied, role-change-on-update denied, invalid role-on-create denied.

## 6. Role-Based Routing

`role.guard.ts` (functional `CanActivateFn`) reads the current `UserProfile` signal from
`user-profile.service.ts` and compares against `route.data['requiredRole']`. Not used within this
auth feature itself (both roles land on a generic authenticated home for now), but established
here so future features (e.g. a physiotherapist-only patient list) can declare
`data: { requiredRole: 'physiotherapist' }` on their routes per requirement 3.6.

## 7. Component/Signal Sketch (illustrative, not literal)

```ts
// data/auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private auth = inject(Auth); // AngularFire Auth
  private userProfileService = inject(UserProfileService);

  readonly user = signal<User | null>(null);           // Firebase Auth user
  readonly isAuthenticated = computed(() => this.user() !== null);

  constructor() {
    onAuthStateChanged(this.auth, (u) => this.user.set(u));
  }

  login(email: string, password: string) { /* signInWithEmailAndPassword, map errors */ }
  signup(input: SignupInput) { /* see §4 sequence */ }
  sendPasswordReset(email: string) { /* sendPasswordResetEmail, always resolve success */ }
  signOut() { /* signOut + clear biometric flag via BiometricService */ }
}
```

```ts
// pages/login/login.page.ts
@Component({
  standalone: true,
  imports: [IonContent, IonButton, /* ... */],
  templateUrl: './login.page.html',
})
export class LoginPage {
  private auth = inject(AuthService);
  private router = inject(Router);

  form = form(/* Signal Forms model: email, password */);
  submitting = signal(false);
  errorMessage = signal<string | null>(null);

  async submit() {
    if (!this.form().valid()) return;
    this.submitting.set(true);
    try {
      await this.auth.login(this.form().value().email, this.form().value().password);
      this.router.navigate(['/home']);
    } catch {
      this.errorMessage.set('Correo o contraseña incorrectos');
    } finally {
      this.submitting.set(false);
    }
  }
}
```

## 8. Error Mapping

Firebase Auth error codes are mapped to the generic, enumeration-safe messages required in
`requirements.md` §3.2/§3.4 inside `auth.service.ts` — never surface raw Firebase error codes/text
to the UI.

| Firebase code | User-facing message (es) |
|---|---|
| `auth/invalid-credential`, `auth/user-not-found`, `auth/wrong-password` | "Correo o contraseña incorrectos" |
| `auth/email-already-in-use` | "Este correo ya está registrado" |
| `auth/too-many-requests` | "Demasiados intentos. Intenta de nuevo más tarde" |
| anything else | "Algo salió mal. Intenta de nuevo" |

## 9. Open Questions Carried Over

- Exact error/success color tokens (blocked on `design-system.md` gap).
- Confirm the dark-neutral vs teal-gradient button distinction is a disabled/enabled state (design assumes yes — see requirements.md §5).
