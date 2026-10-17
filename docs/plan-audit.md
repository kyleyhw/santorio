# Plan Audit

An audit of [`PROJECT_PLAN.md`](../PROJECT_PLAN.md) against the skeleton spec
([`SPEC.md`](../SPEC.md)), the design ([`design.md`](design.md)) and the
mathematics ([`math.md`](math.md)), carried out in October 2026 before any
code exists. It records what the plan omits or contradicts (§1), small edits to
existing tasks (§2), larger proposals (§3) and the decisions the owner is asked
to make (§4). Nothing here is implemented; accepted items move into the plan as
pending tasks, rejected ones are struck through here with the reason, so that
the record of what was considered survives.

The audit's yardstick is the purpose stated in [`design.md`](design.md) §2:
logging must be effortless, estimates must be honest about their accuracy, the
app must stay calm, and personal data must never leave the device. Each
proposal is judged by how much it advances those four aims for the effort it
costs.

---

## 1. Findings

### 1.1 Spec features absent from the plan

| Feature | Spec | Design | In the plan | Proposed action |
|---|---|---|---|---|
| Favourites, recent meals, saved meals, recipes, "copy yesterday's breakfast" | §3.5 | §5.2 (suggestions only) | Absent | Proposal 3.1.3 |
| Restaurant context (name added to the AI prompt) | §3.6 | §7 (`restaurants` pack) | Absent | Proposal 3.1.7 |
| Progress time-lapse and before/after card | §3.11 | §9 | Absent | Proposal 3.4.1 |
| User pack import in Settings | — | §7, use 1 | Absent | Edit E6 |
| Measured accuracy per method shown to the user | — | §6 | Absent | Proposal 3.2.2 |
| Approximate cost per photo in Settings | §6 | §8.4 | Absent | Edit E5 |
| Warning for the free Gemini tier (inputs may be used for training) | §2 | — | Absent | Edit E5 |
| Storage persistence and Safari eviction mitigations | — | §8.3 | Partial (install tip only) | Edit E2 |
| Service-worker update flow | — | §8.1 | Absent | Edit E1 |
| "Ate only part of it" and "shared with others" | §3.2, §3.3 | — | Partial (partial-eaten only) | Edit E4 |
| Tap-to-segment fallback when no usable outline is returned | §3.1 | §8.4 (degrade chain ends at "manual") | Undefined | Edit E9 |
| Delete all data including Health samples | §3.14 | §9 | Task 33 (Health part waits for Phase 9) | None |

### 1.2 Inconsistencies between documents

- [`math.md`](math.md) §1.3 stated that the camera height is only
  approximately known in a browser. The card determines it (§1.4–1.5), which
  also makes the parallax correction exact. Resolved in this round.
- [`design.md`](design.md) §8.3 says all storage is encrypted; public food
  packs need not be, and encrypting several megabytes of public data on every
  search would cost battery for nothing. Resolved in §7.1: packs in Cache
  Storage, the list of installed packs in the vault.
- The spec's volume formula "footprint × average height" and §2.1's
  $V = kAh$ with the *maximum* height $h$ differ by the shape factor $k$. The
  design is right; the consequence is that the AI prompt must ask for the
  maximum height and a shape class, never an average (Edit E10).
- The spec's tilt indicator was the only source of the viewing angle. The
  angle is now measured from the card after capture; the gyroscope is a
  guide only. On iOS Safari `DeviceOrientationEvent` also requires an explicit
  permission request since iOS 13, which the capture flow must handle (Edit
  E10).
- The README described the design as complete while §11 holds open
  questions. Changed to "in review".
- Phase 8 planned "English and Traditional Chinese"; with three regions the
  English copy needs United Kingdom and United States spelling variants, and
  dates need the regional order (day-month in Hong Kong and the United
  Kingdom, month-day in the United States). Plan updated; date order added as
  Edit E13.

### 1.3 Risks

- **AI dependence.** Outlines and corners come from a paid model: cost per
  meal, latency, privacy of the food photograph, and drift between model
  versions. Mitigations already planned: local corner refinement (task 51),
  prompt versioning (§8.4), a manual polygon fallback (E9). Larger
  mitigations are proposals 3.1.1 (no card or AI needed for registered
  tableware), 3.1.9 and 3.1.10.
- **Browser platform.** Safari may evict storage for sites not installed to
  the Home Screen; camera access inside an installed web app works since iOS
  14.3 but must be verified on current iOS; passkey PRF support and OPFS in
  Safari must be verified at build (§4). None of these is a blocker; each is
  a verification step in Phase 2.
- **Data licences.** CoFID requires the Open Government Licence attribution
  statement; Open Food Facts data are ODbL, so any redistributed subset
  (proposal 3.1.9) must carry attribution and the same licence; the Hong Kong
  Centre for Food Safety terms are unknown until task 9 is done.
- **Scope of the single pass.** Phases 2–8 hold about fifty tasks. The order
  already yields a usable text-and-search logger after Phase 3; deploying to
  Pages after every phase (not only at task 68) keeps a usable app in the
  owner's hands throughout.
- **Maintainer load.** Six packages and three toolchains for one maintainer
  is more structure than the minimal-implementation principle suggests;
  proposal 3.3.4 offers a flatter layout.

---

## 2. Proposed edits to existing tasks

Small changes inside the agreed design. Each names the plan task it modifies.

- **E1 (task 18).** Service-worker update flow: a quiet "update ready"
  notice that applies on the next launch, and the build version shown in
  Settings so bug reports can name it.
- **E2 (tasks 23, 33).** Call `navigator.storage.persist()` at first run,
  show the storage estimate in Settings, and remind about backup only when it
  matters: on Safari without installation after five days without a visit
  (inside the seven-day eviction window), otherwise when the last backup is
  older than a user-set interval.
- **E3 (task 26).** Write the input-bar grammar down before building it:
  quantities and units (`180g`, `2 cups`, `half`, `×2`), regional measures
  (兩, pint), modifiers (`少甜`, `skinny`, `no sugar`), sharing (`/3`), and a
  test table of phrases per region.
- **E4 (task 27).** A "shared with N" divisor next to the partial-eaten
  control; dim sum and family-style meals are the norm in Hong Kong and
  common elsewhere.
- **E5 (task 28).** Show the approximate cost per meal (one and two
  photographs); warn when the user says they are on the free Gemini tier; add
  a session-only key mode that never persists the key, for shared computers.
- **E6 (task 30).** The Settings → Food databases screen also imports user
  packs ([`design.md`](design.md) §7, use 1), validated against the schema.
- **E7 (task 37).** Weight-entry validation and an outlier gate: a reading
  whose standardised Kalman innovation exceeds 4 is queried as a possible
  typo (7.5 for 75 kg), never silently dropped and never silently accepted.
- **E8 (task 38).** Record the sources for the default noise parameters when
  they are fixed, and state why no weekday effect is modelled (it is not
  identifiable until months of data exist).
- **E9 (task 54).** Define the manual fallback as a user-drawn polygon that
  snaps to image edges; on-device tap-to-segment is proposal 3.1.10.
- **E10 (tasks 50, 55).** The prompt asks for maximum height and shape class;
  the capture flow requests `DeviceOrientationEvent` permission on iOS; both
  views' outlines are stored with a capture kept for calibration.
- **E11 (task 56).** "I weighed it" is one tap from the Plate item and from
  the day's list; the calibration count appears on the Accuracy screen
  (proposal 3.2.2).
- **E12 (task 61).** Either add the time-lapse (proposal 3.4.1) or strike
  it from the spec explicitly.
- **E13 (tasks 20, 64).** Regional date order and first day of the week via
  `Intl`, added to the conventions table in [`design.md`](design.md) §7.2.
- **E14 (new task, Phase 3).** Time and travel: store each record's UTC
  instant and local offset; a "day" is the local calendar day with a
  user-set boundary (default midnight; a late eater may choose 04:00); the
  weight trend's $\Delta t$ uses true elapsed days, so a flight between Hong
  Kong, London and the United States neither duplicates nor loses a day.
- **E15 (task 57).** A ground-truth protocol for real photographs: ruler
  heights and kitchen-scale masses per item, recorded in the private
  `eval-data/`, so the two-view estimator's claimed precision is checked
  against real plates as well as synthetic ones.
- **E16 (task 70).** HealthKit has no uncertainty field; write the median
  and keep the interval in the app.
- **E17 (Phase 10).** With measured heights on every device, LiDAR's
  incremental value is smaller than the spec assumed; keep the phase but
  evaluate the two-photograph method first and build LiDAR only if the
  evaluation shows a worthwhile gain.

---

## 3. Proposed features

Each proposal states what it is, why it serves the purpose, how it would feel
to use, the mathematics where there is any, and an effort grade (XS, S, M, L)
with a suggested phase. The ambitious items are marked as such; none is in the
plan until accepted.

### 3.1 Logging without friction

#### 3.1.1 Tableware registration: the plate as the reference object

*What.* The user registers each of their plates and bowls once by
photographing it beside the card with the two-shot flow. The app stores the
rim diameter $2a$, the rim height $z_r$ above the table, the well depth and,
for bowls, the interior profile that §2.2 currently takes from a preset. Later
meals on a registered plate need no card at all: the rim's elliptical image
supplies the metric structure of the plane.

*Why.* At home the card is the friction: find it, place it, keep it out of the
sauce. Most home meals are eaten from the same three to six pieces of
tableware, so this removes a step from the most frequent logging path, and
it replaces the bowl preset, the weakest term in §2.2, by a measurement.

*Mathematics.* A circle of known radius $a$ imaged as an ellipse (a conic $C$
in the image), with the focal length known from the device's remembered $f$
(§1.4), determines the supporting plane's normal and distance up to a two-fold
ambiguity and the metric structure of the plane up to a rotation about its
normal [[1]](#ref-safaee), [[2]](#ref-chen). Footprint areas are invariant to
that rotation, so nothing in §1.2 changes; the ambiguity is removed by the
second view or by requiring the camera to be above the table. The plate's well
lies at $z_w < z_r$, so food footprints are mapped to the well plane by the
radial correction of §1.5 with $h \to z_w$. Rim detection: the AI proposes the
ellipse or the user drags five points; a local conic fit to edge points
refines it. Oval and square plates are registered with typed dimensions (a
square plate behaves like a card of known size).

*Effort.* M. Phase 6 extension; depends on task 52.

#### 3.1.2 Leftovers: photograph what you left

*What.* After eating, an optional shot (or pair) of the plate. The app
estimates the remaining volume per item and logs eaten = before − after. The
spec's 25/50/75 % control becomes a measurement.

*Mathematics.* Each item's eaten mass is $m = \rho\,(V_1 - V_2)$. The density
error is shared and remains a log-normal factor; $V_1 - V_2$ is approximately
normal with mean $\mu_{V_1} - \mu_{V_2}$ and variance $\sigma_{V_1}^2 + \sigma_{V_2}^2$
(independent photographs). The product of an independent log-normal and a
normal is summarised by its first two moments,
$\operatorname{Var}(XY) = \operatorname{Var}X\operatorname{Var}Y + \operatorname{Var}X\,(\mathbb E Y)^2 + (\mathbb E X)^2\operatorname{Var}Y$,
and displayed through the same moment-matched log-normal as §3.3. When the
remainder is small the result is dominated by $V_1$, as it should be.

*Why.* Restaurant portions in Hong Kong and the United States are frequently
not finished; a guessed fraction is the coarsest number in the pipeline. Same
pipeline, one extra prompt on the meal ("Done? Photograph what is left").

*Effort.* S–M. Phase 6.

#### 3.1.3 Recipes, saved meals, favourites, and weighing modes

*What.* (a) A recipe is a list of ingredients with weights, from search,
barcode or the scale, plus the cooked yield weight, giving per-100 g values
for the cooked dish; it is logged by portion weight or fraction. (b) A Plate
can be saved as a meal; favourites and "copy yesterday's breakfast" are one
tap from suggestions. (c) Weighing modes for users with a kitchen scale:
grams in the bar (`rice 180g`), and a *cumulative* mode in which the bowl
stays on the scale and the user types each running total (`120, 310, 365`
becomes 120, 190 and 55 g), which is how a mixed bowl is actually weighed.
Weighed items carry only the scale's resolution as uncertainty and so display
as a single number, which is the honest consequence of §3.2.

*Why.* The spec asked for all of it (§3.5) and the plan dropped it. Home
cooking is where logging is both most accurate and most tedious; this is the
largest remaining friction for a user who cooks.

*Effort.* M. Phase 3.

#### 3.1.4 Deep links and Shortcuts

*What.* Routes that pre-fill the universal bar through the same parser:
`#/log?text=black%20coffee`, `#/log?quick=450`, `#/weigh?kg=74.2`. An iOS
Shortcut or Siri phrase ("log coffee"), a home-screen bookmark or a desktop
launcher then logs with no taps inside the app. On Android, Web Share Target
accepts photographs shared from the camera roll; iOS offers no share target
for web apps, which the documentation states plainly.

*Why.* Zero-tap logging for the most repeated items; "everything within two
taps of Today" at its limit. *Effort.* S. Phase 3.

#### 3.1.5 Demo before any key

*What.* First run offers "Try it with a sample plate": bundled synthetic or
licensed photographs with stored AI results, so the user sees the Plate,
intervals and editing within the first minute with no key, no camera
permission and no network. A clearly labelled "sample day" shows Today, the
trend and Plan with synthetic data, cleared on exit.

*Why.* Principle §2.3 wants a first meal logged within a minute; today that is
blocked on obtaining an API key. The key setup guide can then follow a felt
need. *Effort.* S. Phase 3.

#### 3.1.6 Meal thumbnails in the day

*What.* Keep a small (about 256 px) encrypted thumbnail of each meal
photograph; the full image is discarded as now unless kept for calibration.
The day view shows the thumbnails, and "copy this meal" from a picture is
faster than from text.

*Why.* Recall: a picture of yesterday's lunch is recognised faster than its
description, and it anchors the calibration loop ("which meal did I weigh?").
Food photographs are low-sensitivity, encrypted at rest and never sent. The
default (on or off) is decision D4. *Effort.* S. Phase 3.

#### 3.1.7 Restaurant names without a map service

*What.* A restaurant field with autocompletion from the user's own history and
the `restaurants` pack, and "usual order here" suggestions. No location request
and no third party in the web app: MapKit is native-only, and an
OpenStreetMap query would add a third party and disclose coordinates. The
native MapKit picker of the spec can follow in Phase 9.

*Why.* Closes the spec gap (§3.6) at zero privacy cost and gives one-tap
re-logging at frequent places. *Effort.* S. Phase 3.

#### 3.1.8 Printable reference card (ambitious, small)

*What.* A PDF (A4 and US Letter) carrying an ID-1-sized matte rectangle with
a thick black border and asymmetric marks, printed once and kept with the
kitchen scale. It removes reflections (glossy bank cards), rounded corners and
the half-turn ambiguity of §1.6, improves corner detection by the AI or
locally, and needs no new geometry because its size is the card's. The sheet
carries a printed ruler so that print scaling can be checked against a bank
card once.

*Effort.* S for the sheet. A local quad detector in TypeScript that finds it
without any AI call is M–L and belongs after the AI path works.

#### 3.1.9 Offline, private barcode pack per region (ambitious)

*What.* A region-filtered Open Food Facts subset (products with complete
nutrition, ranked by popularity, capped at roughly 100 000 products, about
20–30 MB compressed) installable like a food pack. Lookups then never leave
the device and work offline. The pack is a derived database under the ODbL
and must carry attribution and the same licence.

*Why.* The barcode lookup is the one logging path that reveals purchasing
habits to a third party, and shops have poor signal. *Effort.* M, in
`tools/packs`. Decision D5.

#### 3.1.10 On-device segmentation (exploratory)

*What.* An optional downloadable segmentation model run in the browser
(WebGPU with a WASM fallback) for tap-to-segment: the user taps an item and
receives a mask locally. Candidate "efficient SAM" variants are 20–40 MB;
sizes, licences and outline quality on food must be verified with
`tools/eval` before any commitment.

*Why.* Removes the AI from the geometry path entirely for users who want no
cloud at all, and supplies the fallback the spec asks for when no usable
outline is returned. *Effort.* L, uncertain. After Phase 8.

### 3.2 Honest numbers

#### 3.2.1 "Why this range?"

*What.* Tapping an interval shows each factor's share of $\sigma^2_{\ln E}$
(card scale, outline, height, shape, density, energy density) as a bar, with
one line on what would narrow it most: "a side photograph would roughly halve
the height term", "weighing this once would calibrate rice for you". It is a
direct read-out of the analytic propagation in §3.1 of
[`math.md`](math.md), so it costs almost nothing.

*Why.* Honesty made actionable: the user learns how to get better numbers
without being nagged. *Effort.* S. Phases 3 and 6.

#### 3.2.2 Accuracy screen

*What.* Settings → Accuracy: for each method (card, two-shot, barcode, label,
described) the number of calibration samples, median absolute percentage
error, bias by food category, empirical 80 % coverage and a predicted-versus-
true plot on log axes, computed on the device from the user's own
"I weighed it" pairs. The same statistics `tools/eval` reports (§7).

*Why.* [`design.md`](design.md) §6 promises it, and it is the user-facing
proof that the intervals mean something. *Effort.* S–M. Phase 6.

#### 3.2.3 Energy density of weight change from body composition

*What.* Replace the fixed $\rho_E = 7700$ kcal/kg of §6.2 by a
composition-dependent value. Forbes' cross-sectional relation
$\mathrm{FFM} = 10.4\ln F + 14.2$ (kg, with $F$ the fat mass) implies that a
change in fat mass is accompanied by a lean change
$\mathrm{d}L/\mathrm{d}F = 10.4/F$ [[3]](#ref-forbes). With energy densities
$\rho_F \approx 9400$ kcal/kg for fat and $\rho_L \approx 1800$ kcal/kg for
lean tissue [[4]](#ref-hall2008), the energy density of total weight change is

$$
\rho_E(F) = \frac{\rho_F + \rho_L\,(10.4/F)}{1 + 10.4/F}
          = \frac{9400\,F + 18\,720}{F + 10.4}\ \text{kcal/kg}.
$$

| Fat mass $F$ | 8 kg | 15 kg | 30 kg | $F \to \infty$ |
|---|---|---|---|---|
| $\rho_E(F)$ | 5100 | 6290 | 7440 | 9400 |

For a lean user the fixed 7700 overstates $\rho_E$ by about a third. The
energy-balance estimate $\hat E = \bar I - \rho_E\beta$ is then biased by
$\Delta\rho_E\,\beta$: at a loss of 0.5 kg per week ($\beta \approx -0.071$
kg/day) and $\Delta\rho_E \approx 2600$ kcal/kg this is about 185 kcal/day,
larger than the weekly target step the guardrails allow. Fat mass comes from
body-fat percentage when the user has one (smart scale, later HealthKit) or
from a circumference estimate (the US Navy method needs neck, waist and
height [[5]](#ref-hodgdon)) with its wide uncertainty, which propagates
through $\operatorname{Var}(\rho_E)$ in §6.2 exactly as now. In a surplus with
resistance training the lean share of gain exceeds Forbes' prediction; the
proposal keeps Forbes with a widened uncertainty and lets the guardrails
absorb the rest.

*Why.* The most consequential number in the adaptive engine, replaced by a
sourced model at the cost of one formula and an optional neck measurement
(decision D6). *Effort.* S. Phase 4.

#### 3.2.4 Time to goal as a distribution

*What.* From the filtered state $(\ell, \beta)$ and its covariance (§5), the
time to a target weight $w^\ast$ is $\tau = (w^\ast - \ell)/\beta$. Because
$\tau$ is a ratio with $\beta$ possibly near zero, its distribution is heavy-
tailed and the delta method misleads; instead draw, say, 1000 samples of
$(\ell, \beta)$ from the joint normal with a seeded generator and report the
10th and 90th percentiles of $\tau$: "at the current trend, 72 kg between
3 December and 19 January (80 %)", or "not reachable at this rate" when the
band on $\beta$ includes zero or the wrong sign.

*Why.* Everyone asks this question; existing trackers answer with one
falsely precise date. *Effort.* S. Phase 4.

#### 3.2.5 Weekly review, and targets that never change silently

*What.* One calm weekly screen: trend change with its band; mean intake with
uncertainty and completeness (days fully logged); the maintenance estimate
with the current weights of prior and data (§6.3); the proposed target and
the arithmetic behind it; accept, edit or skip. A target changes only when
the user accepts it. Make this a principle in [`design.md`](design.md) §2.

*Why.* The adaptive engine is the app's most consequential output; a silent
change breaks trust, and showing the reasoning suits a quantitatively literate
owner. *Effort.* S–M. Phase 4.

#### 3.2.6 Goal phases

*What.* A schedule of periods, each with its own goal type, rate and limits
(cut eight weeks, maintain two, cut again; or bulk and maintain). The target
follows the active phase; the review suggests a maintenance break when the
trend has stalled or the rate has been high for long, by a plain user-set
rule.

*Why.* "Any goal" in practice means a sequence of goals, and diet breaks are
standard practice; without phases the user fights the adaptive engine.
*Effort.* S. Phase 4.

#### 3.2.7 Protein basis option

*What.* Protein range per kg of body weight (default), of fat-free mass, or
of target weight; the fat-free-mass option uses the body-fat estimate of
3.2.3. Rationale shown in one line.

*Why.* Grams per kg of total weight over-prescribes at high body fat.
*Effort.* XS. Phase 4.

#### 3.2.8 Secondary nutrients with regional context (opt-in)

*What.* Optional tracking of fibre, salt or sodium, saturated fat, free or
added sugars and alcohol, with regional reference values shown as context
lines, never as targets or red marks: fibre 30 g/day and salt below 6 g/day in
the United Kingdom [[6]](#ref-sacn), sodium below 2300 mg/day in the United
States [[7]](#ref-dga), sodium below 2000 mg/day in Hong Kong, which follows
the WHO guideline [[8]](#ref-who). Off by default; appears only on the review.

*Why.* The data are free from packs and labels, and some users have medical
reasons. The calm principle forbids new daily targets unless asked for.
*Effort.* S. Phases 4–5.

#### 3.2.9 Weight import from CSV

*What.* Import weight, body-fat percentage and waist from exported files
(Apple Health export, Withings, Garmin, Fitbit, Renpho and similar) with a
column-mapping preview and de-duplication by timestamp.

*Why.* The web app has no HealthKit; years of smart-scale history initialise
the Kalman filter and the maintenance prior at once. *Effort.* S. Phase 4.

#### 3.2.10 Cycle-aware weight trend (opt-in, ambitious)

*What.* For users who menstruate, an optional cycle-day entry; the trend model
of §5 gains a periodic component (a per-phase offset estimated from the
user's own data) so that cyclic water retention of roughly 0.5–1 kg is not
read as fat change, and the review names the phase. Entirely local, neutral
copy, off by default.

*Why.* A known, large confounder of the weight trend for half of potential
users. *Effort.* M, including validation on synthetic series. Phase 8.
Decision D7.

### 3.3 Trust and robustness

#### 3.3.1 Diagnostics log

A local ring buffer of errors (never request contents), viewable and
exportable from Settings. It is the only way to debug an app with no
analytics. *Effort.* XS. Phase 2.

#### 3.3.2 Threat model

`docs/threat-model.md`: assets, adversaries (device thief, another user of a
shared computer, compromise of the hosting account, a malicious pack, the AI
provider), and what each control (encryption, lock, Content Security Policy,
pack validation) does and does not protect against; referenced from
`PRIVACY.md` and `SECURITY.md`. *Effort.* S. Phase 2.

#### 3.3.3 Pack integrity

Packs built in CI are published with SHA-256 digests in a manifest signed by a
release key; the app verifies digests before installing, and user packs are
schema-validated and size-capped. *Effort.* S. Phase 3.

#### 3.3.4 Flatter repository (optional)

Keep `packages/core` (pure, tested in isolation) and `packages/ai`; fold
`vault`, `storage` and `platform` into modules of `apps/web/src/` until a
second app (the iOS shell) actually needs to share them. Less structure to
maintain for one maintainer; the interfaces stay where they are. *Effort.*
negative. Decision D8.

### 3.4 Progress photos

#### 3.4.1 Time-lapse and before/after card

On-device video from the pose photographs: frames drawn on a canvas and
encoded with WebCodecs where available, `MediaRecorder` otherwise; alignment
by the ghost overlay (manual) first and by body landmarks later (native Vision
in Phase 9); exposure normalised by histogram matching; exported only on
explicit request, with face blur. *Effort.* M. Phase 7. Decision D9.

---

## 4. Decisions requested

| # | Decision | Recommendation |
|---|---|---|
| D1 | Pack hosting: commit the built UK and US packs, or build them in CI and publish to Pages | CI build from pinned URLs with checksums; commit only the manifest and `SOURCES.md`. Keeps the repository small and the build reproducible |
| D2 | Edits E1–E17 | Accept as a block; strike individually if any is unwanted |
| D3 | Which proposals enter the single pass | See the priority table (§5): accept the first two groups, defer the third |
| D4 | Meal thumbnails (3.1.6) default | On, with a one-line explanation at first run and a switch in Settings |
| D5 | Offline barcode pack (3.1.9) | Defer until the online path has shown how often lookups happen |
| D6 | Optional neck measurement (enables 3.2.3 and 3.2.7 without a smart scale) | Accept; it is one optional field on the measurements screen |
| D7 | Cycle-aware trend (3.2.10) | Accept as opt-in in Phase 8 |
| D8 | Flatter repository (3.3.4) | Accept; the package boundaries are easy to restore when the iOS shell arrives |
| D9 | Time-lapse (3.4.1) in Phase 7, or struck from the spec | Accept in Phase 7 with manual alignment |
| D10 | Calorie floor ([`design.md`](design.md) §11.3) | A target never below the Mifflin–St Jeor resting expenditure, $T \ge \mathrm{REE}$. It needs no external threshold, scales with the person, and matches the safety principle; I am not confident of a citable fixed minimum and would rather not invent one |

## 5. Priority table

| Proposal | Value for the purpose | Effort | Phase |
|---|---|---|---|
| 3.1.4 Deep links and Shortcuts | High (zero-tap logging) | S | 3 |
| 3.1.5 Demo before any key | High (first minute) | S | 3 |
| 3.1.7 Restaurant names | Medium | S | 3 |
| 3.2.1 "Why this range?" | High (honesty) | S | 3, 6 |
| 3.2.3 Composition-dependent $\rho_E$ | High (honesty) | S | 4 |
| 3.2.4 Time to goal as a distribution | High (honesty) | S | 4 |
| 3.2.5 Weekly review; no silent target changes | High (trust) | S–M | 4 |
| 3.2.6 Goal phases | High (any goal) | S | 4 |
| 3.2.9 Weight import from CSV | Medium | S | 4 |
| 3.3.1–3.3.3 Diagnostics, threat model, pack integrity | Medium (trust) | XS–S | 2–3 |
| 3.1.1 Tableware registration | High (no card at home) | M | 6 |
| 3.1.2 Leftovers | High (restaurants) | S–M | 6 |
| 3.1.3 Recipes, saved meals, weighing modes | High (home cooking) | M | 3 |
| 3.2.2 Accuracy screen | High (honesty) | S–M | 6 |
| 3.4.1 Time-lapse | Medium | M | 7 |
| 3.1.6 Meal thumbnails | Medium | S | 3 (decision D4) |
| 3.1.8 Printable reference card | Medium | S (sheet), M–L (local detection) | 6, later |
| 3.2.7 Protein basis option | Low–medium | XS | 4 |
| 3.2.8 Secondary nutrients | Low–medium | S | 4–5 |
| 3.2.10 Cycle-aware trend | Medium for those it serves | M | 8 (decision D7) |
| 3.1.9 Offline barcode pack | Medium (privacy) | M | later (decision D5) |
| 3.1.10 On-device segmentation | Uncertain | L | later |

The first group is small enough to add to the single pass without changing its
shape. The second group is the ambitious core of this audit: tableware
registration and leftovers make the card method usable for every meal rather
than the ones where a card is to hand, and the Accuracy screen is what lets
the owner verify the whole approach. The third group waits for evidence from
the first two.

---

## References

<span id="ref-safaee">[1]</span> Safaee-Rad, R., Tchoukanov, I., Smith, K. C., & Benhabib, B. (1992). Three-dimensional location estimation of circular features for machine vision. *IEEE Transactions on Robotics and Automation*, 8(5), 624–640. [Link](https://doi.org/10.1109/70.163786)

<span id="ref-chen">[2]</span> Chen, Q., Wu, H., & Wada, T. (2004). Camera calibration with two arbitrary coplanar circles. In *Computer Vision — ECCV 2004*, Lecture Notes in Computer Science 3023, 521–532. Springer. [Link](https://doi.org/10.1007/978-3-540-24672-5_41)

<span id="ref-forbes">[3]</span> Forbes, G. B. (1987). Lean body mass–body fat interrelationships in humans. *Nutrition Reviews*, 45(8), 225–231. [Link](https://doi.org/10.1111/j.1753-4887.1987.tb02684.x)

<span id="ref-hall2008">[4]</span> Hall, K. D. (2008). What is the required energy deficit per unit weight loss? *International Journal of Obesity*, 32(3), 573–576. [Link](https://doi.org/10.1038/sj.ijo.0803720)

<span id="ref-hodgdon">[5]</span> Hodgdon, J. A., & Beckett, M. B. (1984). *Prediction of percent body fat for U.S. Navy men from body circumferences and height.* Naval Health Research Center, Report No. 84-11.

<span id="ref-sacn">[6]</span> Scientific Advisory Committee on Nutrition (2015). *Carbohydrates and Health.* The Stationery Office. [Link](https://www.gov.uk/government/publications/sacn-carbohydrates-and-health-report)

<span id="ref-dga">[7]</span> U.S. Department of Agriculture and U.S. Department of Health and Human Services (2020). *Dietary Guidelines for Americans, 2020–2025* (9th ed.). [Link](https://www.dietaryguidelines.gov/)

<span id="ref-who">[8]</span> World Health Organization (2012). *Guideline: Sodium intake for adults and children.* [Link](https://www.who.int/publications/i/item/9789241504836)
