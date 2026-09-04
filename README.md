# Mixed-Reality-Perception-Pipeline-with-Simulator-Generated-Training-Data
A Computer Vision centric project training a robot inside a simulator how to navigate new environments by processing data from external environments.

**Important**: Below is the overall inspiration and guideline deliverables I am shipping in roughly 3-months.
 - I will be listing all relevant and required skills in bullet point format to successfully finish this project

## 1. Summary

This project builds a computer vision pipeline in which a detector is trained entirely on labeled data that a physics simulator generates automatically, then deployed on a real camera and evaluated against physical ground truth. A calibrated laptop webcam observes a physical tabletop scene containing known objects. Detections from that live feed are localized in metric coordinates and mirrored into a simulated environment, where a virtual mobile robot plans around them.

The central research question is quantitative: **how large is the accuracy gap between simulated and real imagery, and which domain randomization parameters reduce it?** The project answers this with a measured ablation study rather than a qualitative demonstration.

---

## 2. Motivation and relevance to course topics

Manual annotation is the dominant cost in supervised detection. Simulators eliminate it, because the renderer already knows every object's three-dimensional pose and the camera's intrinsic parameters, so two-dimensional bounding boxes follow from projection rather than from human labeling. The open problem is that models trained on synthetic imagery degrade on real imagery, a phenomenon known as the domain gap.

The project exercises the following course topics directly:

| Course topic | Where it appears |
|---|---|
| Image formation and the pinhole camera model | Simulated and real camera configuration |
| Camera calibration, intrinsic parameters, lens distortion | Deliverable C2 |
| Homogeneous coordinates and projective geometry | Deliverable C3, projecting three-dimensional poses to image space |
| Planar homography estimation | Deliverable C7, mapping image points to tabletop coordinates |
| Fiducial marker pose estimation | Deliverable C7, AprilTag world reference |
| Object detection and evaluation metrics | Deliverables C5, E2 |
| Dataset construction and experimental design | Deliverables C4, E3 |
| Quantitative error analysis | Deliverables C2, C7, E2, E3 |

---

## 3. System overview

Four subsystems:

1. **Simulation and data generation.** A Gazebo world containing a differential-drive robot, a camera, and four to six object classes. A generator script randomizes lighting, object placement, camera pose, and surface textures, capturing an image and its ground-truth annotation at each iteration.
2. **Detection.** A YOLO-family detector fine-tuned on the synthetic dataset, wrapped as a Robot Operating System 2 node that accepts either the simulated camera stream or the real webcam stream without code modification.
3. **Real-world localization.** A printed AprilTag board fixed to the desk establishes the world coordinate frame. Camera pose is solved from the tags. A ground-plane homography converts each detection's bottom-center pixel to a metric position on the tabletop.
4. **Scene bridge.** Detections in tabletop coordinates are transformed into the simulated world frame, where corresponding object models are spawned, moved, or removed with temporal filtering to suppress noise.

---

## 4. Core deliverables (required for project completion)

| Identifier | Deliverable | Acceptance criterion |
|---|---|---|
| C1 | Simulated environment: Gazebo world with a differential-drive robot, RGB-D camera, and four to six object classes corresponding to physical objects available for testing | A single launch command starts the world, robot, transform tree, and camera topics |
| C2 | Camera calibration: OpenCV checkerboard calibration of the physical webcam; the simulated camera configured with matching intrinsic parameters | Reprojection error below 0.5 pixels, reported with the full intrinsic matrix and distortion coefficients |
| C3 | Automatic annotation: per-frame extraction of object poses from the simulator, projection of three-dimensional bounding boxes through the camera model, and export in a standard detection label format | Projected boxes agree with simulator segmentation masks at intersection over union above 0.85 on a fifty-frame validation sample |
| C4 | Domain randomization generator: configurable randomization of lighting, object placement, camera pose, and textures, driven by a documented configuration schema | At least 2,000 annotated frames generated without manual intervention |
| C5 | Detector training: YOLO fine-tuning on the synthetic dataset, evaluated on a held-out synthetic split | Mean average precision at intersection over union 0.5 reported, with training curves |
| C6 | Dual-source inference node: a single detection node consuming either the simulated camera topic or the webcam topic through topic remapping | Identical code path verified on both sources; frame rate and per-frame latency logged for each |
| C7 | Metric localization: AprilTag-derived world frame plus ground-plane homography mapping detections to tabletop coordinates in meters | Position error below 3 centimeters across ten measured object placements, reported as mean and standard deviation |
| C8 | Reproducibility: documented repository with architecture description, single-command launch for both modes, and continuous integration running unit tests on the projection, homography, and configuration components | Continuous integration passing on the main branch |

**Core deliverable outcome:** a working pipeline that trains without human annotation and localizes real objects to measured accuracy.

---

## 5. Extended deliverables (planned, subject to schedule)

| Identifier | Deliverable | Acceptance criterion |
|---|---|---|
| E1 | Scene bridge: real-time mirroring of webcam detections into the simulated world with confidence thresholding and temporal debounce | Physical object movement reflected in simulation within 500 milliseconds; no spurious appearance or disappearance across a two-minute continuous recording |
| E2 | Sim-to-real evaluation: 150 to 200 hand-annotated real webcam frames used as a held-out real test set | Synthetic accuracy, real accuracy, and the resulting domain gap reported in a table |
| E3 | Domain randomization ablation: lighting, texture, and camera-pose randomization toggled independently; dataset regenerated and detector retrained for each configuration | At least four experimental conditions with the resulting domain gap for each, plus a written analysis of which factors mattered |
| E4 | Navigation integration: detected objects inserted into the simulated robot's occupancy representation so that it replans around physical objects placed in its path | Success rate across ten trials, with video documentation |
| E5 | Teleoperation interface: a userspace human interface device driver for a game controller, publishing standard joystick messages, with command arbitration between manual and autonomous control and a timeout failsafe | Documented input report format, unit tests on the parser, polling rate and timing jitter measured; manual control preempts autonomous control without restarting the system |
| E6 | Demonstration data collection: controller-triggered recording of synchronized image, detection, transform, and command streams | Recorded sessions replay successfully as an automated regression test |
| E7 | Inference benchmarking: floating point 32-bit, floating point 16-bit, and integer 8-bit quantized model variants compared for accuracy, latency, and model size | Complete comparison table across all three precisions |

**Justification for E5 and E6.** These are included because the experimental design benefits from a human control baseline and from a mechanism for capturing difficult real-world cases. The teleoperation path lets a human operator drive the robot through the same courses the autonomous planner attempts, providing a comparison baseline, and the recording hooks capture the frames where detection fails, which are the frames worth annotating. The driver is implemented at the user level rather than using an existing package so that input timing can be measured and controlled. Neither deliverable affects the vision pipeline; if the schedule tightens, both can be omitted without invalidating the core results.

---

## 6. Optional extensions (undertaken only if the schedule permits)

| Identifier | Extension |
|---|---|
| O1 | Behavior cloning: a policy trained on recorded human demonstrations, compared against the planner on identical courses |
| O2 | Semantic segmentation head trained on simulator masks and applied to real imagery |
| O3 | Haptic and visual operator feedback driven by obstacle proximity and system state |
| O4 | Public write-up of the ablation findings |

---

## 7. Experimental methodology

The project produces four measured results:

1. **Calibration accuracy.** Reprojection error and comparison of estimated intrinsic parameters against the simulator's configured values, which serve as ground truth for the synthetic case.
2. **Annotation correctness.** Intersection over union between geometrically projected boxes and simulator segmentation masks, verifying that the automatic labels are correct before any model is trained on them.
3. **Localization accuracy.** Metric position error against physically measured object placements, reported with mean and standard deviation, with a discussion of contributing error sources (calibration residual, marker pose estimate, detection box variance, object height violating the ground-plane assumption).
4. **Domain gap and its reduction.** Detection accuracy on synthetic and real test sets, and the change in that gap across randomization configurations.

All measurements are recorded in the repository with the scripts that produced them.

---

## 8. Schedule

| Weeks | Work | Milestone |
|---|---|---|
| 1 to 2 | C1, C2, C3 | Calibration complete and annotation correctness verified |
| 3 to 4 | C4, C5, C6 | First detector trained and running on both sources |
| 5 to 6 | C7, C8 | Core deliverables complete, tagged release, first demonstration recorded |
| 7 | E1 | Scene bridge operational |
| 8 | E2, E4 | Real test set annotated, domain gap measured, navigation integrated |
| 9 | E5 | Teleoperation interface |
| 10 | E3, E6 | Ablation study, demonstration recording |
| 11 | E7 | Benchmarking, second demonstration recorded |
| 12 | Reserve | Buffer for overruns |
| 13 | Report, slides, final video | Submission |

---

## 9. Risk management

| Risk | Mitigation |
|---|---|
| Development environment friction with camera or peripheral access | Resolved in week 1; a native Linux installation is the fallback if virtualization proves unreliable |
| Automatic annotation proves geometrically incorrect | Verified against segmentation masks in week 2, before any training time is invested |
| Scene bridge instability from noisy detections | Temporal filtering and confidence thresholding are part of the deliverable specification, with a two-minute soak test as the acceptance criterion |
| Schedule overrun | Core deliverables are completed and tagged by week 6 and are independently presentable. Extended deliverables are ordered so that the most scientifically valuable (E1, E2, E3) precede the rest, and optional extensions are omitted first |

---

## 10. Submission artifacts

1. Written report covering method, experiments, results, and analysis
2. Public repository containing all code, configuration, tests, and the scripts that generated every reported number
3. Recorded demonstration video
4. Presentation slides
5. Results appendix: calibration report, annotation verification, localization error analysis, domain gap table, ablation table, and inference benchmarks

---

## 11. Requested feedback
**Important**: Debating and questioning anything I am unsure of goes here.
1. Whether the core deliverables represent appropriate scope for the course project.
2. Whether the ablation study in E3 is the right emphasis, or whether depth in another direction would be more valuable.
3. Whether the teleoperation deliverables (E5, E6) are acceptable as supporting infrastructure or should be omitted as outside the course scope.
4. Any preference regarding report format, length, or presentation requirements.
