# Requirements: Bayesian filter testbed

## Goal

The user is learning Bayesian filters by writing them by hand: first a particle filter and a Kalman filter, later EKF, UKF, histogram filters and others. This story builds everything *around* those filters, but not the filters themselves:

- a simulator that produces ground truth and noisy measurements,
- a small documented plug-in interface for a filter,
- a fixed set of scenarios, each run on many seeds, chosen to show where the filter types differ,
- scoring of every filter on every scenario, with a comparison report,
- a GUI that replays the filters side by side, drawing each one's belief next to ground truth and measurements.

Two placeholder filters show that the whole pipeline works end to end. Terms are used as defined in [GLOSSARY.md](GLOSSARY.md); decisions are recorded in [docs/adr/](docs/adr/).

## Acceptance criteria

### A. Stack choice

- **A1.** `docs/stack-comparison.md` exists in the repository. It compares at least these options on four criteria: ease of setup, how well beliefs can be drawn, how easy it is to plug in a hand-written filter, and repeatability of seeded runs.
  - Options: ROS2 with turtlesim, ROS2 with RViz, Gazebo, plain Python with Matplotlib, plain Python with pygame, and at least one further option found by research.
  - It ends with a recommendation.
- **A2.** The comparison starts from plain Python with Matplotlib as the current recommendation ([ADR-0001](docs/adr/0001-plain-python-matplotlib-not-ros.md)). If it recommends anything else, the switch is put to the user and approved before building starts, and ADR-0001 is updated. The remaining criteria assume Python with Matplotlib; if the switch is approved, they apply to the equivalent pieces of the chosen stack.

### B. Packaging and CLI

- **B1.** The project is a Python package with a `pyproject.toml`, requiring Python 3.11 or later.
  - It lives in the repository `Jcole33/bayes-filters`.
  - `pip install -e .` in a fresh virtual environment installs it and all its dependencies.
- **B2.** Installing provides a `bayes-bench` command with these subcommands:
  - `list`: prints the scenarios and the filters it discovered.
  - `run`: runs the benchmark.
  - `view`: replays recorded runs.
  - `demo`: runs one scenario and opens the replay straight away.
  - `bayes-bench --help` lists the subcommands, and each subcommand's `--help` lists every option with its default.
- **B3.** `README.md` covers:
  - installation;
  - running the GUI from WSL2 on Windows (WSLg);
  - each subcommand, with an example;
  - where to put a new filter.

### C. Scenarios and simulation

- **C1.** Every scenario takes place in a 2D plane and has a unique name. It defines:
  - its state space;
  - its motion model;
  - its measurement model;
  - its initial belief;
  - its map bounds;
  - its landmark positions, if any;
  - its number of steps (between 200 and 500) and its step length `dt`;
  - its control sequence;
  - which steps have a measurement;
  - its recovery threshold (default 1 m).
- **C2.** These 11 scenarios exist, each documented in code and in `docs/writing-a-filter.md` with what it is designed to expose:
  1. `linear-gaussian`: a constant-velocity point. State is (x, y, vx, vy). Controls are accelerations (ax, ay). Measurements are (x, y) with Gaussian noise. Both models are linear and Gaussian.
  2. `nonlinear-landmarks`: a unicycle robot, state (x, y, θ). Controls are (v, ω). Measurements are range and bearing to several landmarks with known identities. Its noise parameters are stated in code. This is position tracking.
  3. `strongly-nonlinear`: like `nonlinear-landmarks`, but with a larger initial heading standard deviation and fewer landmarks in view per measurement step. Both differences are stated as parameters.
  4. `bimodal`: a unicycle robot driving with ω = 0 throughout, parallel to the line through two landmarks, with range-only measurements to those landmarks (known identities). The initial belief is uniform in position over a box that covers both possible starting sides, and Gaussian in heading around the true starting heading, with a documented standard deviation. The scenario is built so that the true trajectory and a second, distinct trajectory both fit every control and every noise-free measurement of the run.
  5. `kidnapped-robot`: like `nonlinear-landmarks`, but at a fixed step no earlier than step 100, the robot is kidnapped to a pose at least 5 m away. The filter is not told.
  6. `outliers`: like `nonlinear-landmarks`, but a fixed fraction (at least 3%) of individual landmark readings are replaced by draws whose standard deviation is at least 10× the nominal measurement noise. The measurement model given to the filter does not describe this. Each reading is recorded as outlier or not, for drawing only; filters never see the flag.
  7. `noise-sweep-low`, `noise-sweep-medium`, `noise-sweep-high`: `nonlinear-landmarks` with motion and measurement noise scaled by documented factors. The factors satisfy low < medium < high, and medium is 1, so `noise-sweep-medium` is identical to `nonlinear-landmarks` apart from its name.
  8. `stationary-pause`: a unicycle robot that drives, then gets zero controls for at least 50 consecutive steps while measurements keep arriving, then drives again.
  9. `global-localization`: the same map, landmarks, ground-truth starting pose, controls, measurement schedule and seeds as `nonlinear-landmarks`. Only the initial belief differs: it is uniform over the whole map in position and uniform over (−π, π] in heading. The landmark layout is asymmetric: no rotation or reflection other than the identity maps the landmark set onto itself, and an automated test checks this.
- **C3.** Every unicycle scenario uses the sampling velocity motion model of Thrun et al., *Probabilistic Robotics* (Table 5.3), with six α-parameters. Each step, the simulator:
  1. draws v̂ = v + ε₁, ω̂ = ω + ε₂ and γ̂ = ε₃. These are independent zero-mean Gaussians with variances α₁v² + α₂ω², α₃v² + α₄ω² and α₅v² + α₆ω²;
  2. moves the pose along the arc given by (v̂, ω̂) for `dt`, using the straight-line limit when |ω̂| is below a documented small threshold;
  3. adds γ̂·dt to the heading and wraps it into (−π, π].

  A zero control therefore gives zero motion noise. The filter is given the same α values (D2).
- **C4.** For `bimodal`, an automated test checks that the true trajectory and the alternative trajectory give identical noise-free measurements and fit the controls at every step.
- **C5.** At least one scenario has measurements on fewer steps than it has controls. Measurements may come at a lower rate or drop out; on steps without a measurement, the harness does not call `update`.
- **C6.** The ground truth and measurements of a run depend only on the scenario and the seed. All filters run on the same scenario and seed see identical ground truth, controls and measurements.

### D. Filter interface

- **D1.** The package exports a `Filter` base class. A filter is a subclass of it with:
  - a `name` attribute;
  - `__init__(self, model, initial_belief, rng)`;
  - `predict(self, u, dt)`;
  - `update(self, z)`;
  - `belief(self)`.

  Nothing else is required. The `rng` is a NumPy random generator derived from the run's seed, so the filter's own randomness is reproducible too.
- **D2.** The `model` passed to a filter provides only raw ingredients. Turning them into a predict step and a measurement update is the user's work.
  - **Motion:** the noise-free transition f(x, u, dt): the C3 step with ε₁ = ε₂ = ε₃ = 0, including the straight-line limit, so it is valid for ω = 0 and for v = ω = 0.
  - **Measurement:** the noise-free measurement function h(x, z), which gives the measurement that state x would produce without noise, for the same readings as the given measurement z. It covers exactly those readings, stacked in the same order, since which landmarks are in view changes from step to step. The simulator forms each reading as h plus independent zero-mean Gaussian noise on each component, with the standard deviations below, and wraps bearings into (−π, π] afterwards.
  - `measurement_residual(a, b)` returns a − b for two stacked measurement vectors of the same layout, with bearing components wrapped into (−π, π].
  - **State layout:** `position_indices`, the indices of x and y in the state vector, and `heading_index`, the index of θ, or `None` when the state has no heading (`linear-gaussian`). `state_residual(a, b)` returns a − b for two state vectors, with the heading component, if any, wrapped into (−π, π].
  - **Noise parameters**, as plain documented values:
    - in unicycle scenarios, the velocity-model α-parameters;
    - in every scenario, the measurement noise standard deviation of each reading component (e.g. range and bearing, or x and y);
    - in `linear-gaussian`, the acceleration-noise standard deviation.
  - **Map and timing:** the landmark positions with their identities, the map bounds, and the step length `dt`.
  - In `linear-gaussian`, f and h are linear. Each step the true acceleration is the control plus independent zero-mean Gaussian noise of the given standard deviation on each axis, held constant over the step. Then x ← x + vx·dt + ½·ax·dt², vx ← vx + ax·dt, and likewise for y. The measurement is (x, y) plus independent Gaussian noise. The model's docstring and H1 state these equations, and the user builds the F, B, Q, H and R matrices from them.
  - The model does not provide Jacobians, process-noise or measurement-noise covariance matrices, a sampler of noisy next states, a measurement likelihood, or finished linear matrices.
  - A measurement consists of one or more readings. In scenarios with landmarks, each reading states which landmark identity it belongs to; in landmark-free scenarios (`linear-gaussian`), the single reading carries no identity.
  - The model docstring and H1 state the full generative equations of every scenario, as in C3 and above, so the user can derive exactly what the simulator does.
  - The simulator's noise generation lives in a private simulator module. It is not exported from the package, not described in `docs/writing-a-filter.md` as something filters may use, and not imported by anything under `filters/`. It reads the same f, h and noise parameter values the model hands to the filter, so there is one source of truth ([ADR-0002](docs/adr/0002-scenarios-hand-models-to-filters.md)). The only exceptions are the deliberate, documented differences in `outliers` and `kidnapped-robot`.
  - The package also offers `check_jacobian(fn, x, jacobian, ...)`, a learning aid. It compares a Jacobian the user wrote against a finite-difference estimate and returns only the largest mismatch and the entry where it occurs, never the estimated Jacobian itself.
- **D3.** The `initial_belief` passed to a filter provides `mean`, `cov` and `sample(n, rng)`.
- **D4.** The package exports three belief classes that filters build and return from `belief()`:
  - `GaussianBelief(mean, cov)`;
  - `ParticleBelief(particles, weights)`;
  - `GridBelief(cells, cell_size, probabilities)`: `cells` holds the centres of the grid cells, one row per cell, spanning the full state (e.g. x, y, θ in unicycle scenarios); `cell_size` gives the width of a cell in each state dimension.

  Filters do not subclass them.
- **D5.** The harness derives each step's estimate and covariance from the returned belief:
  - from a `GaussianBelief`: its mean and covariance;
  - from a `ParticleBelief`: the weighted mean and covariance;
  - from a `GridBelief`: the probability-weighted mean and covariance.

  The heading component (`heading_index`, if any) is averaged as a circular mean, and the covariance is computed from `state_residual` around that mean. A filter may override an optional `estimate()` method to report its own estimate instead. It returns both a mean and a covariance over the full state, and the harness uses that pair for scoring.
- **D6.** Filters are discovered automatically. Any `Filter` subclass defined in a `.py` file under the filters folder, including subfolders, is discovered. The filters folder is `./filters` relative to the current working directory (the repository root in normal use), and `--filters-dir` overrides it. A discovered filter appears in `bayes-bench list` and can be selected by name with `--filter <name>`. `--filter all` selects all of them.
  - If two discovered filters share a name, the command stops with an error naming both files.
  - Adding a filter never requires editing any other file.

### E. Placeholder filters

- **E1.** `filters/placeholders/` in the repository contains two placeholder filters. They do no filter maths. Each one's file and class docstring say plainly that it is not a real filter, only a placeholder for the user's own filters.
  - `dead-reckoning-gaussian` moves its mean with the noise-free f. Its covariance starts as the initial belief's and grows by a fixed documented constant times the identity each step, with no Jacobian. It ignores measurements and returns a `GaussianBelief`.
  - `dead-reckoning-particles` moves each particle with the noise-free f, then adds a simple documented jitter (independent Gaussian noise of fixed standard deviation per state component). It does not use the velocity-model sampler. It uses 1,000 particles, set by a class attribute so a subclass can change it. It never weights or resamples them, keeps equal weights, ignores measurements and returns a `ParticleBelief`.
- **E2.** Both placeholders complete every scenario, on every seed, without becoming a failed run.
- **E3.** The repository contains no particle filter, Kalman filter, EKF, UKF or histogram filter, meaning no filter that uses measurements to revise its belief. The only exception is test code in the `tests/` folder whose only job is to exercise the harness: test doubles such as a filter that returns a fixed `GridBelief`, and Jacobians of toy functions that are not scenario models, used to test `check_jacobian`. The tests contain no Jacobian, sampler, noise covariance or likelihood for any scenario's models. None of that code is importable from the installed package.

### F. Running and scoring

- **F1.** `bayes-bench run` runs the selected filters (`--filter <name>|all`, repeatable, default all) on the selected scenarios (`--scenario <name>|all`, repeatable, default all) for N seeds each (`--seeds N`, default 20, meaning seeds 0 to N−1).
- **F2.** For each run, the harness keeps at every step: ground truth, control, measurement (if any), the estimate, the estimate's covariance and the time spent in `predict` plus `update`; these are used for scoring. A full recording, which adds the belief at every step, is saved for the seeds chosen with `--record-seeds` (default: seeds 0, 1 and 2; `all` records every seed). Recordings are stored compressed, with beliefs in 32-bit floats. With the F8 configuration and default options, one `results/<timestamp>/` folder takes no more than 500 MB. The harness never deletes old results folders.
- **F3.** These scores are computed for each run:
  - **Position RMSE**: the RMSE of the position error, which is the Euclidean distance between estimate and ground truth over the components at `position_indices`. Recovery time, convergence time and G4 use the same position error.
  - **Heading RMSE**, using the heading component of `state_residual`, in scenarios whose `heading_index` is not `None`; otherwise reported as `n/a`.
  - **Mean NEES**, over the full state, with the error estimate − ground truth computed by `state_residual`. Steps where the covariance is singular are left out and counted. If no step is usable, NEES is reported as `n/a`.
  - **Percentage of steps whose NEES lies below the 95% χ² quantile**, with degrees of freedom equal to the state dimension.
  - **Mean runtime per step.**
  - **Recovery time**, in `kidnapped-robot`: steps from the kidnapping step to the first step from which the position error stays under the recovery threshold for the rest of the run. If there is no such step, the run is reported as "did not recover".
  - **Convergence time**, in `global-localization`: steps from step 0 to the first step from which the position error stays under the recovery threshold for the rest of the run. If there is no such step, the run is reported as "did not converge".
  - **Mean and minimum effective sample size** over the run's steps, for particle beliefs. This shows weight degeneracy.
  - **Minimum distinct particle count** over the run's steps, for particle beliefs. This shows particle depletion, which effective sample size does not, since it resets to the full count after each resample.
  - The per-step series of effective sample size and distinct particle count are kept for the GUI (G4).
- **F4.** Each score is reported as mean ± standard deviation over seeds, per filter per scenario. Only seeds that have a value count towards it, and the count of excluded seeds is shown with the reason, e.g. "12.3 ± 4.1 (3/20 did not recover)".
- **F5.** If a filter raises an exception or returns a belief, or an estimate from an overridden `estimate()`, with non-finite values, that run is marked as a failed run, together with the step number and the error message. The remaining runs continue. The report shows "failed (step k)" for that run and leaves it out of the aggregates, stating how many seeds failed.
- **F6.** Running the same command twice with the same seeds gives identical recordings and scores, apart from runtimes.
- **F7.** `run` prints the report as a table in the terminal, followed by its total wall-clock time. It also writes `results/<timestamp>/report.md`, `results/<timestamp>/results.csv` and the recordings in the same folder. The repository's `.gitignore` excludes `results/`. In the report, `global-localization` appears next to `nonlinear-landmarks`.
- **F8.** With both placeholders, all 11 scenarios, 20 seeds and default options, `bayes-bench run` finishes in under 2 minutes, as printed by F7, on the user's laptop under WSL2. The user verifies this on their machine.

### G. Replay GUI

- **G1.** `bayes-bench view results/<timestamp> --scenario <name> --seed <k>` opens a Matplotlib window that replays the recorded runs of that scenario and seed ([ADR-0003](docs/adr/0003-record-then-replay-gui.md)). It shows one panel per filter, side by side; `--filter` restricts which filters are shown. If the folder, scenario or seed has no recording, the command exits non-zero with a message listing the scenarios and seeds that do have recordings.
- **G2.** Each panel shows:
  - the landmarks;
  - the ground-truth path so far and the current true pose;
  - the current measurement (range/bearing drawn as rays or circles, position measurements as points, readings flagged as outliers in a distinct style);
  - the filter's estimate;
  - its belief, drawn by type:
    - particles as points, shaded by weight;
    - a Gaussian as its mean plus a 2σ ellipse from the position block (`position_indices`) of its covariance;
    - a grid as a heatmap of its position marginal, summed over the other state dimensions.
- **G3.** The panels share one set of controls: play/pause, step forward, step back, playback speed (selectable from 0.25× to 4×), and a time scrubber that can jump to any step. All panels always show the same step.
- **G4.** The window also shows each filter's position error over time and, for particle beliefs, the effective sample size and the distinct particle count over time. Both have a marker at the current step, and the kidnapping step is marked in `kidnapped-robot`. A failed run's panel shows where and why it failed.
- **G5.** `bayes-bench demo --scenario <name>` runs every discovered filter on that scenario with one seed (`--seed`, default 0) and opens the replay.
- **G6.** `view --show-fps` displays the achieved frame rate. With 4 particle-belief panels of 2,000 particles each on `nonlinear-landmarks`, playback runs at 20 frames per second or more on the user's laptop under WSL2. The user verifies this on their machine. To set it up, the repository includes `benchmarks/fps-filters/`: a filters folder with four differently named 2,000-particle subclasses of `dead-reckoning-particles`, used with `--filters-dir benchmarks/fps-filters`.
- **G7.** The drawing code can render a given step to an image file without a display (Agg backend). The tests use this for all three belief types, the grid type through a test double.

### H. Documentation for learning

- **H1.** `docs/writing-a-filter.md` explains:
  - the filter interface and the call order (construct, then per step `predict`, then `update` only when there is a measurement, then `belief`);
  - the model object of each scenario, with its state and measurement layout, its noise parameters, and, for `linear-gaussian`, the model equations to build F, B, Q, H and R from;
  - what the user must derive for each filter type, pointing to sources (e.g. Thrun et al., *Probabilistic Robotics*, with chapter references) and the generative equations of D2 and C3. It gives no derived Jacobians, covariances, likelihoods or code. The items to derive are the motion and measurement Jacobians, the mapping from control-space noise (α-parameters) to state-space process noise, sampling from the velocity motion model, and the measurement likelihood;
  - how to use `check_jacobian` on the user's own Jacobians;
  - the three belief types;
  - angle wrapping, `state_residual` and `measurement_residual`, and the state layout (`position_indices`, `heading_index`);
  - how to add a filter to `filters/`;
  - how to run, read the report and view a run;
  - what each score means and what a good value looks like;
  - for each scenario, which filter types it is expected to favour or trouble, including particle depletion in `stationary-pause`; it also explains that effective sample size shows weight degeneracy and that the distinct particle count is what shows depletion.
- **H2.** Every public class and function in the filter interface, models and belief types has a docstring.

### I. Harness tests

- **I1.** `pytest` passes and covers at least the following:
  - same seed gives identical recordings (F6);
  - all filters see identical ground truth, controls and measurements (C6);
  - `bimodal` has an alternative trajectory (C4), and the `global-localization` landmark set has no symmetry (C2.9);
  - the simulator gives zero motion noise at zero control (C3);
  - `check_jacobian` reports a near-zero mismatch for a correct Jacobian and a large one for a wrong one, using Jacobians defined in `tests/` (D2);
  - the filter-facing interface matches exact allowlists: the names exported by the package are exactly those D1, D4 and D2's `check_jacobian` define, the public attributes of model objects are exactly those D2 lists, and those of initial-belief objects are exactly those D3 lists. No file under `filters/` imports the private simulator module (D2, E3);
  - `state_residual` and `measurement_residual` each wrap their angle components across ±π, and leave non-angle components unchanged;
  - circular mean of headings;
  - RMSE, NEES, χ² percentage, recovery time, convergence time, effective sample size and distinct particle count against hand-computed values;
  - outliers are flagged per reading in `outliers` (C2.6);
  - failed-run handling for an exception, a non-finite belief and a non-finite overridden estimate (F5);
  - aggregation with excluded seeds (F4), and a recording missing for `view` (G1);
  - a filter dropped into `filters/` is discovered, and duplicate names are rejected (D6);
  - headless rendering of each belief type (G7);
  - stepping, stepping back and scrubbing the replay programmatically, so that every panel and the time plots show the same requested step (G3, G4);
  - both placeholders complete every scenario (E2).

## Non-goals

- Implementing the user's filters: no particle filter, Kalman filter, EKF, UKF or histogram filter (see E3).
- Unknown data association: landmark identities are always known.
- SLAM, mapping, 3D, multiple robots.
- ROS2, Gazebo or real-robot integration, unless the stack comparison (A2) leads to an approved switch.
- Running filters live in the GUI, or changing filter or scenario parameters from the GUI.
- Exporting videos or animations.
- Speed work beyond the targets in F8 and G6.

## Constraints

- The filter interface stays minimal and documented, because the point of the project is learning (D1 to D5).
- Python 3.11+, installed with `pip install -e .`. Runtime dependencies are limited to NumPy, SciPy and Matplotlib unless the stack comparison justifies more. pytest is a development dependency.
- It must run, GUI included, under WSL2 on Windows using WSLg.
- Code goes in the user's GitHub repository `Jcole33/bayes-filters`, which is currently empty and was renamed from `bayes-filter`, so the old URL redirects. Pushing there needs the user's write access, which the sandbox's GitHub credentials provide. Work lands on a branch and goes into `main` through a pull request; nothing is force-pushed to `main`.
- Network access is needed only during development: web access for the A1 research, and PyPI for installing dependencies. Running the testbed needs no network access. Nothing needs secrets or paid services.
- Disk use is bounded as stated in F2. Old results folders accumulate until the user deletes them.
