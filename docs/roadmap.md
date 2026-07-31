# Roadmap

Where the project is, what comes next, and what we would build with more time. The original unedited planning notes are in [Plan, TODO, Idea, Execution overview](<Plan - TODO - Idea -  Execution overview.md>); this is the tidied version.

## Done

* **Camera node firmware.** ESP32 with an OV2640 serving stills and MJPEG over Wi Fi, tuned for the AI Thinker board. Clock lowered for stability, brownout workaround in place, custom partition table for the larger application. Flashable and running today. See [camera_server.md](camera_server.md).
* **Mechanical design.** Five joint arm, SolidWorks assembly in `robotic arm schematics/`.
* **System design.** The layer split, the network boundaries, and the rule about coordinates, all settled and written down in [architecture.md](architecture.md).
* **Repository structure.** Directories agreed and in place ahead of the code, so parallel work does not collide.

## In progress

**Arm control firmware.** Inverse kinematics solved on device, software joint limits, motion interpolation so the servos move rather than snap, and a watchdog that puts the arm in a safe pose when commands stop arriving. This is the piece that turns a coordinate into a movement and everything downstream waits on it.

**Perception.** Object detection producing pixel boxes, plus the homography from pixels to table centimetres. Calibration data lives in `src/vision-AI/config/`.

**The agent layer.** Planner and worker prompts, the tool schema between them, and the loop that feeds observed outcomes back to the planner. Most of the remaining design risk is here, which is why the coordinate rule exists.

**Operator interface.** A terminal interface first: type a task, watch the plan, watch each step execute. Fast to build and it demonstrates well.

## Next

* **Simulation.** Run the full pipeline against recorded frames with no arm attached. Worth building for its own sake, since it lets the agent work continue while the hardware is occupied, and it is a working fallback if the arm fails during a demo.
* **Browser interface.** The stream, the current plan, and the arm state on one page.
* **Multiple tasks in one session.** Change the task without restarting anything. This is the demo that makes the point.
* **Better state memory.** The planner should remember what it already tried in this session, not just the last step's outcome.

## Later

Ideas we believe in but have not committed to a version.

**Perception**

* A second camera perpendicular to the first, so depth is measured rather than inferred from object class.
* A camera on the gripper for close range approach.
* An ultrasonic sensor above the gripper as a cheap distance check before contact.
* Open vocabulary detection, so the system can find objects it was never trained on and the operator is not limited to a fixed class list.

**Intelligence**

* Tool calling beyond arm control, including search, so the planner can look up how to handle something unfamiliar.
* Retrieval over documentation about our own system, so the agent can reason about its reach, its limits, and its failure modes.
* Learned skills persisted across sessions rather than reasoned from scratch every run.
* Speech input, so the operator talks instead of types.

**Platform**

* Port the pipeline to ROS 2, which is the honest answer to "how would this work on a real arm".
* Multiple arms coordinating on one task.
* A mobile base, so the arm can reach beyond a fixed workspace.
* Joystick override, for the moments when a human should just take the controls.

**Presentation**

* Status display on the arm itself, an LED matrix showing what it is thinking about.
* Visual indicators in both interfaces for what the system is currently doing and why.

## What we would fix first with another week

Not new features. These four:

1. **Depth.** A single overhead camera gives us a plane and a guess. A second view removes the guess and would improve grasp reliability more than any other single change.
2. **Calibration drift detection.** The system should notice that its own calibration has gone stale instead of silently missing by a constant offset.
3. **Grasp verification.** Right now a failed grasp is detected by looking at the next frame. A sensor in the gripper would catch it immediately and cleanly.
4. **Network resilience.** The watchdog makes dropouts safe. It does not make them smooth. Command buffering on the arm side would.
