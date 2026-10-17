# Santorio

A free, open-source calorie and body-composition tracker that runs in the
browser on desktop and phone, keeps all personal data encrypted on the user's
own device, and estimates portions from photos using a reference card of known
size. Named after Santorio Santorio, the 17th-century physician who weighed
himself and everything he ate for decades.

**Status:** design in review, implementation not started. See
[`PROJECT_PLAN.md`](PROJECT_PLAN.md).

**This repository is public.** Never commit personal data, API keys or
identifying configuration.

## Structure

Current contents:

```
santorio/
├── README.md               this file
├── PROJECT_PLAN.md         phases, tasks and status
├── SPEC.md                 original skeleton spec
├── .gitignore
├── .pre-commit-config.yaml detect-secrets hook
├── .secrets.baseline       detect-secrets baseline
└── docs/
    ├── design.md           decisions and rationale
    ├── math.md             mathematical models and references
    └── plan-audit.md       audit of the plan: findings, proposals, decisions requested
```

The planned code layout (`packages/`, `apps/`, `tools/`, `tests/reports/`) is
described in [`docs/design.md`](docs/design.md) §8.1.

## Documentation

| Document | Contents |
|---|---|
| [`PROJECT_PLAN.md`](PROJECT_PLAN.md) | Phases, tasks, status |
| [`docs/design.md`](docs/design.md) | Decisions, principles, architecture, privacy, open questions |
| [`docs/math.md`](docs/math.md) | Card geometry and camera pose, two-photograph height, volume, regional label factors, uncertainty propagation, calibration, weight trend, energy expenditure |
| [`docs/plan-audit.md`](docs/plan-audit.md) | Audit of the plan against the spec and design: gaps, proposed edits, proposed features, decisions requested |
| [`SPEC.md`](SPEC.md) | Original skeleton spec (superseded where it conflicts with the design) |

## Overview

The application is a static web app (React, TypeScript) served from GitHub
Pages, with no backend. Records and photos are stored in the browser and
encrypted with AES-GCM; the only outbound requests go to the AI provider the
user configures (with their own key) and to Open Food Facts for barcode
lookups, and a Content Security Policy enforces this.

Portion estimation rests on three models, derived in
[`docs/math.md`](docs/math.md):

1. **Scale from a card.** The four corners of an ID-1 card
   (85.60 × 53.98 mm) determine a plane-to-image homography $H$, solved by the
   direct linear transform. Food outlines map through $H$ to metric footprint
   areas, using the shoelace formula for polygons and the Jacobian
   determinant $\det H / w^3$ for pixel masks.
2. **Mass and energy.** Volume $V = kAh$ (shape factor $k$, height $h$), or a
   spherical-cap model for bowls, or count × piece mass for discrete foods;
   then $m = \rho V$ and $E = m\,e_{100}/100$.
3. **Honest uncertainty.** Each factor carries a log-normal error, so
   $\sigma_{\ln E}^2$ is the sum of the factors' log variances. Meal and day
   totals combine items by the Fenton–Wilkinson approximation, with shared
   card error treated as a common factor. Estimates are shown with 80 %
   intervals, which are recalibrated against the user's kitchen-scale
   measurements by split-conformal prediction.

The card does more than set the scale. Its two families of edges meet at
vanishing points whose product fixes the camera's focal length, and the
homography then decomposes into the camera's height above the table and its
viewing angle. A second photograph from about 45° turns food height into a
measurement: a point at height $h$ appears displaced from the camera's nadir
by the factor $c_z/(c_z - h)$, so the far edge of an item's outline in the
side view gives $h$ directly, with a precision that improves as $\cot\theta$
of the viewing angle.

Food databases are regional packs (Hong Kong, United Kingdom, United States)
chosen at first run or in Settings; each region also sets units, label
schemas, barcode types and spelling.

Body-weight trend uses a local linear trend Kalman filter. Energy expenditure
combines a Mifflin–St Jeor prior with an energy-balance estimate
$\bar I - \rho_E \beta$ by precision weighting, and calorie targets follow
from it within user-set guardrails.

## Licence

MIT (to be added with the first code).
