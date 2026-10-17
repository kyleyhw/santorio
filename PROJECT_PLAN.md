# Project Development Plan
This document outlines the planned phases and tasks for developing Santorio, a
free, open-source, privacy-first calorie and body-composition tracker that
runs in the browser (desktop and phone) and can later be wrapped as an iOS app.

Design and rationale: [`docs/design.md`](docs/design.md). Mathematics:
[`docs/math.md`](docs/math.md). Plan audit with proposals awaiting decision:
[`docs/plan-audit.md`](docs/plan-audit.md). Original skeleton spec:
[`SPEC.md`](SPEC.md).

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
7. [completed] Regions and two-photograph capture added to the design (`docs/design.md` §7.1–7.2, `docs/math.md` §1.4–1.6, §2.5); plan audited (`docs/plan-audit.md`).
8. [pending] Resolve open questions (`docs/design.md` §11) and decide on the audit proposals (`docs/plan-audit.md` §4).
    - [pending] Cross-device sync approach
    - [pending] Single-pass scope (web only, or include iOS shell code)
    - [pending] Calorie floor value
    - [pending] Pack hosting: commit the built UK and US packs, or build them in CI
    - [pending] Accept, defer or reject each audit proposal
9. [pending] Licence review of regional data sources before any bundling: USDA FoodData Central (CC0), CoFID (Open Government Licence v3.0 and its attribution statement), Hong Kong Centre for Food Safety terms, FAO/INFOODS density database; record outcomes in `packages/packs/SOURCES.md`.

## Phase 1: Style Round
10. [pending] Produce style tiles: type scale, colour (dark and light), spacing, radii, motion.
11. [pending] Mock up key screens at phone and desktop widths.
    - [pending] Today with universal input bar
    - [pending] Plate with per-item uncertainty
    - [pending] Weight trend with uncertainty band
    - [pending] First run, including the region picker
    - [pending] Two-shot capture flow: overhead shot, then side shot with the 30–60° tilt target
    - [pending] Settings → Food databases (install, remove, attribution)
12. [pending] Compare alternative presentations of uncertainty (interval text, range bars, bands).
13. [pending] Iterate with the user until approved; freeze design tokens and breakpoints.

## Phase 2: Foundations
14. [pending] Monorepo skeleton (npm workspaces): `packages/*`, `apps/web`, `tools/eval`, `tools/packs`.
15. [pending] Tooling: TypeScript strict, Biome (lint and format), Vitest, Playwright; Python tools via uv with ruff and ty.
16. [pending] Extend pre-commit: Biome, `tsc --noEmit`, ruff, ty, EXIF check on fixtures.
17. [pending] GitHub Actions: checks on push; pack build if decided in task 8; deploy to GitHub Pages.
18. [pending] App shell: routing, responsive layout (phone and desktop), design tokens, service worker, installability.
19. [pending] Content Security Policy as a meta tag, with an end-to-end test proving an unexpected request is blocked.
20. [pending] Units and regional conventions in `core` (`docs/design.md` §7.2): exact unit constants with sources, salt and sodium, kJ and kcal, household measures, spelling synonyms; round-trip tests.
21. [pending] `README.md` (structure diagram, documentation index, logic and mathematics overview), `PRIVACY.md`, `SECURITY.md`, `LICENSE`.

## Phase 3: Core Logging
22. [pending] Vault: AES-GCM encryption, non-extractable default key, schema-versioned records.
23. [pending] Storage repositories (IndexedDB, OPFS) and an unencrypted Cache Storage area for public packs.
24. [pending] First run: region selection suggested from browser language and time zone, goal, basics, install tip.
25. [pending] Today screen.
26. [pending] Universal input bar: local food matching, quick-add syntax (including grams, e.g. `rice 180g`), suggestions by time of day.
27. [pending] Plate: per-item editing, live nutrition, partial-eaten control, undo.
28. [pending] AI providers (Gemini, Claude): schemas, validation, retry, region hint, just-in-time key setup guide.
29. [pending] Describe-to-items via AI, non-blocking.
30. [pending] Food packs: `tools/packs` pipeline (Python, uv) producing the Hong Kong, United Kingdom (CoFID) and United States (FoodData Central Foundation, SR Legacy and FNDDS, with household-measure weights and derived densities) packs; pack registry; background download, install, remove and attribution in Settings → Food databases; service-worker caching (`docs/design.md` §7.1).
31. [pending] Food search across installed packs: cross-region synonyms, primary-region ranking, household-measure portions; drink modifiers.
32. [pending] Uncertainty engine in `core`: log-normal propagation, Fenton–Wilkinson sums, display rounding (`docs/math.md` §3).
33. [pending] Encrypted backup and restore; JSON and CSV export; delete all data.
34. [pending] Network activity list.
35. [pending] Action-budget end-to-end tests (`docs/design.md` §5.3).
36. [pending] Test report for Phase 3.

## Phase 4: Body and Targets
37. [pending] Weight and waist entry in the region's units.
38. [pending] Kalman trend with smoother and 80 % band (`docs/math.md` §5).
39. [pending] Plan screen: goal, rate, protein range, limits.
40. [pending] Mifflin–St Jeor prior, energy-balance estimate, precision-weighted combination, guardrails (`docs/math.md` §6).
41. [pending] Target history with reasons.
42. [pending] Optional check-in: checklist and energy/performance ratings.
43. [pending] Calendar (`.ics`) reminders.
44. [pending] Optional passkey lock with recovery passphrase.
45. [pending] Scenario tests (loss, gain, gaps, partial logging) and test report.

## Phase 5: Barcodes and Labels
46. [pending] Barcode scanning (camera or webcam; `BarcodeDetector` or WASM fallback) and typed digits; EAN-13, EAN-8, UPC-A and UPC-E, normalised to GTIN-13.
47. [pending] Lookup: local cache, then Open Food Facts (regional subdomain, identifying `User-Agent`); optional FoodData Central Branded Foods with the user's own key, off by default, with its CSP entry.
48. [pending] Label photo extraction with regional schemas (United States Nutrition Facts, United Kingdom and EU back-of-pack with salt, Hong Kong 1+7) and the region-specific energy check (`docs/math.md` §2.5).
49. [pending] Label-parsing tests (English and Chinese; one sample set per region; per serving and per 100 g; kJ and kcal) and test report.

## Phase 6: Photo Portions
50. [pending] Capture: two-shot flow with card guide; overhead shot, then side shot with a 30–60° tilt target and 45° mark; second shot skippable; roles assigned from the measured angle, not the order; desktop pairing of two dropped or pasted files.
51. [pending] Card corners: AI proposal, local sub-pixel refinement by fitting the four edges at full resolution (handles rounded ID-1 corners), draggable adjustment, aspect-ratio check, consistent corner order across the pair with the half-turn ambiguity resolved by outline overlap (`docs/math.md` §1.1, §1.6).
52. [pending] Camera focal length and pose from the card: joint estimate over the pair, EXIF prior for file photographs, per-device running median, refusal of ultra-wide photographs; camera height, nadir and viewing angle (`docs/math.md` §1.4–1.5).
53. [pending] Homography, polygon and mask areas, exact parallax correction from the pose (`docs/math.md` §1.2, §1.5).
54. [pending] Outline parser for masks, polygons and boxes, with items matched across the two images.
55. [pending] Volume models: mounds, bowls, counted items; two-view height estimator with shape-class bisection, occlusion fallback and bowl fill level from the oblique view (`docs/math.md` §1.6, §2).
56. [pending] "I weighed it" calibration: bias and split-conformal width (`docs/math.md` §4).
57. [pending] Accuracy evaluation tool with plots (`tools/eval`, Python), including synthetic two-view scenes with known pose and height (`docs/math.md` §7).
58. [pending] Geometry tests and test report.

## Phase 7: Progress Photos
59. [pending] Encrypted photo store.
60. [pending] Guided capture: poses, ghost overlay, level, timer.
61. [pending] Compare view (side by side, slider).
62. [pending] Test proving progress-photo bytes never appear in a network request; test report.

## Phase 8: Hardening and Release
63. [pending] Accessibility audit (keyboard, screen reader, contrast, reduced motion).
64. [pending] Translations: English with United Kingdom and United States spelling variants, and Traditional Chinese (Hong Kong).
65. [pending] `docs/private-wrapper.md` and overlay build option.
66. [pending] Cross-device sync, as decided in task 8.
67. [pending] Final documentation pass and links; full test report.
68. [pending] Production deploy to GitHub Pages.

## Phase 9: Native iOS (requires a Mac)
69. [pending] Capacitor shell.
70. [pending] Plugins: Keychain, Face ID, HealthKit read/write with deduplication, Vision card corners, VisionKit barcodes, local notifications.
71. [pending] App Store readiness (privacy labels, permission strings); decide on the Apple Developer Program.

## Phase 10: LiDAR (requires a Mac and a LiDAR iPhone)
72. [pending] ARKit depth capture plugin.
73. [pending] Plane fit and depth-volume integration; cross-check with the card and with the two-photograph height.
74. [pending] Depth-volume tests on recorded samples.
