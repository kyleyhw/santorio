# Santorio — Skeleton Spec

Oct 15, 2026 · @mind goblin

> **Status:** original skeleton spec, kept as written. Where it conflicts with
> [`docs/design.md`](docs/design.md) (notably the web-first platform decision),
> the design document wins. Tasks are tracked in [`PROJECT_PLAN.md`](PROJECT_PLAN.md).

A free, open-source iPhone fitness and calorie tracker for any goal: losing fat, maintaining, or building muscle. It merges the best features of paid trackers, estimates portions with a reference card (or LiDAR on iPhone Pro models), syncs with Apple Health, and keeps every user's personal data on their own device. Name: Santorio (repo: santorio), after Santorio Santorio, the 1600s physician who weighed himself and everything he ate for decades.

Note: This is a skeleton, not a finished design. It's meant to be handed to Claude Code as a starting point, and the detailed design will be fleshed out there.

## 1. Goals

- No subscriptions. Each user brings their own pay-as-you-go AI key. No backend, no accounts.
- Public app, private data. The code is public. Each user's stats, logs and photos stay on their own device (except food photos sent to the AI they chose).
- Accurate-enough portions. A card of known size gives real-world scale on any iPhone; LiDAR depth adds precision on Pro models.
- Apple Health. Nutrition written to Health; weight, waist, steps and workouts read from it.
- Works in Hong Kong. Barcode lookup with a nutrition-label fallback; understands local dishes.
- Any goal: cut, maintain or bulk. Adaptive maintenance estimate, protein tracking, weight trend, waist and progress photos.

## 2. Key decisions

### Platform: native iOS (SwiftUI)

- Apple Health (HealthKit) and LiDAR depth are only available to native apps, so this is a SwiftUI app targeting iOS 17+.
- Building it needs a Mac with Xcode.
- Works on all iPhones; LiDAR features appear only on models that have the sensor.

### Distribution (owner to decide)

- Free: public GitHub repo (MIT license). Anyone with a Mac can build and install it. With a free Apple ID, the installed app must be re-signed from Xcode every 7 days.
- US$99/year: Apple Developer Program, needed for TestFlight or a public App Store listing. Note this is itself a yearly fee.
- Design the app to be App Store-ready either way (privacy policy, permission strings, no bundled keys).

### AI provider

- An `AIProvider` protocol with `GeminiProvider` and `ClaudeProvider` implementations, chosen in Settings. Default: Gemini, since it can return per-item outlines (segmentation masks), which both the card and LiDAR pipelines use.
- Use a paid tier. Paid Gemini and the Claude API don't use inputs for training by default. The free Gemini tier may, so show a warning if the user says they're on it.
- A Claude Pro/Max subscription does not include API access; the Claude API is billed separately, pay-as-you-go.
- Keys stored only in the iOS Keychain, never in code or the repo.

### Privacy model

- No server, no accounts, no analytics, no ads, no third-party SDKs that phone home.
- All data stored locally with SwiftData. Files use iOS Data Protection (`.complete`).
- Optional sync via the user's own iCloud private database (CloudKit). Off by default. Only that user's Apple ID can read it.
- Apple Health data stays in Health on the device; the app only reads and writes the types the user approves.
- Optional Face ID / passcode lock for the app.
- Progress photos are never sent to any AI service, never saved to the Photos library, and stored only in the app's sandbox.
- Food photos sent to the AI are downscaled and stripped of all metadata (EXIF, GPS) first.
- Never hardcode anyone's personal stats in the code. All personal values come from onboarding.

## 3. Features

Inspiration noted in brackets. Build these as our own implementations; don't copy any app's branding or assets.

### 3.1 Photo logging: portion methods

The camera screen picks the best available method automatically, and the user can switch:

1. Reference card (default, every iPhone)
2. LiDAR (optional, iPhone Pro models only)
3. AI visual estimate (fallback when neither is available; marked low confidence)

If LiDAR is on and a card is also in the shot, use both and cross-check: large disagreement lowers the confidence badge.

#### Method A: reference card

How the user shoots:

- Place an Octopus card or credit card flat on the table next to the plate. Both are ISO ID-1 size: 85.6 × 54 mm.
- Take one photo from roughly above (a slight angle is fine). Optional second photo from about 45° for tall or mounded food.
- An in-camera guide shows where to put the card, plus a tilt indicator from the gyroscope.

How the app estimates portions:

1. Detect the card's four corners with Apple's Vision rectangle detection (fall back to asking the AI for the corners).
2. Send the photo to the AI and ask for JSON: each food item's label, outline (mask or polygon) or bounding box, container type (flat plate, bowl, cup), and an estimated height.
3. Use the card's corners to compute a perspective correction (homography) for the table surface. This turns each item's outline into a real footprint area in cm².
4. Volume ≈ footprint area × average height (AI estimate, refined by the optional 45° photo).
5. Grams = volume × density from a bundled table (rice, noodles, leafy greens, meat, tofu, soup, etc.). Unknown food: ask the AI for a density and mark it as estimated.
6. Nutrition from per-100 g values (food database or AI), shown on the editable Plate (3.3).

#### Method B: LiDAR [SnapCalorie-style depth]

- Off by default; offered in Settings and on the camera screen only on devices with LiDAR.

1. Run an ARKit session with scene depth enabled.
2. On capture, save the RGB image, depth map, depth confidence map, camera intrinsics and gravity vector.
3. Fit the table surface: ARKit horizontal plane detection, or a RANSAC plane fit to the depth points.
4. Get per-item outlines from the AI as in Method A.
5. For each item, sum (height above the plane) × (pixel footprint area from depth + intrinsics) over the pixels in its outline, skipping low-confidence depth pixels. This measures real volume, so no height estimate is needed.
6. Convert to grams and nutrition as in Method A.

#### Edge cases (both methods)

- Bowls and soups: neither method sees below the rim. Use the rim diameter (from the card or depth) plus a bowl-shape model, and mark as lower confidence.
- AI outline formats vary between model versions. Write a parser that accepts the known formats, validates results, and falls back to bounding boxes.
- No usable outline: let the user tap each food item to segment it (on-device, using depth when available).
- Card not found and no LiDAR: fall back to AI visual estimation and suggest retaking with the card.
- Users can register a different reference object with known dimensions.

### 3.2 Photo + text context

- Optional note sent with the photo: "extra sauce", "shared with one other person", "fried not steamed".
- Optional restaurant name (3.6) added to the prompt.

### 3.3 Editable Plate [MacroFactor-style]

- Every AI result is split into separate items (e.g. rice 220 g, char siu 90 g, greens 80 g), never one opaque total.
- Each item shows grams, calories, protein, carbs and fat, plus a confidence badge and the method used.
- Grams editable by slider or keypad; nutrition updates live.
- Meal total shown as a range (e.g. 650–780 kcal) when confidence is low.
- "Ate only part of it" control: 25/50/75/100%.

### 3.4 Barcode scanning (Hong Kong)

1. Scan with VisionKit's `DataScannerViewController` (EAN-13, EAN-8, UPC-A, UPC-E).
2. Look up in order:
   - Local cache (products this user has scanned before)
   - Open Food Facts API (free, no key). Hong Kong coverage is limited, so expect misses.
3. Not found: photograph the nutrition label. The AI extracts values into a fixed schema. Hong Kong labels list energy plus 7 core nutrients (protein, total fat, saturated fat, trans fat, carbohydrates, sugars, sodium), in English or Chinese, per 100 g or per serving.
4. User confirms the values; save to the local cache under that barcode.
5. Optional, off by default: contribute the product to Open Food Facts. Make clear this is public.

### 3.5 Other ways to log

- Describe in text ("two eggs, toast with butter, milk tea") → AI returns Plate items.
- Food search across local foods, cached products and the bundled starter database.
- Quick add (calories and protein only).
- Favorites, recent meals, saved meals and recipes. Copy yesterday's breakfast in one tap.

### 3.6 Restaurant context without storing location

- Off by default. When on, tapping "Where am I?" makes a single one-time location request.
- Query nearby restaurants and cafés within ~100 m with MapKit's points-of-interest search. (The query goes to Apple's servers.)
- The user picks one; only the restaurant name is kept and added to the AI prompt.
- Coordinates are discarded immediately, never written to disk, never sent to the AI.
- Typing a restaurant name manually is always available instead.

### 3.7 Hong Kong food database

- Bundle a starter JSON of common local dishes with typical portions (e.g. char siu rice, wonton noodles, milk tea, pineapple bun, cart noodles).
- Candidate source: the Hong Kong Centre for Food Safety's nutrient database. Check its terms before bundling.
- Every value records its source; AI-estimated values are labeled as estimates.

### 3.8 Weight trend and adaptive targets [MacroFactor-style]

- Daily weight from Apple Health (e.g. a smart scale) or entered in-app.
- Smoothed trend line and weekly average, not just raw daily weights.
- Initial maintenance estimate from onboarding (Mifflin-St Jeor with an activity level).
- Adaptive maintenance: weekly, estimate actual maintenance from logged intake and trend weight change over the last 2–3 weeks; suggest a new calorie target for the chosen goal and rate (loss, maintenance or gain).
- Guardrails: change targets by at most ~100–150 kcal per week; never suggest a deficit or surplus beyond the user's limits (defaults: 500 kcal deficit, 300 kcal surplus); require at least ~10 logged days before adapting.
- Protein target in g/kg (default range 1.6–2.2 g/kg) shown alongside calories.

### 3.9 Body measurements

- Weekly waist measurement at the navel; optional hips, chest, arm.
- Waist trend charted next to weight trend.

### 3.10 Apple Health

Each data type is a separate permission the user can approve or skip.

Write to Health:

- Per meal: dietary energy, protein, carbohydrates, total fat, plus saturated fat, sugar and sodium when known.
- Weight and waist entered in the app.
- Tag samples with a meal ID; editing or deleting a meal updates or removes its Health samples.

Read from Health:

- Body weight and body fat % (e.g. from a smart scale).
- Waist circumference.
- Steps and active energy (display only; not added to calorie targets by default).
- Workouts: a logged climbing session or strength workout auto-ticks the matching item in the daily checklist.

Rules:

- Avoid duplicates: don't re-import samples the app wrote itself.
- Works fully without Health access; everything can be entered in-app.

### 3.11 Progress photo studio

- Guided capture of front, side and back poses.
- Ghost overlay of the previous photo for that pose to match position and distance.
- Level indicator so the phone is upright, not tilted up or down.
- Timer (3/5/10 s) for propping the phone on a shelf.
- Optional flexed variant for the front pose.
- Compare view: side by side, or a slider between two dates.
- Stored locally only (see Privacy model).

#### Progress time-lapse

A short video that morphs through every progress photo for a pose, so changes are easy to see.

- Auto-align: detect body landmarks (Apple Vision body pose: shoulders, hips) in each photo and scale, rotate and crop so the body sits in the same place every frame. Fall back to manual alignment with the ghost overlay if detection fails.
- Even out lighting: normalize exposure and white balance across frames so the video shows body changes, not lighting changes.
- Layouts: one pose full-screen, or front/side/back side by side in a grid.
- Options: frame duration, crossfade between frames, optional overlays for date, weight and waist (off by default).
- Privacy: built entirely on-device (AVFoundation); never sent to any AI service. Optional face blur or crop-below-chin for sharing. Saved to Photos or shared only when the user taps Export.
- Before/after card: a still image of the first and latest photo side by side, using the same alignment.

### 3.12 Training and energy check-in

- Daily checklist the user configures (default: climbing, push/legs session, ab work, daily walk), auto-ticked from Health workouts where possible.
- Daily 1–5 ratings for energy and training performance.
- If energy or performance averages drop noticeably for a week, gently suggest eating a bit more. Never suggest eating less in response.

### 3.13 Reminders

- Local notifications (no server needed): weekly waist, progress photos every 2 weeks on a chosen weekday morning, optional daily weigh-in, backup reminder if iCloud sync is off.

### 3.14 Export and backup

- Export all data to CSV and JSON.
- Encrypted backup file (password-protected) including photos, for moving phones without iCloud.
- Restore from a backup file.
- "Delete all my data" button (with an option to also remove the app's samples from Health).

## 4. Screens

- Today: calories and protein remaining, meals logged, weight, checklist.
- Log (camera): modes for Photo (card / LiDAR), Barcode, Label, Describe, Search.
- Plate: editable result from any logging method, before saving.
- Progress: weight trend, waist, target history, photo studio and compare.
- Plan: goal type (cut, maintain, bulk), targets, weekly rate, protein range, deficit/surplus limits.
- Settings: AI provider and key, portion method (card / LiDAR), Apple Health permissions, privacy options, iCloud sync, reminders, units, backup/restore, delete data.
- Onboarding: height, weight, age, sex, activity level, goal, Apple Health connection, AI key setup with a privacy explanation.

Design notes: dark mode, large numbers for at-a-glance use, one-handed logging, everything within two taps of Today.

## 5. Data model (SwiftData)

- `UserProfile`: height, sex, birth year, activity level, units.
- `Goal`: type (cut, maintain, bulk), target body fat or weight, weekly rate, protein g/kg range, deficit/surplus limits, start date.
- `Food`: name, source (OFF, label scan, HK starter, AI, user), per-100 g nutrition, density, barcode (optional).
- `Meal`: date, meal type, optional restaurant name, optional note, Health sample IDs.
- `MealItem`: food, grams, nutrition snapshot, confidence, method (LiDAR, card, AI visual, barcode, manual).
- `WeightEntry`, `MeasurementEntry`: date, value, source (app or Health).
- `ProgressPhoto`: date, pose, flexed flag, file reference.
- `CheckIn`: date, checklist items, energy and performance ratings.
- `TargetHistory`: date, maintenance estimate, calorie target, reason.

No model stores coordinates. Meal photos are not kept after logging by default.

## 6. AI integration

- All calls go directly from the phone to the chosen provider. No proxy server.
- Structured JSON output with a fixed schema; validate every response and retry once on invalid JSON.
- Images downscaled to ~1024 px on the long edge, re-encoded as JPEG with no metadata.
- Prompts include: card scale or LiDAR volumes when available, optional restaurant name and note, and an instruction to return per-item results with confidence.
- Model names kept in one config file, picked from current provider docs at build time.
- Timeouts and clear errors. Without a key or connection, manual, barcode-cache and search logging still work.
- Show approximate cost per photo in Settings.

## 7. Security checklist

- API keys only in Keychain; `.gitignore` covers any local config; no keys in sample code.
- No logging of request contents, images or keys.
- Location: "When In Use" only, single request, coordinates discarded.
- Permission strings explain exactly why camera, location, Health and photo access are needed.
- Privacy policy in the repo: nothing is collected by the developer; what goes to the AI provider and Apple; how to delete data.

## 8. Build phases

1. Core: onboarding, Today, food search, quick add, text describe, editable Plate, AI provider settings with Keychain.
2. Barcodes: scanner, Open Food Facts lookup, label-photo fallback, local product cache.
3. Photo logging: reference-card capture guide, homography and volume pipeline, density table, restaurant picker.
4. Apple Health and progress: HealthKit read/write, weight trend, adaptive maintenance, waist tracking.
5. Photo studio and check-ins: guided capture with ghost overlay, compare view, time-lapse, check-ins, local reminders.
6. LiDAR (optional method): depth capture, plane fit, depth volume pipeline, cross-check with the card.
7. Polish: iCloud sync, Face ID lock, export/backup, accessibility, App Store readiness.

Each phase ends with a working app and tests for its core logic (nutrition math, homography and area calculation, depth volume, trend and adaptive targets, JSON parsing, Health deduplication).

## 9. Open decisions

- Distribution: free self-install via Xcode, or US$99/year for TestFlight / App Store.
- Default AI: Gemini (recommended for outlines) or Claude.
- iCloud sync on or off by default.
- Whether to allow contributing products to Open Food Facts.

## 10. Prompt for Claude Code

Export this doc as Markdown, save it in a new repo as `SPEC.md`, then paste:

> Read SPEC.md. It's a skeleton spec, not a final design. Before writing any code, work through it with me section by section: point out gaps, ask questions, and propose options and trade-offs so we can flesh out the design together. The app is called Santorio. Once we've agreed on the design, we'll build a native SwiftUI iOS 17+ app starting with Phase 1. Store API keys only in the Keychain and never commit secrets. Ask me before adding any third-party dependency.
