# Mathematics

This document sets out the mathematical models behind Santorio's estimates.
Each section states the model, derives the quantities the app computes, and
identifies the error terms that feed the uncertainty shown to the user. The
design consequences are summarised in [`design.md`](design.md) §6.

Constants whose values are not yet fixed are marked **(to be fixed at
build)**, together with the method that will fix them; the build may not
hard-code a value without recording its justification here.

---

## 1. Reference-card geometry

### 1.1 Plane-to-image homography

Let $(u, v)$ be pixel coordinates and $(x, y)$ metric coordinates on the table
plane. A pinhole camera maps the plane to the image by a projective
transformation (homography) $H \in \mathbb{R}^{3\times 3}$, defined up to
scale [[1]](#ref-hartley):

$$
\begin{pmatrix} x' \\ y' \\ w \end{pmatrix} = H \begin{pmatrix} u \\ v \\ 1 \end{pmatrix},
\qquad x = \frac{x'}{w}, \quad y = \frac{y'}{w}.
$$

The card supplies four correspondences: its detected image corners
$(u_i, v_i)$ and its true corners
$(0,0),\ (L,0),\ (L,W),\ (0,W)$ with $L = 85.60$ mm and $W = 53.98$ mm, the
ISO/IEC 7810 ID-1 dimensions shared by Octopus and payment cards
[[2]](#ref-iso7810). Eliminating $w$ gives two linear equations per
correspondence in the entries $\mathbf{h} = (h_{11}, \dots, h_{33})^\top$:

$$
\begin{pmatrix}
u_i & v_i & 1 & 0 & 0 & 0 & -x_i u_i & -x_i v_i & -x_i \\
0 & 0 & 0 & u_i & v_i & 1 & -y_i u_i & -y_i v_i & -y_i
\end{pmatrix} \mathbf{h} = \mathbf{0}.
$$

Stacking the four correspondences gives $A\mathbf{h} = \mathbf{0}$ with
$A \in \mathbb{R}^{8 \times 9}$. The solution (up to scale) is the right
singular vector of $A$ for its smallest singular value (direct linear
transform). Coordinates are first normalised (translated to zero centroid and
scaled to mean distance $\sqrt{2}$) to keep $A$ well conditioned, as
recommended in [[1]](#ref-hartley).

The card's orientation (landscape or portrait in the image) is resolved by
matching the longer image edge to $L$. As a sanity check, the rectified card
must reproduce the aspect ratio $L/W \approx 1.586$; a large deviation means
the corners are wrong and the user is asked to adjust them.

### 1.2 Footprint area

A homography maps straight lines to straight lines, so a polygonal outline
with image vertices $(u_k, v_k)$ maps to a polygon with vertices
$(x_k, y_k)$, whose area follows from the shoelace formula:

$$
A = \frac{1}{2}\left| \sum_{k=1}^{n} \left( x_k\, y_{k+1} - x_{k+1}\, y_k \right) \right|,
\qquad (x_{n+1}, y_{n+1}) \equiv (x_1, y_1).
$$

For a pixel mask $M$, the local area scale varies across the image. The area
element is $dA = |\det J(u,v)|\, du\, dv$, where $J$ is the Jacobian of
$(u,v) \mapsto (x,y)$. For a homography this determinant has the closed form

$$
\det J(u, v) = \frac{\det H}{w(u,v)^3}, \qquad w(u,v) = h_{31} u + h_{32} v + h_{33},
$$

so the mask area is the sum $A = \sum_{(u,v) \in M} |\det H| / |w(u,v)|^3$
over mask pixels (unit pixel area).

### 1.3 Error sources in the area

- **Corner error.** A corner displacement $\delta$ pixels on a card spanning
  $s$ pixels changes the inferred scale by roughly $\delta/s$ per axis, hence
  area by about $2\delta/s$. This is propagated by perturbing the corners and
  recomputing (deterministically, over the $2^4$ sign combinations of a fixed
  displacement), giving $\sigma_{\ln A,\mathrm{card}}$.
- **Parallax.** The homography is exact only on the table plane. A point at
  height $h$ above the plane, seen from camera distance $Z$, is magnified by
  $Z/(Z-h)$, so an outline traced at the top of a mound overstates the
  footprint area by a factor of approximately $(1 - h/Z)^{-2} \approx 1 + 2h/Z$
  for $h \ll Z$. This is applied as a bias correction with its own
  uncertainty, since $Z$ is only approximately known in a browser (no camera
  intrinsics are exposed).
- **Outline error.** The AI outline's boundary error contributes
  $\sigma_{\ln A,\mathrm{outline}}$, larger for boxes than polygons, and larger
  for polygons than masks **(to be fixed at build** from the accuracy
  evaluation set, §7**)**.

The card error is **shared** by every item in the same photo; the outline
error is item-specific. This distinction matters when items are summed (§3.3).

---

## 2. Volume, mass and energy

### 2.1 Mounded food on a flat plate

For a mound with footprint area $A$ and maximum height $h$,

$$
V = k\, A\, h,
$$

where the shape factor $k$ depends on the profile: $k = 1$ for a slab,
$k = 1/2$ for a paraboloid or a shallow spherical cap, $k = 1/3$ for a cone.
The spherical-cap case follows from $V_{\mathrm{cap}} = \tfrac{\pi h}{6}(3a^2 + h^2)$
with $A = \pi a^2$, giving $V = \tfrac{1}{2} A h + \tfrac{\pi h^3}{6}$, which
tends to $\tfrac{1}{2} A h$ when $h \ll a$. The AI supplies $h$ and a shape
class; both carry uncertainty, and $h$ is the dominant error term of the card
method. The optional 45° photo constrains $h$ directly.

### 2.2 Food in a bowl

Model the bowl interior as a spherical cap with rim radius $a$ (measured from
the homography) and depth $D$ (from a bowl-type preset). The sphere radius is

$$
R = \frac{a^2 + D^2}{2D}.
$$

Filled to a height $h = fD$ above the lowest point ($0 < f \le 1$, chosen by
the user on a fill-level slider), the food occupies a cap of the same sphere:

$$
V(h) = \frac{\pi h^2 (3R - h)}{3}.
$$

At $f = 1$ this reduces to $\tfrac{\pi D}{6}(3a^2 + D^2)$, the full-cap volume,
as it should. The uncertainty in $D$ (preset) and $f$ (user judgement) is
larger than for a plate, so bowl items are marked lower confidence.

### 2.3 Discrete items

For countable foods, $m = N\, \bar m_{\mathrm{piece}}$, where $N$ is the count
and $\bar m_{\mathrm{piece}}$ the typical piece mass from the food pack. The
error is that of $\bar m_{\mathrm{piece}}$ only (counts are exact once
confirmed), which is why counting is preferred where it applies.

### 2.4 Mass and energy

$$
m = \rho\, V, \qquad E = \frac{m\, e_{100}}{100},
$$

with $\rho$ the density (g/ml) from the density table and $e_{100}$ the
energy per 100 g. Macronutrients scale identically.

### 2.5 Label consistency check

Label energy is checked against the Atwater general factors
[[3]](#ref-merrill):

$$
E_{\mathrm{Atwater}} = 4P + 4C + 9F \quad (+\,7\,\mathrm{alcohol}),
$$

with $P, C, F$ in grams and $E$ in kcal. A relative discrepancy
$|E_{\mathrm{label}} - E_{\mathrm{Atwater}}| / E_{\mathrm{label}}$ above a
threshold flags a likely misread. Label rounding rules and components outside
the formula (fibre, polyols) make some discrepancy normal; the threshold is
**to be fixed at build** from a sample of real labels.

---

## 3. Uncertainty propagation

### 3.1 Log-normal multiplicative errors

Every factor in §2 is a product, and its errors are proportional to its size.
Each estimated factor $X$ is therefore modelled as log-normal:
$\ln X \sim \mathcal{N}(\mu_X, \sigma_X^2)$. For
$m = k A h \rho$ with independent factor errors,

$$
\ln m = \ln k + \ln A + \ln h + \ln \rho,
\qquad
\sigma_{\ln m}^2 = \sigma_{\ln k}^2 + \sigma_{\ln A}^2 + \sigma_{\ln h}^2 + \sigma_{\ln \rho}^2,
$$

and for energy $\sigma_{\ln E}^2 = \sigma_{\ln m}^2 + \sigma_{\ln e}^2$. This is
the standard first-order propagation of the GUM [[4]](#ref-gum), exact here
because the model is linear in log space.

### 3.2 Displayed interval

The point estimate shown is the median $\tilde E = e^{\mu}$. The central 80 %
interval is

$$
\left[\, \tilde E\, e^{-z\sigma},\ \tilde E\, e^{z\sigma} \,\right],
\qquad z = \Phi^{-1}(0.9) \approx 1.2816.
$$

The 80 % level follows forecasting convention [[5]](#ref-fpp3). It is chosen
over 95 % because 95 % intervals on single-photo estimates would be too wide
to inform a decision, and because 80 % coverage can be verified with fewer
calibration samples: the standard error of an empirical coverage estimate is
$\sqrt{p(1-p)/n}$, and checking a 95 % claim requires many more misses to be
observed than checking an 80 % one.

**Rounding.** With interval half-width $w$, values are rounded to the largest
step in $\{1, 5, 10, 25, 50\}$ kcal not exceeding $w/4$. The rounding error is
then at most $w/8$, small relative to the uncertainty, and no digits are shown
that the estimate cannot support.

### 3.3 Sums: meals and days

Totals add log-normal variables, whose sum is not log-normal. It is
approximated by the Fenton–Wilkinson method [[6]](#ref-fenton), which matches
the first two moments. For independent items $i$,

$$
M = \sum_i e^{\mu_i + \sigma_i^2/2}, \qquad
V = \sum_i \left(e^{\sigma_i^2} - 1\right) e^{2\mu_i + \sigma_i^2},
$$

$$
\sigma_S^2 = \ln\!\left(1 + \frac{V}{M^2}\right), \qquad
\mu_S = \ln M - \frac{\sigma_S^2}{2}.
$$

Errors shared by all items in one photo (the card scale, §1.3) are not
independent. They are factored out as a common multiplier $e^{\varepsilon}$
with $\varepsilon \sim \mathcal{N}(0, \sigma_{\mathrm{card}}^2)$, the sum is
formed from the item-specific parts, and because a product of log-normals is
log-normal, the total has $\sigma_{\mathrm{total}}^2 = \sigma_S^2 + \sigma_{\mathrm{card}}^2$.
The approximation is verified in tests against a Monte Carlo reference that
uses a seeded NumPy generator.

---

## 4. Calibration against ground truth

When the user weighs an item, the pair $(\hat m_j, m_j)$ yields a log residual
$r_j = \ln(m_j / \hat m_j)$ and a standardised residual $s_j = r_j / \sigma_j$,
where $\sigma_j$ is the model's predicted log standard deviation.

- **Bias.** Within a food category, the median residual $b$ estimates a
  systematic multiplicative bias, corrected by $\hat m \leftarrow \hat m\, e^{b}$.
  With few samples the correction is shrunk towards zero; the shrinkage
  strength is **to be fixed at build**.
- **Width (split-conformal calibration).** Let $s'_j = |r_j - b| / \sigma_j$
  over $n$ calibration samples and $\alpha = 0.2$. Take $c$ as the
  $\lceil (n+1)(1-\alpha) \rceil$-th smallest $s'_j$. Intervals then use
  $c\,\sigma$ in place of $z\sigma$. Under exchangeability this guarantees
  marginal coverage of at least $1-\alpha$ [[7]](#ref-conformal). The rescaling
  is applied only once $n$ reaches a minimum **(to be fixed at build)**; the
  quantile requires $n \ge 4$ even to exist at $\alpha = 0.2$.

The app reports each method's measured median absolute percentage error and
empirical coverage, so its stated accuracy can be checked.

---

## 5. Weight trend

Daily weigh-ins fluctuate with water and gut content, so the trend is
estimated with a local linear trend model [[8]](#ref-harvey) and a Kalman
filter [[9]](#ref-kalman). The state is level $\ell_t$ (kg) and slope
$\beta_t$ (kg/day); for an observation after a gap of $\Delta t$ days,

$$
\begin{pmatrix} \ell_t \\ \beta_t \end{pmatrix}
= \begin{pmatrix} 1 & \Delta t \\ 0 & 1 \end{pmatrix}
\begin{pmatrix} \ell_{t-1} \\ \beta_{t-1} \end{pmatrix} + \boldsymbol{\eta}_t,
\qquad
y_t = \ell_t + \varepsilon_t,
$$

with $\boldsymbol{\eta}_t \sim \mathcal{N}(\mathbf{0}, Q\,\Delta t)$ and
$\varepsilon_t \sim \mathcal{N}(0, r)$. The standard predict and update steps
give the filtered level (the trend line), the slope, and their variances;
a Rauch–Tung–Striebel smoother gives the plotted history. Irregular sampling
and missing days are handled naturally through $\Delta t$. The noise
parameters $Q$ and $r$ are estimated per user by maximum likelihood once
enough weigh-ins exist; until then conservative defaults are used **(to be
fixed at build, with sources)**.

The trend is drawn with a shaded 80 % band, so a "change" smaller than the
band is visibly not a change.

---

## 6. Energy expenditure and targets

### 6.1 Initial estimate

Resting energy expenditure from the Mifflin–St Jeor equation
[[10]](#ref-mifflin), with mass $W$ (kg), height $H$ (cm) and age $a$ (years):

$$
\mathrm{REE} = 10W + 6.25H - 5a + s,
\qquad
s = \begin{cases} +5 & \text{male equation} \\ -161 & \text{female equation} \\ -78 & \text{average of the two} \end{cases}
$$

Total expenditure is $\mathrm{TDEE}_0 = \mathrm{REE} \times \mathrm{PAL}$, with
the physical activity level from the user's activity choice. In a systematic
review, Mifflin–St Jeor was the equation most often within 10 % of measured
resting expenditure [[11]](#ref-frankenfield); the prior standard deviation
starts from that ±10 % and is widened for activity-level uncertainty **(to be
fixed at build)**.

### 6.2 Energy-balance estimate

Over a window of complete-log days $\mathcal{D}$ ($n = |\mathcal{D}|$), with
mean logged intake $\bar I$ and trend slope $\beta$ from §5,

$$
\hat{E}_{\mathrm{data}} = \bar I - \rho_E\, \beta,
\qquad
\operatorname{Var}\!\big(\hat E_{\mathrm{data}}\big) \approx \operatorname{Var}(\bar I) + \rho_E^2 \operatorname{Var}(\beta) + \beta^2 \operatorname{Var}(\rho_E),
$$

where $\rho_E \approx 7700$ kcal/kg is the energy content of body-mass change.
The value derives from the classic 3500 kcal/lb rule [[12]](#ref-wishnofsky)
(3500 × 2.2046 ≈ 7716). It is an approximation, since the energy density of
weight change depends on its fat and lean composition [[13]](#ref-hall), and
so it carries its own uncertainty. Days not fully logged are excluded, never
treated as zero intake.

Consistent under-logging biases $\bar I$, and hence $\hat E_{\mathrm{data}}$,
by the same factor. Because targets are expressed in the same logged
calories, a consistent bias cancels in the feedback loop; only
*inconsistent* logging degrades the estimate. This is why the method adapts
correctly even for imperfect loggers.

### 6.3 Combining prior and data

The two estimates are combined by precision weighting (the normal–normal
Bayesian update):

$$
\hat E = \frac{\mu_0/\sigma_0^2 + \hat E_{\mathrm{data}}/\sigma_{\mathrm{data}}^2}{1/\sigma_0^2 + 1/\sigma_{\mathrm{data}}^2},
\qquad
\frac{1}{\sigma_{\hat E}^2} = \frac{1}{\sigma_0^2} + \frac{1}{\sigma_{\mathrm{data}}^2}.
$$

With little data the prior dominates; with several weeks of consistent logging
the data dominate. This replaces an arbitrary switch-over with a continuous
transition.

### 6.4 Target update with guardrails

For a goal rate $g$ (kg/day, negative for loss), the raw target is
$T^\ast = \hat E + \rho_E\, g$. The suggested target is $T^\ast$ clipped by the
spec's guardrails, which are user-adjustable defaults rather than physiological
constants:

- weekly change $|T - T_{\mathrm{prev}}| \le 100$–$150$ kcal;
- deficit $\hat E - T \le 500$ kcal, surplus $T - \hat E \le 300$ kcal;
- $T \ge$ the calorie floor ([`design.md`](design.md) §11);
- at least 10 complete-log days, and $\sigma_{\hat E}$ below a threshold
  **(to be fixed at build)**; otherwise no change is suggested.

### 6.5 Protein

The daily protein range is $[1.6, 2.2]\ \mathrm{g/kg} \times W$. The lower
bound is the meta-analytic estimate beyond which additional protein gave no
further gain in fat-free mass with resistance training; the upper bound is the
upper 95 % confidence limit of that estimate [[14]](#ref-morton).

---

## 7. Accuracy evaluation

The evaluation tool (`tools/eval`, Python) runs the photo pipeline over images
with kitchen-scale ground truth and reports, per method, model and prompt
version: median absolute percentage error, bias, and empirical coverage of the
80 % intervals, with plots of predicted against true mass on log axes. The
public repository ships synthetic and licensed images; personal images stay in
the user's private wrapper. The error constants marked **(to be fixed at
build)** in §1–§2 are set from this evaluation.

---

## References

<span id="ref-hartley">[1]</span> Hartley, R., & Zisserman, A. (2004). *Multiple View Geometry in Computer Vision* (2nd ed.). Cambridge University Press. [Link](https://doi.org/10.1017/CBO9780511811685)

<span id="ref-iso7810">[2]</span> ISO/IEC 7810:2019. *Identification cards — Physical characteristics.* International Organization for Standardization. [Link](https://www.iso.org/standard/70483.html)

<span id="ref-merrill">[3]</span> Merrill, A. L., & Watt, B. K. (1973). *Energy Value of Foods: Basis and Derivation* (Agriculture Handbook No. 74). United States Department of Agriculture.

<span id="ref-gum">[4]</span> JCGM 100:2008. *Evaluation of measurement data — Guide to the expression of uncertainty in measurement.* Joint Committee for Guides in Metrology. [Link](https://www.bipm.org/en/committees/jc/jcgm/publications)

<span id="ref-fpp3">[5]</span> Hyndman, R. J., & Athanasopoulos, G. (2021). *Forecasting: Principles and Practice* (3rd ed.). OTexts. [Link](https://otexts.com/fpp3/)

<span id="ref-fenton">[6]</span> Fenton, L. F. (1960). The sum of log-normal probability distributions in scatter transmission systems. *IRE Transactions on Communications Systems*, 8(1), 57–67. [Link](https://doi.org/10.1109/TCOM.1960.1097606)

<span id="ref-conformal">[7]</span> Angelopoulos, A. N., & Bates, S. (2021). *A gentle introduction to conformal prediction and distribution-free uncertainty quantification.* arXiv:2107.07511. [Link](https://arxiv.org/abs/2107.07511)

<span id="ref-harvey">[8]</span> Harvey, A. C. (1989). *Forecasting, Structural Time Series Models and the Kalman Filter.* Cambridge University Press. [Link](https://doi.org/10.1017/CBO9781107049994)

<span id="ref-kalman">[9]</span> Kalman, R. E. (1960). A new approach to linear filtering and prediction problems. *Journal of Basic Engineering*, 82(1), 35–45. [Link](https://doi.org/10.1115/1.3662552)

<span id="ref-mifflin">[10]</span> Mifflin, M. D., St Jeor, S. T., Hill, L. A., Scott, B. J., Daugherty, S. A., & Koh, Y. O. (1990). A new predictive equation for resting energy expenditure in healthy individuals. *American Journal of Clinical Nutrition*, 51(2), 241–247. [Link](https://doi.org/10.1093/ajcn/51.2.241)

<span id="ref-frankenfield">[11]</span> Frankenfield, D., Roth-Yousey, L., & Compher, C. (2005). Comparison of predictive equations for resting metabolic rate in healthy nonobese and obese adults: a systematic review. *Journal of the American Dietetic Association*, 105(5), 775–789. [Link](https://doi.org/10.1016/j.jada.2005.02.005)

<span id="ref-wishnofsky">[12]</span> Wishnofsky, M. (1958). Caloric equivalents of gained or lost weight. *American Journal of Clinical Nutrition*, 6(5), 542–546. [Link](https://doi.org/10.1093/ajcn/6.5.542)

<span id="ref-hall">[13]</span> Hall, K. D., Sacks, G., Chandramohan, D., Chow, C. C., Wang, Y. C., Gortmaker, S. L., & Swinburn, B. A. (2011). Quantification of the effect of energy imbalance on bodyweight. *The Lancet*, 378(9793), 826–837. [Link](https://doi.org/10.1016/S0140-6736(11)60812-X)

<span id="ref-morton">[14]</span> Morton, R. W., Murphy, K. T., McKellar, S. R., et al. (2018). A systematic review, meta-analysis and meta-regression of the effect of protein supplementation on resistance training-induced gains in muscle mass and strength in healthy adults. *British Journal of Sports Medicine*, 52(6), 376–384. [Link](https://doi.org/10.1136/bjsports-2017-097608)
