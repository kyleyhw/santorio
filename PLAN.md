# Santorio — Plan

> **Status:** agreed direction, not built yet. Source: [`SPEC.md`](SPEC.md).
> Where the two disagree, this plan wins.

> ⚠️ **This repository is public.** Everything committed here (code, issues, PR
> text, commit messages, test data, screenshots) is visible to anyone, and git
> history keeps deleted files. **No personal data, stats, photos, API keys,
> real meal logs or identifying config ever goes in this repo.** Personal
> setup belongs in a private wrapper that each person creates for themselves
> (§5), and personal data belongs only on the device.

---

## 1. Decisions

| Topic | Decision |
|---|---|
| Platform | **Web app first** (installable, works offline), same codebase wrapped as a native iOS app later |
| Hosting | **GitHub Pages**, deployed by GitHub Actions from this repo |
| UI | **Svelte + TypeScript + Vite**: small, fast, little boilerplate |
| Native later | **Capacitor** shell with small Swift plugins (HealthKit, LiDAR, Keychain, Vision, notifications) |
| Data | Stored only in the browser (IndexedDB + OPFS), **always encrypted** |
| Lock | Optional **Face ID lock** (passkey) with a recovery passphrase; offered once, never forced |
| AI | Bring your own key. **Gemini** default, **Claude** available. Not needed to start using the app |
| Private wrapper | The public repo provides the hooks and a setup guide; **no private repo is created by us** |
| Sync | Not in v1. Encrypted backup file instead. End-to-end-encrypted sync later if wanted |

## 2. Principles

1. **Calm and friendly.** A new user should log their first meal within a
   minute of opening the app. One clear action per screen, plain language, no
   jargon, no streaks, badges, confetti or nagging. Features appear when
   they're relevant, not all at once (progressive disclosure).
2. **Do the essentials well.** Fewer features, each done properly. When in
   doubt, leave it out or put it under "More".
3. **Private by construction.** No server, no accounts, no analytics, no
   third-party scripts. The app is technically unable to send data anywhere
   except the AI provider you chose and Open Food Facts (barcode lookups).
4. **Honest numbers.** Every estimate shows its confidence; low-confidence
   meals show a range. Nothing pretends to be more precise than it is.
5. **Safe.** Targets never go below a sensible calorie floor; the app never
   suggests eating less in response to low energy; copy is neutral, never
   shaming.

## 3. What a new user sees

**First run (three short screens, all skippable except the first):**

1. *Your goal:* lose fat, maintain, or build muscle.
2. *About you:* height, weight, age, activity level, units. One sentence on
   why these are needed and that they never leave the device.
3. *Add to Home Screen:* a short illustrated tip (it keeps data safe on
   iPhone; see §4.3).

Then straight to **Today**. Nothing else is asked up front.

**Today** shows only: calories and protein left (large numbers), meals logged,
and a big **Log** button. Weight appears once you've entered one. The daily
checklist appears only if you set one up.

**Log** opens on the fastest method you've used before. Options: Search, Quick
add, Describe, Barcode, Photo. Search, Quick add and Barcode-from-cache work
without any AI key.

**AI key setup happens just in time:** the first time you tap Photo or
Describe, a short guide explains what's sent, what it roughly costs and how to
get a key (about two minutes). The free-tier training caveat is mentioned
there in one line.

**Face ID lock** is offered once, after a few days of use, in a single dismissible
card. It's always available in Settings.

## 4. Architecture

### 4.1 Layout

```
santorio/
  README.md  PLAN.md  SPEC.md  PRIVACY.md  SECURITY.md  LICENSE (MIT)
  docs/
    private-wrapper.md      how to set up your own private wrapper (§5)
    threat-model.md
    algorithms/             trend, adaptive maintenance, homography, bowls
  packages/
    core/       pure TypeScript, no browser APIs: nutrition math, units,
                homography, volume, uncertainty, trend, adaptive targets,
                schemas, pack validation
    ai/         provider interface, Gemini + Claude, prompts, response parsers
    vault/      encryption, keys, backup file format
    storage/    IndexedDB/OPFS repositories (behind interfaces)
    platform/   device capabilities (camera, motion, health, notifications,
                barcode, secure storage): web versions now, native later
    packs/      public data packs: hk-starter, densities, reference objects
  apps/
    web/        the app (screens, components, service worker)
    ios/        (later) Capacitor shell + Swift plugins
  fixtures/     synthetic test data only
  tools/        pack builder, accuracy eval runner, EXIF checker
```

`core` has no browser dependencies, so it's identical on web and iOS and
easy to test.

### 4.2 Device capabilities

Each capability is an interface with a web version now and a native version
later. The UI checks availability and hides what isn't there instead of
showing disabled buttons.

| Capability | Web | Native (later) |
|---|---|---|
| Camera, tilt indicator | `getUserMedia`, DeviceOrientation | same / CoreMotion |
| Card corners | AI finds them; user can drag to adjust | Apple Vision |
| Barcode | `BarcodeDetector` if available, else a small self-hosted WASM decoder loaded on demand | VisionKit |
| Apple Health | Not available (optional import of Health's export file, later) | HealthKit |
| LiDAR | Not available | ARKit depth |
| Lock | Passkey (Face ID) | Face ID + Keychain |
| Reminders | Add-to-calendar file (`.ics`) | Local notifications |

### 4.3 Storage and encryption

- Records in IndexedDB, photos in OPFS, **all encrypted** with AES-GCM via the
  browser's built-in WebCrypto.
- Without a lock: the key is a non-extractable WebCrypto key kept by the
  browser. It protects against copying raw files, not against someone using
  your unlocked phone.
- With a lock: the key is wrapped by a passkey (Face ID, via WebAuthn PRF)
  and by a recovery passphrase (PBKDF2, built-in). Locked data is unreadable.
- Every record has an ID, timestamps, a deleted flag and a schema version, so
  migrations and future sync don't need a rewrite.
- **Safari caveat:** Safari may clear a website's data after 7 days without a
  visit unless it's added to the Home Screen. Hence: the Home Screen tip in
  onboarding, `navigator.storage.persist()`, and a gentle monthly backup
  reminder in Settings (not a pop-up).

### 4.4 AI

- One `AIProvider` interface: analyze plate, read label, describe → items,
  find card corners.
- Responses use a fixed JSON schema, are validated, retried once, then fall
  back (masks → polygons → boxes → manual).
- Only what's needed is sent: the food photo (downscaled to ~1024 px,
  re-encoded so all metadata is gone), the optional note and restaurant name.
  Never body stats, goals, history, progress photos or location.
- Model names live in one config file. Prompt version is stored with each
  result.
- Direct calls from the browser: Claude needs the
  `anthropic-dangerous-direct-browser-access` header; Gemini accepts browser
  calls with a key. The setup guide recommends a dedicated key with a spending
  limit.
- Approximate cost per photo shown in Settings.

## 5. Public platform, private wrapper

The public repo stays generic. Personal or regional content plugs in as
**packs**: JSON files checked against a published schema.

| Pack | Contains |
|---|---|
| `foods` | Foods, per-100 g nutrition, typical portions, source and license |
| `reference-objects` | Known-size objects for scale (a card is built in) |
| `checklist` | Daily checklist items, optionally matched to Health workouts |
| `restaurants` | Places you go and what you usually order (names only) |
| `presets` | Default targets, units, reminder schedule |

Two ways to use packs:

1. **Import in the app** (Settings → Packs). Stored inside the encrypted
   vault. No git, no build. This is enough for most people.
2. **Private wrapper repo**, for people who want their own deployment.
   `docs/private-wrapper.md` will explain how to create one: a private repo
   with this repo as a pinned git submodule, an `overlay/` folder of packs
   and settings, and a build that runs with `SANTORIO_OVERLAY=overlay`.

The guide will also say plainly: **a private GitHub repo is not a place for
health data.** Logs, weights, measurements and progress photos stay on the
device and in encrypted backup files.

Public-repo rules:

- Defaults are generic (e.g. the checklist ships empty, with suggestions).
- Tests use clearly synthetic people and data.
- `.gitignore` covers `.env*`, `*.local.*`, `overlay/`, `private/`,
  `*.santorio-backup`, `eval-data/`.
- CI rejects image fixtures with EXIF/GPS data and key-shaped strings.
- Enable GitHub secret scanning and push protection in repo settings.
- Commit with your GitHub `noreply` email address.

## 6. Privacy and security

- **Content Security Policy** (as a `<meta>` tag, because GitHub Pages can't
  set headers): scripts only from the app itself; network only to the
  configured AI provider and Open Food Facts. No CDNs, no web fonts from
  elsewhere.
- **GitHub Pages origin:** the app is served from
  `<user>.github.io/santorio/`. Every other GitHub Pages project on that
  account shares the same origin, so in principle can read the same browser
  storage. Fine while all of those sites are your own; encryption + lock
  limits the damage; a custom domain removes the issue later. Anyone who
  forks and deploys gets their own origin.
- **Network activity** (Settings → Privacy): a plain list of every outbound
  request: time, destination, purpose, size, never contents.
- **Progress photos:** never sent anywhere, never saved to Photos, exported
  only when you tap Export (with optional face blur or crop below chin).
- **Meal photos:** discarded after logging unless you keep them.
- **Location:** only used if you turn on restaurant lookup (not in v1).
  Coordinates are never stored or sent to the AI.
- **No logging** of request contents, images or keys.
- **Delete all data:** one button, clears everything including caches.
- `PRIVACY.md` (plain-language policy) and `SECURITY.md` (how to report
  issues) ship in Phase 0.

## 7. Scope

### 7.1 v1 (web)

| Area | What's included | Notes on the spec |
|---|---|---|
| Logging | Search, quick add, describe, favorites, recents, repeat a meal, saved meals | |
| Plate | Per-item grams/kcal/macros, confidence, live edit, "ate part of it", ranges when unsure, **"I weighed it"** field | Weighed entries quietly improve your personal estimates over time |
| Barcode | Scan → local cache → Open Food Facts → label photo → confirm and save | Label check: energy ≈ 4P + 4C + 9F, flags mismatches; handles kJ/kcal, per-serving/100 g, EN/中文 |
| Photo (card) | Capture guide, tilt indicator, card corners (AI + drag), perspective correction, area × height × density, bowls, count-based items | See §7.3 |
| HK food | Starter pack of common dishes; milk-tea/coffee options (少甜, 走甜, 茶走, 走冰) as item options | Check CFS data terms before bundling; USDA FoodData Central (public domain) as a generic fallback |
| Body | Weight and waist entry, smoothed trend, weekly average | |
| Targets | Mifflin–St Jeor start, adaptive maintenance with guardrails, protein g/kg, target history with reasons | See §7.3 |
| Progress photos | Guided front/side/back, ghost overlay, level, timer, compare slider | Time-lapse later |
| Check-in | Optional checklist + optional 1–5 energy/performance | Off until you set it up |
| Reminders | Add-to-calendar (`.ics`) for weigh-in, waist, photos, backups | |
| Data | Encrypted backup/restore, JSON/CSV export, delete all | Backup is in v1 because of Safari storage clearing |
| Languages | English + 繁體中文 | Built for translation from the start |

### 7.2 Later

- **Native iOS app:** HealthKit read/write with deduplication, Face ID,
  automatic card detection, native barcode scanner, local notifications,
  App Store readiness.
- **LiDAR** depth volume, cross-checked with the card.
- **Progress time-lapse** and before/after card (manual two-tap alignment
  first; automatic pose alignment if it proves worth a dependency).
- **Restaurant lookup** by location (opt-in).
- **Apple Health export import** on web.
- **Weekly review**, a quiet summary of trend, expenditure, adherence and
  protein.
- **Importers** from other trackers' CSV exports.
- **End-to-end-encrypted sync** to storage you own.
- **Leftovers photo** (subtract what's left on the plate).
- **More regional packs**, contributed via PR.

### 7.3 Spec gaps, resolved

- **Height is the weakest input** in the card method. Use per-food height
  priors, a simple "flat / heaped / mounded" choice the user can change, and
  the optional 45° photo for tall food.
- **Bowls:** rim diameter from the card + bowl type (rice / noodle / soup) +
  a fill-level slider. Marked lower confidence.
- **Discrete foods** (dumplings, siu mai, buns, eggs, slices) are counted, not
  measured: count × typical piece weight is more accurate.
- **Claude returns polygons/boxes, not masks.** The pipeline accepts all three
  and sets confidence accordingly.
- **Uncertainty** flows through the math (corner, height, density ranges) to a
  kcal range per item and per meal.
- **Adaptive maintenance:** smoothed trend weight; intake minus trend change ×
  ~7,700 kcal/kg, blended with the Mifflin–St Jeor estimate according to how
  much data there is. Days not fully logged are excluded, never counted as
  zero (a day can be marked "partially logged"). Guardrails as specced plus a
  calorie floor.
- **Sex field** asks which formula to use (male / female / average), since
  that's all the equation needs.
- **Day boundary** is configurable (e.g. a late dinner at 1 am counts
  toward the previous day).
- **Energy-dip rule:** a clear, testable threshold, and only suggests eating
  more when you're in a deficit.
- **App Store note to verify later:** Apple's review guidelines have limited
  storing health information in iCloud, which affects the spec's CloudKit idea.
- **Open Food Facts contributions** need an OFF account; deferred.

## 8. Build phases

Each phase ends with a deployed, working app on GitHub Pages and tests for its
logic.

| # | Phase | Done when |
|---|---|---|
| 0 | **Foundations:** README, LICENSE, PRIVACY, SECURITY, `.gitignore`, CI (typecheck, tests, EXIF/secret checks), Pages deploy, monorepo skeleton, design tokens (dark mode, large type), CSP, threat model, private-wrapper guide | An empty app installs, works offline, and a test proves the CSP blocks an unexpected request |
| 1 | **Core logging:** onboarding, Today, search, quick add, describe, Plate, AI settings (Gemini, Claude), HK starter pack, vault, backup/restore, export, delete all, network activity list | You can log a full day, back up, wipe and restore in a fresh browser |
| 2 | **Body and targets:** weight, waist, trend, Plan screen, adaptive maintenance, target history, check-ins, calendar reminders, Face ID lock | Trend/adaptive logic passes scenario tests (cut, bulk, gaps, partial logging) |
| 3 | **Barcodes and labels** | Label-reading tests pass (EN/中文, per-serving/100 g, kJ/kcal) |
| 4 | **Photo portions:** card capture, corners, perspective correction, outlines, volume, bowls, counts, ranges, "I weighed it" calibration, accuracy eval tool | Geometry tests pass; the eval tool reports error per method |
| 5 | **Progress photos:** encrypted store, guided capture, ghost overlay, compare | A test proves photo bytes never appear in any network request |
| 6 | **Native iOS shell** (needs a Mac; App Store needs US$99/yr) | Same app runs natively with HealthKit |
| 7 | **LiDAR** | Depth-volume tests pass on recorded samples |

## 9. Testing

- **Unit (core):** nutrition math, units, homography and polygon area, bowl
  volume, uncertainty, trend, adaptive maintenance, guardrails, label checks,
  AI response parsing (saved sample responses per provider), pack validation,
  backup round-trip.
- **End-to-end (Playwright, Chromium + WebKit):** onboarding → log → backup →
  wipe → restore; network tests against the CSP and the activity list.
- **Accuracy:** an eval tool runs the photo pipeline over photos with
  kitchen-scale ground truth. The repo ships it with synthetic/licensed
  images; your own photos stay in your private wrapper.

## 10. Dependencies

Chosen to keep the app small and self-contained. Nothing loads from a CDN;
nothing phones home.

| Purpose | Choice | Runtime? |
|---|---|---|
| UI | Svelte | Compiles away to plain JS |
| Build | Vite, TypeScript | Dev only |
| Tests | Vitest, Playwright | Dev only |
| Barcode on Safari | Small WASM barcode decoder (ZXing-based), self-hosted, loaded only when scanning | Yes, lazy |
| Native shell | Capacitor | Later |

Built in, no dependency: encryption and key derivation (WebCrypto), charts
(SVG), storage wrapper, translations, video export (canvas + MediaRecorder).
Any further dependency gets asked about first.

## 11. Remaining questions

None blocking. Later, when relevant:

- A custom domain instead of `github.io`? (Removes the shared-origin caveat.)
- Restaurant lookup by location: worth it, and which service?
- Apple Developer Program (US$99/yr) when we reach the native phase.
