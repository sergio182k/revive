# Design System Steering — ReVive

> Visual tokens for the ReVive app, reverse-engineered by pixel-sampling the provided screenshots
> (`Fisioterapia_inteligente`, `Bienvenido`, `Crear_cuenta`, `Recuperar`) and reviewing the
> `Revive_Design_system_.pdf` export. Values marked **(sampled)** are measured from pixels and are
> close approximations, not guaranteed hex-exact — confirm against the Figma file if pixel-perfect
> accuracy matters. Values marked **(confirm)** could not be reliably determined and need a source
> file (Figma/Tailwind config) to nail down exactly.

## 1. Color Tokens

| Token | Value (sampled) | Usage |
|---|---|---|
| `--color-bg` | `#08090D` | App/card background — near-black, not pure black |
| `--color-surface-input` | `#0D1013` | Text input fill — barely lighter than bg |
| `--color-surface-input-alt` | `#091317` | Alternate input fill seen on Recuperar screen (slight teal tint) — **(confirm)** whether this is a distinct "highlighted/example" state or a different input variant |
| `--color-border-input` | ~`#1A1E22` (10–15% white) | Input border — very subtle, 1px — **(confirm)** exact value, too subtle to pixel-sample reliably |
| `--color-primary-start` | `#4BA7B4` | Primary button gradient — top-left |
| `--color-primary-end` | `#3A8490` | Primary button gradient — bottom-right |
| `--color-action-neutral` | `#202225` | "Iniciar sesión" / "Crear cuenta" / "Enviar instrucciones" button fill — dark neutral, distinct from the teal gradient used on the Welcome screen's "Ingresar"/"Acceso Rápido" — **(confirm)** whether this is a disabled/inactive state (button becomes teal once the form is valid) or an intentional secondary button style. This materially affects the component spec — see Open Question in the auth design doc. |
| `--color-accent` | `#54B7C6` | Links ("¿Olvidaste tu contraseña?", "Regístrate", "Inicia sesión") and the glowing ring on the Welcome screen |
| `--color-text-primary` | `#FFFFFF` | Headings, primary button labels |
| `--color-text-secondary` | `#77787B` | Placeholder text, helper/subtitle text |
| `--color-error` | **(confirm)** | Not visible in any provided screen — no error/validation state was shown. Needs a source (PDF page or Figma) with a form-error state. |
| `--color-success` | **(confirm)** | Same — not present in provided material |

> **Gap to flag:** none of the 4 screenshots show a validation-error state, a loading/spinner state on a button, or a success confirmation. Before building the auth UI, either pull those states from Figma or I'll propose reasonable ones (red ~`#FF6B6B`, green ~`#4ADE80` — these appeared in the raw color-function data inside the PDF, suggesting they exist somewhere in the fuller design system, just not on these 4 pages).

## 2. Typography

| Role | Style observed | Example |
|---|---|---|
| Display / Heading | Bold serif, high-contrast strokes (thick verticals, thin horizontals) — visually similar to **Playfair Display** or **Lora Bold** | "ReVive", "Bienvenido", "Crear cuenta", "Recuperar contraseña" |
| Body / UI text | Clean geometric sans-serif, rounded terminals — visually similar to **Inter**, **Poppins**, or **Manrope** | Subtitles, input labels, button text, links |

**(confirm)** exact font family/weights — this is a visual match, not extracted from font metadata (the PDF's fonts are Type 3/outlined, so font names aren't embedded as text). Recommend confirming with whoever owns the Figma file, or picking one serif + one sans from Google Fonts that matches closely (e.g. `Playfair Display` + `Inter`) if no source is available.

## 3. Spacing & Shape

| Token | Value (sampled, relative to a 903×1701 canvas ≈ a 390–430px-wide phone at 2.2x scale) |
|---|---|
| Screen horizontal padding | ~46px canvas → **~21px / ~1.3rem** at 1x phone width |
| Card/phone corner radius | large, ~36px canvas → **~16px** at 1x |
| Button height | ~104px canvas → **~48px** at 1x (comfortably clears the 44/48px touch-target minimum from `tech.md` §9) |
| Button corner radius | fully rounded pill on primary/secondary Welcome buttons; large-radius rounded-rect (not full pill) on form buttons (Iniciar sesión, Crear cuenta) — **(confirm)** whether this distinction is intentional |
| Input height | ~104px canvas → **~48px** at 1x |
| Input corner radius | same large rounded-rect as form buttons |
| Vertical gap between stacked inputs/buttons | ~50–56px canvas → **~24px / 1.5rem** at 1x |

## 4. Component Notes (from the 4 auth screens)

- **Back button**: top-left simple chevron (`←`), no label, no background — maps to `IonBackButton` or a plain icon button calling `Router`/`NavController` back.
- **Primary CTA button** has (at least) two visual states seen across screens: a teal gradient (Welcome screen) and a flat dark neutral (form screens). Needs clarification — see auth `design.md`.
- **Recuperar (forgot password) screen** uses an input with a leading mail icon and placeholder example text (`correo@ejemplo.com`) — the only screen with an icon inside the input; other screens' inputs have no icon.
- **Inline links** ("¿No tienes cuenta? Regístrate", "¿Ya tienes cuenta? Inicia sesión", "Volver a iniciar sesión") are a two-color sentence: secondary-gray lead-in text + accent-colored actionable word.

## 5. How This File Is Used

- `tech.md` §9 (Mobile-First Styling Rules) defers to this file for actual token values.
- Implement these as CSS custom properties in `src/styles/theme.scss`, layered on top of Ionic's own `--ion-color-*` variables (map `--ion-color-primary` etc. to the tokens above rather than inventing a parallel system).
- Any `design.md` for a UI-bearing feature should reference token names from this file (`--color-primary-start`, etc.) instead of hardcoding hex values.
