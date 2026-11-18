# Bayes Filter Testbed

A testbed for learning Bayesian filters: it simulates a robot, feeds its noisy data to hand-written filters, scores them on standard scenarios, and shows them side by side.

## Language

### The simulation

**Scenario**:
A named setup that fixes the state space, motion model, measurement model, initial belief, landmarks, controls and measurement schedule; run with a seed, it produces ground truth and measurements.
_Avoid_: Test, case, environment

**Ground truth**:
The state the simulated robot really has at each step, known to the simulator and the scorer but never to a filter.
_Avoid_: True state, actual

**Control**:
The command applied to the robot over one step (e.g. velocity and turn rate), which the filter uses in its predict step.
_Avoid_: Action, input, odometry

**Measurement**:
The noisy sensor data the simulator produces about the robot's state at a step, made of one or more readings, which the filter uses in its update.
_Avoid_: Observation, sensor reading, z (in prose)

**Reading**:
One component of a measurement that refers to a single landmark, such as that landmark's range and bearing, or the whole measurement when no landmarks are involved.
_Avoid_: Sensor reading, observation

**Motion model**:
The scenario's description of how the state evolves given a control, including the parameters of its noise.
_Avoid_: Transition model, dynamics, process model

**Measurement model**:
The scenario's description of how a measurement relates to the state, including the parameters of its noise.
_Avoid_: Observation model, sensor model

**State residual**:
The difference between two states, with the heading difference wrapped into one turn around zero so that nearby headings on either side of ±π count as close.

**Measurement residual**:
The difference between two measurements of the same layout, with each bearing difference wrapped into one turn around zero.

**Nonlinear model**:
A motion or measurement model whose output is not a fixed matrix times the state (plus noise), e.g. one involving sin/cos of heading or a range computed with a square root.

### Filters

**Filter**:
An implementation of a Bayesian filter that plugs into the testbed, keeping a belief about the state and revising it from controls and, usually, measurements.
_Avoid_: Estimator, tracker, algorithm

**Initial belief**:
The belief about the starting state that a scenario gives each filter before the first step.
_Avoid_: Prior (in prose), initial state

**Belief**:
A filter's probability distribution over the state at a step, in whatever form the filter uses (Gaussian, weighted particles, grid).
_Avoid_: Posterior, distribution, state estimate

**Estimate**:
The single state taken as a filter's best guess at a step, derived from its belief unless the filter reports its own, and compared with ground truth for scoring.
_Avoid_: Prediction, output

**Placeholder filter**:
A deliberately simple filter shipped with the testbed only to prove the harness works, standing in for the user's own filters.
_Avoid_: Reference filter, baseline, sample filter

**Effective sample size**:
A measure, computed from the weights alone, of how many particles in a particle belief are really contributing; a fall toward 1 signals weight degeneracy, not particle depletion.
_Avoid_: ESS, N_eff (in prose)

**Weight degeneracy**:
The state of a particle belief in which nearly all the weight sits on a few particles, so most particles contribute nothing.
_Avoid_: Particle collapse

**Distinct particle count**:
The number of different states among the particles of a particle belief, counting exact copies once; a fall well below the particle count is a direct sign of particle depletion.

**Particle depletion**:
The loss of diversity in a particle belief when the same few particles are copied repeatedly, leaving it unable to represent the ground truth.
_Avoid_: Sample impoverishment, particle collapse

**Dead reckoning**:
Estimating the state from controls and the motion model alone, ignoring measurements.

### Running and scoring

**Seed**:
The number that fixes all randomness in one run of a scenario, so the same seed always reproduces the same ground truth, measurements and filter behaviour.

**Harness**:
The part of the testbed that drives filters through scenarios, scores them and keeps their recordings.
_Avoid_: Framework, runner

**Benchmark**:
One invocation that runs a chosen set of filters on a chosen set of scenarios over a set of seeds and produces a report.
_Avoid_: Test suite, batch

**Recording**:
The stored per-step ground truth, controls, measurements, beliefs and estimates of a run, which the replay GUI shows.
_Avoid_: Log, trace

**Run**:
One filter driven through one scenario with one seed, producing an estimate at every step, and a recording if its seed is recorded.
_Avoid_: Trial, episode, test

**Position tracking**:
A scenario type in which the initial belief is concentrated near the true starting state.

**Global localization**:
A scenario type in which the initial belief is uniform over the whole map and all headings, so the filter must find the robot from scratch.

**Landmark**:
A fixed point in the map with a known position and identity, which measurements refer to.
_Avoid_: Beacon, feature

**Map**:
The bounded rectangular region of the plane a scenario takes place in.
_Avoid_: World, arena

**Outlier**:
A reading deliberately generated from a distribution far wider than the measurement model describes.

**Kidnapping**:
Moving the robot instantly to a new pose mid-run without telling the filter.
_Avoid_: Teleport, reset

**Recovery time**:
The number of steps from a kidnapping to the first step from which a filter's position error stays under the recovery threshold for the rest of the run.

**Convergence time**:
The number of steps from the start of a run to the first step from which a filter's position error stays under the recovery threshold for the rest of the run.

**Position error**:
The straight-line distance in the plane between a filter's estimated position and the ground-truth position at a step.

**Recovery threshold**:
The position error below which a filter counts as having found the robot, used by recovery time and convergence time.

**Failed run**:
A run stopped because the filter raised an error or reported a non-finite belief or estimate, recorded with the step at which it happened.

**Report**:
The comparison of every filter on every scenario produced by one benchmark, as a table of scores aggregated over seeds.
_Avoid_: Results table, leaderboard
