# Santorio — Plan

> **Status:** proposal for discussion. Nothing here is built yet.
> Source: [`SPEC.md`](SPEC.md) (skeleton spec). Where the two disagree, this plan
> is the proposal on the table; decisions get recorded as ADRs in `docs/adr/`
> once agreed.

> ⚠️ **This repository is public.** Everything committed here, including issues,
> PR text, commit messages, test fixtures and this plan, is visible to anyone,
> forever (git history keeps deleted files). **No personal data, stats, photos,
> API keys, real meal logs or identifying config ever goes in this repo.**
> Personal configuration lives in a separate private wrapper (§2) or, better,
> only on the device.

---

## 0. TL;DR

1. **Web first, native later, one codebase.** Build Santorio as an installable,
   offline-first **PWA** in TypeScript. Keep all logic (nutrition math,
   homography, volume, trend, adaptive targets, AI parsing, crypto) in a pure,
   DOM-free `core` package. Later, wrap the same app in a **native iOS shell**
   (Capacitor) with small Swift plugins for HealthKit, LiDAR, Keychain, Vision
   and local notifications, then ship to the App Store if it's worth it.
2. **Privacy by architecture, not by promise.** No backend, no accounts, no
   analytics. Data is **encrypted at rest in the browser** with a key that never
   leaves the device (passkey/Face ID via WebAuthn PRF, or a passphrase). Strict
   Content-Security-Policy so the app *cannot* talk to any host except the AI
   provider the user chose and Open Food Facts.
3. **Public engine, private wrapper.** The public repo is generic and has
   extension points ("packs"). A private repo (e.g. `santorio-private`) holds
   personal presets, a private accuracy-evaluation dataset and deploy config.
   Actual health data lives in **neither** repo: only on the device and in
   end-to-end-encrypted backups.
4. **Gaps the web pivot creates** (HealthKit, LiDAR, card detection without
   Apple Vision, MapKit, reminders without a server, Safari storage eviction)
   all have concrete workarounds below.
5. **Ambitious additions:** personal accuracy calibration with a kitchen scale,
   an accuracy-evaluation harness, uncertainty carried end-to-end, count-based
   logging for dim sum and shared dishes, cha chaan teng drink modifiers,
   leftovers photo, local-model AI provider (zero data leaves your network),
   Apple Shortcuts bridge to Health from the web app, importers from other
   trackers, and more (§7).

---

## 1. What changes versus the spec

| Spec says | Plan proposes | Why |
|---|---|---|
| Native SwiftUI, iOS 17+, needs a Mac | **PWA first** (TypeScript), native iOS shell later | You want it usable on the web now; one codebase scales to iOS later without a rewrite |
| SwiftData + iOS Data Protection | IndexedDB/OPFS + **app-level AES-GCM encryption** | Browsers have no Data Protection; we encrypt ourselves |
| Keys in Keychain | Keys encrypted inside the local vault (web); Keychain via plugin (native) | Same guarantee: never in code, never on a server |
| Face ID lock | **Passkey with WebAuthn PRF** unlocks the vault (Face ID on iPhone), passphrase fallback | Real cryptographic lock, not just a UI gate |
| HealthKit read/write | Web: **Apple Health export import** + **Apple Shortcuts bridge**; native: HealthKit plugin | HealthKit is native-only |
| Apple Vision card detection | AI returns corners + **user-draggable corner handles**; on-device detector later | No Vision framework on the web |
| VisionKit barcode scanner | Web barcode decoder (dependency decision, §10) or photo-of-barcode via AI | `BarcodeDetector` is not available in Safari (verify at build time) |
| MapKit POI search | Manual name + "recent restaurants" first; opt-in OpenStreetMap lookup or MapKit JS later | MapKit JS needs an Apple developer token |
| Local notifications | **Calendar (.ics) reminders** + Shortcuts automations on web; real local notifications native | Web push needs a push server; we have none |
| iCloud CloudKit sync | **End-to-end-encrypted** sync adapter later (user's own storage); E2EE backup file first | Provider-agnostic; also see App Store note in §6.10 |
| LiDAR | Native-only phase (Swift plugin) | ARKit depth isn't exposed to the web |

Things that stay exactly as specced: BYO AI key, no backend, editable Plate,
per-item results, reference-card method, adaptive maintenance with guardrails,
progress photos never sent to AI, metadata stripping, delete-all.

---

## 2. Public repo, private wrapper

### 2.1 Rules for the public repo

- No personal values anywhere: not in defaults, tests, screenshots, fixtures,
  commit messages or issues. Tests use **synthetic personas** (e.g.
  `persona-a.json`, clearly fake).
- The spec's default checklist ("climbing, push/legs, ab work, daily walk") is
  personal. Public defaults become generic (e.g. "Workout", "Walk",
  "Stretch"); your exact list moves to a private pack.
- `.gitignore` covers `.env*`, `*.local.*`, `private/`, `overlay/`,
  `*.santorio-backup`, `eval-data/`, and any build output.
- CI guardrails: secret scanning (enable GitHub secret scanning + push
  protection in repo settings), a check that image fixtures have no EXIF/GPS,
  and a lint rule blocking hardcoded API-key-shaped strings.
- Commit authorship: commits expose the author email. Use your GitHub
  `noreply` address for commits to this repo.
- Screenshots in docs use synthetic data only.

### 2.2 Extension points the public app exposes

The public code is a generic engine. Anything personal or regional plugs in via
**packs**: versioned JSON files validated against a public schema.

| Pack type | Contents | Example |
|---|---|---|
| `foods` | Foods with per-100 g nutrition, typical portions, provenance, license | `hk-starter` (public), `my-usuals` (private) |
| `densities` | g/ml by food category | public default table |
| `reference-objects` | Known-size objects for scale | ID-1 card (public), your own lunchbox (private) |
| `checklist` | Daily checklist templates + Health workout mappings | your training list (private) |
| `restaurants` | Named places + their usual items | your regular spots (private) |
| `prompt-addenda` | Extra context appended to AI prompts | "assume HK portion sizes" (public) |
| `settings-presets` | Default targets, units, reminder schedule | private |

Packs load two ways:

1. **Runtime import (primary):** import a pack file in Settings; it's stored
   inside the encrypted vault. No git, no build, nothing on any server.
2. **Build-time overlay (optional):** the private repo builds the public app
   with `SANTORIO_OVERLAY=../santorio-private/overlay` to bundle your packs and
   deploy config into your personal deployment.

### 2.3 What the private wrapper repo contains

```
santorio-private/            (private GitHub repo)
  overlay/packs/*.json       personal presets: checklist, usual foods, restaurants
  overlay/deploy/            your hosting config (custom domain, headers)
  eval-data/                 your meal photos + kitchen-scale ground truth (§9.3)
  scripts/                   build/deploy wrapper around the public repo
  santorio/                  public repo as a git submodule (pinned commit)
```

**Not** in the private repo either: daily logs, weights, body measurements,
progress photos, API keys. A private GitHub repo is access-controlled, not
end-to-end encrypted, so it's not a health-data store. Those stay on-device
and in your encrypted backups. (If you ever want them versioned, use the app's
encrypted backup file, which is safe to store anywhere.)

---

## 3. Privacy and threat model

To be written up properly as `docs/threat-model.md` in Phase 0. Summary:

| Adversary | What they could want | Mitigation |
|---|---|---|
| **The developer / repo** (us) | Anything | No backend, no telemetry, no accounts; public code is auditable; CSP forbids unexpected hosts |
| **AI provider** | Food photos, prompts | Data minimization: send only the food photo (downscaled, re-encoded, no metadata), the optional note and restaurant name. **Never** body stats, goals, history, progress photos or location. Paid tiers / no-training; free-tier warning. Local-model option (§7) |
| **Network observer** | Traffic | HTTPS only; HSTS; no third-party CDNs or fonts |
| **Someone with your phone** | Logs, photos | Vault encrypted at rest; passkey/Face ID or passphrase unlock; auto-lock timeout; "hide numbers" mode |
| **Cloud/backup provider** | Backups, sync data | Client-side encryption; provider only sees ciphertext |
| **Compromised hosting / supply chain** | Inject exfiltration code | Minimal dependencies, lockfile, strict CSP with `connect-src` allowlist, Subresource Integrity, reproducible builds with published hashes, **self-hosting supported** (anyone can serve their own build) |
| **Other web pages / extensions** | Read data | Same-origin isolation; no data in URLs; no `postMessage` listeners without origin checks; dedicated origin (don't host on a shared `github.io` origin with other apps) |
| **Apple / OS** | Health data | Health data stays in Health; we only read/write approved types (native phase) |

Concrete privacy features:

- **Vault:** all records encrypted with AES-GCM-256 via WebCrypto. The vault
  key is wrapped by (a) a passkey's PRF output, (b) a passphrase via a slow KDF,
  or both. Decrypted data lives in memory only while unlocked.
- **Progress photos:** encrypted blobs in OPFS/IndexedDB, never sent anywhere,
  never added to Photos. Export is explicit, with optional face blur or
  crop-below-chin.
- **Meal photos:** discarded after logging by default; opt-in retention only
  for calibration (§7.1).
- **Location:** one-shot request, coordinates held in memory only for the
  lookup, never persisted, never sent to the AI.
- **Logging:** no request bodies, images or keys in console logs; a lint rule
  bans `console.*` in shipped code paths that touch them.
- **Privacy ledger (new):** a local, user-visible log of every outbound request
  (time, host, purpose, byte size, never contents) so you can audit exactly
  what left the device.
- **Prompt preview (new):** an "Inspect what will be sent" toggle shows the
  exact image and text before each AI call.
- **Delete all:** wipes the vault, OPFS, caches and service-worker storage;
  native also removes the app's Health samples if chosen.

---

## 4. Architecture

### 4.1 Stack (recommendation; dependencies need your OK, see §10)

- **Language:** TypeScript (strict).
- **UI:** React + Vite (largest ecosystem, easiest Capacitor path). Preact or
  Svelte are lighter alternatives if bundle size matters more.
- **Packaging:** npm workspaces (built in, no extra tooling).
- **Storage:** IndexedDB (records) + OPFS (blobs) behind a repository
  interface; a tiny wrapper rather than a database library.
- **Charts:** hand-rolled SVG (weight trend, waist, expenditure); no chart
  dependency.
- **Tests:** Vitest (unit) + Playwright (end-to-end; Chromium is already in
  this environment; WebKit for Safari behaviour).
- **Native later:** Capacitor iOS shell + Swift plugins.

### 4.2 Repository layout

```
santorio/
  SPEC.md  PLAN.md  README.md  LICENSE  PRIVACY.md  SECURITY.md  CONTRIBUTING.md
  docs/
    threat-model.md
    adr/                      decision records
    algorithms/               trend, adaptive TDEE, homography, bowl model
  packages/
    core/        pure TS, no DOM: units, nutrition math, homography, volume,
                 density, uncertainty, trend, adaptive targets, schemas,
                 Health-dedup logic, pack validation
    ai/          AIProvider interface; gemini, claude, openai-compatible;
                 prompts; response parsers (masks/polygons/boxes); cost estimates
    vault/       key management, encryption envelopes, backup file format
    storage/     repository interfaces + IndexedDB/OPFS implementation
    platform/    capability adapters (camera, motion, location, health,
                 notifications, secure storage, barcode) with web impls now
                 and native impls later
    packs/       public packs: hk-starter, densities, reference-objects
  apps/
    web/         the PWA (screens, components, service worker)
    ios/         (later) Capacitor shell + Swift plugins
  fixtures/      synthetic personas + synthetic/licensed test images only
  tools/         pack builders, eval runner, EXIF checker
```

### 4.3 Platform adapters (how web and native share one app)

Each device capability is an interface in `platform/` with a capability flag.
The UI asks "is X available?" and degrades gracefully.

| Capability | Web (now) | Native iOS (later) |
|---|---|---|
| Camera | `getUserMedia` / file capture | same, or native camera plugin |
| Tilt / level | `DeviceOrientationEvent` (permission prompt on iOS) | CoreMotion |
| Card corners | AI corners + manual handles; optional JS detector | Apple Vision rectangles |
| Barcode | JS/WASM decoder or AI from photo | VisionKit `DataScannerViewController` |
| Health read | Apple Health export import; Shortcuts bridge | HealthKit |
| Health write | Shortcuts bridge | HealthKit |
| Depth | n/a | ARKit scene depth plugin |
| Secure key storage | Vault (PRF/passphrase) | Keychain + Secure Enclave |
| Biometric lock | Passkey (WebAuthn PRF) | LocalAuthentication |
| Reminders | `.ics` calendar export; Shortcuts automations | UNUserNotificationCenter |
| Body pose (time-lapse) | WASM pose model (dependency) or manual align | Apple Vision body pose |
| Video export | WebCodecs or canvas + MediaRecorder | AVFoundation |

### 4.4 Data model

Same entities as the spec (`UserProfile`, `Goal`, `Food`, `Meal`, `MealItem`,
`WeightEntry`, `MeasurementEntry`, `ProgressPhoto`, `CheckIn`,
`TargetHistory`), plus these changes:

- Every record: `id` (UUIDv7), `createdAt`, `updatedAt`, `deletedAt`
  (tombstone), `schemaVersion`. This makes **future sync and migrations** possible
  without a rewrite.
- `MealItem.nutrition` is a snapshot **with uncertainty**: `{kcal: {est, lo, hi}, ...}`.
- `MealItem.provenance`: method (card, lidar, ai-visual, barcode, label, describe,
  search, manual), model + prompt version, pack + version.
- `Food.source` + `Food.license` (OFF is ODbL, USDA is public domain, CFS terms TBD).
- New: `DayLog` (completeness flag: "fully logged" / "partial" / "not logged";
  this matters for adaptive maintenance), `Pack`, `CalibrationSample` (§7.1),
  `PrivacyLedgerEntry`, `ReferenceObject`, `Restaurant` (name only, no coordinates).
- Day boundary is configurable (e.g. day ends at 4 am for late dinners) and
  timezone-aware for travel.

### 4.5 AI layer

- `AIProvider` interface: `analyzePlate`, `readLabel`, `describeToItems`,
  `findCardCorners`, `estimateDensity`. Each returns typed, schema-validated
  results with per-field confidence.
- Providers: **Gemini** (default; segmentation masks), **Claude** (polygons/boxes,
  strong label reading and reasoning), **OpenAI-compatible** (covers local
  models via Ollama/LM Studio/llama.cpp and other hosts).
- Browser specifics: Claude API needs the
  `anthropic-dangerous-direct-browser-access` header for direct browser calls;
  Gemini supports browser calls with an API key. Recommend users create a
  **restricted key** (API-scoped, spending cap/budget alert) dedicated to Santorio.
- Model names live in one config file, chosen from current provider docs at
  build time; prompt templates are versioned and recorded on every result.
- Validate → retry once with the validation error → fall back to boxes →
  fall back to manual.
- Cost estimate per call from token counts, shown per photo and as a monthly
  running total (stored locally).
- Offline queue: a photo taken offline is held (encrypted) and analyzed when
  back online, or logged manually now.

---

## 5. Web versus native capability matrix

| Feature | Web PWA | Native shell |
|---|---|---|
| Logging (search, quick add, describe, Plate) | ✅ | ✅ |
| Reference-card portions | ✅ (manual-adjustable corners) | ✅ (auto corners) |
| Barcode | ✅ (decoder dependency) / AI fallback | ✅ |
| Label photo | ✅ | ✅ |
| Weight / waist trend, adaptive targets | ✅ | ✅ |
| Apple Health | ⚠️ import + Shortcuts bridge | ✅ full |
| Progress photo studio | ✅ | ✅ |
| Time-lapse | ⚠️ (pose model dependency or manual align) | ✅ |
| Reminders | ⚠️ calendar / Shortcuts | ✅ |
| LiDAR | ❌ | ✅ (Pro models) |
| Biometric lock | ✅ passkey PRF (iOS 18+ for PRF; verify) | ✅ |
| Works offline | ✅ (service worker) | ✅ |

**Safari storage caveat:** Safari can delete website storage after 7 days
without interaction **unless the PWA is installed to the Home Screen**.
Onboarding must push installing it, call `navigator.storage.persist()`, and
nudge backups. This is the biggest data-loss risk of web-first.

---

## 6. Section-by-section review of the spec: gaps and proposals

### 6.1 Goals (§1)
- Add explicit **non-goals**: no social features, no medical advice, no
  meal-plan generation that overrides the user, no ads ever.
- Add a **safety goal**: eating-disorder-aware design (calorie floor, no
  shaming copy, a "numbers-light" mode that hides calories and shows only
  protein and adherence). Suggest a floor of max(BMR estimate, a sensible
  minimum), and never let adaptive targets go below it.

### 6.2 Platform and distribution (§2)
- Web-first resolves "needs a Mac" for now. Hosting: static host on a
  **dedicated domain** (e.g. Cloudflare Pages or GitHub Pages with a custom
  domain). Anyone can also self-host the same build.
- App Store later: US$99/yr; Capacitor apps with real native features
  (HealthKit, LiDAR) are fine under the minimum-functionality guideline.

### 6.3 AI provider (§2)
- Claude doesn't return segmentation masks. The card pipeline must therefore
  accept **polygons or boxes** with honest confidence, not assume masks.
- Add a **local model provider**: the strongest privacy option.
- Free-tier warning: also detect it where possible (error codes / quotas), not
  only when the user says so.

### 6.4 Privacy model (§2)
- Web replacement for Data Protection = the vault (§3).
- Clarify **what the AI sees** in a one-screen explainer during onboarding,
  with the prompt-preview toggle.
- Food photos may incidentally include faces, other people or receipts. Add an
  optional on-device **crop-to-plate** step before sending.

### 6.5 Photo portions (§3.1)
- **Card detection on web:** step 1 ask AI for corners; step 2 show draggable
  handles snapped to edges; step 3 (later) on-device detector. Check card
  aspect ratio (85.6:54 ≈ 1.585) after homography as a sanity check.
- **Height is the weakest link** in Method A. Proposals: the optional 45° photo
  becomes a guided step for mounded food; a "height hint" slider ("flat /
  heaped / mounded") the user can tweak; per-category height priors.
- **Bowls:** model as a spherical cap or truncated cone from rim diameter
  (from the homography) and a user-set **fill level** slider (¼ … full) plus
  bowl-depth presets (rice bowl, noodle bowl, soup bowl). Lower confidence.
- **Count-based items** beat volume for discrete food: dumplings, siu mai,
  slices, eggs, buns. AI returns counts; per-piece weights come from the pack.
- **Uncertainty propagation:** carry intervals (corner error → area error;
  height prior range; density range) through to a kcal range per item and
  per meal. Low-confidence items widen the meal range.
- **Mask/polygon parser** gets golden tests across known model output formats.
- **Reference objects:** register any rectangle (or circle: a coin or a known
  plate) with dimensions; stored as a pack.

### 6.6 Editable Plate (§3.3)
- Add **swap food** (pick a different database match), **split item**, and
  **merge items**.
- Show *why* a number is what it is: "220 g = 180 cm² × 1.6 cm × 0.75 g/ml".
- Unit toggles: g / oz, kcal / kJ (HK labels often show both).

### 6.7 Barcodes and labels (§3.4)
- Open Food Facts lookups send the barcode to OFF; list it in the privacy
  ledger. Cached results are personal use; bundling OFF data in the repo would
  bring ODbL attribution/share-alike obligations.
- Label reading: handle per-100 g vs per-serving vs per-package, kJ/kcal, and
  bilingual labels; validate (energy ≈ 4P + 4C + 9F within tolerance) and flag
  mismatches.
- OFF contribution: keep off by default; an OFF account is needed to submit,
  which conflicts mildly with "no accounts". Suggest deferring.

### 6.8 Restaurant context (§3.6)
- Web phase: manual entry + recent restaurants + per-restaurant "usuals".
- Opt-in lookup options to decide: OpenStreetMap Overpass (free, no key, sends
  coordinates to OSM's server) or MapKit JS (needs an Apple developer token).
  Either way, coordinates are never stored and never sent to the AI.

### 6.9 Weight trend and adaptive targets (§3.8)
- Trend: exponentially smoothed moving average or a simple Kalman filter;
  handle missing days and multiple weigh-ins per day.
- Maintenance estimate: average intake minus energy implied by trend change
  (≈7,700 kcal/kg, configurable), **blended with the Mifflin–St Jeor prior**
  weighted by data quantity and logging completeness. Unlogged days are
  excluded, never treated as zero.
- Show expenditure as a **range** and the target history with reasons.
- Sex field for Mifflin–St Jeor: offer "use male formula / female formula /
  average of both" so the field is about the equation, not identity.
- Guardrails as specced, plus the calorie floor (§6.1) and a "diet break /
  maintenance phase" option.

### 6.10 Apple Health (§3.10)
- Web phase:
  - **Import Apple Health export** (`export.zip` from the Health app) for
    weight, body fat, waist, steps, workouts: parsed entirely on-device.
  - **Shortcuts bridge:** a published Apple Shortcut reads today's weight /
    workouts and opens Santorio with the data in the URL **fragment** (never
    sent to the host), which the app imports and immediately strips from
    history. A second Shortcut writes nutrition back to Health. Verify Shortcut
    capabilities when we get there.
- Native phase: full HealthKit as specced; dedup by tagging samples with
  external UUID metadata and filtering by our source on import.
- **App Store note to verify:** App Review guidelines have restricted storing
  personal health information in iCloud. That may affect the CloudKit sync idea;
  E2EE sync via a user-chosen provider sidesteps the question either way.

### 6.11 Progress photo studio (§3.11)
- Web: ghost overlay via semi-transparent previous photo; level from
  DeviceOrientation; timer; compare slider. Time-lapse alignment needs a pose
  model (WASM, self-hosted, no network) or manual two-point alignment
  (tap shoulders + hips). Lighting normalization is plain canvas math.
- Add **consistency score** (pose match, distance, lighting) at capture time.

### 6.12 Check-ins and reminders (§3.12–3.13)
- Reminders on web: generate a **recurring `.ics` calendar file** (weigh-in,
  waist, photos, backup) that the user adds to their own calendar. No server.
- Energy/performance dip rule: define "noticeably" (e.g. 7-day mean down ≥ 0.75
  vs prior 14-day mean) and require the user to be in a deficit before
  suggesting more food.

### 6.13 Export and backup (§3.14)
- **Move this to Phase 1** because of Safari storage eviction.
- Backup format: versioned container, AES-GCM, key from passphrase via a slow
  KDF; includes photos; restorable on web and native.
- Exports: JSON (complete), CSV (per entity), and optional open formats.

### 6.14 Screens (§4)
- Add a **Lock** screen, **Privacy ledger** (in Settings), **Packs** manager,
  and **Weekly review**.
- i18n from day one: English + Traditional Chinese (Hong Kong). Food names
  stored bilingually in packs.

### 6.15 Security checklist (§7)
- Add: CSP, SRI, `Referrer-Policy: no-referrer`, `Permissions-Policy`,
  dependency review, reproducible build hash, `SECURITY.md` with a disclosure
  contact (a dedicated address, not a personal one).

---

## 7. Additional feature ideas (ambitious)

### 7.1 Accuracy
- **Kitchen-scale calibration:** "I weighed it" mode stores (estimate, truth)
  pairs per food category; the app learns **personal correction factors** and
  shows your real accuracy per method ("card method: median error 14%").
- **Leftovers photo:** shoot the plate after eating; the app subtracts what's
  left.
- **Accuracy evaluation harness:** a script runs the full pipeline over a
  ground-truth photo set and reports error by method/model/prompt version.
  Public repo ships the harness + synthetic/licensed images; your real set
  lives in the private repo. Prompt or model changes must not regress it.
- **Cross-method consensus:** when card + AI visual + database typical-portion
  disagree, show the spread and let the user pick.

### 7.2 Hong Kong specifics
- **Cha chaan teng drink modifiers:** 少甜 / 走甜 / 茶走 / 走冰 / 飛沙走奶, hot vs
  iced cup sizes, all adjusting sugar/milk.
- **Dim sum / shared-table mode:** photograph the table, set number of diners,
  or count pieces you ate ("3 har gow, 2 siu mai").
- **Menu photo:** photograph a menu; the app pre-fills candidates.
- **Set-meal templates:** e.g. "breakfast set A" combos.

### 7.3 Insights (all computed on-device)
- **Weekly review:** trend, expenditure, adherence, protein days hit, waist
  change, check-in averages. Optional AI-written summary sending **only the
  aggregate numbers shown on screen**, opt-in, previewable.
- **Weight-noise annotations:** flag high-sodium or high-carb days and
  training days on the weight chart so spikes make sense.
- **Goal projection** with an uncertainty cone.
- **Waist-based body-fat estimate** (e.g. relative fat mass from height and
  waist), shown as an estimate trend.
- **"What fits?"**: suggests favorites/saved meals that fit the remaining macros.
- Optional (native, opt-in, sensitive): sleep and cycle data from Health to
  explain energy dips and water-weight swings.

### 7.4 Logging speed
- One-tap repeat yesterday's meal; recurring meals; "usuals" per restaurant.
- Recipe builder with cooked-yield adjustment (raw → cooked weight).
- Native later: home-screen widgets, Siri/App Intents, share-sheet import,
  Apple Watch quick add.

### 7.5 Privacy-maximal options
- **Local AI provider** (OpenAI-compatible endpoint on your own machine):
  photos never leave your network.
- **Quick-lock** gesture and auto-lock timer.
- **Privacy ledger** and **prompt preview** (§3).
- **E2EE sync** to user-owned storage later (WebDAV, Google Drive app folder,
  Dropbox, or iCloud Drive files on native), with records encrypted before upload.

### 7.6 Portability
- **Importers** from other trackers' CSV exports (MyFitnessPal, MacroFactor,
  Cronometer, Lose It!), so switching costs nothing.
- Pack system enables **other regions** (e.g. `sg-hawker`, `jp-konbini`)
  contributed by the community via PR, with provenance and license per value.

---

## 8. Revised build phases

Each phase ends with a deployable app, tests for its core logic, and an
updated `PRIVACY.md`.

| # | Phase | Contents | Exit criteria |
|---|---|---|---|
| 0 | **Foundations** | Repo hygiene (LICENSE, PRIVACY, SECURITY, CONTRIBUTING, .gitignore, CI, secret scanning, EXIF check), threat model, ADRs for stack/deps, monorepo skeleton, design tokens (dark mode, big numbers), CSP | CI green; empty PWA installs and works offline; CSP blocks an unexpected fetch in a test |
| 1 | **Core logging** | Vault (passphrase + passkey PRF), onboarding (generic), Today, search, quick add, describe-with-AI, editable Plate, AI settings (Gemini, Claude), HK starter pack, **encrypted backup/restore + JSON/CSV export**, delete-all, privacy ledger | Can log a full day offline (except AI), restore a backup on a fresh browser, unit tests for nutrition math and AI parsers |
| 2 | **Body and targets** | Weight & waist entry, trend, Plan screen, Mifflin–St Jeor, adaptive maintenance with guardrails, target history, check-ins, `.ics` reminders, Apple Health export import | Trend/adaptive algorithms pass synthetic-scenario tests (cut, bulk, missing days, partial logging) |
| 3 | **Barcodes and labels** | Barcode decode, OFF lookup + cache, label photo with energy-consistency check, cha chaan teng modifiers | Label parser golden tests (EN/中文, per-serving/100 g, kJ/kcal) |
| 4 | **Photo portions (card)** | Capture guide + tilt, corners (AI + handles), homography, outline parser, volume/density, bowl model, count items, uncertainty ranges, crop-to-plate, calibration mode, eval harness | Homography/area tests against known geometry; eval harness reports per-method error |
| 5 | **Progress studio** | Encrypted photo store, guided capture, ghost overlay, compare, manual align, time-lapse + before/after export with face blur | Photos verifiably never in any network request (Playwright test against the ledger + CSP) |
| 6 | **Native iOS shell** | Capacitor wrap, plugins: Keychain, Face ID, HealthKit read/write + dedup, Vision corners, VisionKit barcode, local notifications; App Store readiness (privacy labels, permission strings) | Same web code runs native; Health dedup tests |
| 7 | **LiDAR** | ARKit depth plugin, plane fit, depth-volume, card cross-check | Depth-volume tests on recorded fixtures |
| 8 | **Sync and polish** | E2EE sync adapter, i18n (EN / 繁中), accessibility audit, importers, weekly review, widgets | — |

Phases 3–5 can be reordered to taste; 0–2 should come first.

---

## 9. Testing and quality

### 9.1 Unit (core)
Nutrition math, unit conversion, homography + polygon area (known synthetic
quads), bowl volumes, uncertainty propagation, trend smoothing, adaptive
maintenance (scenario fixtures), guardrails, label-energy consistency, pack
schema validation, backup round-trip, Health dedup, AI response parsers
(golden files per provider format).

### 9.2 End-to-end
Playwright: onboarding → log → backup → wipe → restore; CSP/ledger tests that
assert no request goes to an unexpected host and no progress photo bytes ever
appear in a request.

### 9.3 Accuracy
The eval harness (§7.1). Public: synthetic geometry and licensed images.
Private: your real weighed meals.

---

## 10. Dependencies that need your approval

Per the spec, nothing gets added without asking. Proposed, all as decisions to
make in Phase 0 (each an ADR):

| Need | Option(s) | Alternative with zero deps |
|---|---|---|
| UI framework | React (or Preact / Svelte) | Vanilla web components (slower to build) |
| Build | Vite | — |
| Unit tests | Vitest | Node's built-in test runner |
| E2E | Playwright (dev only) | — |
| Barcode decode (web) | ZXing-based WASM decoder | AI reads the barcode digits from a photo |
| Passphrase KDF | Argon2 (WASM) | PBKDF2 via WebCrypto (built in, weaker per cost) |
| Pose detection (time-lapse) | Self-hosted WASM pose model | Manual alignment |
| Video encoding | MP4 muxer for WebCodecs | Canvas + MediaRecorder (WebM/MP4 varies) |
| Native shell | Capacitor | SwiftUI rewrite of the UI using the same algorithms/fixtures |

Runtime dependencies are kept tiny; nothing that phones home; everything
self-hosted (no CDN).

---

## 11. Open decisions

My recommendation is listed first.

1. **Web-first PWA + Capacitor later** vs native SwiftUI now (needs a Mac) vs
   React Native. → *PWA + Capacitor.*
2. **UI framework:** React vs Preact vs Svelte. → *React.*
3. **Vault unlock:** passkey PRF + passphrase fallback vs passphrase only.
   → *Both; passphrase required as recovery.*
4. **Default AI:** Gemini (masks) vs Claude. → *Gemini default, Claude
   one tap away, local option for privacy maximalists.*
5. **Hosting:** dedicated custom domain on a static host. Do you have a domain?
6. **Restaurant lookup on web:** manual only vs opt-in OSM vs MapKit JS.
   → *Manual + recents now; decide on lookup later.*
7. **Sync:** E2EE to user-owned storage (later) vs CloudKit. → *E2EE.*
8. **OFF contributions:** defer.
9. **Private wrapper:** create `santorio-private` now (submodule layout in
   §2.3)? Name it something else?
10. **Language:** English + 繁體中文 from Phase 1, or English first?
11. **Food data sources:** check CFS terms; USDA FoodData Central (public
    domain) as a generic fallback?

---

## 12. Next step

Once you've answered (or overridden) the open decisions, Phase 0 starts:
repo hygiene and guardrails first, then the monorepo skeleton and the ADRs,
with no feature code until those land.
