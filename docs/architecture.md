# Architecture

This document explains how IRobots is put together and, more usefully, why. Several of these choices look strange in isolation and only make sense once you know which failure we were designing around.

## The problem we are solving

A conventional arm executes a fixed program. Changing the job means rewriting that program. We want the job to be a sentence, and we want the arm to work out the rest.

The obvious approach is to hand a language model a camera feed and let it drive the servos directly. We tried reasoning through that and it falls apart in two specific ways. First, models are confidently wrong about spatial coordinates; ask one where an object is in centimetres and it will produce a plausible number that is off by a decimeter. Second, a model in the control loop is far too slow for anything resembling smooth motion.

So the architecture is mostly a set of walls built to contain those two problems.

## The layers

### Perception

A camera looks at the workspace from a fixed position. Object detection produces bounding boxes in pixel space. A homography, computed once during calibration from four known markers, converts pixel coordinates into centimetres relative to the arm base.

This layer is deterministic. Same frame in, same numbers out, every time. That property is what lets us trust it downstream.

### Planning

The planner receives the operator's task and a description of the current scene, and returns an ordered list of steps. It does not touch hardware and does not run per frame. It runs when the task is given and again when a step fails or the scene changes in a way that invalidates the plan.

Keeping the planner slow and occasional is deliberate. Reasoning is expensive, and most of the time nothing has changed enough to justify it.

### Execution

The worker takes one step from the plan plus the current detections and produces a single tool call. Something like: pick up the object identified as the red block, using the coordinates perception already provided.

The critical detail is the shape of that call. The worker chooses an object by identity, not by position. The coordinates attached to the call are copied verbatim from what perception measured. The model is selecting from a list, which models are good at, rather than estimating geometry, which they are bad at.

### Control

The arm controller receives a target in centimetres over the network, runs inverse kinematics to get joint angles, clamps every angle to a mechanically safe range, and interpolates the motion so the servos do not snap. It also holds a watchdog: if commands stop arriving, the arm settles into a safe pose rather than freezing mid reach.

The controller trusts nothing it receives. Out of range coordinates are rejected, not attempted. This is the last line of defence between a confused model and stripped servo gears.

## Why two models instead of one

Splitting planning from execution buys three things.

**Latency where it matters.** The worker runs a small, fast prompt. The planner, which is slow, runs rarely. A single model doing both would pay planning cost on every action.

**Failure isolation.** When the worker cannot complete a step it reports why, and the planner adapts. One model doing everything tends to either give up or loop, because it has no external perspective on its own output.

**Auditability.** We can read the plan before the arm moves. During a demo that is the difference between a confident explanation and a shrug.

## The rule about coordinates

Stated plainly, because it is the single most important design decision here:

> The language model never produces a spatial coordinate. It receives coordinates from perception and it selects among them.

Every number that reaches a servo can be traced back to a measurement. When the arm misses, the cause is calibration or kinematics, both of which are debuggable. If we let the model estimate positions, every miss would be unattributable, and on a hackathon clock that is fatal.

## Data flow through one action

1. The camera node serves a frame over Wi Fi.
2. Detection returns boxes in pixel space.
3. The homography converts box centres to centimetres from the arm base.
4. The worker receives the current step and the detection list, and emits a tool call naming a target.
5. The host attaches the measured coordinates and a height derived from the object class, then sends JSON to the arm controller.
6. The controller solves for joint angles, clamps them, and interpolates the movement.
7. The next frame shows the outcome. If the object did not move as expected, the planner is told and revises.

Step seven is the whole point. Without it this is a slow, expensive way to run a fixed program.

## Where the network boundaries sit

The camera node is a standalone board on Wi Fi serving HTTP. The arm controller is a separate board accepting JSON commands. The host machine runs perception, planning, and execution, and is the only component that talks to both.

Splitting camera from arm cost us a board but bought independence: the camera can be repositioned or upgraded without touching arm firmware, and the arm can be driven from a recorded feed or a simulator when the camera is not available. During development that has been worth more than the board cost.

## Known weak points

We would rather name these than have someone find them.

* **Calibration is fragile.** Bump the camera and the homography is wrong, silently. Positions stay plausible while being consistently off. Recalibrating before a session is not optional.
* **Detection is the ceiling on reliability.** If perception cannot see an object, no amount of good reasoning recovers. Lighting matters more than anything else in the system.
* **Depth is inferred, not measured.** A single overhead camera gives us a plane. Object height comes from the object class rather than a sensor. Two cameras or a distance sensor on the gripper would fix this and both are on the roadmap.
* **Wi Fi is a shared, hostile medium.** Conference halls are the worst case. The watchdog handles dropouts safely, but safely is not the same as smoothly.
