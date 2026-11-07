# Product Steering

> Fill this file in before starting spec work. This document answers "why are we building this"
> so that every requirements.md / design.md written later inherits the right context automatically.

## 1. Product Vision
<!-- One paragraph. What does this product do, and for whom? -->
TODO: e.g. "A [type of app] that helps [target user] to [core outcome] by [key mechanism]."

## 2. Target Users
| Persona | Description | Primary Need |
|---|---|---|
| TODO | TODO | TODO |

## 3. Core Problem Being Solved
<!-- The pain point that justifies the product's existence. Be specific and falsifiable. -->
TODO

## 4. Key Features (MVP scope)
- TODO: Feature 1
- TODO: Feature 2
- TODO: Feature 3

> Anything not listed here is explicitly out of scope for the MVP unless a spec says otherwise.

## 5. Non-Goals
<!-- Things we are deliberately NOT building, to prevent scope creep in generated specs. -->
- TODO

## 6. Success Metrics
<!-- How we'll know the product is working. Ties back to Firebase Analytics events if relevant. -->
- TODO: e.g. activation rate, retention, task completion time

## 7. Constraints
- **Platform**: Web app (Angular), responsive, installable as PWA — TODO confirm
- **Backend**: Firebase only (Firestore/RTDB, Auth, Cloud Functions, Storage, Hosting) — no custom server
- **Budget/Tier**: TODO (Spark vs Blaze plan — affects whether Cloud Functions/outbound network calls are usable)
- **Compliance**: TODO (GDPR? data residency? none?)

## 8. Tone & Design Language
<!-- Feeds into UI generation later. -->
TODO: e.g. minimal, playful, enterprise-serious, dark-mode-first, etc.

---
**Usage note for the agent:** When generating `requirements.md` or `design.md` for a new feature,
always re-read this file first. If a proposed feature conflicts with a Non-Goal or Constraint above,
flag it explicitly rather than silently proceeding.
