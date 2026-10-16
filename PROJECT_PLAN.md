# Project Development Plan
This document outlines the planned phases and tasks for developing Santorio, a
free, open-source, privacy-first calorie and body-composition tracker that
runs in the browser (desktop and phone) and can later be wrapped as an iOS app.

Design and rationale: [`docs/design.md`](docs/design.md). Mathematics:
[`docs/math.md`](docs/math.md). Original skeleton spec: [`SPEC.md`](SPEC.md).

**Process.** Phase 1 is an interactive style round with the user. Phases 2–8
are then built in a single pass at the user's explicit request, which replaces
the usual per-phase confirmation; each phase is still committed and pushed on
completion, with tests and a markdown test report. Phases 9–10 require a Mac
and are out of the single pass unless decided otherwise.

**This repository is public.** No personal data, keys or identifying
configuration may be committed (see [`docs/design.md`](docs/design.md) §7).

## Phase 0: Planning
1. [completed] Record the skeleton spec (`SPEC.md`).
2. [completed] Review the spec, identify gaps, propose additions.
3. [completed] Agree platform and hosting: web-first PWA on GitHub Pages, Capacitor iOS later.
4. [completed] Choose framework (React + TypeScript + Vite) and dependencies.
5. [completed] Write design document and mathematical models (`docs/design.md`, `docs/math.md`).
6. [completed] Repository security baseline before any code.
    - [completed] `.gitignore` (including `.env`, `.DS_Store`, `venv/`, `.venv/`)
    - [completed] `pre-commit` with `detect-secrets` and a baseline
7. [pending] Resolve open questions (`docs/design.md` §11).
    - [pending] Cross-device sync approach
    - [pending] Single-pass scope (web only, or include iOS shell code)
    - [pending] Calorie floor value

## Phase 1: Style Round
8. [pending] Produce style tiles: type scale, colour (dark and light), spacing, radii, motion.
9. [pending] Mock up key screens at phone and desktop widths.
    - [pending] Today with universal input bar
    - [pending] Plate with per-item uncertainty
    - [pending] Weight trend with uncertainty band
    - [pending] First run
10. [pending] Compare alternative presentations of uncertainty (interval text, range bars, bands).
11. [pending] Iterate with the user until approved; freeze design tokens and breakpoints.

## Phase 2: Foundations
12. [pending] Monorepo skeleton (npm workspaces): `packages/*`, `apps/web`, `tools/eval`.
13. [pending] Tooling: TypeScript strict, Biome (lint and format), Vitest, Playwright; Python tools via uv with ruff and ty.
14. [pending] Extend pre-commit: Biome, `tsc --noEmit`, ruff, ty, EXIF check on fixtures.
15. [pending] GitHub Actions: checks on push; deploy to GitHub Pages.
16. [pending] App shell: routing, responsive layout (phone and desktop), design tokens, service worker, installability.
17. [pending] Content Security Policy as a meta tag, with an end-to-end test proving an unexpected request is blocked.
18. [pending] `README.md` (structure diagram, documentation index, logic and mathematics overview), `PRIVACY.md`, `SECURITY.md`, `LICENSE`.

## Phase 3: Core Logging
19. [pending] Vault: AES-GCM encryption, non-extractable default key, schema-versioned records.
20. [pending] Storage repositories (IndexedDB, OPFS).
21. [pending] First run: goal, basics, install tip.
22. [pending] Today screen.
23. [pending] Universal input bar: local food matching, quick-add syntax, suggestions by time of day.
24. [pending] Plate: per-item editing, live nutrition, partial-eaten control, undo.
25. [pending] AI providers (Gemini, Claude): schemas, validation, retry, just-in-time key setup guide.
26. [pending] Describe-to-items via AI, non-blocking.
27. [pending] Hong Kong starter pack (licence-checked sources) and drink modifiers.
28. [pending] Uncertainty engine in `core`: log-normal propagation, Fenton–Wilkinson sums, display rounding (`docs/math.md` §3).
29. [pending] Encrypted backup and restore; JSON and CSV export; delete all data.
30. [pending] Network activity list.
31. [pending] Action-budget end-to-end tests (`docs/design.md` §5.3).
32. [pending] Test report for Phase 3.

## Phase 4: Body and Targets
33. [pending] Weight and waist entry.
34. [pending] Kalman trend with smoother and 80 % band (`docs/math.md` §5).
35. [pending] Plan screen: goal, rate, protein range, limits.
36. [pending] Mifflin–St Jeor prior, energy-balance estimate, precision-weighted combination, guardrails (`docs/math.md` §6).
37. [pending] Target history with reasons.
38. [pending] Optional check-in: checklist and energy/performance ratings.
39. [pending] Calendar (`.ics`) reminders.
40. [pending] Optional passkey lock with recovery passphrase.
41. [pending] Scenario tests (loss, gain, gaps, partial logging) and test report.

## Phase 5: Barcodes and Labels
42. [pending] Barcode scanning (camera or webcam; `BarcodeDetector` or WASM fallback) and typed digits.
43. [pending] Lookup: local cache, then Open Food Facts.
44. [pending] Label photo extraction with Atwater consistency check (`docs/math.md` §2.5).
45. [pending] Label-parsing tests (English and Chinese, per serving and per 100 g, kJ and kcal) and test report.

## Phase 6: Photo Portions
46. [pending] Capture: phone camera with card guide and tilt indicator; desktop drag-and-drop, paste, webcam.
47. [pending] Card corners: AI proposal, draggable adjustment, aspect-ratio check.
48. [pending] Homography, polygon and mask areas, parallax correction (`docs/math.md` §1).
49. [pending] Outline parser for masks, polygons and boxes.
50. [pending] Volume models: mounds, bowls, counted items (`docs/math.md` §2).
51. [pending] "I weighed it" calibration: bias and split-conformal width (`docs/math.md` §4).
52. [pending] Accuracy evaluation tool with plots (`tools/eval`, Python).
53. [pending] Geometry tests and test report.

## Phase 7: Progress Photos
54. [pending] Encrypted photo store.
55. [pending] Guided capture: poses, ghost overlay, level, timer.
56. [pending] Compare view (side by side, slider).
57. [pending] Test proving progress-photo bytes never appear in a network request; test report.

## Phase 8: Hardening and Release
58. [pending] Accessibility audit (keyboard, screen reader, contrast, reduced motion).
59. [pending] English and Traditional Chinese (Hong Kong) translations.
60. [pending] `docs/private-wrapper.md` and overlay build option.
61. [pending] Cross-device sync, as decided in task 7.
62. [pending] Final documentation pass and links; full test report.
63. [pending] Production deploy to GitHub Pages.

## Phase 9: Native iOS (requires a Mac)
64. [pending] Capacitor shell.
65. [pending] Plugins: Keychain, Face ID, HealthKit read/write with deduplication, Vision card corners, VisionKit barcodes, local notifications.
66. [pending] App Store readiness (privacy labels, permission strings); decide on the Apple Developer Program.

## Phase 10: LiDAR (requires a Mac and a LiDAR iPhone)
67. [pending] ARKit depth capture plugin.
68. [pending] Plane fit and depth-volume integration; cross-check with the card.
69. [pending] Depth-volume tests on recorded samples.
