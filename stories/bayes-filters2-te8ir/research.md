# Research: Bayesian filter testbed

Discovery for [requirements.md](requirements.md). It covers what exists, the stack comparison that A1 asks for, the design decisions the requirements leave open, measured performance, and risks. Terms follow [GLOSSARY.md](GLOSSARY.md).

## 1. What exists

- **`Jcole33/bayes-filters`** (GitHub, public, default branch `main`, at `b239a9e` on 2026-11-22): holds no code. `main` has a one-line `README.md` (`# bayes-filters`), plus `GLOSSARY.md`, `docs/adr/0001` to `0003` and `stories/bayes-filters-uwi3q/`, which the earlier story's approved documents added. There are no other branches and no pull requests. `Jcole33/bayes-filter` returns a 301 to the same repository id (1461799708), which confirms the rename. The build replaces only the README with the B3 one and leaves the other files alone, including this story's `stories/bayes-filters2-te8ir/` once it is added (Constraints).
- **No existing code** to reuse. The libraries that have filters (FilterPy, PythonRobotics) must not be vendored or imported, because of E3. They are useful only as background reading for the user.
- **Dependency versions on PyPI (2026-11-18, re-checked unchanged on 2026-11-22)**, checked against Python 3.11 (B1):

  | Package | Latest | Latest requires | Newest that supports 3.11 |
  |---|---|---|---|
  | numpy | 2.5.5 | ≥3.12 | 2.4.6 |
  | scipy | 1.18.1 | ≥3.12 | 1.17.1 |
  | matplotlib | 3.11.2 | ≥3.11 | 3.11.2 |
  | pytest (dev) | 9.2.0 | ≥3.10 | 9.2.0 |

  pip honours `Requires-Python`, so it picks numpy 2.4.x and scipy 1.17.x on 3.11 by itself. **Use lower bounds only, with no upper caps:** `numpy>=1.26`, `scipy>=1.11`, `matplotlib>=3.8`, and `pytest>=8` as a dev extra. An upper cap such as `numpy<2.5` would gain nothing and would hold back Python 3.12+ users.
- **Build sandbox facts** for later agents. The sandbox runs Ubuntu 26.04 with Python 3.14 and `uv` in `/usr/local/bin`, and `python3 -m venv` fails because `python3.14-venv` is missing.
  - Use `uv venv --seed` for a fresh environment that has pip, then run `pip install -e .` in it to check B1 literally.
  - `uv python install 3.11` fetches CPython 3.11.17 from `releases.astral.sh` in about 1 s. Use it to check the 3.11 floor.
  - sudo works without a password, but apt needs Ubuntu mirrors that are not on the site list. Nothing here needs apt.

## 2. Stack comparison (A1, A2)

`docs/stack-comparison.md` should be written from this section. The criteria are those of A1.

| Option | Ease of setup | Drawing beliefs | Plugging in a hand-written filter | Repeatability of seeded runs |
|---|---|---|---|---|
| **ROS2 + turtlesim** | Poor. A ROS 2 install is needed (current LTS: Lyrical Luth, May 2026, with Ubuntu 26.04 as Tier 1; Jazzy on 24.04), plus a colcon workspace and sourcing. Under WSL2 it is workable but heavy. | None. turtlesim draws only turtles and pen trails, so it cannot show particles, ellipses or grids. | Each filter becomes a node with subscriptions and publishers, or sits behind a wrapper node. That is boilerplate between the user and the maths. | Possible only with a single-threaded executor, synchronous calls or rosbag replay. DDS timing is not deterministic by default. |
| **ROS2 + RViz2** | As above, plus RViz2 (and WSLg OpenGL). | Good primitives: `PoseArray` for particles, `PoseWithCovarianceStamped` for a covariance ellipse, `OccupancyGrid` or `MarkerArray` for grids. Time plots (error, effective sample size) need a separate tool such as rqt_plot or PlotJuggler. Side-by-side panels mean one RViz display set per filter. | Same node boilerplate as turtlesim. | Same as turtlesim. Replay from bags gives scrubbing, but only through bag tooling. |
| **Gazebo (Jetty, the LTS paired with Lyrical)** | Heaviest option: a physics simulator with 3D rendering, usually installed with ROS 2 through `ros_gz`. GPU rendering under WSLg is fragile. | Built for 3D worlds, not distributions. Beliefs would have to go out to RViz anyway. | Through ROS 2, the same as above. | Physics stepping can be made deterministic, but the 2D kinematic models of C3 do not need a physics engine, and the engine adds sources of nondeterminism. |
| **Plain Python + Matplotlib** (ADR-0001) | `pip install -e .` with NumPy, SciPy and Matplotlib only. The GUI on WSLg uses the TkAgg backend, which needs Tk for the interpreter (`sudo apt install python3-tk` on Ubuntu's Python; bundled in python.org, uv and pyenv builds). | Direct: scatter for particles, `Ellipse` patches, `pcolormesh`/`imshow` for grid marginals, and line plots for errors, all in one figure. Widgets (`Button`, `Slider`, `RadioButtons`) cover the G3 controls. Drawing speed needs care (§5). | A filter is a plain Python class with four methods (D1). Discovery is an import by file path. | Fully deterministic: one process, `numpy.random.SeedSequence`, no threads. |
| **Plain Python + pygame** | Easy: one pip wheel. | Manual. Points, circles and lines are fast, but there is no ellipse-from-covariance, no heatmap colormap, no axes and no time plots, and widgets (sliders, buttons) must be written by hand. | Same as Matplotlib (the harness is the same). | Same as Matplotlib. |
| **Rerun** (`rerun-sdk` 0.43, found by research) | One pip wheel, but it pulls in pyarrow, pillow and psutil and starts a separate native viewer. The viewer uses wgpu (Vulkan by default). Under WSLg it may fall back to software rendering, and the documented workaround is `WGPU_BACKEND=gl` / `--renderer=gl` or the web viewer. | Strong. Its built-in timeline gives record-then-replay with a scrubber for free, it has `Points2D` with per-point colour, line strips for ellipses, image/tensor views for grids, and time-series views. Its blueprints arrange the side-by-side views. | Same harness. Logging calls replace drawing code. | Same harness. Rerun only displays recordings. |
| **PyQtGraph** (0.14, found by research) | Needs a Qt binding: PySide6 6.12 is a large set of wheels (the metapackage pulls WebEngine and more), and PyQt6 is smaller. Qt under WSLg works but needs extra xcb libraries on some distributions. | Very fast scatter (thousands of points at high frame rates), image items for grids, and plot widgets. Ellipses must be drawn by hand as curves. | Same harness. | Same harness. |
| **IR-SIM** (`ir-sim` 2.12, found by research) | Pip install. It depends on shapely, pyyaml, imageio and loguru. | Matplotlib-based robot simulator. Its world and robot rendering is built for planning and control, so beliefs and time plots would still be custom Matplotlib code. | It has its own YAML world and robot abstractions, which would wrap or compete with the C1 scenario definition. | It takes a seed, but its noise models are not the C3 velocity model, so the simulation would be replaced anyway. |

**Recommendation: keep plain Python with Matplotlib (ADR-0001). No switch, so A2 needs no user approval, the build leaves ADR-0001 unchanged, and the site list needs no widening.**

- ROS2, RViz and Gazebo fail on setup and plug-in ease for a single-machine learning project. Only RViz draws beliefs well, and it still needs a second tool for the time plots. What ROS is good at (drivers, distributed nodes, real robots) is outside this story's scope (see Non-goals).
- Rerun is the strongest alternative: its timeline would replace much of G3. It loses on three counts. It adds runtime dependencies beyond the Constraints' NumPy/SciPy/Matplotlib, its rendering on WSLg is the least predictable piece of the whole stack, and G7 (headless rendering of a step to an image file, used by tests) would need a second rendering path anyway.
- pygame and PyQtGraph are faster than Matplotlib but make the user's drawing work (ellipses, heatmaps, axes, widgets) hand-written code. Matplotlib meets the G6 frame rate if particles are drawn the way §5 describes.
- The comparison doc should say Rerun is the upgrade path if Matplotlib playback ever becomes the bottleneck, because the record-then-replay design (ADR-0003) means only the viewer would change.

Sources for the doc: ROS 2 Lyrical Luth release notes (docs.ros.org/en/lyrical/Releases/Release-Lyrical-Luth.html), the ros_gz pairing table (github.com/gazebosim/ros_gz), Rerun troubleshooting (rerun.io/docs/getting-started/troubleshooting), and the PyPI pages of `rerun-sdk`, `pyqtgraph`, `pygame`, `ir-sim` and `PySide6` for versions and dependencies.

## 3. Package layout

```
pyproject.toml            # setuptools backend, src layout, Python >=3.11
README.md                 # B3
.gitignore                # includes results/
src/bayes_bench/
  __init__.py             # exports exactly: Filter, GaussianBelief, ParticleBelief, GridBelief, check_jacobian
  _filter.py              # Filter base class (D1, D5 estimate())
  _beliefs.py             # the three belief classes (D4)
  _jacobian.py            # check_jacobian (D2)
  _models.py              # model classes handed to filters, f/h, residuals (D2)
  _initial.py             # initial-belief classes: mean, cov, sample (D3)
  _scenarios.py           # the 11 scenarios (C1, C2), parameters documented in code
  _sim.py                 # PRIVATE simulator: noise generation, ground truth, measurements (D2)
  _harness.py             # runs filters, timing, failed runs (F1, F2, F5)
  _scoring.py             # RMSE, NEES, chi2, recovery/convergence, effective sample size, distinct count (F3)
  _report.py              # aggregation, terminal table, report.md, results.csv (F4, F7)
  _recording.py           # save/load recordings (F2)
  _discovery.py           # filter discovery (D6)
  _gui/                   # replay controller, drawing, widgets (G1-G7)
  _cli.py                 # argparse, entry point `bayes-bench`
filters/placeholders/     # dead_reckoning_gaussian.py, dead_reckoning_particles.py (E1)
benchmarks/fps-filters/   # fps_filters.py: four 2,000-particle subclasses (G6)
docs/writing-a-filter.md, docs/stack-comparison.md   # docs/adr/ exists; the build does not edit it
tests/
```

- **Every internal module name starts with an underscore.** I1 checks that "the names exported by the package are exactly" five names. Python adds every imported submodule as an attribute of the package, so a public `bayes_bench.harness` would show up in `dir(bayes_bench)` once it is imported. With private submodule names, the test can assert both `bayes_bench.__all__` and `{n for n in vars(bayes_bench) if not n.startswith("_")}` equal the allowlist.
- The distribution name is `bayes-bench` and the import name is `bayes_bench`. The entry point is `[project.scripts] bayes-bench = "bayes_bench._cli:main"`. The dev extra is `[project.optional-dependencies] dev = ["pytest>=8"]`.
- Do not put `from __future__ import annotations` (or any other import that binds a public name) in `__init__.py`: it would add a public `annotations` attribute and break the allowlist test. Python 3.11 does not need it.
- `filters/`, `benchmarks/` and `tests/` sit outside `src/`, so they are not importable from the installed package (E3) and a filter can never pick up the simulator by accident.

## 4. Design decisions the requirements leave open

### 4.1 Model object, measurements and vectorization (D2)

- **`f(x, u, dt)` and `h(x, z)` broadcast over leading dimensions:** `x` may be shape `(d,)` or `(N, d)`, and the result has the matching shape. This is not optional in practice. The particle placeholder (E1) moves 1,000 particles each step, and a per-particle Python loop over the F8 workload (66,000 particle-filter steps × 1,000 particles, §5) is around 66 million calls, far over the 2-minute budget. Measured vectorized cost: about 130 µs per step for f plus jitter on 1,000 particles (§5). `state_residual(a, b)` and `measurement_residual(a, b)` broadcast the same way (`a` of shape `(N, …)` against `b` of shape `(…)`), which particle filters and the harness's covariance computation (§4.6) need. A test covers the broadcast case. This is an interface convenience, not derived maths, so documenting it in `writing-a-filter.md` is within H1.
- **Two model classes:** `UnicycleModel` and `LinearGaussianModel`, each with a fixed public attribute set. The I1 allowlist test then checks each class against its own D2 list:
  - both: `f`, `h`, `measurement_residual`, `state_residual`, `position_indices`, `heading_index`, `measurement_std`, `landmarks`, `bounds`, `dt`;
  - unicycle only: `alphas` (a length-6 tuple α₁…α₆);
  - linear only: `accel_std`.

  Implement them as plain classes with read-only properties, not dataclasses, so no stray public names (such as `replace`) appear. Units are metres, seconds and radians.
- **`landmarks`:** an `(L, 2)` float array whose row index is the landmark identity. `linear-gaussian` has shape `(0, 2)`.
- **`bounds`:** `(xmin, xmax, ymin, ymax)`.
- **`measurement_std`:** per reading component, for example `(σ_r, σ_b)` in landmark scenarios, `(σ_r,)` in `bimodal` (range-only) and `(σ_x, σ_y)` in `linear-gaussian`.
- **The measurement `z` passed to `update`:** a small read-only object with
  - `values`, the stacked 1-D array (range-bearing: `[r₁, b₁, r₂, b₂, …]`);
  - `landmark_ids`, a 1-D int array, or `None` in `linear-gaussian`.

  `h(x, z)` uses `z.landmark_ids` to pick and order readings. `measurement_residual(a, b)` takes two arrays of the same layout, and the model knows which positions are bearings. The outlier flag (C2.6) is never on this object. The simulator keeps it in a parallel array that goes into the recording only. The class is not exported, because filters receive instances and never construct them.
- **Straight-line threshold** for C3/D2: `|ω̂| < 1e-6` rad/s, a documented module constant shared by the simulator and `f`.
- **Arc step**, as in Thrun Table 5.3:
  - x' = x − (v̂/ω̂) sin θ + (v̂/ω̂) sin(θ + ω̂·dt);
  - y' = y + (v̂/ω̂) cos θ − (v̂/ω̂) cos(θ + ω̂·dt);
  - θ' = wrap(θ + ω̂·dt + γ̂·dt).

  The straight-line limit is x' = x + v̂·dt·cos θ, y' = y + v̂·dt·sin θ.
- **Single source of truth (ADR-0002):** `_sim.py` takes the scenario's model and calls its own `f`:
  - unicycle: `model.f(x, (v̂, ω̂), dt)` with the noisy control from C3, then adds γ̂·dt to the heading and wraps it;
  - `linear-gaussian`: `model.f(x, u + w, dt)`, with w the acceleration noise drawn using `model.accel_std`;
  - measurements: `model.h(x, z_layout)` plus Gaussian noise drawn using `model.measurement_std`, then the bearings are wrapped.

  The simulator never re-implements f or h. Its noise values come from `model.alphas`, `model.accel_std` and `model.measurement_std`.

- **`check_jacobian(fn, x, jacobian, eps=1e-6, residual=None)`:**
  - it uses central differences and returns `(max_abs_mismatch, (row, col))`, nothing else;
  - `residual` (for example `model.measurement_residual` or `model.state_residual`) is used to difference outputs, so a bearing that crosses ±π between x − eps and x + eps does not give a spurious mismatch of about 2π/eps;
  - H1's how-to shows passing it.

### 4.2 Initial beliefs (D3)

There are three private classes, each with exactly `mean`, `cov` and `sample(n, rng)`, and analytic mean and covariance:

- `GaussianInitialBelief(mean, cov)`;
- `UniformPoseBelief(bounds)` for `global-localization`: position uniform over the map, heading uniform on (−π, π]. Its mean heading is 0 by convention and its heading variance is π²/3. Document that the mean is not meaningful for a uniform belief;
- `BoxPositionGaussianHeadingBelief(box, heading_mean, heading_std)` for `bimodal`.

### 4.3 Seeds and determinism (C6, F6, D1)

- Use `numpy.random.SeedSequence`. Never use Python's `hash()`, which is salted per process. Use `zlib.crc32(name.encode())` as a stable id.
- **Simulation streams:** `SeedSequence([seed, crc32(sim_key)])`, then `.spawn(4)` into independent motion, measurement, outlier and start-pose generators (children 0 to 3, in that order). `SeedSequence.spawn` keys child i by its index, so the start-pose child does not shift the other three. Changing the measurement schedule cannot change the trajectory, and drawing the start cannot change the noise.
  - `sim_key` is the scenario name, except that every scenario defined as "like `nonlinear-landmarks`" declares `sim_key = "nonlinear-landmarks"`: `strongly-nonlinear`, `kidnapped-robot`, `outliers`, the three `noise-sweep-*` and `global-localization`.
  - That gives C2.9 ("same … seeds") and C2.7 ("identical apart from its name") exact equality of the true starting pose, ground truth, controls and measurements. Add a test that asserts this for both.
  - It also means `strongly-nonlinear`, `kidnapped-robot` (before the kidnapping) and `outliers` share the same ground truth. Only their stated differences change what a filter sees. `noise-sweep-low` and `-high` use the same motion and measurement draws scaled by their factors. The start draw is not scaled.
- **Simulate once per (scenario, seed)** and hand the same arrays to every filter. That makes C6 true by construction, and the C6 test compares what each filter received.
- **The filter `rng`:** `default_rng(SeedSequence([seed, crc32(sim_key), crc32(filter_name), 1]))`. It is reproducible and it differs between filters. Keying it on `sim_key`, not the scenario name, makes `noise-sweep-medium` give exactly the same scores as `nonlinear-landmarks` for every filter, including the particle placeholder, as C2.7 leads the user to expect.
- **Byte-identical recordings:** `np.savez_compressed` output was checked to be byte-identical across two writes 2 s apart (NumPy writes a fixed zip timestamp). Recordings can therefore be compared byte for byte, provided runtimes are kept out of the recording files. Store per-step runtimes in a separate `timings.npz` per run. F6 excludes runtimes, so the F6 test compares everything except `timings.npz`, the runtime column of `results.csv` and the runtime score in `report.md`. `manifest.json` holds no timestamp, and the folder's timestamp lives only in its name.

### 4.4 Scenario parameters (C1, C2)

These are suggested values. The build agent may tune them, but the structural choices matter. Every value lives as a named, documented constant in `_scenarios.py`.

- **Common:** `dt = 0.1` s, 300 steps (inside 200 to 500), recovery threshold 1 m.
- **True starting pose drawn per seed:** the true starting pose x₀ is a seeded draw from the start-pose stream (§4.3), not a constant.
  - If the ground truth started exactly at the initial-belief mean on every seed, early errors would be smaller than the stated covariance. Mean NEES would then read low and the χ² percentage high in the first steps, which misleads anyone checking consistency.
  - Each scenario has a nominal start (a constant) and a **start distribution**, which is the Gaussian x₀ is drawn from. For a scenario with its own Gaussian initial belief, it is that same Gaussian, so the initial belief describes the real uncertainty. The deliberate exceptions are `strongly-nonlinear`, whose initial belief has heading std 1.0 rad against the 0.05 rad the start is drawn from, and `global-localization`, whose initial belief is uniform. Their initial beliefs are wider than the real start uncertainty, so NEES reads low early on. `writing-a-filter.md` says so.
  - Scenarios that share `sim_key = "nonlinear-landmarks"` all draw from the base `nonlinear-landmarks` Gaussian (std 0.3 m, 0.3 m, 0.05 rad around (0, −10, 0)), not from their own initial belief. `global-localization` (uniform initial belief) and `strongly-nonlinear` (heading std 1.0 rad) therefore get exactly the same start as `nonlinear-landmarks` on each seed, which keeps C2.9's "same ground-truth starting pose" and C2.7's identity. `strongly-nonlinear`'s initial belief is centred on the nominal start, with the wider heading std it states, and `global-localization`'s covers the whole map.
  - **Exception: `bimodal` keeps a fixed true start**, because the C4 reflection argument relies on it (see below). The requirements and glossary say nothing about a fixed start, so nothing upstream changes.
- **Controls:** a deterministic function of the step index, not of the seed (C1 "its control sequence").
- **The `nonlinear-landmarks` family:**
  - map [−15, 15] × [−15, 15] m;
  - nominal start pose (0, −10, 0), with the true start drawn around it as above;
  - controls v = 1 m/s, ω = 0.1 rad/s, a 10 m-radius circle around the origin. 300 steps cover about half the circle;
  - α = (0.01, 0.001, 0.001, 0.01, 0.0005, 0.0005). At v = 1 m/s this gives std(v̂) ≈ 0.1 m/s and std(ω̂) ≈ 0.03 rad/s;
  - σ_r = 0.3 m and σ_b = 0.05 rad. The outliers' 20σ_b = 1.0 rad stays well under π, so wrapping barely shrinks the stated outlier std;
  - initial belief Gaussian around the nominal start, with std (0.3 m, 0.3 m, 0.05 rad). The same Gaussian is the start distribution.
- **Landmarks:** 6 landmarks, for example (−13, −5), (−5, 13), (4, 13.5), (13.5, 3), (3, −4.5), (−4, 2). Four are outside the nominal 10 m circle and two inside, each at least 3.8 m from it. That distance matters because a landmark on the path drives the range to 0, makes the bearing erratic and makes the range-bearing Jacobian blow up (∝ 1/r). The drawn start shifts the circle (centre shift ≈ 10 m × heading error), so the margin has to cover that. Over 2,000 seeds (exact figures depend on the implementation's noise draw order), the closest the ground truth came to any landmark (per-run minimum) had a median of 3.7 m and a worst case of 1.7 m at ×1, 0.8 m at `noise-sweep-high` and 1.2 m in `kidnapped-robot`. An independent re-check with a different draw order found 0.4 m at `noise-sweep-high`. That is acceptable for a rare worst case at double noise: the Jacobian blow-up needs r → 0. The closest pair of the 15 distances differs by 15 cm, and no three landmarks are near collinear.
  - **Asymmetry test (C2.9):** all 15 pairwise distances are distinct (by more than 1 cm) and the points are not collinear. This is sufficient. An isometry that maps the set onto itself preserves distances, so with all distances distinct it maps each pair onto itself. Each point lies in two pairs whose only shared point is itself, so every point is fixed, and an isometry that fixes three non-collinear points is the identity.
- **Landmarks "in view":** the K nearest landmarks, with K a scenario parameter. K = 6 (all) in `nonlinear-landmarks` and K = 1 in `strongly-nonlinear`. Nearest-K is never empty, so a measurement step always has at least one reading.
- **`strongly-nonlinear`:** initial heading std 1.0 rad instead of 0.05 rad, and K = 1. These are the two parameters C2.3 names. Its ground truth and controls are those of `nonlinear-landmarks` (shared `sim_key`, §4.3).
- **C5 (fewer measurements than controls):** put it in `linear-gaussian`. It measures every 2nd step and has a 40-step dropout window (steps 150 to 189). This shows a Kalman filter's covariance growing between updates, and it keeps the `nonlinear-landmarks` family on the plain every-step schedule. The harness skips `update` on steps without a measurement.
- **`linear-gaussian`:**
  - map [−25, 25] × [−25, 25] m;
  - nominal start state (−7.5, 0, 0, −1.5), with a Gaussian initial belief (and start distribution) of std (0.5 m, 0.5 m, 0.05 m/s, 0.05 m/s);
  - controls a(t) = 0.3·(cos 0.2t, sin 0.2t) m/s², which with that start traces (approximately, see below) a 7.5 m circle around the origin;
  - `accel_std` 0.05 m/s², `measurement_std` (0.5, 0.5) m;
  - generative equations as in D2.

  The circle identities (0.3 = 7.5 · 0.2² and 1.5 = 7.5 · 0.2) hold for the nominal start. A drawn start adds its position offset plus its velocity offset times t, so the ground truth is the circle displaced by δp + δv·t. The velocity std is kept at 0.05 m/s because δv·t integrates: 0.05 m/s gives about 1.5 m after 30 s, while 0.2 m/s would give about 6 m and leaves the map on some seeds. Evaluate the control at the mid-step, a((k + ½)·dt), so the noise-free path stays on the circle. Holding a(k·dt) drifts it up to about 0.45 m off. Position drift from acceleration noise held over each step grows as σ·dt²·√(N³/3): about 1.5 m std after 300 steps at 0.05 m/s², and about 6 m at 0.2 m/s², where the ground truth occasionally leaves the map.
- **`bimodal`:**
  - two landmarks on the line y = 0, at (−5, 0) and (5, 0), with range-only readings (σ_r = 0.3 m);
  - start (−8, 3, θ = 0), v = 0.5 m/s, ω = 0, the family's α values, map [−15, 15] × [−10, 10] m;
  - **the true start stays fixed at (−8, 3, 0) on every seed**, the one exception to the drawn start. Heading 0 is the only heading that is its own mirror, so it keeps the symmetric initial belief below symmetric, and it keeps the true path parallel to the landmark line, as C2.4 demands. A drawn heading would tilt the path off parallel, and drawing only the position would gain little. The doc states this exception and why.

  The alternative trajectory is the reflection R(x, y, θ) = (x, −y, −θ) of the true one.
  - **Measurements:** ranges to points on y = 0 are invariant under R, so the noise-free measurements agree exactly.
  - **Controls:** R maps an arc step with noise draws (ε₁, ε₂, ε₃) onto the arc step with (ε₁, −ε₂, −ε₃). These have the same density, because the noises are zero-mean Gaussian. The C4 test replays the ground truth's recorded draws and the mirrored draws through the private simulator step and checks the reflected path. It also asserts the two paths are distinct (|y| > 1 m throughout).
  - **Initial belief:** uniform over the box x ∈ [−10, −6], y ∈ [−5, 5]. It is symmetric about y = 0 and covers both starting sides. The heading is Gaussian around 0 with documented std 0.1 rad. Heading 0 is its own mirror, so the initial belief is symmetric too. This is why C2.4 insists on driving parallel to the landmark line.
  - Mirrored draws are what makes "fits the controls" checkable: the simulator keeps each step's ε draws in its private output for the test. Tests may import `_sim`; only `filters/` may not.
- **`kidnapped-robot`:** the kidnapping happens at step 150. After that step's motion, the true pose is rotated by π about the noise-free circle centre implied by the drawn start x₀ = (x₀, y₀, θ₀): c = (x₀ − R sin θ₀, y₀ + R cos θ₀), with R = v/ω = 10 m. The new pose is (2c − p, θ + π, wrapped).
  - The robot lands on the opposite side of the same circle, heading the same way round it, so the unchanged controls keep it on that circle and inside the map.
  - The jump is 2·|p − c| ≈ 20 m (smallest over 2,000 seeds: 17.7 m), far over the required 5 m. The simulator asserts |p − c| ≥ 2.5 m at that step (a failed assertion is a harness bug, not a failed run), and the in-bounds test covers it.
  - This is a deliberate, documented difference from the model (D2), like the outliers.
- **`outliers`:** exactly `ceil(0.05 · n)` of the run's n readings, chosen without replacement from the outlier stream (`rng.choice(n, k, replace=False)`). That keeps the fraction fixed at 5% (≥ 3%, C2.6) on every seed. They are replaced by h plus Gaussian noise with 20× the nominal std on each component. Bearings are wrapped, and ranges are left unclipped, which is documented. A negative range is a deliberate outlier, and clipping would change the stated std. The C2.6 test checks that the per-reading flags exist and that their count is exactly k.
- **Placeholder constants (E1):**
  - `dead-reckoning-gaussian` adds `0.01 · I` to its covariance each step;
  - `dead-reckoning-particles` jitters with std 0.02 m on position, 0.01 rad on heading (re-wrapped into (−π, π]) and 0.02 m/s on velocity in `linear-gaussian`.

  Both are class attributes with docstrings, and both are documented as arbitrary.
- **`noise-sweep-*`:** factors s = 0.5, 1, 2 applied to noise standard deviations. The α-parameters multiply variances (C3), so they scale by s², while `measurement_std` scales by s. The doc must state this explicitly, because it is an easy mistake.
- **`stationary-pause`:** the same map, landmarks, nominal start, start distribution and noise values as `nonlinear-landmarks`, with its own `sim_key`, so its draws differ. It drives the circle for 100 steps, gets zero control for 100 steps (more than the 50 required) while measuring every step, then drives for 100 steps. C3 gives zero motion noise at zero control, so a particle filter that resamples without jitter collapses here. This is the depletion case H1 asks the doc to explain.
- **Ground truth inside the map:** the `global-localization` initial belief covers only the map, so the ground truth must stay inside it. Measured over 2,000 seeds with the drawn starts and the values above (about; an independent re-check with a different draw order agreed within 0.5 m) (largest |x| or |y| reached, against an edge at 15 m):
  - `nonlinear-landmarks` ×1: 12.1 m;
  - `noise-sweep-high`: 12.9 m;
  - `stationary-pause`: 12.3 m;
  - `kidnapped-robot`: 12.7 m;
  - `linear-gaussian`: 14.5 m (99th percentile 12.5 m), against its edge at 25 m;
  - `bimodal`: the largest x was 8.0 against an edge at 15, and the smallest |y| was 1.9 m.

  Ground truth leaving the map is not guaranteed impossible for every seed, because the noise is Gaussian. A test checks every scenario on seeds 0 to 99. If a larger seed ever left the map, the run would still complete; only the drawing would clip it. State this in the scenario docs.

### 4.5 Filter discovery (D6)

- Walk `filters_dir.rglob("*.py")` in sorted order. Import each file with `importlib.util.spec_from_file_location`, under a unique module name derived from its resolved path, and register it in `sys.modules`.
- A file's filters are the `Filter` subclasses whose `__module__` is that module's name, meaning those defined there, not those imported into it. That stops a file that imports a placeholder from counting it twice.
- A subclass with no string `name` raises an error naming the file.
- If two discovered classes share a name, raise an error listing both file paths (D6).
- If a file fails to import, stop with an error naming the file. A silent skip would hide the user's mistakes.
- **`benchmarks/fps-filters/fps_filters.py`:** loads `filters/placeholders/dead_reckoning_particles.py` by path, relative to `Path(__file__)`, and defines four subclasses (`fps-particles-1` … `fps-particles-4`) with `num_particles = 2000`. Only the four are discovered, because the base class's `__module__` is the placeholder's module. This adds nothing to the package's exports.

### 4.6 Harness, scoring and failed runs (F2 to F5)

- **Step indexing:** a run has N steps, k = 0…N−1, and N + 1 ground-truth states x₀…x_N. At step k, u_k moves x_k to x_{k+1}, and the measurement z_k, if any, is generated from x_{k+1}. The filter calls `predict(u_k, dt)`, then `update(z_k)` if there is one, then `belief()`. The estimate at step k is scored against x_{k+1}. Convergence time counts from k = 0, and the kidnapping "at step 150" replaces x_151 after the motion of step 150. All per-step arrays in a recording have length N, plus x₀ stored separately.
- **Per step:** call `predict(u, dt)`, call `update(z)` only on measurement steps, then `belief()`. Time `predict + update` with `time.perf_counter`. The estimate comes from `estimate()` if it is overridden, otherwise from the belief (D5).
- **Particle weights** are normalized by the harness for every statistic, and that is documented. A weight (or grid probability) sum that is zero or non-finite, or any negative weight or probability, is a contract violation. NumPy would only warn and give NaN, so the harness checks explicitly and raises a `ValueError` saying which, and that ends the run as failed with that message (F5's "raises an exception" covers it, because the harness treats an error while deriving the estimate like one from the filter).
- **Circular mean** of the heading: `atan2(Σ wᵢ sin θᵢ, Σ wᵢ cos θᵢ)`. The covariance is then `Σ wᵢ rᵢ rᵢᵀ`, with `rᵢ = state_residual(xᵢ, mean)`. For a grid, `wᵢ` is the cell probabilities.
- **Failed runs (F5):** any exception from the filter, or from deriving its estimate (shape errors included), and any non-finite belief or estimate value, ends the run as failed at step k, with the message. Steps before k are kept so the GUI can replay up to the failure (G4). In the replay, a failed run's panel draws normally up to step k − 1. From step k on, it greys out and overlays the text `failed at step k: <message>`, truncated to the panel width. Its time plots stop at k with a red ✕ marker there.
- **NEES:** eᵀP⁻¹e, with e = `state_residual(estimate, ground_truth)`. Treat P as singular when `np.linalg.cholesky` raises, or when its condition number is above 1e12. Leave those steps out and count them. The χ² 95% quantile comes from `scipy.stats.chi2.ppf(0.95, d)`, and the percentage is over usable steps.
- **Recovery and convergence time:** scan the position errors backwards, over steps from the kidnapping step (recovery) or step 0 (convergence) onward, to find the last step at or above the threshold. The stay-under step is the one after it, or the start of the range if there is none, so recovery time is never negative. If the final step is at or above the threshold, report "did not recover" / "did not converge".
- **Effective sample size:** 1/Σwᵢ², from normalized weights.
- **Distinct particle count:** `len(np.unique(rows_as_void))`. View the contiguous particle array as one `np.void` item per row. This took 130 µs against 440 µs for `np.unique(axis=0)` (§5). Normalize −0.0 to 0.0 first (add 0.0), so bit-identical copies are what gets compared.
- **Results folder:** `results/<YYYYmmdd-HHMMSS>/`, with a `-2`, `-3` suffix on collision. It is never deleted. `demo` writes one too, the same way, with `--record-seeds` set to its single seed, then opens `view` on it. That way a demo run can be replayed later.
  - `report.md`: one table per scenario, rows are filters and columns are scores. Scenarios appear in a fixed order with `global-localization` right after `nonlinear-landmarks` (F7).
  - `results.csv`: one row per run with filter, scenario, seed, status, failure step, every score, and the runtime per step.
- **Terminal table:** the same tables, formatted with plain string widths. No `tabulate` dependency is needed. The last line is `Total wall-clock time: N.N s`, measured with `time.perf_counter` from the start of `run` to after the files are written. That is the number F8 is judged by.
- **CLI:** use `argparse.ArgumentDefaultsHelpFormatter` on every subparser, which lists each option with its default (B2).
  - Repeatable options (`--filter`, `--scenario`) use `action="append"` with `default=None`, are resolved to `all` after parsing, and state "(default: all)" in their help text.
  - Plain `ArgumentDefaultsHelpFormatter` would then print "(default: all) (default: None)". Use a small subclass whose `_get_help_string` appends `(default: %(default)s)` only when the default is not `None` and the help does not already mention a default.
  - Do not use `default=["all"]`: argparse appends user values to that list, so `--filter foo` would become `["all", "foo"]`.
  - `--record-seeds` takes a comma list or `all`, as a string with `default="0,1,2"`, so the formatter prints it as is. A missing display for `view`/`demo` (non-interactive backend) exits with a pointer to the README's WSLg section.

### 4.7 Recordings (F2)

- **Layout:**
  - `recordings/<scenario>/seed-<k>/shared.npz`: ground truth, controls, measurement values, landmark ids, the outlier flags and the measurement-step mask. It is shared by all filters, which saves space and makes C6 visible.
  - `<filter>.npz`: estimates, covariances, belief data, the effective-sample-size and distinct-count series, and the failure info.
  - `<filter>.timings.npz`: kept apart for F6.
  - `manifest.json`: lists what was recorded, for G1's error message.
- **Beliefs in float32, stored ragged** so particle or cell counts may change between steps (a user's adaptive particle filter). Each belief array is a concatenation, with an `offsets` int array and a per-step type code.
- **Non-recorded seeds** keep only the scoring arrays in memory (F2), not on disk beyond `results.csv`.

### 4.8 Replay GUI (G1 to G7)

- **Split the code** into a `ReplayController` (current step, play state, speed, `step_forward`, `step_back`, `goto(k)`), drawing code per panel, and the Matplotlib widgets that call the controller. Tests drive the controller under the Agg backend and read back each panel's and time plot's displayed step (I1 G3/G4). G7 is `render_step(recording, k, path)` on a figure created without pyplot.
- **Controls:** `Button` for play/pause, step forward and step back; `RadioButtons` for speed (0.25×, 0.5×, 1×, 2×, 4×); a `Slider` with `valstep=1` as the scrubber. All panels share the one controller, which is what keeps them on the same step (G3).
- **Playback timing (G3, G6):** a `fig.canvas.new_timer` fires every 40 ms (a 25 fps target). Every tick is a frame: it blits, even when the step has not changed. 1× means one step per frame (25 steps/s, faster than real time at `dt` = 0.1 s, which keeps 300-step runs short to watch). A fractional accumulator advances `speed` steps per tick: 0.25× changes the step every 4th frame, and 4× advances 4 steps per frame. `--show-fps` reports frames actually blitted per second (a moving average over the last second), so it measures drawing cost at every speed. G6 is checked at 1×, and `writing-a-filter.md`/README say so.
- **Layout:** one column per filter. The belief panel is on top, and below it a position-error plot and, for particle beliefs, the effective sample size and distinct particle count over time. Each time plot has an animated `axvline` at the current step, and `kidnapped-robot` adds a static line at step 150.
- **Drawing:**
  - grid: bin each cell's (x, y) centre to an integer index with `round((c − c_min) / cell_size)`, so float noise cannot split a column. Sum the probabilities per bin with `np.add.at`, then `pcolormesh` on the resulting 2D array. The harness normalizes grid probabilities to sum to 1, as it does particle weights (§4.6 covers zero and negative sums);
  - Gaussian: a 2σ `Ellipse` from the eigen-decomposition of the position block;
  - measurements: rays from the true pose for range-bearing, circles around landmarks for range-only, and points for (x, y). Outliers are drawn red and dashed.

## 5. Measured performance (sandbox: 8-core Xeon, NumPy 2.5.5, Matplotlib 3.11.2, Agg)

| What | Cost |
|---|---|
| f (arc step) + jitter, 1,000 particles × 3, vectorized | 132 µs/step |
| weighted mean, circular mean, covariance, effective sample size | 84 µs/step |
| distinct count, `np.unique(axis=0)` vs void view | 442 vs 130 µs/step |
| one 400-step, 1,000-particle recording, `savez_compressed` float32 | 4.4 MB (6.4 MB raw) |
| 4 panels × 2,000 particles, blitted, **scatter with per-point colormap** | **77 ms/frame (13 fps)** |
| same, single-colour scatter | 9.7 ms/frame |
| same, weights quantized into 8 `Line2D` alpha levels | 12.4 ms/frame |

Conclusions:

- **F8 fits serially.** The workload is 11 scenarios × 20 seeds × 300 steps = 66,000 steps per filter. The particle placeholder costs about 0.35 ms per step plus harness overhead, so about 25 to 35 s. The Gaussian placeholder and the simulation add a few seconds. That leaves about a 3× margin on 2 minutes, so no multiprocessing is needed. Add `--jobs` only if a measured run on the user's laptop exceeds the target.
- **F2's 500 MB holds.** The F8 configuration records 3 seeds × 11 scenarios × 2 filters. The 33 particle recordings come to about 33 × 3.5 MB (300 steps) ≈ 115 MB. The `linear-gaussian` one is about 4/3 of that, because its state is 4-D, which is negligible, and the Gaussian recordings are well under 1 MB each. Add a test that runs a reduced configuration and extrapolates, or simply asserts the folder size after a full F8 run in CI-like conditions.
- **G6 needs the right particle drawing.** A colormapped `scatter`, the obvious way to shade by weight, misses 20 fps by a wide margin. **Draw particles as K = 8 `Line2D` marker artists per panel, one per weight bin, with increasing alpha. The bins split `w / w.max()` into 8 fixed equal-width intervals, so shading follows the weight value, not its rank, and equal weights (as in `dead-reckoning-particles`) all fall into the top bin. Blit every animated artist.** Only the full-figure draw on resize and scrub release should be unblitted. That measured 12 ms/frame for the particles, leaving room for the ground-truth path, measurement rays, ellipse and time-plot cursors inside the 40 ms frame interval (25 fps target; G6 needs 50 ms or less). TkAgg's blit to a WSLg window adds an unknown cost (it cannot be measured here). That is why G6 is a user-verified criterion, and the `--show-fps` overlay is the instrument.

## 6. WSLg notes for the README (B3, Constraints)

- Requirements: Windows 11 or Windows 10 21H2+ with the Store version of WSL, and `wsl --update`. WSLg sets `DISPLAY` and `WAYLAND_DISPLAY` itself.
- `sudo apt install python3-venv python3-tk` for Ubuntu's own Python. Builds from uv, pyenv or python.org bundle Tk. Check with `python -c "import tkinter; tkinter.Tk()"` inside the venv. A venv made from an interpreter without Tk falls back to Agg, and `view` must say so (§4.6).
- Matplotlib picks TkAgg automatically when no Qt or GTK binding is installed. `MPLBACKEND=TkAgg` forces it.

## 7. Risks

| Risk | Mitigation |
|---|---|
| G6 frame rate under WSLg is unknown | Bucketed `Line2D` particles plus blitting (§5). The `--show-fps` overlay. User-verified by design. |
| Ground truth leaves the map under noise or a drawn start, breaking the `global-localization` initial belief and drawing | Circle controls well inside the bounds. Re-measured with drawn starts over 2,000 seeds: the largest coordinate reached was 12.9 m of 15 (§4.4). In-bounds test over seeds 0 to 99 per scenario. Documented as not guaranteed beyond that. |
| Reflection test for `bimodal` needs the simulator's noise draws | `_sim` returns its ε draws in a private field. Only tests read it. |
| Accidental public names in the package or on model objects | Underscore submodules (§3), property-based model classes, exact allowlist tests (I1). |
| Python 3.11 floor not exercised (sandbox has 3.14) | `uv python install 3.11` + `uv venv --seed -p 3.11` + `pip install -e .[dev]` + `pytest` once before the PR. |

## 8. What later stages need

- **Repo:** `Jcole33/bayes-filters` only. Cloning, pushing the work branch and opening the PR go through that repo grant, so `sites.txt` needs no `github.com` entry.
- **Sites:** `proposed/sites.txt`, each limited to the path glob pip or uv uses (PyPI for the packages and setuptools, `releases.astral.sh` for Python 3.11). pip's self-version check uses `/pypi/pip/json`, and failing it does no harm. The build writes `docs/stack-comparison.md` from §2 and does not re-fetch its sources.
