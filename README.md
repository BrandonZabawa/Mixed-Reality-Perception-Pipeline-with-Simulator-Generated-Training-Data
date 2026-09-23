# Project Super-AV-Racer, part 1: Human-versus-Autonomous Racing in Simulation

Ten deliverables per tier. Each carries an acceptance test, the artifact it produces, and the job-relevant skills it builds. Solo.

**Checkpoint 1: December 1, 2026.** Minimum viable product plus the core Advanced items, with a demonstration video. That is 10 working weeks from September 21.

**Full project: at least a year**, ending with real 1/10-scale cars on a real track (see "Beyond December 1").

**Stack:** Ubuntu 26.04, ROS 2 Lyrical Luth, Gazebo Jetty (the recommended pairing for Lyrical), C++ and Python, PyTorch, Ultralytics YOLO26.

---

## The pitch

A human drives one car around a Gazebo race track with a PlayStation 5 DualSense controller, through a driver you write yourself. An autonomous car races them under fixed race conditions. The autonomous car knows only what its own simulated cameras show it: a You Only Look Once (YOLO) detector trained on automatically labeled simulator images finds the cones and the opponent, a Kalman filter tracks the opponent, and a model trained on the human's past races predicts where they will go next.

Every race is logged. Between race blocks, the prediction model, a learned copy of the human's driving, and the car's racing policy retrain on the new data. A new model replaces the current one only if it wins more in simulation and passes a regression suite built from old races. The target is a car that wins at least 80 percent of races at matched pace, so the only way for the human to win is to invent something new, and whatever they invent becomes the next round of training data.

The same code, data formats, and race rules later move to real 1/10-scale cars and grow to six autonomous cars racing each other and human drivers.

**Why pure simulation first.** Everything that makes the real version hard, except the physical car, is built and measured here: camera perception, opponent prediction, race rules, logging, the learning loop, the human interface, and safety behavior. What changes on real hardware (vehicle dynamics, sensor noise, actuation) sits behind interfaces designed to be swapped, listed under "Designed-in sim-to-real seams."

---

## How the pieces fit

Read in one line: **the human races, every race becomes data, the data teaches the car what the human will do next, and the car races again slightly better.** Every deliverable serves one of five stages.

| Stage | What happens | Deliverables |
|---|---|---|
| 1. Build | The track, two identical cars, the human's controller, and a referee that scores every race. | M1, M2, M3, M4 |
| 2. See | Gazebo labels its own images, YOLO learns from them, detections become positions in meters, and filters turn positions into tracked state. A public real-image dataset measures how much of this survives real pixels. This stage is the computer vision contribution. | M6, M7, M8, A1, A2, A3, A4 |
| 3. Race | A classical racer sets the baseline, a prediction-aware planner adds racecraft, and shared control lets the human take over the autonomous car. | M9, A6, A9 |
| 4. Learn | Races become datasets; a prediction model, a clone of the human, and a reinforcement learning policy train on them; a gated loop repeats this every race block. | M5, A5, A7, A8, A10 |
| 5. Prove | A fixed race protocol, win rates with confidence intervals, ablations, held-out drivers, demonstration videos, and a design journal. | M10, S1, S2 |

The loop that runs from the Advanced tier onward:

```text
10-race block -> log (M5) -> tag hard cases -> retrain (A5, A7, A10) -> gate (A8) -> next block
```

> [!IMPORTANT]
> **Scope decisions**
>
> **Pure simulation until December 1.** No car hardware is bought for this checkpoint.
>
> **The autonomous car races on perception only.** In scored races it never reads the opponent's true pose or the human's controller inputs. Ground truth is for labeling, training, and scoring, never for driving. This keeps the computer vision load-bearing and the win rate honest. The car's own pose comes from ground truth only until A2 replaces it.
>
> **The headline win rate is pace-matched.** One speed-scale parameter sets the car's solo lap time equal to the human's warm-up median. Without it, 80 percent measures top speed, not prediction or racecraft. Open-pace results are reported too, labeled as such.
>
> **Two simulators, one track file.** Gazebo runs human races, perception, and scoring in real time. A lightweight vectorized bicycle-model simulator runs reinforcement learning far faster than real time. Both load the same track file, and the gap between them is measured as a rehearsal for sim-to-real.
>
> **No end-to-end pixels-to-steering learning before the checkpoint.** Learning decides when and where to overtake or defend; a classical controller underneath keeps the car on the track. S10 revisits end-to-end driving.
>
> **Built at 1/10 scale.** Car size, speeds, camera heights, and track widths match RoboRacer (formerly F1TENTH) cars, so the port to hardware is a parameter file, not a rewrite.
>
> **Camera calibration is given by the simulation.** Intrinsics come from the car's model definition. One rendered-checkerboard exercise in M2 keeps the skill verified against a known answer; real calibration returns with the real camera.

---

## Defined race conditions

The 80 percent claim is only as good as the rules it was measured under. The race controller (M4) enforces these rules, and the scenario file stores them with every race.

| Rule | Setting | Why |
|---|---|---|
| Cars | Identical: one vehicle parameter file (mass, wheelbase, power limit, steering limit, tire friction) loaded by both | A win comes from driving, not from a better car |
| Tracks | Two training tracks and one held-out track, each versioned by file hash | The held-out track catches a model that memorized corners |
| Format | 5 laps, standing start, about 1 to 2 minutes at 1/10 scale | Long enough to attempt passes, short enough for many races per session |
| Grid | Starting position alternates every race | Starting ahead is worth a lot; alternating cancels it out |
| Track limits | All four wheels outside the boundary: time penalty | Cutting corners is not racecraft |
| Contact | At-fault contact (hitting a car from behind, turning into a car alongside): time penalty. A win with at-fault contact by the autonomous car counts as a loss | Winning by ramming is not winning |
| Autonomous car inputs | Its own front and rear cameras, inertial measurement unit, and wheel odometry | The perception-only rule |
| Autonomous car timing | Tactical decisions capped at 10 hertz; camera-to-command latency measured and reported next to the human's input-to-screen latency | No superhuman reaction time |
| Human setup | Same controller, chase-camera view, and head-up display every session; 5 warm-up laps; at most 20 races per session | Removes setup and fatigue as variables |
| Model freeze | Models frozen for a block of 10 races; training happens only between blocks | Every result maps to exactly one model version |
| Pace matching | The car's speed scale is set so its solo lap equals the median of the human's warm-up laps | Separates racecraft from raw speed |
| Scoring | Win rate with a 95 percent Wilson score interval, per driver and per model version | Small samples need honest error bars |
| Targets | Target met: observed win rate of at least 80 percent over at least 50 pace-matched races. Target proven: the interval's lower bound is at least 80 percent | See the table below |

**How many races the claim needs** (95 percent Wilson score interval):

| Races | Wins | Observed | 95 percent interval | Proves at least 80 percent? |
|---|---|---|---|---|
| 20 | 16 | 80% | 58% to 92% | No |
| 50 | 40 | 80% | 67% to 89% | No |
| 50 | 46 | 92% | 81% to 97% | Yes |
| 100 | 80 | 80% | 71% to 87% | No |
| 100 | 88 | 88% | 80% to 93% | Yes, barely |
| 200 | 172 | 86% | 81% to 90% | Yes |

Hitting 80 percent is the target; proving it takes about 46 wins in 50 races or 88 in 100. Report the interval either way. This is the function M10 and every later scoring script should share:

```python
from math import sqrt


def wilson_interval(wins: int, races: int, z: float = 1.96) -> tuple[float, float]:
    """95 percent Wilson score interval for a win rate."""
    if races == 0:
        return (0.0, 1.0)
    p = wins / races
    denom = 1 + z * z / races
    center = (p + z * z / (2 * races)) / denom
    half = z * sqrt(p * (1 - p) / races + z * z / (4 * races * races)) / denom
    return (center - half, center + half)


if __name__ == "__main__":
    for wins, races in [(16, 20), (40, 50), (46, 50), (88, 100)]:
        low, high = wilson_interval(wins, races)
        print(f"{wins}/{races}: {low:.1%} to {high:.1%}")
```

---

## How to read the deliverable tables

Each row has five columns. **Deliverable** is what to build. **Acceptance test** is how you know it is done. **Artifact produced** is what exists afterward. **Skills built** is the column to read when deciding what to study next: it lists the skills, phrased the way a robotics job posting phrases them, that finishing the row earns. If a skill you need is not in any row, see the gap section below for how to add it.

---

## Minimum viable product, weeks 1 to 5 (September 21 to October 25)

Required. Target 6 out of 10 on its own. A human and an autonomous car race under the defined conditions, the car perceives through a detector trained on free labels, and every race is logged.

| #   | Deliverable                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Acceptance test                                                                                                                                                                                                    | Artifact produced                                                                                                                            | Skills built                                                                                                                                                                                                                                              |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| M1  | **Track generator and race world.** A track is data: a centerline with left and right widths in a comma-separated values file (the Technical University of Munich racetrack-database format). A generator places Formula Student-style cones (blue left, yellow right, orange at start and finish) along the boundaries and writes a Gazebo world in Simulation Description Format (SDF), plus an occupancy image for the fast simulator. Three tracks: two for training, one held out.                    | One launch command brings up any track; regenerating from the same file gives an identical world; every cone within 1 centimeter of the track definition                                                           | Versioned track files that Gazebo, the fast simulator, and later the real track all load                                                     | Gazebo world authoring (SDF), procedural world generation, spline parameterization, programmatic entity spawning, ROS 2 launch files, configuration as data                                                                                               |
| M2  | **Race car model, two identical instances.** A 1/10-scale Ackermann-steered car in Unified Robot Description Format (URDF) with xacro, driven by Gazebo's Ackermann steering system; front and rear color cameras, inertial measurement unit, wheel odometry. Spawned as `/ego` and `/human` from one vehicle parameter file, with a ros_gz_bridge configuration. Calibrate the simulated camera from a rendered checkerboard.                                                                             | Both cars spawn from one launch with separate transform trees; steering and speed step responses plotted; checkerboard calibration recovers the intrinsics set in the car description within 1 percent             | A reusable car description, and the vehicle parameter file that system identification later overwrites with real-car values                  | Robot description (URDF, xacro), Ackermann steering geometry, Gazebo sensor configuration, ros_gz_bridge, namespaced multi-robot tf2 trees, camera intrinsic calibration (OpenCV) checked against known truth, step-response characterization             |
| M3  | **DualSense driver, USB path.** Your own userspace driver in C++ with hidapi: parse USB input report `0x01` (sticks, triggers, buttons, touchpad, inertial measurement unit) in a fixed 250 hertz loop; publish `sensor_msgs/Joy` plus a custom message; a udev rule for non-root access. A teleoperation node maps triggers and stick to speed and steering for `/human`; a controller timeout stops the car.                                                                                             | Parser unit tests pass on recorded report bytes; loop jitter measured (99th percentile reported); a human completes 5 clean laps; unplugging the controller stops the car within 200 milliseconds                  | The human's input device as a standalone driver package, and the first measured number on the human side of the race                         | USB human interface device protocol, hidapi and hidraw, udev rules, byte-level report parsing in C++, fixed-rate loop and jitter measurement, custom ROS 2 messages, failsafe timeout, driver unit testing                                                |
| M4  | **Race control and defined race conditions.** A race controller node reads a scenario file (track, laps, grid, rules, model versions, random seed) and runs start lights, lap timing, positions, track-limit and contact detection, at-fault rules (ambiguous contacts flagged for review), penalties, and one result record per race. Start and reset are a ROS 2 service and action. A head-up display shows the human position, lap, gap, and penalties.                                                | 10 scripted races produce correct results checked against ground truth; any race reproduces from its scenario file and seed                                                                                        | The referee: every win rate in the project comes from this node, never from hand counting                                                    | ROS 2 node design, finite state machines, ROS 2 services and actions, contact detection, YAML scenario configuration, reproducibility with seeds, head-up display (OpenCV overlay or Foxglove)                                                            |
| M5  | **Race logging and dataset pipeline.** Every race is recorded with rosbag2 (MCAP storage) plus metadata (scenario, model versions, driver, result). A controller button tags hard-case moments. A converter turns bags into per-race episode files (Parquet) with synchronized ego state, opponent state, commands, and camera frames at a fixed rate. Datasets are versioned with Data Version Control.                                                                                                   | Conversion is deterministic and covered by tests; 20 races logged; a dataset card (races, laps, drivers, tracks, hard-case tags) generated automatically                                                           | The first stage of the data flywheel: a growing, versioned dataset of human racing                                                           | rosbag2 and MCAP, message time synchronization, data engineering (Parquet), dataset versioning (Data Version Control), dataset cards, hard-case tagging                                                                                                   |
| M6  | **Automatic labels and domain randomization.** Gazebo's bounding box camera and segmentation camera emit labels for every cone and car that carries the Label system; your own projection of three-dimensional model poses into the image cross-checks them; output in YOLO format. A randomization file varies lighting, sun angle, ground textures, cone placement noise, car colors, camera pose jitter, and motion blur.                                                                               | Your projected boxes and Gazebo's boxes agree at intersection over union above 0.85; 3,000 or more labeled frames generated unattended; any dataset regenerates from its configuration and seed                    | The labeling tool that replaces about forty hours of manual annotation, plus a versioned synthetic dataset                                   | Projective geometry (three-dimensional pose to two-dimensional box), simulation ground-truth extraction, YOLO annotation format, segmentation masks, domain randomization, synthetic data generation, unattended dataset pipelines                        |
| M7  | **YOLO detector: fine-tune and deploy.** Fine-tune YOLO26 nano on the synthetic data, with class names matching the Formula Student Objects in Context (FSOCO) real-image dataset (`blue_cone`, `yellow_cone`, `orange_cone`, `large_orange_cone`) plus `race_car`, using default hyperparameters. Export to Open Neural Network Exchange (ONNX) format; a detector node runs both cameras through ONNX Runtime and publishes `vision_msgs/Detection2DArray`.                                              | Mean average precision at 0.5 and at 0.5 to 0.95 intersection over union reported on held-out synthetic frames; frames per second and latency logged with Gazebo running; at least 15 frames per second per camera | Trained weights, the baseline accuracy number every later result is measured against, and a real-time detector node                          | Transfer learning and fine-tuning, PyTorch training workflow, mean average precision evaluation, train and validation split hygiene, experiment tracking, ONNX export and ONNX Runtime, ROS 2 perception node design, inference latency profiling         |
| M8  | **Detections to meters.** Project each detection's bottom-center pixel onto the ground plane using the known camera extrinsics (ground-plane homography); chain camera, base, odometry, and map frames with tf2; publish cone positions and opponent measurements with a covariance that grows with range.                                                                                                                                                                                                 | Position error against ground truth plotted against range; median error under 5 percent of range out to 5 meters, for cones and for the opponent                                                                   | The pixels-to-meters layer and an error-versus-range plot, the headline perception number of the minimum viable product                      | Ground-plane homography, camera extrinsics, tf2 frame chaining, metric localization from one camera, measurement covariance modeling, error-versus-range analysis                                                                                         |
| M9  | **Classical autonomous racer, the baseline.** Offline: a minimum-curvature raceline (the Technical University of Munich global_racetrajectory_optimization tool, or your own quadratic program) and a velocity profile from friction limits. Online: pure pursuit steering, proportional-integral-derivative speed control, and a follow mode that holds a gap behind a slower opponent using M8's measurements. One speed-scale parameter for pace matching.                                              | 10 consecutive clean solo laps; lap time within 5 percent of the velocity profile's predicted lap time; no contact in 10 follow tests; the pace scale hits a target lap time within 1 percent                      | The baseline racer, a predicted best lap time you can quote before driving, and the pace-matching knob that keeps the headline metric honest | Raceline optimization (minimum curvature, quadratic programming), velocity profile generation (friction circle), pure pursuit, proportional-integral-derivative tuning, track-relative (Frenet) coordinates, adaptive cruise control, lap time prediction |
| M10 | **Race Day One: protocol, demonstration, repository.** 20 races, human versus baseline, under the protocol (10 open pace, 10 pace-matched), scored by M4. A 60 to 120 second demonstration video. A README with an architecture diagram and an interface table (every topic, service, and message with its rate and quality of service profile). A Dockerfile, and GitHub Actions running pytest on the report parser, projection, raceline, and scoring. A design journal entry per deliverable M1 to M9. | Continuous integration green; results table with Wilson intervals; journal entries with at least one measured number each; release tagged                                                                          | The first demonstration, the baseline win rate every learned model must beat, and the written record that interview answers come from        | Technical communication, demonstration video production, ROS 2 package structure (colcon, CMake, package.xml), Docker, pytest, GitHub Actions continuous integration, interface contract documentation, ROS 2 quality of service, small-sample statistics |

> **Resume line after the minimum viable product**
>
> "Built a ROS 2 and Gazebo racing simulator in which a custom PlayStation 5 DualSense driver (USB human interface device, 250 hertz) lets a human race an autonomous car that sees only through a YOLO detector trained on automatically labeled simulator images; opponent localized within X percent of range; baseline win rate of Y percent over Z races under a fixed race protocol."

---

## Advanced, weeks 6 to 9 (October 26 to November 22)

Target 8 out of 10. The car becomes perception-only end to end and starts learning from the human. This tier is sized for more than four weeks on purpose: whatever is unfinished on November 22 becomes Season 1 after the checkpoint.

| # | Deliverable | Acceptance test | Artifact produced | Skills built |
|---|---|---|---|---|
| A1 | **Opponent tracking.** An extended Kalman filter in C++ (Eigen) with a constant turn rate and velocity model per opponent, fed by M8 measurements from both cameras; gating by Mahalanobis distance, track identifiers, and birth and death rules. Output in track coordinates (progress, lateral offset, speed) for prediction and planning. | Filtered position error at least 30 percent below raw detections; a track survives the side blind spot during an overtake (about one second with no detections); no identity swaps over a two-minute recording with two opponents (the human plus a ghost car replaying a logged race) | Stable opponent state from noisy detections, and a measured error reduction you can quote | Extended Kalman filtering (constant turn rate and velocity), multi-object tracking, data association and gating, track lifecycle management, occlusion handling, modern C++ with Eigen |
| A2 | **Ego localization on the cone map.** Replace the ground-truth ego pose: an extended Kalman filter predicts with wheel odometry and the inertial measurement unit (with realistic simulated noise) and corrects with YOLO cone detections matched to M1's known cone map, associated by color plus gating. Compare against robot_localization's odometry-and-inertial fusion as the no-landmark baseline. | Position and heading error over 10 laps, with and without landmark updates; the M9 racer completes 10 clean laps on the estimate, with lap time within 3 percent of the ground-truth baseline | A perception-driven localization layer; the car is now perception-only end to end | Landmark-based extended Kalman filter localization, sensor fusion (odometry, inertial measurement unit, camera), landmark data association, sensor noise modeling, robot_localization |
| A3 | **Sim-to-real gap on real images.** Evaluate the synthetic-only detector on real pixels: a 500-image test split from FSOCO (real cone photos from dozens of Formula Student teams) with matching class names, plus 100 photos you take and label of a small radio-controlled or toy car for the `race_car` class. Report synthetic accuracy, real accuracy, and the gap, broken down by class, distance, and lighting. | Table in README; a failure gallery of the 20 worst false negatives and false positives with categorized causes | The single quantitative computer vision claim of the project, measured on real pixels without owning a car | Evaluation methodology, sim-to-real gap quantification, dataset curation and class mapping, manual annotation with a labeling tool, error analysis, mean average precision on real data, results reporting |
| A4 | **Domain randomization ablation.** Toggle randomization groups one at a time (lighting, textures, camera jitter and blur, distractor objects); regenerate, retrain with fixed hyperparameters and budget, and remeasure A3's real-image gap. Train the full configuration with 3 seeds to measure run-to-run noise. | Four or more rows and a conclusions paragraph that claims only differences larger than the seed-to-seed spread | A results table showing which randomization parameters actually mattered, which is publishable material | Ablation study design, controlled experiments, variance across seeds, automated retraining pipeline, comparing results across runs, technical writing of conclusions |
| A5 | **Opponent prediction.** From the last 2 seconds of tracked opponent state plus upcoming track curvature, predict the human's next 2 seconds at 10 hertz and their intent (follow, overtake left, overtake right, defend). Intents are labeled automatically from logged geometry with documented rules. Baselines: constant velocity, the A1 filter's own prediction, and "follows the raceline." Learned model: a gated recurrent unit encoder-decoder producing 3 candidate trajectories with probabilities. | Average and final displacement error, and minimum average displacement error over 3 candidates, versus every baseline on held-out races (split by race, never by window); intent F1 score; runs at 10 hertz or faster in the stack | A model of the human's future, with baselines proving it adds something | Trajectory prediction, sequence models (gated recurrent unit or long short-term memory, optional Transformer), multimodal prediction, displacement error metrics, intent classification, weak supervision with rule-based labels, leakage control in time-series splits |
| A6 | **Prediction-aware racecraft planner.** A Frenet-frame sampling planner in C++: candidate trajectories (lateral offsets times speed profiles over 2 seconds) scored for progress, collision risk against the predicted opponent trajectories, and track limits; the tactical mode (follow, overtake left or right, defend) falls out of the scores. Then a pace-matched race block with prediction on versus off (off means constant-velocity prediction). | At least 20 pace-matched races per condition; win rate difference reported with intervals; zero at-fault contacts by the car across the block | The first evidence that prediction, not speed, wins races | Motion planning (Frenet-frame sampling), cost function design, collision checking against predicted trajectories, behavior planning, controlled comparison with statistics, real-time C++ |
| A7 | **Human behavior model, "the ghost."** A driving policy cloned from the human's logged races: inputs (speed, track-relative pose, upcoming curvature, opponent relative state), outputs (steering, throttle, brake). It runs as an opponent in both simulators. Collect recovery data by injecting small steering disturbances while you drive, and add correction segments from A9 takeovers (dataset aggregation). | The clone completes laps on the held-out track; its lap-time distribution is reported next to the human's; a classifier trained to tell human from clone on 5-second snippets is reported (accuracy near 50 percent means human-like) | A sparring partner that drives like the human and is available around the clock for training and testing | Behavior cloning, imitation learning, covariate shift and compounding error, dataset aggregation (DAgger, human-gated DAgger), disturbance injection, discriminator-based fidelity evaluation, PyTorch |
| A8 | **Learning loop: rounds, gate, regression suite.** Race block, log, retrain prediction and clone, gate, next block. A challenger replaces the champion only if it wins more in 50 overnight Gazebo races against the clone and passes the regression suite: scenario snapshots cut from tagged hard cases, where both cars start from the logged state and the human car replays the logged inputs for 5 to 10 seconds. A model registry records the data, code, and configuration behind every model. | At least 3 rounds; a learning curve of win rate per round with intervals; at least one exploit you found as the human tagged, trained against, and shown fixed; old scenarios still pass (no forgetting); exploit half-life reported (rounds until a human strategy stops working) | The loop the pitch describes, with evidence that it improves per round and does not forget | Continual learning, catastrophic forgetting checks with replay of old data, champion-challenger promotion, model registry and experiment tracking (MLflow or Weights & Biases), scenario-based regression testing, machine learning operations |
| A9 | **Full DualSense driver and shared control.** Bluetooth path (input report `0x31` with a cyclic redundancy check, and the feature report read that switches the controller into full mode); output reports for rumble on contact and off-track, adaptive trigger resistance on the brake, and a light bar showing race position. A command multiplexer with priority and timeout, and a takeover mode where the human drives the autonomous car mid-race, with takeover segments logged as corrective labels for A7. | Bluetooth passes the USB parser tests; checksums verified; switch between human and autonomous control mid-race without restart; a disconnect stops the car within 200 milliseconds; at least 30 takeover segments logged | Safe shared control, a second source of training data, and a driver that reads as genuine embedded work | Bluetooth human interface device protocol, cyclic redundancy check (CRC32), output report construction, haptic feedback design, command arbitration, failsafe design, shared autonomy, intervention-based learning |
| A10 | **Reinforcement learning racecraft.** A vectorized Gymnasium environment with a dynamic bicycle model on the same track files (or f1tenth_gym); the A7 clone as opponent; perception noise fitted to A1 and A3 measurements; the A6 planner compiled in through pybind11 so both simulators run identical planning code. Proximal Policy Optimization (Stable-Baselines3) learns a residual on the planner: target lateral offset and speed scale every 0.1 seconds. Reward: progress relative to the opponent and overtakes, minus penalties for at-fault contact, track limits, and jerky commands. Curriculum from a slower clone up to pace-matched. | Beats the A6 planner head to head against the clone in the fast simulator with non-overlapping intervals; Gazebo win rate reported next to the fast-simulator win rate (the sim-to-sim gap); at-fault contact rate no higher than the planner's | A learned racecraft policy and a measured sim-to-sim gap, the rehearsal for sim-to-real | Reinforcement learning (Proximal Policy Optimization, Stable-Baselines3, Gymnasium), reward design, curriculum learning, residual policy learning, vectorized simulation, pybind11, sim-to-sim transfer evaluation |

> **Resume line after Advanced**
>
> "...the car localizes itself on the cone map and tracks the human with extended Kalman filters, predicts the human's next two seconds (X percent lower displacement error than constant velocity), and plans overtakes against those predictions; at matched pace, prediction raised its win rate from A to B percent over N races, and three rounds of retraining on logged races took it to C percent; synthetic-only detector measured on real Formula Student images with a sim-to-real gap of Z mean average precision points, reduced by W through randomization ablation."

---

## Stretch, after December 1 or if ahead

Target 9 out of 10. Most of this tier is Season 1 and Phases 2 and 3 of the roadmap below.

| # | Deliverable | Notes | Artifact produced | Skills built |
|---|---|---|---|---|
| S1 | **Held-out drivers.** 3 to 5 drivers from your robotics clubs race the car under the protocol; none of them contributed training data. | A model that only beats its author is overfit to its author. If results go into a paper, check with the university's institutional review board first. | Per-driver win rates with intervals, and the gap between the author and everyone else | Held-out human evaluation, generalization measurement, experiment protocol design |
| S2 | **Write-up.** A short paper or article: how much prediction is worth at matched pace, and which randomization parameters mattered on real cone images. | Public artifact | A published article or workshop-style paper, the highest-leverage artifact per hour spent | Technical writing, results communication, public portfolio artifact |
| S3 | **Six cars in simulation.** Race control, launch, and tracking scale to seven cars (six autonomous plus the human) from the scenario file, with per-car namespaces and transform trees. Cars whose cameras are not rendered use a perception noise model fitted to A1 and A3 measurements. | The noise-model trick is how large fleets are simulated cheaply | A multi-car race stack and a measured real-time factor with seven cars | Multi-robot ROS 2 (namespaces, tf2 prefixes), scalable simulation, sensor error modeling, multi-object tracking at scale, middleware discovery tuning |
| S4 | **Self-play league.** A population of policies (A10 variants and clones of several humans) matched against each other; ratings with Elo or TrueSkill; training samples opponents from the whole league, not just the latest. | How six cars learn to race each other without overfitting to one rival | A league ladder and rating history | Self-play, league training, multi-agent reinforcement learning, skill rating systems, population-based evaluation |
| S5 | **ros2_control migration.** Swap Gazebo's Ackermann system for gz_ros2_control with the Ackermann steering controller; the car description gains ros2_control tags; nothing above the controller changes. | The sim-to-real seam: the real car then needs only a new hardware interface | A car that runs through ros2_control in simulation, ready for a hardware interface | ros2_control, hardware interface abstraction, controller configuration, gz_ros2_control |
| S6 | **Jetson hardware-in-the-loop.** The detector, filters, predictor, and planner run on the NVIDIA Jetson; the laptop runs only Gazebo; one ROS 2 graph over the network. TensorRT engines in 32-bit floating point, 16-bit floating point, and 8-bit integer compared on accuracy, latency, and size. | The car's future computer drives the simulated car. Build the 8-bit calibration set from frames of the domain you deploy in. | A two-machine deployment and a quantization benchmark | NVIDIA Jetson deployment, TensorRT, 16-bit and 8-bit quantization with calibration sets, multi-machine ROS 2, network latency budgeting, hardware-in-the-loop testing |
| S7 | **System identification and dynamics randomization.** Treat the Gazebo car as if it were real: identify its steering map, acceleration limits, and friction from logged data, then randomize those parameters in the fast simulator and measure policy robustness. | Exactly the procedure the first real car will need | An identification script that writes the vehicle parameter file, and a robustness table | System identification, parameter estimation, dynamics randomization, robustness evaluation |
| S8 | **Overhead race-control camera.** A camera above the track; YOLO26 oriented bounding boxes give every car's position and heading; lap timing and track limits come from vision instead of ground truth. | Oriented boxes give heading only up to 180 degrees; the tracker's velocity resolves it. Becomes the real track's positioning and timing system. | A vision-based positioning and timing system reusable on the real track | Oriented bounding box detection, overhead camera calibration, multi-object tracking, vision-based timing |
| S9 | **Model predictive control.** Replace pure pursuit with model predictive control (or model predictive contouring control) on the bicycle model; compare lap time, tracking error, and compute time. | The standard controller in autonomous racing research | A controller comparison table | Model predictive control, optimization solvers, real-time optimization, controller benchmarking |
| S10 | **End-to-end student.** A convolutional network maps both camera images plus speed to steering and speed, trained to imitate the full modular stack (the teacher) with dataset aggregation; compared against the modular stack on lap time, win rate, and failure modes. | The "learning by cheating" teacher-student setup; answers the end-to-end versus modular question with your own numbers | An end-to-end policy and a modular-versus-end-to-end comparison | End-to-end driving, policy distillation, teacher-student training, imitation learning from an algorithmic expert |

---

## Skills this project puts on your resume

Consolidated from the Skills built column, grouped the way a resume skills section is grouped. The deliverable identifiers tell you which row earns each line.

### Perception and computer vision

| Skill | Earned by |
|---|---|
| Synthetic data generation and automatic labeling from simulation | M6 |
| Domain randomization and ablation | M6, A4 |
| Object detection fine-tuning and real-time deployment (YOLO26) | M7 |
| Sim-to-real evaluation on real images | A3 |
| Ground-plane homography and metric localization from one camera | M8 |
| Camera intrinsic calibration checked against known truth | M2 |
| Multi-object tracking with an extended Kalman filter | A1 |
| Landmark-based localization from camera detections | A2 |
| Oriented bounding box detection | S8 |

### Machine learning and robot learning

| Skill | Earned by |
|---|---|
| PyTorch training, evaluation, and experiment tracking | M7, A5, A7 |
| ONNX export, ONNX Runtime, TensorRT, 16-bit and 8-bit quantization | M7, S6 |
| Trajectory prediction and intent classification | A5 |
| Behavior cloning and dataset aggregation | A7, A9 |
| Reinforcement learning, reward design, curriculum, residual policies | A10 |
| Self-play and multi-agent reinforcement learning | S4 |
| Teacher-student distillation and end-to-end driving | S10 |

### Planning, control, and state estimation

| Skill | Earned by |
|---|---|
| Raceline optimization, velocity profiles, lap time prediction | M9 |
| Pure pursuit and proportional-integral-derivative control | M9 |
| Frenet-frame sampling planner and behavior planning | A6 |
| Extended Kalman filtering for tracking and localization | A1, A2 |
| Sensor fusion (odometry, inertial measurement unit, camera) | A2 |
| System identification | S7 |
| Model predictive control | S9 |

### ROS 2 and simulation

| Skill | Earned by |
|---|---|
| Gazebo worlds, sensors, procedural generation, ros_gz_bridge | M1, M2, M6 |
| URDF and xacro, Ackermann steering, namespaced multi-robot tf2 | M2, S3 |
| Services, actions, custom messages | M3, M4 |
| rosbag2 with MCAP | M5 |
| Quality of service profiles and interface contracts | M10 |
| Command multiplexing and failsafe stop | M3, A9 |
| ros2_control and gz_ros2_control | S5 |
| Multi-machine ROS 2 and hardware-in-the-loop testing | S6 |

### Embedded and driver work

| Skill | Earned by |
|---|---|
| USB and Bluetooth human interface device protocol | M3, A9 |
| Byte-level report parsing and cyclic redundancy checks in C++ | M3, A9 |
| Fixed-rate polling loop and jitter measurement | M3 |
| Output reports for lights, rumble, and adaptive triggers | A9 |
| udev rules and hidraw | M3 |
| NVIDIA Jetson deployment | S6 |

### Data and machine learning operations

| Skill | Earned by |
|---|---|
| Robot-log-to-dataset pipeline (rosbag2 to Parquet) | M5 |
| Dataset versioning and dataset cards | M5 |
| Model registry and champion-challenger promotion | A8 |
| Scenario-based regression testing | A8 |
| Continual learning with forgetting checks | A8 |

### Evaluation and engineering practice

| Skill | Earned by |
|---|---|
| Race protocol design and fairness controls | M4, M10 |
| Win rates with Wilson intervals and controlled comparisons | M10, A6 |
| Held-out human evaluation | S1 |
| Skill ratings (Elo, TrueSkill) | S4 |
| Modern C++ with Eigen, pybind11 | M3, A1, A6, A10 |
| colcon, CMake, Docker, pytest, GitHub Actions | M10 |
| Design journal, demonstration video, technical writing | M10, S2 |

---

## Skills a robotics job wants that this project does not earn yet

### Carried over from the webcam version

Listed so you can see the lineage when you revisit this section after more research.

| Webcam-version item | Where it lives now |
|---|---|
| M1 world and differential-drive robot | M1, M2 (race track and Ackermann cars) |
| M2 webcam calibration, M7 AprilTag board | M2 (rendered checkerboard), Phase 3 (real camera) |
| M3, M4 labels and randomization | M6 |
| M5, M6 detector training and node | M7 |
| M7 ground-plane homography | M8 |
| M8, M9, M10 repository, interface contract, demonstration, journal | M10 |
| A1 scene bridge | Retired: nothing physical to mirror in pure simulation |
| A2 sim-to-real gap | A3, measured on FSOCO instead of webcam frames |
| A3 randomization ablation | A4 |
| A4, A10 Nav2 work and costmap plugin | A different project (below); lifecycle nodes are still addable |
| A5 DualSense driver, S2 haptics | M3 (USB), A9 (Bluetooth and haptics) |
| A6 command multiplexer | M3 (failsafe), A9 |
| A7 data collection mode | M5, A8 |
| A8 export and quantization | M7 (ONNX), S6 (TensorRT) |
| A9 Kalman tracking | A1 |
| S1 behavior cloning | A7 |
| S3 controller inertial input | Retired; sensor fusion now lives in A2 |
| S4 segmentation, S5 real robot data, S9 real-data injection | Still addable (below) |
| S6 write-up | S2 |
| S7 depth obstacle layer | Retired with Nav2; camera and lidar fusion is the still-addable replacement |
| S8 Jetson deployment | S6 |
| S10 second camera | M2 (front and rear cameras) |
| ros2_control and model predictive control (were "different project") | S5, S9 |

### Still addable inside this project

| Skill | How to add it | Cost |
|---|---|---|
| Camera and lidar fusion | Add a simulated two-dimensional lidar; take the opponent's range from lidar points that project inside its box; compare against M8's camera-only error. | About two days, extends M8 and A1 |
| Tracking metrics | Add multiple object tracking accuracy and identity F1 score to A1 so the tracker has a standard number, not just an error reduction. | About half a day, extends A1 |
| Real-data injection | Mix 0, 25, 50, and 100 FSOCO frames into training and remeasure the gap to find the point of diminishing returns. | About one day, extends A4 |
| Image-to-image translation | Restyle synthetic frames toward FSOCO with unpaired translation (CycleGAN); one more ablation row. | About three days, extends A4 |
| Foundation-model labeling | An open-vocabulary detector plus Segment Anything auto-labels real frames; you verify them. | About one day, for Phase 3 real frames |
| Semantic segmentation | A drivable-surface and track-limit segmentation head trained on segmentation-camera masks. | About three days, extends M6 and M8 |
| Managed lifecycle nodes | Detector, filters, and race controller as lifecycle nodes, so the race stack comes up and down in a defined order. | About one day, extends M10 |
| Hardware-in-the-loop continuous integration | The Jetson as a self-hosted GitHub Actions runner replaying A8's regression scenarios on every commit. | About one day, extends A8 and S6 |
| Real robot data | A 2025 RoboRacer platform survey came out of the University of Central Florida's Department of Electrical and Computer Engineering; ask that group for real 1/10-scale rosbags and run the pipeline on them. | About two days, extends A3 |

### Belongs to a different project
**Ignore some**: SLAM needed for this project given AV Racecars need to build maps of environment to determine best racelines. Add anything else that comes up when tackling this project, and schedule separate time blocks to learn anything else that blocks my progress from getting a deliverable done on time. The more time I spend mopping or unsure of myself the more time I lose. I need this project on my resume pronto, and frankly speaking if possible, I need the Embedded aspect part sooner being implemented and workon overtime.

Do not force these in. They are real gaps, but they belong to other anchor projects in the roadmap (`robotics-target-candidate-profile.md`), and bolting them onto a racing project makes both worse.

- Bare-metal C, interrupts, direct memory access, real-time operating systems (roadmap Project 1)
- Motor control, current sensing, cascaded proportional-integral-derivative loops (roadmap Project 1); the real car uses an off-the-shelf speed controller
- Controller area network and other field buses (roadmap Project 1, existing Teledyne FLIR and Project Storm experience)
- Simultaneous localization and mapping, `slam_toolbox`, particle-filter localization, and Nav2 (roadmap Project 2); racing at the limit is not Nav2's job, and the real track uses the S8 overhead camera instead of mapping
- micro-ROS (roadmap Project 2)
- Forward and inverse kinematics, Jacobians, MoveIt 2 (roadmap Project 4)
- Linear quadratic regulator for balancing (roadmap Project 4 or a balancing robot); model predictive control now lives here as S9

---

## Interview questions this project lets you answer

Write the answer to each one in the design journal as the corresponding deliverable is finished, not at the end.

| Deliverable | Question you can now answer |
|---|---|
| M3, A9 | Explain the DualSense Bluetooth report format and why it needs a checksum that the USB path does not. |
| M4, M10 | How did you make races fair and reproducible? |
| M6 | Where did your training labels come from, and how did you know they were correct? |
| M7, A3 | Your detector never saw a real image in training. How badly did it do on real cones, and why? |
| A4 | Which randomization parameters mattered, and were the differences bigger than seed noise? |
| M8 | How do you get distance from one camera, and how does the error grow with range? |
| A1 | What happens to your estimate of the opponent while it is alongside you and invisible to both cameras? |
| A2 | How does your car know where it is without ground truth, and what happens when a cone is misdetected? |
| A5 | How do you predict what a human driver will do next, and how do you know it beats a constant-velocity guess? |
| A6 | How do you know your car wins because of prediction and not because it is faster? |
| A7 | Why does behavior cloning drift off the track, and how did you fix it? |
| A8 | How do you stop a retrained model from forgetting how to beat last month's strategies? |
| A9 | What happens if the controller disconnects mid-race? |
| A10 | Why train in a second simulator, and what broke when the policy came back to Gazebo? |
| Protocol | Your car wins 80 percent of the time. How sure are you, and against whom? |
| S5, S6 | What changes when this runs on a real car? |
| S3, S4 | How would you scale this to six cars? |

---

## Feasibility notes

### Gazebo Jetty with Lyrical

`sudo apt install ros-lyrical-ros-gz` installs the recommended Gazebo for Lyrical on Ubuntu 26.04. The bounding box camera (visible and full boxes) and the segmentation camera have existed since Gazebo Fortress, and see only models that carry the Label system. Gazebo's Ackermann steering system takes velocity commands as a Twist, so the teleoperation node converts steering angle to yaw rate:

```python
from math import tan


def steering_to_yaw_rate(speed_mps: float, steering_rad: float, wheelbase_m: float) -> float:
    """Bicycle-model yaw rate for the Twist that Gazebo's Ackermann steering system expects."""
    return speed_mps * tan(steering_rad) / wheelbase_m
```

### Frame rates and compute

Human races must run at real-time factor 1.0 with rendering on, so measure the budget in week 1: Gazebo rendering, two cameras, and the detector on one laptop. YOLO26 nano through ONNX Runtime on the processor alone is the starting point; the model was designed for fast inference without a graphics card. If the budget is tight, lower camera resolution, run the rear camera at half rate, and show the human the Gazebo window rather than an extra video stream. Labeling runs offline, so only races must be real time. The processor handles M7 overnight and the small networks in A5, A7, and A10; A4's repeated retrains want the planned CUDA-capable computer or a free cloud notebook.

### Python environments

ROS 2 binds to the system Python. Keep training and ROS 2 nodes in separate virtual environments, and keep node inference in ONNX Runtime so node dependencies stay small:

```bash
# Training environment: no ROS 2 inside
python3 -m venv ~/venvs/train
~/venvs/train/bin/pip install ultralytics

# Node environment: sees the system rclpy, adds only ONNX Runtime
python3 -m venv --system-site-packages ~/venvs/node
~/venvs/node/bin/pip install onnxruntime
```

### Why two cameras

A car that cannot see behind it cannot defend, and the side blind spot during an overtake is exactly where A1's tracker earns its keep.

### Scale

Build at 1/10 scale. Size cones for 1/10 cars (about 15 centimeters, a mini-cone size you can buy for the real track) but keep Formula Student colors and stripes, so FSOCO images stay a fair appearance test. FSOCO is shot from full-size cars, so name the camera-height difference when you explain the gap.

### FSOCO

Now publicly available without the contribute-first requirement it launched with; over 11,000 labeled images of real cones. Use it as a test set (and for the still-addable real-data injection), and check its license before redistributing any images.

### DualSense

Linux ships the hid-playstation kernel driver, so the joy package works out of the box and is the fallback if your driver ever blocks a race; your driver reads the hidraw node alongside it. USB input report `0x01` is 64 bytes. Over Bluetooth the controller sends a reduced report until a feature report is read, then switches to the 78-byte `0x31` report. Bluetooth reports carry a CRC32 computed over a one-byte prefix plus the report (`0xA1` for input, `0xA2` for output), and the controller ignores output reports whose checksum is wrong. The kernel's hid-playstation source is the most reliable format reference. Rule file for non-root access over both links:

```text
# /etc/udev/rules.d/70-dualsense.rules
# DualSense over USB
KERNEL=="hidraw*", ATTRS{idVendor}=="054c", ATTRS{idProduct}=="0ce6", MODE="0660", TAG+="uaccess"
# DualSense over Bluetooth
KERNEL=="hidraw*", KERNELS=="*054C:0CE6*", MODE="0660", TAG+="uaccess"
```

```bash
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### Pace matching

Scale the whole velocity profile by one parameter and bisect on solo laps in simulation until lap time is within 1 percent of the human's warm-up median. Recompute every session, because the human gets faster too.

### Prediction data

A 5-lap race at 10 hertz gives roughly 700 overlapping 4-second windows, all highly correlated. Split training and validation by race, never by window, or the model memorizes laps. Train on the tracker's output (or ground truth with matched noise), because at race time the model only ever sees tracker output.

### Two learned models of the human

A5 predicts the human online so the car can plan against them. A7 is a stand-in human for training and testing offline. Same logs, different jobs.

### Fast simulator

f1tenth_gym (Python, single-track dynamics, multiple agents) or your own bicycle-model environment; either loads M1's centerline and occupancy image. Gazebo stays the judge: no result counts until it holds up there.

### Detector license

Ultralytics YOLO is licensed under the GNU Affero General Public License (AGPL-3.0): fine for a public portfolio repository. If the fleet ever becomes a product, swap in a permissively licensed detector or buy a license.

---

## Designed-in sim-to-real seams

Each row is an interface that stays fixed while what sits behind it changes.

| Interface | In simulation | On the real car |
|---|---|---|
| Vehicle parameter file | Gazebo values | System identification output (S7) |
| Actuation | Gazebo Ackermann system, then ros2_control (S5) | ros2_control hardware interface for the speed controller and steering servo |
| Camera intrinsics (`camera_info`) | From the car's model definition | Checkerboard calibration of the real camera |
| Race control positioning | Ground truth, then the overhead camera (S8) | Overhead camera above the real track |
| DualSense driver and teleoperation topics | Unchanged | Unchanged; pair the controller directly to the car's computer to skip a Wi-Fi hop |
| Scenario file and race rules | Unchanged | Unchanged |
| Dataset schema | Tagged as simulation | Tagged as real; both feed the same flywheel |

---

## Week-by-week guardrails

| Window | Deliverables | Guardrail |
|---|---|---|
| Week 1, September 21 to 27 | M1, M2 | Build at 1/10 scale from day one; one vehicle parameter file for both cars. Measure the frame-rate budget now. |
| Week 2, September 28 to October 4 | M3, M4, M5 | USB only. If the driver blocks racing, race with the joy package and keep writing the driver. Log every lap from here on. |
| Week 3, October 5 to 11 | M6, M7 | Do not tune the detector. |
| Week 4, October 12 to 18 | M8, M9 | Plot error against range before trusting any distance. |
| Week 5, October 19 to 25 | M10 | Freeze the minimum viable product, tag a release, run Race Day One, record demonstration one. |
| Week 6, October 26 to November 1 | A1, A2 | Both are extended Kalman filters; write the shared filter code once. |
| Week 7, November 2 to 8 | A3, A4, A5 | Ablation retrains run overnight while you build prediction. |
| Week 8, November 9 to 15 | A6, A7 | If A5 slipped, A6 runs with A1's filter prediction; the learned model is an upgrade, not a dependency. |
| Week 9, November 16 to 22 | A8, then A9 | Three rounds of the loop beat one more feature. |
| Week 10, November 23 to December 1 | Checkpoint | Written deliverables first: README results, journal, demonstration two (prediction on versus off, plus the learning curve). Thanksgiving is November 26. |
| After December 1 | A10, leftovers, Stretch | Season 1 begins. |

> [!WARNING]
> **Defer order if behind schedule**
>
> On a year-long timeline nothing is cut, only deferred past December 1. Defer in this order: S10 through S1, then A10, A9, A7, A4, A8, A2, A5. If A7 is deferred, A8 gates on offline prediction metrics plus a short live block instead of overnight races against the clone. **Never defer A3 or A6**, or A1, which A6 stands on. A3 is the computer vision measurement on real pixels, and A6 is the evidence that the car wins by predicting the human rather than by speed. Without them, the checkpoint is a racing game with a detector attached.

---

## Bill of materials, checkpoint 1

| Item | Cost | Notes |
|---|---|---|
| DualSense controller | $0 if owned, roughly $70 to $80 new | Any USB-C data cable is enough for M3; Bluetooth waits for A9. |
| Compute | $0 | Current laptop through M10; the planned CUDA-capable computer or a free cloud notebook for A4's retrains; the Jetson you already own for S6. |
| Real-image data | $0 | FSOCO for cones; your own photos of any small car for the `race_car` class. |
| Checkerboard and AprilTag board | $0 | Not needed in simulation; they return in Phase 3. |
| Car hardware | $0 | Nothing before the checkpoint; priced in the racing fleet plan. |

---

## Beyond December 1: from one simulated rival to six real cars

At least a year in total, matching the racing fleet plan. Every phase has an exit test so progress stays measurable.

| Phase | Window | Goal | Exit test | New skills |
|---|---|---|---|---|
| Season 1 | December to February | Finish Advanced leftovers and A10; run the loop until the target; S1 and S2 | Observed win rate of at least 80 percent over at least 50 pace-matched races against you, interval reported; held-out drivers measured | Continual learning at scale, human evaluation |
| Phase 2: six cars in simulation | February to April | S3 and S4: six autonomous cars plus the human in one race; league training | 20-lap races with seven cars and at most one at-fault contact per 10 races; league ratings stable | Multi-agent learning, multi-robot ROS 2, middleware tuning (the Zenoh middleware versus the default Data Distribution Service) |
| Phase 3: one real car | April to June | S5 to S8 on hardware: ros2_control hardware interface, system identification into the parameter file, real camera calibration, overhead-camera positioning, the Jetson running the stack | 10 clean autonomous laps on a real track; sim-versus-real lap time gap measured; the A3 gap table repeated on your own real frames | Hardware bring-up, system identification, real calibration, sim-to-real transfer |
| Phase 4: real races | June to August | Human versus car on the real track under the same protocol; the human drives a second real car with the same DualSense driver | Win rate on hardware with intervals; real logs feeding the same flywheel | Real-world data collection, track safety operations |
| Phase 5: six real cars | Fall 2027 onward | Six autonomous cars plus human-driven cars | A 10-lap six-car race with no interventions; every race logged | Fleet operations, networking at scale, multi-car safety |
| Later | After Phase 5 | Drift cars and mixed traffic: rule-following cars driving point to point while racers weave through | Defined when the phase starts | Drift dynamics, traffic rules, interaction-aware planning |

> [!CAUTION]
> **Real cars need safety that does not depend on software.** A hardware kill switch on its own radio channel that cuts motor power without Wi-Fi or ROS 2, a speed cap enforced below the autonomy stack, and a geofence. Six cars multiply every failure mode, so the kill switch comes before the second car.

---

## Risk register

| Risk | Severity | Response |
|---|---|---|
| Gazebo, two cameras, and the detector cannot hold real time on the laptop | Medium | Lower resolution, rear camera at half rate, perception on the Jetson early (pull S6 forward), or wait for the CUDA-capable computer. Labeling runs offline, so only races must be real time. |
| Win rate rises because the car got faster, not smarter | High for validity | Pace matching plus A6's prediction-off comparison. Always report both open-pace and pace-matched results. |
| The model only beats its author | High for the claim | Held-out drivers (S1), with results reported per driver. |
| The human keeps learning, so the target moves | Medium | Models frozen per block; the human's own lap-time trend logged next to the win rate. Co-adaptation is the point of the project, not a flaw. |
| The behavior clone drifts off the track | Medium | Disturbance injection while you drive, takeover corrections (A9), dataset aggregation. |
| Retraining forgets old strategies | Medium | Replay old data every round, regression scenarios from past exploits, and the champion-challenger gate. |
| The FSOCO gap is huge | Medium | This is a result, not a failure. Report it, then use A4 to show what closes it. A large measured gap with a clear ablation is a stronger deliverable than a small unexplained one. |
| Reinforcement learning exploits the fast simulator or never converges | Medium | Residual actions on top of the planner, reward sanity checks, curriculum. The A6 planner stays the shipped path: A6 is the deliverable, A10 is the credential. |
| Contact physics unstable at speed | Medium | Smaller physics step, tuned friction, and geometric overlap checks in race control rather than contact sensors alone. |
| Bluetooth friction on Linux | Low | USB covers everything a race needs; Bluetooth is A9. |
| The Advanced tier does not fit four weeks | Certain, low impact | By design. Leftovers become Season 1. |
| Detector license (AGPL-3.0) | Low | Fine for a public repository; swap detectors before any commercial use. |

---

## Resources that feed specific deliverables

| Resource | Feeds |
|---|---|
| Gazebo Jetty documentation: worlds, Ackermann steering system, bounding box and segmentation cameras; ros_gz | M1, M2, M6 |
| Technical University of Munich racetrack-database and global_racetrajectory_optimization; Heilmeier et al., minimum curvature trajectory planning (2019) | M1, M9 |
| Edinburgh University Formula Student simulator (cone models, track generator; Gazebo Classic, so reference only) | M1, M6 |
| RoboRacer (formerly F1TENTH) course materials and f1tenth_gym | M9, A6, A10 |
| RoboRacer platform survey by Charles, Maghsoumi, and Fallah (University of Central Florida, 2025) | Phase 3, real robot data |
| Linux kernel hid-playstation driver source; community DualSense protocol notes | M3, A9 |
| Ultralytics YOLO26 documentation (detection, oriented boxes, export) | M7, S6, S8 |
| FSOCO dataset and paper (Vödisch, Dodel, Schötz, 2022) | A3, A4 |
| Deep Learning Specialization, Course 4 (convolutional networks) and Course 5 (sequence models) | M7, A5 |
| Stanford CS231n notes | M7, A4 |
| Roger Labbe, Kalman and Bayesian Filters in Python (free); Brian Douglas, MATLAB Tech Talks | A1, A2 |
| Probabilistic Robotics (Thrun, Burgard, Fox), extended Kalman filter localization | A2 |
| Self-Driving Cars Specialization (Toronto): State Estimation and Localization; Motion Planning | A1, A2, A6 |
| Werling et al., optimal trajectory generation in a Frenet frame (2010) | A6 |
| Argoverse and Waymo Open Motion forecasting papers (metric definitions, baselines) | A5 |
| DAgger (Ross, Gordon, Bagnell, 2011); HG-DAgger (Kelly et al., 2019); DART disturbance injection (Laskey et al., 2017) | A7, A9 |
| Stable-Baselines3 and Gymnasium documentation; OpenAI Spinning Up | A10, S4 |
| Gran Turismo Sophy (Wurman et al., Nature, 2022); Swift drone racing (Kaufmann et al., Nature, 2023) | A10, S4 |
| Learning by Cheating (Chen et al., 2019) | S10 |
| Liniger, Domahidi, and Morari, optimization-based racing of 1:43-scale cars (2015) | S9 |
| ros2_control and gz_ros2_control documentation | S5 |
| NVIDIA jetson-inference and TensorRT samples | S6 |

---

# Project Super-AV-Racer, Part 2: Building the Real AV Cars (Electrical and Embedded)

Ten deliverables per tier, same format as Part A: each carries an acceptance test, the artifact it produces, and the job-relevant skills it builds. Solo.

**Window: 26 weeks, March 1 to August 29, 2027**, the second half of the master's year. Part A (simulation) runs September through February.

**Checkpoints:** minimum viable product May 30 (one car, 10 clean autonomous laps on its own sensors); Advanced August 15 (a fleet racing under the protocol); wrap-up August 29.

**Effort assumption:** about 15 to 20 hours a week through April, 25 to 30 from May. If a summer internship takes those hours, the tiers stay and the calendar stretches into fall.

**Stack:** STM32G474 microcontroller, FreeRTOS, C++17 firmware, KiCad, controller area network (CAN), ROS 2 with ros2_control, and the Jetson and Raspberry Pi 5 you already own for the first two cars.

---

## The pitch

Part A proves the autonomy in simulation. Part B builds the cars it runs on: your own vehicle control board and firmware, the power and safety systems, sensors and calibration, a real track with race control, and then six copies of the car. The simulation-trained models move onto the real cars through the seams Part A designed in, get tuned and retrained on real data, and race people and each other under the same rules.

**Buy the mechanics, build the electronics.** A hobby rolling chassis and an off-the-shelf motor controller carry the car. Everything that decides what the car does, and everything that stops it, is your design.

**Why this reads as autonomous vehicle work.** Current Waymo Hardware Engineering postings ask for C++ on microcontrollers, board-level telemetry and fault handling, schematic review and debugging with JTAG (the standard debug port), oscilloscopes, and logic analyzers, vehicle networks described in CAN database files, verification and validation plans, functional safety, and fleet software delivery and monitoring. Part B is that job at 1/10 scale; "Where this lines up with autonomous vehicle job postings" maps each item to the deliverable that earns it.

---

## What Part B needs from Part A

Needed on day one: M3 (DualSense driver), M4 (race control), M5 (logging and dataset pipeline), M7 (detector and ONNX export), M8 (ground-plane projection), M9 (baseline racer), A1 and A2 (filters), S5 (ros2_control in simulation), S7 (system identification procedure), and S8 (overhead camera in simulation). Needed by week 22: S3 (the multi-car stack). If S5, S7, or S8 are unfinished on March 1, they come first and push this schedule about two weeks.

| Part A seam | Real side, built in Part B |
|---|---|
| Vehicle parameter file | BM9 system identification on the real car |
| Actuation through ros2_control | BM2, BM4, BM6: firmware, board, and the hardware interface |
| Camera intrinsics (`camera_info`) | BM7 calibration of the real cameras |
| Race control positioning | BM8 overhead camera above the real track |
| DualSense driver and teleoperation topics | BA5 human-driven car |
| Scenario file and race rules | BM8 and BA10, plus the real-world rules below |
| Dataset schema | BA9 real logs flowing into the same pipeline |

---

## Vehicle architecture

```text
 RACE CONTROL LAPTOP (overhead camera, timing, flags, logs)
          |  Wi-Fi (Zenoh middleware)
 CAR COMPUTER (Jetson or Raspberry Pi 5, one container)  <-- front and rear cameras
          |  <-- DualSense over Bluetooth (human-driven role)
          |  USB-to-CAN adapter
 ===== AUTONOMY BUS: CAN, 1 megabit per second ===============================
          |
 VEHICLE CONTROL BOARD (STM32G474, FreeRTOS, C++17)
   gateway: command limits, heartbeat, vehicle state machine, fault log
   inertial sensor, power monitor, servo output
          |
 ===== VEHICLE BUS: CAN, 500 kilobits per second =============================
          |
 MOTOR CONTROLLER (VESC-class)  -->  sensored brushless motor

 MARSHAL RADIO  ~~>  ExpressLRS receiver  -->  SAFETY MICROCONTROLLER
   on kill or signal loss: silence the vehicle bus, alert the board,
   switch off drive power once the car has stopped

 3S LiPo --> protection and power monitor --> motor controller (high current)
                                          --> computer, servo, and 3.3 volt logic rails
```

> [!IMPORTANT]
> **Scope decisions**
>
> **Buy the mechanics, build the electronics.** Rolling chassis and a VESC-class motor controller are bought. Custom motor control (current sensing, field-oriented control) stays in roadmap Project 1.
>
> **The board is a gateway.** The computer sits on the autonomy bus and the motor controller on the vehicle bus; only the board touches both. It enforces limits, heartbeats, and the vehicle state machine, the same split production autonomous vehicles keep between the autonomy computer and the vehicle platform.
>
> **One CAN database (DBC) file is the source of truth.** Firmware packing code is generated from it, logs are decoded with it, and the ROS 2 hardware interface is built against it.
>
> **Hard real-time on the microcontroller, soft real-time on Linux.** Anything that must happen within milliseconds (limits, timeouts, controlled stop) lives in firmware.
>
> **Safety in three independent layers:** software on the computer, firmware on the board, and a separate safety microcontroller with a radio kill that still works when both are hung. Nothing moves on the floor before BM3 passes.
>
> **STM32 with FreeRTOS in C++17.** Project Storm already shows ESP32, FreeRTOS, and CAN. An STM32 adds a second microcontroller family while FreeRTOS and CAN carry over, and C++ matches what autonomous vehicle hardware teams ask for on microcontrollers.
>
> **Two board revisions are planned, not hoped for.** Revision A uses an off-the-shelf module for the high-current computer rail.
>
> **Identical cars, roles assigned by race control.** Any car can be autonomous or human-driven. A cheaper human-only variant is optional (see the bill of materials).
>
> **Choose the fleet's computer with data** (BA2), not before, and run one ROS 2 distribution everywhere, shipped in containers.

---

## Real-world race rules

These extend Part A's defined race conditions. Race control enforces what it can; technical inspection covers the rest.

| Rule | Setting | Why |
|---|---|---|
| Marshal | One person holds the kill transmitter during every session and is the only person who can start one | The hardware kill needs a human with authority |
| Track entry | Nobody inside the track while any car is armed | Cars at speed hurt ankles |
| Red flag | Race control broadcasts a red flag and every car makes a controlled stop; losing race control's heartbeat triggers the same stop | Stops the fleet without touching the radio |
| Speed caps | Set per session class (practice, race) from measured stopping distance and runoff, enforced in board firmware | Software on the computer cannot exceed them |
| Technical inspection | Before each session: kill test, brake test, speed-cap test, battery voltage, firmware and calibration versions; results logged | Catches a bad car before it moves |
| Human drivers | Drive by line of sight from a driver stand; throttle requires a held enable button | Letting go of the controller stops the car |
| Perception-only | Autonomous cars never use overhead-camera positions in scored races (Part A's rule) | The overhead camera is the referee, not a sensor |
| Batteries | Minimum pack voltage to start a session; charging only in fireproof containers, attended | Brownouts and fires |
| Incidents | Any contact, kill, or red flag creates an incident record from logs | Every incident becomes a test case |

---

## How to read the deliverable tables

Same as Part A. **Deliverable** is what to build, **Acceptance test** is how you know it is done, **Artifact produced** is what exists afterward, and **Skills built** lists the skills, phrased the way job postings phrase them, that finishing the row earns.

---

## Minimum viable product, weeks 1 to 13 (March 1 to May 30)

Required. Target 6 out of 10 on its own. One car with your board and firmware, a working kill chain, a real track, and 10 clean autonomous laps on its own sensors.

| # | Deliverable | Acceptance test | Artifact produced | Skills built |
|---|---|---|---|---|
| BM1 | **Requirements, architecture, and safety concept.** Vehicle requirements (speed caps, mass, run time, sensors, control rates); electrical block diagram; power budget per rail with peak and continuous current; the CAN database file for both buses with a bus-load estimate; a lightweight hazard analysis and risk assessment with safety goals (for example, "a controlled stop within the runoff from any speed cap"); an interface document tying each Part A seam to its real side. | Reviewed by at least one outside engineer (club member or lab mate) with comments closed; estimated bus load under 30 percent; every safety goal traces to at least one planned test | The design baseline every later deliverable is checked against | Requirements engineering, system architecture, power budgeting, CAN network design (database files, bus load), hazard analysis and risk assessment, interface documents, design reviews |
| BM2 | **Drive-by-wire bench prototype.** On a NUCLEO-G474RE with CAN transceiver and inertial sensor breakouts: FreeRTOS firmware in C++17 with both CAN buses, servo output, motor controller commands and status on the vehicle bus, the inertial sensor over Serial Peripheral Interface with a data-ready interrupt and direct memory access, a heartbeat monitor, the vehicle state machine (boot, standby, armed, driving, controlled stop, fault), command limits, and a rolling counter and checksum on every command frame. A laptop drives it through a USB-to-CAN adapter. | Wheels and steering follow commands with the car on a stand; a missing heartbeat triggers a controlled stop within 100 milliseconds; state machine and limit logic covered by host unit tests (GoogleTest); inertial data at 400 hertz with microcontroller timestamps; measured bus load compared against BM1 | Working drive-by-wire firmware before any custom hardware exists | STM32 peripherals (FDCAN, the built-in CAN controller; timers; Serial Peripheral Interface with direct memory access; interrupts), FreeRTOS tasks, queues, and priorities, modern C++ on microcontrollers, CAN drivers, end-to-end protection (counters and checksums), state machine design, host-based unit testing |
| BM3 | **Independent kill chain and controlled stop.** An ExpressLRS receiver with servo-signal outputs and failsafe feeds a small safety microcontroller that owns only the kill chain. On kill or signal loss it silences the vehicle bus (transceiver standby) so the motor controller's own command timeout brakes the car, alerts the main microcontroller, and switches off drive power once the car has stopped. One marshal transmitter controls every car through a shared binding phrase. | Kill switch, transmitter off, receiver unplugged, main microcontroller hung, and computer crash each tested 10 times at three speeds; stopping distance versus speed table; zero failures to stop | The safety system every later test depends on, with measured stopping distances | Safety architecture (defense in depth, fail-safe defaults, independent channels), radio-control failsafe behavior, watchdogs, controlled stop as the minimal risk condition, safety test design |
| BM4 | **Vehicle control board, revision A.** A 4-layer KiCad board: STM32G474; two CAN transceivers with switchable termination; inertial sensor; digital power monitor on the battery input; reverse-polarity and transient protection; logic and servo power rails (the computer rail from an off-the-shelf module on revision A); the safety microcontroller; receiver input; servo and spare outputs; debug header; test points on every rail and bus; status lights. Fabricated and assembled by a board house. | Electrical and design rules checks clean; review against a written checklist with at least one outside reviewer; every part in stock with a second source noted; order placed by the end of week 6 | Your own automotive-style controller board and its design package (schematic PDF, layout, bill of materials, review record) | Schematic capture and 4-layer layout (KiCad), power distribution and protection circuits, CAN physical layer, mixed-signal layout, design for manufacturing and assembly, design reviews, bill-of-materials management |
| BM5 | **Board bring-up and firmware port.** Bring-up on a current-limited supply: rails, clocks, programming over Serial Wire Debug, then each interface in turn. Port the firmware from the Nucleo. Add board management (rail voltages and currents, temperature, power-sequencing checks) and fault codes stored in flash, all reported on CAN. | Bring-up log with measured rail voltages and ripple; every interface passes its test; errata list for revision B; board telemetry streaming at 10 hertz | A working board, a bring-up report, and the errata list that drives revision B | Hardware bring-up, debugging with JTAG and Serial Wire Debug, oscilloscopes, and logic analyzers, board management (power sequencing, rail monitoring), fault codes, board-level telemetry |
| BM6 | **Car computer integration.** The car computer (your Jetson, for car 1) runs the Part A stack in a container. A C++ ros2_control hardware interface talks to the board over SocketCAN using the CAN database file. Clocks are synchronized between computer and microcontroller over CAN. Latency from camera exposure to steering motion is measured with a board-controlled light in the camera's view. | Part A's M9 racer drives the real car with no code changes above the hardware interface; clock offset under 1 millisecond (distribution reported); latency budget table from light to wheel | The Part A seam made real: one stack, two worlds | ros2_control hardware interfaces, SocketCAN, code generation from CAN database files (cantools), containers on embedded Linux, clock synchronization, end-to-end latency measurement |
| BM7 | **Sensors and calibration.** Front and rear global-shutter cameras with exposure set for indoor lighting (motion blur versus light flicker); intrinsic calibration; camera-to-car extrinsics; camera-to-inertial calibration (Kalibr); inertial noise parameters from a 3-hour static log (Allan variance); wheel odometry scale from measured runs; calibration files versioned per car. | Reprojection error under 0.5 pixels; camera-to-inertial residuals reported; Allan variance plot, with its noise parameters now used in Part A's filters; odometry distance error under 2 percent over 10 meters | A calibrated car and a repeatable calibration procedure for five more | Camera calibration (intrinsics, extrinsics), camera-to-inertial calibration (Kalibr), inertial noise characterization (Allan variance), odometry calibration, calibration data management, exposure and flicker handling |
| BM8 | **Real track and race control.** The track laid out from the same track file Gazebo uses (printed cone coordinates, floor tape marks); 15-centimeter cones; soft barriers at fast corners; overhead camera or cameras calibrated to the floor with AprilTags; Part A's race control on real positions (S8's oriented boxes); a dedicated router; a marshal station holding the kill transmitter. | Cone positions within 3 centimeters of the track file; overhead position error under 5 centimeters at taped check points; lap timing within 50 milliseconds of video frame counts; the track rebuilds from its file in under an hour | A real track that is a measured twin of the simulated one, and a race-control station | Multi-view geometry and ground-plane calibration, fiducial markers, vision-based tracking and timing, test facility design, network setup |
| BM9 | **System identification and simulation update.** Measure the steering map (command to curvature), acceleration and braking limits, top speed, tire grip from constant-radius circle tests, and actuation latency; write them into the vehicle parameter file; rerun Gazebo and the fast simulator with it. | Simulated and real paths agree within a stated tolerance for the same recorded inputs; the reality gap reported as lap-time and path error | A parameter file measured on the real car, and a quantified reality gap | System identification, vehicle dynamics (bicycle model, friction circle), experiment design, model validation |
| BM10 | **First autonomous laps: Race Day Zero.** Part A's baseline racer on the real car, first on overhead-camera positions (used like motion capture), then on onboard localization only (Part A's A2 on real sensors). A first scored session, a demonstration video, and a tagged hardware and firmware release. | 10 consecutive clean laps on onboard localization; lap time within 15 percent of the simulator's prediction at the same speed cap; laps per intervention reported; zero safety incidents | The first real autonomous laps, a measured sim-versus-real gap, and demonstration three | Sim-to-real transfer, localization on real sensors, field test procedures, test reports, intervention metrics |

> **Resume line after the minimum viable product**
>
> "Designed a vehicle control board (STM32, dual CAN, 4-layer KiCad) and C++ FreeRTOS drive-by-wire firmware with an independent radio kill chain for a 1/10-scale autonomous race car; calibrated its cameras and inertial sensing, identified its dynamics, and ran simulation-trained autonomy on it for X clean laps within Y percent of simulated lap time."

---

## Advanced, weeks 14 to 24 (May 31 to August 15)

Target 8 out of 10. Two cars racing, validated safety, real data in the loop, then the fleet.

| # | Deliverable | Acceptance test | Artifact produced | Skills built |
|---|---|---|---|---|
| BA1 | **Board revision B and production test.** Revision A errata fixed; the computer rail integrated on the board if the module design proved out; connectors and mounting reworked for assembly. An end-of-line test: a fixture and script that checks every interface of a new board automatically over CAN. | Revision B passes bring-up with no rework wires; the end-of-line test catches a deliberately introduced fault; at least 6 boards built and passed | A production-ready board, and the test that proves each copy works | Design iteration, design for manufacturing and test, production test fixtures, test automation, quality records |
| BA2 | **Compute bake-off and edge deployment.** Run the real perception stack on the Jetson (TensorRT) and on your Raspberry Pi 5 with an AI HAT+ (Hailo compiler): frames per second, latency, power, heat, and cost. Choose the fleet's computer from the numbers, with 8-bit quantization calibrated on real frames. | Decision table with measured numbers; the chosen computer runs both cameras at the required rate; accuracy lost to quantization reported | A data-backed compute decision for six cars | Edge inference (TensorRT, Hailo), 8-bit quantization with real calibration data, power and thermal measurement, cost-performance analysis, technical decision records |
| BA3 | **Fault injection and safety case.** A failure mode and effects analysis of the car; a fault-injection campaign (CAN wire cut, a babbling node flooding the bus, brownout, sensor dropout, computer freeze, radio loss, stuck servo); a lightweight safety case linking hazards, mitigations, and test evidence; the technical inspection checklist in use. | Every hazard from BM1 has a mitigation and passing evidence; every injected fault ends in a controlled stop; inspection results logged per session | A safety case with evidence, the document autonomous vehicle safety teams actually produce | Failure mode and effects analysis, fault injection, safety case construction, ISO 26262 and UL 4600 concepts, requirements traceability, operational safety procedures |
| BA4 | **Hardware-in-the-loop bench and firmware continuous integration.** The real board in the loop with Gazebo: a bridge feeds simulated wheel speed and inertial data onto CAN and returns the board's actuator commands to the simulation. pytest and python-can scenarios cover timeouts, limits, kill, and bus-off recovery. GitHub Actions builds firmware, runs host unit tests and static analysis (clang-tidy), and publishes images; a self-hosted runner executes the bench suite on hardware. | At least 20 bench scenarios; every firmware change passes the bench before reaching a car; at least one regression caught by the bench documented | A bench that makes firmware changes safe, the core of verification and validation | Hardware-in-the-loop testing, test automation (pytest, python-can), firmware continuous integration, static analysis, self-hosted runners, verification and validation practice |
| BA5 | **Human-driven car and the 1v1 series.** Car 2 uses a spare revision A board and the Raspberry Pi 5 you own; race control assigns it the human-driven role. The DualSense pairs with the car's computer through Part A's driver; drivers race by line of sight from the driver stand; board firmware enforces the pace-matching speed cap. | 20 protocol races with zero safety incidents; win rate with its interval; every race logged on both cars and at race control | The first real human-versus-autonomous races | Human-in-the-loop system operation, low-latency wireless control, race operations, field data collection |
| BA6 | **Real-image perception and offboard auto-labeling.** The surveyed cone map and each car's pose from the overhead camera, projected into onboard frames, give free labels for cones and other cars. Measure the real gap against Part A's FSOCO gap, fine-tune on real frames, and redeploy. | Auto-label precision checked on 200 hand-verified frames; real-image mean average precision before and after fine-tuning; the onboard detector meets its rate on the chosen computer | A real-data labeling pipeline and a closed sim-to-real perception gap | Offboard perception and auto-labeling, cross-view geometry, dataset curation, fine-tuning with real data, domain gap measurement |
| BA7 | **Firmware updates and diagnostics over CAN.** A bootloader with two image slots and automatic rollback; updates from the car computer over CAN using the ISO 15765-2 transport (ISO-TP) and a small Unified Diagnostic Services subset (read data by identifier, read fault codes, request download, transfer data, reset); version and git hash reported on the bus. | One command from race control updates every car; a corrupted image rolls back on its own; fault codes readable remotely | Over-the-air updates and diagnostics for the fleet, the way vehicle modules are serviced | Bootloaders, flash memory management, ISO-TP, Unified Diagnostic Services (ISO 14229) basics, diagnostic trouble codes, over-the-air update design, rollback |
| BA8 | **Fleet build: cars 3 to 6.** Batch assembly from a written, photographed procedure; per-car calibration with the BM7 procedure; car identity (an identifier in firmware, calibration files keyed by it); the end-of-line test on every car; a fleet telemetry dashboard (battery, temperatures, fault codes, firmware versions) with alerts; battery logistics (labels, charge logs, storage voltage). | Each new car goes from parts to passing inspection in under one working day; all cars report healthy telemetry at the same time | A six-car fleet built like a small production run | Manufacturing procedures, calibration management, fleet telemetry and alerting, battery operations, configuration management |
| BA9 | **Real-data flywheel.** Logs offloaded after each session into Part A's dataset pipeline (tagged as real); prediction, clone, and perception retrained on mixed real and simulated data; redeployed through the container and firmware update paths; win rate tracked per round on the real track. | At least 2 real rounds with a win-rate learning curve; every deployed model traceable to its data and code; Part A's regression scenarios still pass | The learning loop running on real hardware | Real-world data pipelines, mixed simulated and real training, model deployment to edge fleets, field-data analysis |
| BA10 | **Multi-car racing.** Race control for any number of cars; the red flag and race-control heartbeat as networked stops, with the hardware kill as backup; the Zenoh middleware for multi-car networking; Part A's multi-car stack (tracking several opponents) on real cars; races with 3 to 6 cars mixing autonomous and human drivers. | 10 multi-car races with at most one at-fault contact; a red flag stops every car within its measured distance; laps per intervention reported per car | Real multi-car autonomous racing with humans in the mix | Multi-robot systems, networked safety functions, ROS 2 middleware configuration (Zenoh), multi-agent field testing, race operations |

> **Resume line after Advanced**
>
> "...built a six-car fleet with over-the-air firmware updates, Unified Diagnostic Services diagnostics, and fleet telemetry; validated safety with failure mode and effects analysis and a fault-injection campaign; auto-labeled real images from an overhead camera and surveyed map; raced humans against autonomous cars under a fixed protocol, retraining on real logs to raise win rate from A to B percent."

---

## Stretch, after August or if ahead

Target 9 out of 10.

| # | Deliverable | Notes | Artifact produced | Skills built |
|---|---|---|---|---|
| BS1 | **Six autonomous cars, no humans.** Part A's self-play league (S4) on hardware. | The six-car goal in its purest form | A real league ladder and rating history | Multi-agent racing on hardware, fleet-scale testing |
| BS2 | **Residual dynamics learning.** Learn the difference between the bicycle model and the real car from logs, add it to the fast simulator, retrain, and measure transfer. | The approach behind the Swift drone-racing result | A learned correction model and a smaller reality gap | Learned dynamics models, sim-to-real methods, model-based evaluation |
| BS3 | **Hardware-triggered cameras.** The board triggers global-shutter exposures and timestamps them. | Camera-to-inertial sync under 100 microseconds | Exact exposure timestamps for fusion | Hardware triggering, sensor synchronization, timing analysis |
| BS4 | **Event data recorder.** A ring buffer of the last 10 seconds of commands, states, and faults, written to flash on any kill or impact (inertial threshold). | Every incident gets a black box | Incident reports generated from recorder data | Crash-safe logging, flash wear management, incident analysis |
| BS5 | **Secure CAN commands.** Message authentication on drive commands with a freshness counter, in the style of AUTOSAR Secure Onboard Communication. | A replayed or forged frame is rejected and logged | An authenticated vehicle bus | Automotive network security, message authentication codes, key handling |
| BS6 | **Drift variant.** Rear-wheel-drive drift chassis with a yaw-rate controller in firmware (gyro-assisted countersteer) and drift-angle estimation. | First step of the street-racing extension | A drifting car and its controller | Yaw-rate control, vehicle dynamics at the limit, embedded control loops |
| BS7 | **Real-time Linux on the car computer.** PREEMPT_RT (in the mainline kernel since Linux 6.12), latency histograms before and after, and thread priority tuning. | Measure first, then tune | A latency report and a tuned configuration | Real-time Linux, latency measurement, scheduling |
| BS8 | **Lidar and camera fusion.** A low-cost two-dimensional lidar fused with camera detections for class-agnostic collision avoidance. | Part A's still-addable item, now on hardware | A second obstacle source and a fusion comparison | Sensor fusion, lidar processing, collision avoidance |
| BS9 | **Infrared lap-timing transponders.** One transponder per car and a start-and-finish receiver, on your own boards, as redundant timing. | Radio-controlled car timing systems use the same idea | An independent timing system | Infrared signaling, timing hardware, redundancy |
| BS10 | **Open hardware release and a public race day.** Publish the board design, firmware, and track kit; host a race day with the robotics clubs. | The public artifact recruiters can see | A release, a video, and a write-up | Technical communication, open-source release management |

---

## Week-by-week schedule and ordering

Hardware waits on shipping. The Order or receive column is what keeps a solo schedule from stalling.

| Week | Dates | Build | Order or receive | Guardrail |
|---|---|---|---|---|
| 0 | February (optional, overlaps Part A) | BM1 draft | Order: two NUCLEO-G474RE boards, CAN transceiver and inertial breakouts, USB-to-CAN adapter, chassis, sensored motor, two motor controllers (one spare), servo, 3S batteries, charger and fireproof bag, ExpressLRS transmitter and three receivers | Optional, but it removes about three weeks of waiting from March |
| 1 | March 1 to 7 | BM1 | Receive the week 0 order | Write the CAN database file before any firmware |
| 2 | March 8 to 14 | BM2 on the bench | | Wheels off the ground |
| 3 | March 15 to 21 | BM2 on the chassis, BM3 | Order: two global-shutter cameras, cones, AprilTag prints | Nothing touches the floor until the kill chain passes on the stand |
| 4 | March 22 to 28 | BM3 stop tests, BM4 schematic | | Review the schematic against a checklist with one outside reviewer |
| 5 | March 29 to April 4 | BM4 layout | | Test points on every rail and bus; module for the computer rail |
| 6 | April 5 to 11 | BM4 order, BM6 starts on the prototype | Order: revision A (5 boards, 3 assembled) | Order by Friday; fabrication plus shipping is 2 to 3 weeks |
| 7 | April 12 to 18 | BM6, BM7 intrinsics | Order: overhead camera and mount, router | |
| 8 | April 19 to 25 | BM7, BM8 layout | | Run the 3-hour inertial log overnight; lay the track out from the Gazebo track file |
| 9 | April 26 to May 2 | BM5 bring-up, BM8 race control | Receive revision A | Current-limited supply, one rail at a time |
| 10 | May 3 to 9 | BM5 port and telemetry, BM9 | | If revision A is late, run BM9 on the prototype car |
| 11 | May 10 to 16 | BM9, simulation update | Order: car 2 drivetrain (chassis, motor, controller, servo, batteries) | Rerun Part A's baseline in simulation with the measured parameters before driving it for real |
| 12 | May 17 to 23 | BM10 on overhead positions | | Speed cap set from measured stopping distance and runoff |
| 13 | May 24 to 30 | BM10 on onboard localization | | Freeze, tag, record demonstration three |
| 14 | May 31 to June 6 | BA1 revision B design, BA2 bake-off | Order: one AI HAT+ for the bake-off (your Pi 5 is the host) | Bake off before buying five computers |
| 15 | June 7 to 13 | BA3 | Order: revision B (8 boards), parts for cars 3 and 4 | Order the fleet in two batches |
| 16 | June 14 to 20 | BA4 | | Every firmware change goes through the bench from now on |
| 17 | June 21 to 27 | BA5 | | Other human drivers only after BA3 has passed |
| 18 | June 28 to July 4 | BA6 | Receive revision B | |
| 19 | July 5 to 11 | BA1 bring-up and production test, BA7 | | Production test before building more cars |
| 20 | July 12 to 18 | BA8 cars 3 and 4, BA9 round 1 | Order: parts for cars 5 and 6, if cars 3 and 4 pass inspection | |
| 21 | July 19 to 25 | BA7 across the fleet, BA8 dashboard | | |
| 22 | July 26 to August 1 | BA10 with 3 to 4 cars, BA9 round 2 | Receive parts for cars 5 and 6 | Test the red flag before the first multi-car race |
| 23 | August 2 to 8 | BA8 cars 5 and 6 | | Every car passes the end-of-line test and inspection |
| 24 | August 9 to 15 | BA10 with up to 6 cars | | Advanced checkpoint |
| 25 | August 16 to 22 | Documentation package, final safety case, videos | | Written deliverables first |
| 26 | August 23 to 29 | Race day with the robotics clubs; buffer | | The buffer is real: hardware slips |

> [!WARNING]
> **Defer order if behind schedule**
>
> Defer in this order: BS10 through BS1, then cars 5 and 6 (BA8 stops at four cars), BA9's second round, BA7 (flash with the debugger instead), BA6 (hand-label 200 real frames instead), then BA10 (stay with 1v1 races). A four-car fleet with a real safety case is a complete result. **Never defer BM3 or BA3.** Nothing moves without the kill chain, and nobody else drives, marshals, or stands trackside before the fault-injection campaign passes.

---

## Bill of materials and budget

Rough prices as of September 2026; recheck at order time. Memory prices pushed single-board computer prices up several times in 2025 and 2026.

### Per autonomous car

| Item | Rough cost | Notes |
|---|---|---|
| 1/10 rolling chassis, four-wheel drive (touring or rally) | $150 to $350 | Room for electronics matters more than top speed |
| Sensored brushless motor | $50 to $100 | Sensored for smooth low-speed control |
| VESC-class motor controller | $80 to $200 | CAN built in; clone quality varies, so keep a spare |
| Steering servo | $30 to $60 | Fast, metal gears |
| Two 3S LiPo batteries | $60 to $120 | VESC-class controllers usually need at least 3S (three cells in series) |
| Car computer | $195 to $285 | Jetson Orin Nano Super ($249 list), or a Raspberry Pi 5 plus AI HAT+ 26 TOPS (26 tera-operations per second, $110 list) |
| Two global-shutter cameras | $80 to $120 | Front and rear |
| Vehicle control board, assembled | $50 to $100 | Small-batch price per board |
| ExpressLRS receiver with servo-signal outputs | $13 to $25 | Failsafe on signal loss |
| USB-to-CAN adapter | $30 to $60 | Must be supported by SocketCAN |
| Wiring, connectors, mounts | $40 to $80 | 3D-printed mounts from the lab |
| **Per car** | **about $800 to $1,500** | |

### Shared track, race control, and tools

| Item | Rough cost | Notes |
|---|---|---|
| About 80 cones (blue, yellow, orange) | $60 to $120 | 15-centimeter mini cones |
| Soft barriers | $40 to $100 | Fast corners and the driver stand |
| Overhead camera or cameras, wide lens, mount | $100 to $300 | Two cameras if the track exceeds about 10 by 6 meters |
| Dedicated router | $80 to $150 | Keep the track network off campus Wi-Fi |
| ExpressLRS transmitter for the marshal | $60 to $150 | The only radio on the kill binding phrase |
| Multi-port LiPo charger and fireproof storage | $100 to $250 | Check lab rules first |
| Spares (controller, servo, board, motor) | $150 to $300 | One of each |
| Development boards, breakouts, logic analyzer, current-limited supply | $100 to $450 | Skip what the lab already has |

### Totals

| Scope | Rough total |
|---|---|
| Minimum viable product: car 1 on your Jetson, car 2 drivetrain for your Pi 5, revision A boards, track, tools | about $1,900 to $4,300 |
| Full fleet of six autonomous cars | about $5,500 to $10,500 |

Cost-down options: a human-driven variant without cameras or an accelerator (a Raspberry Pi Zero 2 W, whose price the memory increases did not touch, runs the DualSense driver) saves roughly $300 per human car; a fleet of four autonomous and two human cars; buy cars 5 and 6 only after the first multi-car race. Club budgets, independent study funds, and board-house student sponsorship programs are worth asking about once car 1 has a video.

---

## Skills Part B puts on your resume

### Embedded firmware

| Skill | Earned by |
|---|---|
| C++17 on STM32 with FreeRTOS: tasks, queues, priorities | BM2 |
| Interrupts, direct memory access, timers, Serial Peripheral Interface and Inter-Integrated Circuit (I2C) drivers | BM2, BM5 |
| CAN drivers, bus-off recovery, end-to-end protection | BM2, BA4 |
| State machines, watchdogs, controlled stop | BM2, BM3 |
| Bootloaders with two image slots and rollback | BA7 |
| Host unit testing (GoogleTest) and static analysis | BM2, BA4 |

### Electrical and board design

| Skill | Earned by |
|---|---|
| 4-layer schematic and layout in KiCad | BM4 |
| Power distribution, protection circuits, power budgeting | BM1, BM4 |
| CAN physical layer and mixed-signal layout | BM4 |
| Bring-up and debugging with JTAG, oscilloscopes, logic analyzers | BM5 |
| Board management: power sequencing, rail monitoring, telemetry | BM5 |
| Design for manufacturing and test, production test fixtures | BA1 |

### Vehicle networks and diagnostics

| Skill | Earned by |
|---|---|
| CAN database files, code generation, bus-load analysis | BM1, BM6 |
| Gateway architecture between autonomy and vehicle buses | BM1, BM2 |
| ISO-TP, Unified Diagnostic Services, diagnostic trouble codes | BA7 |
| Over-the-air firmware updates | BA7 |
| Authenticated CAN messages | BS5 |

### Safety, verification, and validation

| Skill | Earned by |
|---|---|
| Hazard analysis and risk assessment, safety goals | BM1 |
| Independent kill chain and fail-safe design | BM3 |
| Failure mode and effects analysis, fault injection, safety case | BA3 |
| Hardware-in-the-loop testing and firmware continuous integration | BA4 |
| Requirements traceability and inspection procedures | BM1, BA3 |

### Sensors, calibration, and timing

| Skill | Earned by |
|---|---|
| Camera intrinsics and extrinsics, camera-to-inertial calibration | BM7 |
| Inertial noise characterization (Allan variance) | BM7 |
| Clock synchronization and end-to-end latency measurement | BM6 |
| Ground-plane calibration and vision-based timing | BM8 |
| Hardware-triggered cameras | BS3 |

### Autonomy on real hardware

| Skill | Earned by |
|---|---|
| ros2_control hardware interfaces over SocketCAN | BM6 |
| Containers on embedded Linux | BM6 |
| Edge inference (TensorRT, Hailo) with 8-bit quantization | BA2 |
| System identification and sim-to-real transfer | BM9, BM10 |
| Offboard auto-labeling and real-data fine-tuning | BA6 |
| Real-data retraining and fleet deployment | BA9 |

### Fleet and operations

| Skill | Earned by |
|---|---|
| Production test and calibration management | BA1, BA8 |
| Fleet telemetry, alerting, battery operations | BA8 |
| Multi-car networking (Zenoh) and networked safety functions | BA10 |
| Race operations: marshal, red flag, technical inspection | BM8, BA3, BA10 |

---

## Where this lines up with autonomous vehicle job postings

Pulled from Waymo Hardware Engineering postings, checked in September 2026.

| What current Waymo Hardware Engineering postings ask for | Where Part B earns it |
|---|---|
| C++ on microcontrollers and on-vehicle software | BM2, BM6 |
| Board management: power sequencing, rail monitoring, fault handling, board-level telemetry | BM5, BA8 |
| Schematic review and debugging with JTAG, oscilloscopes, and logic analyzers | BM4, BM5 |
| Vehicle networks (CAN and CAN database files) and the embedded systems behind them | BM1, BM6 |
| Verification and validation plans, test execution, defect resolution | BA3, BA4 |
| Functional safety (ISO 26262) | BM3, BA3 |
| Fleet software delivery and fleet monitoring | BA7, BA8 |
| Vehicle diagnostics and security (Unified Diagnostic Services, secure onboard communication) | BA7, BS5 |
| Analyzing field logs and simulation results | BM9, BA9 |

No single project guarantees an offer. This one gives you evidence in the areas these teams interview on, at a scale you can demonstrate in a room.

---

## Course tie-ins

One robotics project per class works best when the classes feed the flagship.

| Course (if it falls in spring or summer 2027) | Point its project at |
|---|---|
| EAS 5407 Mechatronic Systems | BM2 and BM4: drive-by-wire firmware and the board |
| EEL 6667 Mobile Robotic Systems | BM7 and BM10: inertial calibration and localization on real sensors |
| CAP 6419 3D Computer Vision | BM7, BM8, BA6: calibration, overhead geometry, cross-view auto-labeling |
| EEL 6812 Neural Networks and Deep Learning | BA2 and BA6: edge deployment and real-data fine-tuning |
| CAP 5610 Machine Learning | BA9: the real-data flywheel |
| Independent study | All of Part B: this document, trimmed to the tiers and acceptance tests, is most of the proposal |

---

## Updates to Part A's gap list

Part A placed bare-metal C, interrupts, direct memory access, and real-time operating systems under roadmap Project 1. Part B pulls interrupts, direct memory access, a real-time operating system, and CAN into this project through the vehicle control board. Still elsewhere: custom motor control with current sensing and field-oriented control (Project 1); simultaneous localization and mapping, Nav2, and micro-ROS (Project 2); manipulation (Project 4).

---

---

## Feasibility notes

### Power

VESC-class controllers usually start at 3S; the FSESC 6.6, for example, lists 8 to 60 volts. A 3S pack spans roughly 9.0 to 12.6 volts and sags under hard acceleration. Plan four power domains: motor (battery straight to the motor controller, the highest current), servo (its own regulator, since servos draw spikes of several amps), computer (the largest steady load; check its input range against battery sag, and add hold-up capacitance or a separate pack if it browns out), and 3.3 volt logic (filtered from everything else). Protect the input with reverse-polarity protection, transient voltage suppression, a fuse per branch, and the power monitor.

**Kill sequence: brake first, cut drive power after the car stops.** Opening the battery line while the motor is still spinning lets the motor's generated voltage pump up the controller's supply rail.

### Stopping distance

Stopping distance is roughly speed times latency plus speed squared over twice the braking deceleration. At 4 meters per second, with 100 milliseconds of latency and 4 meters per second squared of braking, that is 0.4 plus 2.0, or 2.4 meters. Set each session's speed cap so this fits inside the runoff before the barriers, using braking measured in BM9.

### CAN design

Two buses, each terminated with 120 ohms at both ends (switchable on the board), on twisted pair routed away from motor phase wires. Estimate bus load in BM1 and measure it in BM2. The VESC uses 29-bit extended identifiers, with the command number in the upper bits and the controller identifier in the low byte; describe them in the database file so logs decode. Configure the motor controller's command timeout and timeout brake current, because they are part of the kill chain. Handle bus-off recovery and report error counters.

Start on a virtual bus on your laptop, then move to the adapter:

```bash
# Virtual bus for development without hardware
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0

# Real USB-to-CAN adapter at 1 megabit per second
sudo ip link set can0 type can bitrate 1000000
sudo ip link set up can0

# Watch traffic
candump vcan0

# Python tools used below
pip install python-can cantools
```

A starting CAN database file for the autonomy bus (save as `vehicle_autonomy_bus.dbc`):

```text
VERSION ""

NS_ :

BS_:

BU_: COMPUTER VEHICLE_BOARD

BO_ 256 DriveCommand: 8 COMPUTER
 SG_ SteeringAngle : 0|16@1- (0.0001,0) [-0.5|0.5] "rad" VEHICLE_BOARD
 SG_ SpeedTarget : 16|16@1- (0.001,0) [-10|10] "m/s" VEHICLE_BOARD
 SG_ Counter : 32|4@1+ (1,0) [0|15] "" VEHICLE_BOARD
 SG_ Checksum : 56|8@1+ (1,0) [0|255] "" VEHICLE_BOARD

BO_ 257 Heartbeat: 2 COMPUTER
 SG_ ComputerState : 0|8@1+ (1,0) [0|255] "" VEHICLE_BOARD
 SG_ Counter : 8|8@1+ (1,0) [0|255] "" VEHICLE_BOARD

BO_ 512 VehicleStatus: 8 VEHICLE_BOARD
 SG_ VehicleState : 0|4@1+ (1,0) [0|15] "" COMPUTER
 SG_ FaultActive : 4|1@1+ (1,0) [0|1] "" COMPUTER
 SG_ WheelSpeed : 8|16@1- (0.001,0) [-20|20] "m/s" COMPUTER
 SG_ BatteryVoltage : 24|16@1+ (0.001,0) [0|30] "V" COMPUTER
 SG_ Counter : 40|4@1+ (1,0) [0|15] "" COMPUTER
```

A bench sender that plays the car computer at 50 hertz, with a cyclic redundancy check (CRC-8, the SAE J1850 variant that AUTOSAR end-to-end protection uses) in the last byte of each drive command:

```python
"""Bench sender: plays the car computer on the autonomy bus at 50 hertz."""
import time

import can
import cantools

DB = cantools.database.load_file("vehicle_autonomy_bus.dbc")
DRIVE = DB.get_message_by_name("DriveCommand")
HEARTBEAT = DB.get_message_by_name("Heartbeat")


def crc8_sae_j1850(data: bytes) -> int:
    """Cyclic redundancy check, CRC-8 SAE J1850: polynomial 0x1D, initial 0xFF, final XOR 0xFF."""
    crc = 0xFF
    for byte in data:
        crc ^= byte
        for _ in range(8):
            crc = ((crc << 1) ^ 0x1D) & 0xFF if crc & 0x80 else (crc << 1) & 0xFF
    return crc ^ 0xFF


def drive_frame(steering_rad: float, speed_mps: float, counter: int) -> can.Message:
    payload = bytearray(DRIVE.encode({
        "SteeringAngle": steering_rad,
        "SpeedTarget": speed_mps,
        "Counter": counter % 16,
        "Checksum": 0,
    }))
    payload[7] = crc8_sae_j1850(bytes(payload[:7]))
    return can.Message(arbitration_id=DRIVE.frame_id, data=bytes(payload), is_extended_id=False)


def heartbeat_frame(counter: int) -> can.Message:
    payload = HEARTBEAT.encode({"ComputerState": 1, "Counter": counter % 256})
    return can.Message(arbitration_id=HEARTBEAT.frame_id, data=payload, is_extended_id=False)


def main() -> None:
    period_s = 0.02
    bus = can.Bus(channel="vcan0", interface="socketcan")
    counter = 0
    next_send = time.monotonic()
    try:
        while True:
            bus.send(drive_frame(steering_rad=0.10, speed_mps=1.0, counter=counter))
            bus.send(heartbeat_frame(counter))
            counter += 1
            next_send += period_s
            time.sleep(max(0.0, next_send - time.monotonic()))
    except KeyboardInterrupt:
        pass
    finally:
        bus.shutdown()


if __name__ == "__main__":
    main()
```

### Firmware structure

| Task | Rate | Priority | Does |
|---|---|---|---|
| Safety monitor | 1 kilohertz | Highest | Heartbeat and command timeouts, limits, vehicle state machine, watchdog check-in |
| Actuation | 100 hertz | High | Servo output and motor controller commands on the vehicle bus |
| Inertial | 400 hertz, interrupt plus direct memory access | High | Read, timestamp, publish |
| Telemetry | 10 to 50 hertz | Medium | Status, rails, temperatures, fault codes |
| Diagnostics and update | On request | Low | ISO-TP, Unified Diagnostic Services, bootloader handoff |

Rules: interrupts only timestamp and queue; no heap allocation after start-up; the independent watchdog is fed only when every task has checked in with the safety monitor.

### Car computer and ROS 2 distribution

JetPack 7.2 brings Ubuntu 24.04 to the Jetson Orin Nano Super, where Lyrical is a source build; much third-party Jetson software still targets JetPack 6, so build what you need against the local CUDA toolchain. TensorRT ships with JetPack, so the ONNX-to-engine path works natively. The AI HAT+ is supported on Raspberry Pi OS, so ROS 2 on that computer runs in a container. Build one container image per computer type with the same ROS 2 distribution inside both. If containers fight you, move the whole project, simulation included, to Jazzy, which is Tier 1 on Ubuntu 24.04 and supported until May 2029. After the 2025 and 2026 memory-driven price increases, a Pi 5 plus AI HAT+ costs about as much as a Jetson Orin Nano Super, so BA2 is a decision about software, power, and accuracy rather than price.

### Cameras

Global shutter avoids the skew a rolling shutter shows at speed. Short exposures fight motion blur (at 5 meters per second, a 5 millisecond exposure smears 2.5 centimeters) but expose flicker from light-emitting diode (LED) lighting at 120 hertz. Test under the real lights, and prefer flicker-free lighting over longer exposures.

### Clock synchronization

A two-way exchange over CAN: the computer sends a sync frame and records its send time, the board timestamps reception with the FDCAN peripheral's receive timestamp and replies, and the computer estimates offset and drift from many exchanges. Timestamp inertial samples on the board at the data-ready interrupt and convert them to computer time.

### ExpressLRS

Receivers sharing a binding phrase all follow one transmitter; if two transmitters share a phrase, which one a receiver follows is random, so the marshal's radio is the only one on the kill phrase. Turn telemetry off on all but one receiver. Configure failsafe so signal loss reads as a kill, and verify every car's failsafe on the bench before its first session.

### LiPo safety

Charge only in a fireproof container, attended. Store packs at storage voltage (about 3.8 volts per cell) between sessions, retire any puffed pack, and label every battery with a charge log. Ask about lab rules on charging before the first order.

### Board house practicalities

Fabrication, assembly, and shipping usually take 2 to 3 weeks, and import costs and lead times vary. Confirm stock at order time, and assemble only 3 of the 5 revision A boards.

---

## Risk register

| Risk | Severity | Response |
|---|---|---|
| The kill chain fails to stop the car | High | Three independent layers, 10 trials at three speeds per failure path, speed caps from measured stopping distance; BM3 and BA3 are never deferred |
| Someone is hit by a car | Low likelihood, severe | Marshal, barriers, speed caps, a spectator zone behind the driver stand, nobody inside the track while cars are armed |
| LiPo fire | Low likelihood, severe | Attended charging in fireproof containers, storage voltage, retire damaged packs, lab rules |
| Revision A has a fatal error | Medium | Dev-board prototype first, test points, checklist review; revision B is planned; rework wires are acceptable on revision A |
| Parts out of stock or late | Medium | In-stock parts with second sources, early orders, and the Nucleo prototype kept running so work continues |
| Brownouts reset the computer under hard acceleration | Medium | Power budget, scope the rails during launches, hold-up capacitance or a separate computer pack |
| Motor noise corrupts CAN or inertial data | Medium | Twisted pair, termination, layout separation, ferrites; error counters logged |
| Clone motor controllers fail | Medium | Keep a spare, buy from a reputable seller, log controller temperatures |
| ROS 2 distribution mismatch across computers | Medium | Containers per computer type; Jazzy fallback |
| Wi-Fi drops mid-race | Medium | Losing race control's heartbeat triggers a controlled stop; the hardware kill needs no Wi-Fi |
| Budget runs out before six cars | Medium | Two purchase batches; four cars is a complete result; the human-only variant; sponsorship |
| A summer internship takes the hours | Medium | Tiers stay, the calendar stretches; May 30 is the protected milestone |

---

## Resources that feed specific deliverables

| Resource | Feeds |
|---|---|
| STM32G4 reference manual (RM0440) and the FDCAN introduction application note (AN5348) | BM2, BM5 |
| Mastering the FreeRTOS Real Time Kernel (free book) | BM2 |
| Elecia White, Making Embedded Systems (second edition) | BM2, BM5 |
| Phil's Lab videos on KiCad and STM32 board design; Rick Hartley's talks on grounding and layout | BM4 |
| Texas Instruments, Introduction to the Controller Area Network (SLOA101) | BM1, BM4 |
| cantools, python-can, can-utils, and the Linux SocketCAN documentation | BM1, BM6, BA4 |
| VESC firmware and VESC Tool documentation (CAN commands, timeouts) | BM2, BM3 |
| ExpressLRS documentation (binding phrases, failsafe, receivers with servo outputs) | BM3 |
| ros2_control documentation (hardware interfaces) | BM6 |
| Kalibr (camera-to-inertial calibration) and allantools | BM7 |
| udsoncan and can-isotp (Python), ISO 14229 overviews | BA7 |
| Memfault's Interrupt blog (firmware updates, fault handling, firmware testing) | BA4, BA7, BS4 |
| ISO 26262, ISO 21448, and UL 4600 overviews; Waymo's published safety methodology reports | BM1, BA3 |
| Coursera, covered by your subscription: University of Colorado Boulder's Real-Time Embedded Systems and Embedding Sensors and Motors specializations | BM2, BM4, BM7 |
| RoboRacer build documentation, as the reference architecture you are departing from | BM1 |
| Kaufmann et al., champion-level drone racing (Nature, 2023), for residual dynamics | BS2 |
| AUTOSAR end-to-end protection and Secure Onboard Communication overviews | BM2, BS5 |
| Board-house design-for-manufacturing guidelines (JLCPCB, PCBWay) | BM4, BA1 |

---

The target is one car on its own sensors by May 30, and a six-car race day by the end of August.


The target is portfolio-ready at the checkpoint, and a real six-car race at the end.
