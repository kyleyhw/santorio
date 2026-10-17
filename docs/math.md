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
  for $h \ll Z$. Here $Z$ is the camera's height above the table, which
  §1.5 recovers from the card itself, and §1.5 also gives the exact,
  position-dependent form of this correction. Only when the pose cannot be
  recovered (a view taken straight down, with no focal-length prior) is a
  typical shooting height assumed, with a wide uncertainty **(to be fixed at
  build)**.
- **Outline error.** The AI outline's boundary error contributes
  $\sigma_{\ln A,\mathrm{outline}}$, larger for boxes than polygons, and larger
  for polygons than masks **(to be fixed at build** from the accuracy
  evaluation set, §7**)**.

The card error is **shared** by every item in the same photo; the outline
error is item-specific. This distinction matters when items are summed (§3.3).

### 1.4 Focal length from the card

Section 1.1 defines $H$ as the map from image to plane. Its inverse
$G = H^{-1} = [\,\mathbf g_1\ \mathbf g_2\ \mathbf g_3\,]$ maps plane to
image. For a pinhole camera with intrinsic matrix $K$ and pose $(R, \mathbf t)$
from the table frame to the camera frame, a plane point $(x, y, 0)$ images at

$$
\mathbf p \sim K\,[\,\mathbf r_1\ \ \mathbf r_2\ \ \mathbf t\,]\,(x, y, 1)^\top,
\qquad\text{so}\qquad
G \sim K\,[\,\mathbf r_1\ \ \mathbf r_2\ \ \mathbf t\,],
$$

where $\mathbf r_1, \mathbf r_2$ are the first two columns of $R$
[[15]](#ref-zhang). The columns $\mathbf g_1$ and $\mathbf g_2$ are the images
of the plane's points at infinity along $x$ and $y$: the vanishing points of
the card's two families of edges.

Phone cameras have square pixels, negligible skew and a principal point at
the image centre $(u_0, v_0)$. In centred coordinates $u' = u - u_0$,
$v' = v - v_0$ the intrinsic matrix is $K = \operatorname{diag}(f, f, 1)$ with
the focal length $f$ (in pixels) as the only unknown. The columns
$\mathbf r_1, \mathbf r_2$ of a rotation are orthogonal and of equal length,
so with $\omega = K^{-\top}K^{-1} = \operatorname{diag}(f^{-2}, f^{-2}, 1)$,

$$
\mathbf g_1^\top\,\omega\,\mathbf g_2 = 0,
\qquad
\mathbf g_1^\top\,\omega\,\mathbf g_1 = \mathbf g_2^\top\,\omega\,\mathbf g_2 .
$$

Writing $\mathbf g_i = (a_i, b_i, c_i)^\top$ and expanding,

$$
f^2 = -\frac{a_1 a_2 + b_1 b_2}{c_1 c_2}
\quad\text{(orthogonal edges)},
\qquad
f^2 = -\frac{a_1^2 + b_1^2 - a_2^2 - b_2^2}{c_1^2 - c_2^2}
\quad\text{(equal scale)}.
$$

The first equation is the classical statement that the vanishing points
$\mathbf v_i = (a_i/c_i,\ b_i/c_i)$ of two orthogonal directions satisfy
$\mathbf v_1\cdot\mathbf v_2 + f^2 = 0$ in centred pixel coordinates
[[1]](#ref-hartley). The second exists only because the card's dimensions,
hence its aspect ratio, are known: a rectangle of unknown proportions fixes
the plane only up to a stretch along its edges and yields the first equation
alone. Both equations are linear in $f^2$, so they are solved together by
least squares, and their disagreement, with the aspect-ratio check of §1.1,
diagnoses badly placed corners.

**Degeneracy.** For a view looking straight down the optical axis is parallel
to the table normal, the vanishing points recede to infinity, $c_1 = c_2 = 0$,
and both equations reduce to $0 = 0$: $f$ is unobservable. For a view tilted
by $\theta$ from the vertical the vanishing point of the depth direction lies
at distance $f\cot\theta$ from the principal point, so its position, and
hence $f$, is poorly determined for small $\theta$. The two photographs of a
meal (§1.6) come from the same camera and share one $f$, which is therefore
estimated from both views jointly, each view's equations weighted by the
inverse variance obtained by propagating a corner displacement through them;
the oblique view dominates, as it should.

**Prior.** When a photograph comes from the file system its EXIF
`FocalLengthIn35mmFilm` value $f_{35}$ (mm) gives
$f = f_{35}\,N_{\mathrm{long}}/36$, with $N_{\mathrm{long}}$ the image's long
side in pixels and 36 mm the long side of the 35 mm frame. It is combined with
the card estimate by precision weighting (§6.3). Camera frames taken in the
browser carry no EXIF. Across sessions the device's $f/N_{\mathrm{long}}$ is
remembered as a running median, so a later single photograph, even one taken
straight down, still has a usable focal length. The viewfinder is held at 1×
zoom; file photographs whose EXIF indicates the ultra-wide camera are declined
with a hint, because its distortion exceeds this model.

**Lens distortion.** The camera pipeline corrects the main lens before the
image is saved; the residual is about 1 % at the frame edge and is included
in $\sigma_{\ln A}$ as a term **(to be fixed at build** from the evaluation
set, §7**)**. The capture guide keeps the card and plate within the central
two-thirds of the frame.

### 1.5 Camera pose: height above the table and viewing angle

With $K$ known, let $B = K^{-1}G = [\,\mathbf b_1\ \mathbf b_2\ \mathbf b_3\,]$.
Then $\mathbf b_1 = \lambda\mathbf r_1$, $\mathbf b_2 = \lambda\mathbf r_2$ and
$\mathbf b_3 = \lambda\mathbf t$ for one unknown scale $\lambda$, so

$$
\lambda = \tfrac12\big(\|\mathbf b_1\| + \|\mathbf b_2\|\big),
\qquad
\mathbf r_1 = \frac{\mathbf b_1}{\lambda},\quad
\mathbf r_2 = \frac{\mathbf b_2}{\lambda},\quad
\mathbf r_3 = \mathbf r_1\times\mathbf r_2,\quad
\mathbf t = \frac{\mathbf b_3}{\lambda},
$$

with the overall sign of $G$ chosen so that $t_z > 0$ (the card lies in front
of the camera). Corner noise makes $[\,\mathbf r_1\ \mathbf r_2\ \mathbf r_3\,]$
only approximately orthonormal; it is replaced by the nearest rotation
$R = UV^\top$ from its singular value decomposition $U\Sigma V^\top$
[[15]](#ref-zhang). Because the card's coordinates are in millimetres,
$\mathbf t$ and everything derived from it are in millimetres.

The camera centre in the table frame is

$$
\mathbf C = -R^\top\mathbf t = (c_x,\ c_y,\ c_z)^\top :
$$

$c_z$ is the camera's height above the table and $\mathbf n = (c_x, c_y)$ is
the **nadir**, the point on the table directly beneath the camera. The optical
axis in the table frame is the third row of $R$, and the viewing angle from
the vertical is

$$
\theta = \arccos\!\big(-R_{33}\big),
$$

near $0$ for a photograph taken from above and near $45°$ for an oblique one.
The gyroscope tilt shown in the viewfinder guides the user before the shot;
the card's $\theta$ is what the computation uses afterwards.

**Exact parallax.** A point at height $h$ above table position $\mathbf q$
lies on the ray from $\mathbf C$; that ray meets the table where
$c_z + \lambda(h - c_z) = 0$, i.e. at $\lambda = c_z/(c_z - h)$, so the plane
homography places the point at

$$
\mathbf q' = \mathbf n + \frac{c_z}{c_z - h}\,(\mathbf q - \mathbf n),
$$

displaced radially away from the nadir by the factor $c_z/(c_z - h)$. This
replaces the approximate $(1 - h/Z)^{-2}$ of §1.3 (which is the special case
of an item centred under the camera) by an exact, position-dependent
correction: once $h$ is known (§1.6), each outline vertex that belongs to the
item's top surface is mapped back by the inverse,
$\mathbf q = \mathbf n + \tfrac{c_z - h}{c_z}(\mathbf q' - \mathbf n)$, while
vertices on the item's base need no correction. Which vertices are which
follows from the shape class (§2.1).

### 1.6 Two photographs: measured height from parallax

**Protocol.** The user takes one photograph from above ($\theta_1$ small) and
one from the side at $\theta_2$ between about $30°$ and $60°$, with the card
fully visible in both and nothing moved between the shots. Roles are assigned
from the measured angles, not from the order of capture. Because each view's
pose is recovered from the card alone (§1.5), both are expressed in the card's
frame directly: no feature matching between the photographs and no
structure-from-motion is needed. The method extends unchanged to more than two
views; two is the budgeted default ([`design.md`](design.md) §5.3).

**Common frame.** A rectangle is unchanged by a half-turn, so the
correspondence between the card's corners in the two views is ambiguous by a
rotation of $180°$ about the card's centre; the wrong choice rotates one
view's whole scene by a half-turn. Both choices are tried and the one kept for
which the two views' food outlines, mapped to the table plane, overlap (largest
intersection over union), with the card's printed face (chip, text) as a
tie-break. Reflections are excluded because the corners are traversed in the
same rotational sense in both views, which fixes the handedness.

**Items.** Both images are sent to the AI in one request, which returns, per
item, an outline in each image under one label; the geometric overlap test
above verifies the matching. The AI's own height impression from the pair is
retained only as a prior and combined with the measurement below by precision
weighting (§6.3).

**Estimator.** For an item, let $F$ be its footprint from photograph 1 (the
outline through $H_1$; for $\theta_1 \approx 0$ and $h \ll c_{z,1}$ the
parallax of §1.5 is small and is applied iteratively). Let $S$ be its outline
in photograph 2 mapped through $H_2$ to the table plane, $\mathbf n_2$ the
nadir of photograph 2 and $\hat{\mathbf d}$ the unit vector from $\mathbf n_2$
towards the centroid of $F$. Define the far extents

$$
e_F = \max_{\mathbf q\in F}\ (\mathbf q - \mathbf n_2)\cdot\hat{\mathbf d},
\qquad
e_S = \max_{\mathbf q\in S}\ (\mathbf q - \mathbf n_2)\cdot\hat{\mathbf d}.
$$

The far edge of $S$ is the image of the item's far limb, which is raised above
the table. By §1.5 a point of the item at height $z$ whose horizontal section
reaches radial extent $\rho(z)$ appears at radial extent
$\rho(z)\,c_{z,2}/(c_{z,2} - z)$, so

$$
e_S = \max_{0 \le z \le h}\ \frac{c_{z,2}}{c_{z,2} - z}\,\rho(z).
$$

For a flat-topped item ($\rho(z) = e_F$ for all $z$) the maximum is at the top:

$$
e_S = e_F\,\frac{c_{z,2}}{c_{z,2} - h}
\quad\Longrightarrow\quad
h = c_{z,2}\left(1 - \frac{e_F}{e_S}\right).
$$

For a mound, $\rho(z) = (e_F - a) + a\,\psi(z/h)$, where $a$ is the item's
half-extent along $\hat{\mathbf d}$ and $\psi$ is the profile of its shape
class in §2.1 ($\psi(s) = \sqrt{1 - s}$ for a paraboloid, $1 - s$ for a cone,
$\sqrt{1 - s^2}$ for a hemispherical cap). The right-hand side increases
monotonically with $h$, since a larger $h$ lifts every section further, so $h$
is found by bisection. The far edge of $S$ therefore measures $h$; the AI
supplies only the shape class.

**Precision.** For a flat top with $h \ll c_{z,2}$,
$e_S \approx e_F\,(1 + h/c_{z,2})$, hence

$$
\sigma_h \approx \sigma_{e_S}\,\frac{c_{z,2}}{e_F} = \sigma_{e_S}\cot\theta_{\mathrm{far}},
\qquad \tan\theta_{\mathrm{far}} = \frac{e_F}{c_{z,2}},
$$

where $\theta_{\mathrm{far}}$ is the angle from the vertical at which the far
edge is seen. The height resolution improves with obliqueness: at $20°$ the
outline error is magnified $2.7\times$, at $45°$ it passes through unchanged,
at $60°$ it is halved. Beyond about $60°$ the card is foreshortened (its short
side shrinks as $\cos\theta$, degrading the corners) and items occlude one
another, hence the $30°$–$60°$ window with a $45°$ target. $\sigma_{e_S}$
follows from the outline's pixel error through the Jacobian of $H_2$ at the
far edge (§1.2), and $\sigma_{\ln h} = \sigma_h/h$ replaces the AI's
guessed-height term in §3.1. A worked case: at a camera height of 45 cm and
$\theta_2 = 45°$, a 1024 px image places about 1.4 px per millimetre near the
card, so an outline error of 3 px is 2 mm on the table and 2 mm in $h$, i.e.
$\sigma_{\ln h} \approx 0.07$ for a 3 cm mound against roughly 0.3–0.5 for a
height guessed from one image **(both to be confirmed at build** from the
evaluation set**)**.

**Checks and failure modes.**

- The near edges of $F$ and $S$ coincide for an item lying wholly beyond the
  nadir (its near silhouette is its base), which checks the frame
  correspondence and the outline quality.
- Where another item's outline overlaps the far edge in photograph 2, the
  extent is truncated and $h$ would be underestimated; the item falls back to
  the AI height with its wider uncertainty, and the capture guide asks for the
  side photograph to be taken with the tallest items furthest from the camera.
- If $\theta_2 < 20°$ or the card is not found in one view, the second
  photograph adds little and the single-view model is used, with a note to
  the user.

**Bowls.** The oblique photograph shows the far inner wall down to the food
surface. The far rim, at height $z_r$ and radial extent $\rho_r$, and the
surface's far edge, at height $z_s$ and radial extent $\rho_s \le \rho_r$
given by the bowl profile, appear at $\rho_r\,c_{z,2}/(c_{z,2} - z_r)$ and
$\rho_s\,c_{z,2}/(c_{z,2} - z_s)$. Their separation fixes $z_r - z_s$, the
depth of the surface below the rim, and hence the fill fraction $f$ of §2.2
without a slider.

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
tends to $\tfrac{1}{2} A h$ when $h \ll a$. With two photographs, $h$ is
measured from the parallax of the oblique view (§1.6) and the AI supplies only
the shape class. With a single photograph the AI supplies $h$ as well, and $h$
is then the dominant error term of the card method.

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

Label energy is checked against the energy computed from the declared
nutrients. The conversion factors are set by each region's labelling law, and
the check must mirror the region of the label being read:

| Region | Protein | Carbohydrate | Fat | Fibre | Alcohol | Energy unit | Sodium |
|---|---|---|---|---|---|---|---|
| United States [[17]](#ref-fda) | 4 kcal/g | 4 kcal/g, *including* fibre | 9 kcal/g | within carbohydrate | 7 kcal/g | kcal only | sodium (mg) |
| United Kingdom and EU [[16]](#ref-eu1169) | 17 kJ/g (4 kcal) | 17 kJ/g (4 kcal), *excluding* fibre | 37 kJ/g (9 kcal) | 8 kJ/g (2 kcal) | 29 kJ/g (7 kcal) | kJ and kcal | salt = 2.5 × sodium |
| Hong Kong [[18]](#ref-hkcfs) | 4 kcal/g | 4 kcal/g, available carbohydrate | 9 kcal/g | 2 kcal/g | 7 kcal/g | kcal or kJ | sodium (mg) |

The factors are the Atwater general factors [[3]](#ref-merrill) in every
region; the regions differ in whether fibre is inside the carbohydrate figure
and in units. With $C'$ the carbohydrate excluding fibre ($C' = C - \mathrm{Fi}$
for a United States label), the check is

$$
E_{\mathrm{calc}} = 4P + 4C' + 9F + 2\,\mathrm{Fi} + 7\,\mathrm{Alc}
\quad\text{(kcal)},
\qquad 1\ \mathrm{kcal} = 4.184\ \mathrm{kJ},
$$

and a relative discrepancy $|E_{\mathrm{label}} - E_{\mathrm{calc}}| / E_{\mathrm{label}}$
above a threshold flags a likely misread. Label rounding rules, polyols and
organic acids make some discrepancy normal, and United States manufacturers
may lawfully use specific factors or subtract insoluble fibre, so the
threshold is region-specific and **to be fixed at build** from a sample of
real labels from each region. The Hong Kong factors are to be confirmed
against the current Technical Guidance Notes at build.

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
build)** in §1–§2 are set from this evaluation. The two-view geometry of
§1.4–§1.6 is additionally tested on synthetic scenes: mounds of known height
rendered through pinhole cameras of known pose, with seeded corner and outline
noise, so that the recovered $f$, $\theta$, $c_z$ and $h$ are compared with
their true values before any real photograph is used.

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

<span id="ref-zhang">[15]</span> Zhang, Z. (2000). A flexible new technique for camera calibration. *IEEE Transactions on Pattern Analysis and Machine Intelligence*, 22(11), 1330–1334. [Link](https://doi.org/10.1109/34.888718)

<span id="ref-eu1169">[16]</span> Regulation (EU) No 1169/2011 of the European Parliament and of the Council on the provision of food information to consumers, Annex I (definition of salt) and Annex XIV (conversion factors); retained in United Kingdom law. [Link](https://eur-lex.europa.eu/eli/reg/2011/1169/oj)

<span id="ref-fda">[17]</span> United States Code of Federal Regulations, Title 21 §101.9(c), *Nutrition labeling of food*: calorie calculation methods and the definition of total carbohydrate. [Link](https://www.ecfr.gov/current/title-21/chapter-I/subchapter-B/part-101/subpart-A/section-101.9)

<span id="ref-hkcfs">[18]</span> Centre for Food Safety, Hong Kong. *Technical Guidance Notes on Nutrition Labelling and Nutrition Claims.* Food and Environmental Hygiene Department. [Link](https://www.cfs.gov.hk/english/food_leg/food_leg_Technical_Guidance_Notes.html)
