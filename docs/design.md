# Design

This document records the agreed design of Santorio and the reasoning behind
each decision. The task list and status live in
[`PROJECT_PLAN.md`](../PROJECT_PLAN.md); the mathematics lives in
[`math.md`](math.md); the original skeleton spec is [`SPEC.md`](../SPEC.md).
Where this document and the spec disagree, this document is authoritative.

**This repository is public.** Everything committed here (code, issues, pull
request text, commit messages, test data, screenshots) is visible to anyone,
and git history retains deleted files. No personal data, body statistics,
photos, API keys, real meal logs or identifying configuration may be
committed. Personal configuration belongs in a private wrapper that each user
creates for themselves (§7); personal data belongs only on the device.

---

## 1. Decisions

| Topic | Decision | Status |
|---|---|---|
| Platform | Web application first (installable PWA, offline-capable), wrapped as a native iOS app later from the same codebase | Agreed |
| Browsers | Chrome on desktop is the primary development target; Safari and Chrome on iPhone must also work fully | Agreed |
| Hosting | GitHub Pages, deployed by GitHub Actions from this repository | Agreed |
| UI framework | React + TypeScript + Vite, with our own components and design tokens (no UI kit) | Chosen (§3) |
| Native shell | Capacitor with small Swift plugins (HealthKit, LiDAR, Keychain, Vision, notifications) | Agreed |
| Storage | Browser only (IndexedDB for records, OPFS for blobs), always encrypted | Agreed |
| Lock | Optional passkey lock (Face ID / Touch ID / Windows Hello) with a recovery passphrase; offered once, never forced | Chosen |
| AI | Bring your own key; Gemini default, Claude available; not required to start using the app | Agreed |
| Private wrapper | Public repo provides the hooks and a setup guide only; no private repository is created by the project | Agreed |
| Process | Style round first, then a single-pass build of the whole web app (§10) | Agreed |
| Cross-device sync | **Open**, see §11 | Open |

---

## 2. Principles

1. **Logging must be effortless.** This is the single most important
   requirement. Every logging path is measured in taps and keystrokes and has a
   budget (§5). If a step can be inferred, it is inferred; if it can be
   undone, it is not confirmed.
2. **Accuracy, and honesty about accuracy.** Every estimate carries a
   quantified uncertainty derived from an explicit error model (§6,
   [`math.md`](math.md)). Displayed precision never exceeds actual precision.
   Intervals are calibrated against ground truth when the user supplies it.
3. **Calm and friendly.** A new user logs a first meal within a minute. One
   primary action per screen, plain language, defined jargon, no streaks,
   badges, confetti or nagging. Features appear when relevant (progressive
   disclosure).
4. **Private by construction.** No server, no accounts, no analytics, no
   third-party scripts. The Content Security Policy makes it technically
   impossible for the page to contact any host other than the AI provider the
   user chose and Open Food Facts.
5. **Minimal implementation.** The simplest design that fully meets the
   requirement; no speculative abstraction or configuration.
6. **Safe.** Targets never go below a calorie floor; low energy never
   triggers a suggestion to eat less; copy is neutral and never shaming.

---

## 3. Framework choice

The request was for the most customisable framework. The degree of visual
customisation is determined less by the framework than by whether a
pre-built component library is used: with hand-written components and design
tokens (CSS custom properties), every pixel is ours under React, Svelte, Vue
or Solid alike. Among those, **React** is chosen because:

- it has the most escape hatches for unusual interactions (refs, custom hooks,
  fine control of rendering), which the camera overlays, corner-drag handles
  and live sliders need;
- it has the largest ecosystem of *headless* (unstyled) primitives, should a
  complex widget ever justify one;
- it has the most direct path to native: Capacitor runs it unchanged, and
  React Native remains an option if a fully native UI is ever wanted.

The trade-off is bundle size: React adds roughly 45 KB (gzipped) of runtime
that Svelte would not. For an app loaded once and then served offline from
the service worker, this is acceptable.

Styling: plain CSS with custom-property design tokens (colour, type scale,
spacing, radii, motion), defined after the style round. No CSS framework.

---

## 4. Desktop and mobile

The same app serves both form factors; layout adapts, behaviour does not
diverge.

| Aspect | Phone (< 768 px) | Desktop (≥ 1024 px) |
|---|---|---|
| Layout | Single column, bottom navigation, thumb-reachable primary action | Two panes: Today on the left, Log / Plate on the right; side navigation |
| Logging input | Universal input bar, camera button | Same bar, always focused; `/` focuses it from anywhere |
| Photos | Camera capture | Drag-and-drop, paste from clipboard, file picker, or webcam |
| Barcode | Camera | Webcam, or type the digits |
| Charts | Compact | Expanded, with hover read-outs |
| Install | Add to Home Screen | Chrome "Install app" |

The 768 px and 1024 px breakpoints are the conventional tablet-portrait and
small-laptop widths; the range between uses the phone layout with wider
margins. These are to be confirmed during the style round.

Browser capability notes (to be verified at build time, because support
changes):

- `BarcodeDetector` is available in Chrome on some operating systems only and
  not in Safari; a small self-hosted WASM decoder is the fallback everywhere.
- Passkey PRF (needed for the passkey lock) is supported by recent Chrome and
  Safari with platform authenticators; where unavailable, the lock falls back
  to the passphrase alone.
- `DeviceOrientationEvent` (tilt indicator) is absent on desktop; the
  indicator is hidden there.

---

## 5. Logging friction

Logging difficulty in existing trackers is the main problem this app exists
to fix. The design therefore treats friction as a measurable quantity with a
budget, enforced by end-to-end tests that script each path and count user
actions.

### 5.1 Universal input bar

One input on Today accepts everything:

- **Text**: `2 eggs, toast with butter, milk tea 少甜`. Known foods are
  matched locally and instantly; unknown fragments go to the AI (if a key is
  set) while the known ones are already on the Plate.
- **Numbers**: `450` logs a quick-add of 450 kcal; `450 30p` adds 30 g protein.
- **Photos**: drop, paste or shoot; analysis runs in the background.
- **Barcodes**: digits typed or scanned.

Dictation through the operating system's keyboard microphone works with no
extra code.

### 5.2 Inference instead of questions

- Meal type is inferred from the time of day (editable later, never asked).
- Portion defaults to the last portion used for that food.
- Suggestions on an empty bar are ranked by what the user usually eats at
  this time of day and day of week.
- AI analysis never blocks: a photo is logged immediately as *pending* and
  fills in when the result arrives.
- No confirmation dialogs; every action has undo.

### 5.3 Action budgets

| Path | Budget |
|---|---|
| Re-log a usual food or meal | 1 tap (from suggestions) |
| Log a known food by search | type ≤ 4 characters + 1 tap |
| Quick add calories | type number + Enter |
| Photo | shutter + 1 tap to accept |
| Barcode already scanned before | scan, auto-logged with undo |
| Edit a gram amount | 1 tap + drag or type |

Budgets are asserted in Playwright tests so regressions fail CI.

---

## 6. Accuracy and uncertainty

The full derivations are in [`math.md`](math.md). The design consequences:

- **Explicit error model.** Each estimated quantity (card-derived area, food
  height, density, energy density, portion from a description) carries a
  log-normal multiplicative uncertainty. These propagate analytically to each
  item's mass and energy, then to meal and day totals.
- **Display.** Values are shown as a point estimate with an 80 % interval
  when the interval is wide, e.g. `650 kcal (540–790)`, and rounded so that
  displayed digits do not exceed the precision (e.g. to the nearest 10 kcal
  when the interval half-width exceeds 50 kcal). Narrow-interval values (a
  weighed item, a barcode with a label) show a single number.
- **Calibration.** When the user enters a kitchen-scale weight ("I weighed
  it"), the pair (estimate, truth) is stored. These pairs are used to (a)
  correct systematic bias per food category and (b) rescale interval widths
  so that empirical coverage matches the nominal 80 % (split-conformal
  calibration). The app reports its measured accuracy per method.
- **Honesty in downstream estimates.** The adaptive maintenance estimate and
  the weight trend carry their own intervals, and target suggestions are made
  only when the interval is narrow enough to justify a change.
- **Discrete foods are counted, not measured.** Dumplings, buns, eggs and
  slices use count × typical piece mass, which is more accurate than volume.
- **Provenance.** Every value records its source (label, database, AI, user),
  the model and prompt version when applicable, and the method.

---

## 7. Public platform and private wrapper

The public repository is generic. Personal or regional content plugs in as
**packs**: JSON files validated against a published schema.

| Pack | Contents |
|---|---|
| `foods` | Foods, per-100 g nutrition, typical portions, source and licence |
| `reference-objects` | Objects of known size for scale (an ID-1 card is built in) |
| `checklist` | Daily checklist items, optionally matched to Health workouts |
| `restaurants` | Places and usual orders (names only, never coordinates) |
| `presets` | Default targets, units, reminder schedule |

Packs are used in one of two ways:

1. **Import in the app** (Settings, Packs). Stored in the encrypted vault; no
   git and no build. Sufficient for most users.
2. **Private wrapper repository**, for users who want their own deployment.
   `docs/private-wrapper.md` (written during the build) will describe it: a
   private repository containing this one as a pinned git submodule, an
   `overlay/` folder of packs and settings, and a build that runs with
   `SANTORIO_OVERLAY=overlay`.

A private GitHub repository is access-controlled but not end-to-end encrypted,
so it is not a place for health data. Logs, weights, measurements and progress
photos stay on the device and in encrypted backup files.

Public-repository rules:

- Defaults are generic; the checklist ships empty with suggestions.
- Tests use clearly synthetic people and data.
- `.gitignore` covers `.env*`, `.DS_Store`, `venv/`, `.venv/`, `*.local.*`,
  `overlay/`, `private/`, `*.santorio-backup`, `eval-data/`.
- `pre-commit` runs `detect-secrets` (configured) plus linters once code
  exists; CI rejects image fixtures containing EXIF or GPS metadata.
- GitHub secret scanning and push protection should be enabled in the
  repository settings.

---

## 8. Architecture

### 8.1 Repository layout (planned)

```
santorio/
├── README.md               overview, structure, documentation index
├── PROJECT_PLAN.md         phases, tasks, status
├── SPEC.md                 original skeleton spec
├── PRIVACY.md              plain-language privacy policy
├── SECURITY.md             vulnerability reporting
├── LICENSE                 MIT
├── docs/                   design, mathematics, guides, algorithm notes
├── packages/
│   ├── core/               pure TypeScript, no browser APIs: nutrition,
│   │                       units, homography, volume, uncertainty, trend,
│   │                       adaptive targets, schemas, pack validation
│   ├── ai/                 provider interface, Gemini, Claude, prompts, parsers
│   ├── vault/              encryption, key management, backup format
│   ├── storage/            IndexedDB/OPFS repositories behind interfaces
│   ├── platform/           device capabilities (web now, native later)
│   └── packs/              public data packs
├── apps/
│   ├── web/                the React application and service worker
│   └── ios/                (later) Capacitor shell and Swift plugins
├── tools/
│   └── eval/               accuracy evaluation (Python, uv)
├── fixtures/               synthetic test data only
└── tests/reports/          markdown test reports with runtimes
```

`core` has no browser dependencies, so it is identical on web and iOS and is
tested in isolation.

### 8.2 Device capabilities

Each capability is an interface with a web implementation now and a native
one later; the UI hides what is unavailable rather than showing disabled
controls.

| Capability | Web | Native (later) |
|---|---|---|
| Camera | `getUserMedia`, file input, drag-and-drop, paste | same |
| Tilt indicator | `DeviceOrientationEvent` (phones only) | CoreMotion |
| Card corners | AI proposes, user can drag | Apple Vision |
| Barcode | `BarcodeDetector` or WASM decoder | VisionKit |
| Apple Health | unavailable (optional import of Health's export file, later) | HealthKit |
| LiDAR | unavailable | ARKit scene depth |
| Lock | passkey (WebAuthn PRF) + passphrase | Face ID + Keychain |
| Reminders | add-to-calendar (`.ics`) | local notifications |

### 8.3 Storage and encryption

- Records in IndexedDB, photos in OPFS, all encrypted with AES-GCM-256 through
  WebCrypto (built into every browser; no dependency).
- Without a lock, the data key is a non-extractable WebCrypto key held by the
  browser. This protects against copying raw files, not against someone using
  the unlocked device.
- With a lock, the data key is wrapped by a passkey's PRF output and by a
  recovery passphrase (PBKDF2-HMAC-SHA256, 600 000 iterations, the OWASP
  recommendation for that construction [[1]](#ref-owasp)). Locked data is
  unreadable.
- Each record has a UUIDv7 identifier, creation and update timestamps, a
  deletion flag and a schema version, so migrations and future sync require no
  restructuring.
- Safari may clear a site's storage after seven days without a visit unless
  the app is on the Home Screen. Mitigations: an install tip during
  onboarding, `navigator.storage.persist()`, and a quiet backup reminder in
  Settings.

### 8.4 AI

- One `AIProvider` interface: analyse plate, read label, parse description,
  find card corners.
- Fixed JSON schemas; every response is validated, retried once with the
  validation error, then degraded (mask → polygon → box → manual).
- Data minimisation: only the food photo (downscaled to about 1024 px on the
  long edge and re-encoded, which removes all metadata), the optional note and
  the restaurant name are sent. Never body statistics, goals, history,
  progress photos or location.
- Model names live in one configuration file; the prompt version is stored
  with each result.
- Calls go directly from the browser. Claude requires the
  `anthropic-dangerous-direct-browser-access` header; Gemini accepts browser
  calls with a key. The setup guide recommends a dedicated key with a spending
  limit.
- An approximate cost per photo is shown in Settings.

---

## 9. Privacy and security

- **Content Security Policy** as a `<meta>` tag (GitHub Pages cannot set
  response headers): scripts only from the app's own origin; network only to
  the configured AI provider and Open Food Facts; no CDNs or external fonts.
- **GitHub Pages origin.** The app is served from
  `<user>.github.io/santorio/`. Every GitHub Pages project on the same account
  shares that origin and could therefore read its browser storage. This is
  acceptable while all such sites belong to the same owner; encryption and the
  lock limit exposure; a custom domain would remove the issue.
- **Network activity list** (Settings, Privacy): every outbound request with
  time, destination, purpose and size, never contents.
- **Progress photos** never leave the device, are never saved to the photo
  library, and are exported only on explicit request (optional face blur or
  crop below the chin).
- **Meal photos** are discarded after logging unless kept for calibration.
- **No logging** of request contents, images or keys.
- **Delete all data** clears every store, including caches.

---

## 10. Process

1. **Style round** (after the user's "go"). Interactive style tiles and
   key-screen mock-ups at phone and desktop widths: type, colour (dark and
   light), number display, uncertainty display, the input bar, the Plate.
   Iterated with the user until the design tokens are fixed.
2. **Single-pass build** of the whole web app against
   [`PROJECT_PLAN.md`](../PROJECT_PLAN.md), with commits and pushes after each
   phase, tests and markdown test reports throughout. This deliberately
   replaces per-phase confirmation, at the user's request.
3. **Native phases** (iOS shell, HealthKit, LiDAR) require a Mac and are
   outside the single pass unless decided otherwise (§11).

---

## 11. Open questions

1. **Cross-device sync.** Using the app on both a phone and a desktop means
   two separate encrypted stores. Options:
   - *(a)* Encrypted backup file moved by hand. No new infrastructure; clumsy
     for daily use.
   - *(b)* End-to-end-encrypted sync through the user's own Google Drive
     (hidden app folder). Requires registering an OAuth client for the Pages
     origin; Google sees only ciphertext plus timing and size.
   - *(c)* Direct device-to-device transfer, paired by QR code. No third
     party holds data, but both devices must be online together, and
     connection set-up uses a public STUN server, which sees IP addresses.

   Recommendation: (a) in the first build as the baseline, (b) next.
2. **Single-pass scope.** Recommendation: the complete web app; the iOS shell
   and LiDAR follow on a Mac.
3. **Calorie floor value.** It is a design guardrail rather than a
   physiological threshold. Candidate: the larger of the estimated resting
   energy expenditure and a fixed minimum, to be agreed in the style round.

---

## References

<span id="ref-owasp">[1]</span> OWASP Foundation. *Password Storage Cheat Sheet*, section on PBKDF2. [Link](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
