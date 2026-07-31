# IRobots

A robotic arm you talk to instead of program.

Most industrial arms are taught one job and then repeat it forever. Changing that job means someone rewrites the motion sequence, teaches every waypoint again, and takes the cell offline for a day. We think that is the wrong shape for the way small factories, labs, and workshops actually operate, where the task changes every week and nobody has a robotics engineer sitting around.

IRobots is our attempt at a general controller that sits on top of any arm. You describe the job in plain language. A planning model breaks it into steps. A worker model turns each step into arm movements, watches what happens through a camera, and adjusts when reality does not match the plan. Swap the description, and the same hardware does a completely different job with no new code.

This repository holds the work for our hackathon submission: firmware, mechanical design, vision pipeline, and the agent layer.

## Why we think this matters

The pitch is not "an arm that sorts blocks". Block sorting is just what fits on a table in a demo hall. The pitch is that the gap between "I know what I want done" and "the robot is doing it" collapses from days to a sentence.

Three places that gap actually hurts:

* **Adaptive assembly.** Short production runs and custom orders never justify the cost of reprogramming a cell. A prompt does.
* **Places people should not be.** Space, deep sea, contaminated sites. You cannot send a technician out to teach waypoints again, and the round trip latency makes teleoperation painful. An arm that can plan and correct itself locally is worth a lot.
* **Robotics without robotics engineers.** A biology lab, a repair shop, a school. The hardware is affordable now. The programming is what stops people.

## What makes our approach different

* **Two model architecture.** A planner reasons about the goal and produces an ordered list of steps. A worker executes one step at a time and reports back what it observed. Splitting them keeps the reasoning model out of the tight control loop, which is where latency and hallucination hurt most.
* **The model never invents coordinates.** Object positions come from the vision stack as real numbers. The language model chooses which object and what to do with it, and nothing else. This one rule kills the most common failure mode in language driven robotics.
* **Portable across bodies.** Nothing in the agent layer assumes a specific arm. Kinematics live behind an interface, so a different arm means a new solver, not a new brain.
* **It learns from the run.** State and outcomes are fed back to the planner, so a failed grasp becomes information rather than a stuck loop.
* **No action model required.** We get task generality out of prompting and tool calling rather than a fine tuned vision language action model, which means no training run and no dataset to collect.

## Where the project stands right now

We would rather tell you what actually runs than paint a picture we cannot demo. Current state:

| Piece | Status | Notes |
|---|---|---|
| ESP32 camera node | Working | Streams MJPEG and stills over Wi Fi, tuned for the AI Thinker board |
| Arm mechanical design | Working | SolidWorks assembly under `robotic arm schematics/` |
| Inverse kinematics and servo control | In progress | Target is on device solving with software joint limits |
| Vision and coordinate mapping | In progress | Detection plus a homography from pixels to table centimetres |
| Planner and worker agents | In progress | Prompt design and tool schema |
| Operator interface | Planned | Terminal interface first, browser interface after |

The camera node is the part you can flash and see working in about ten minutes. Instructions are below.

## How the system fits together

```
                 operator types a task
                          │
                          ▼
             ┌────────────────────────┐
             │     planner model      │   turns the task into ordered steps
             └────────────────────────┘
                          │
   camera frames          ▼
        │    ┌────────────────────────┐
        └───▶│    perception layer    │   detection, then pixels to centimetres
             └────────────────────────┘
                          │
                          ▼
             ┌────────────────────────┐
             │      worker model      │   picks the target, emits one tool call
             └────────────────────────┘
                          │
                          ▼   JSON over Wi Fi
             ┌────────────────────────┐
             │     arm controller     │   inverse kinematics, joint limits, motion
             └────────────────────────┘
                          │
                          ▼
                   the arm moves, and
                 the outcome is reported
                     back to the planner
```

The loop matters more than any single box. The worker acts, the camera sees the outcome, and the planner decides whether the step succeeded or needs another attempt. A longer walkthrough lives in [docs/architecture.md](docs/architecture.md).

## Repository layout

```
docs/                          project documentation
pics/                          photos and captures for the writeup
presentation/                  slides and demo material
robotic arm schematics/        SolidWorks assembly for the arm
src/
  micro_controller/
    esp32/                     arm side firmware
    ardunio/                   auxiliary board sketches
  vision-AI/
    vison/                     ESP32 camera web server firmware
    config/                    calibration and runtime configuration
    simulation/                offline testing without hardware
```

Directories holding a lone `temp.txt` are placeholders for work in flight. They exist so the structure is settled before the code lands.

## Running the camera node

The ESP32 camera node is the piece that runs today. Full detail is in [docs/camera_server.md](docs/camera_server.md); the short version:

1. Install the Arduino IDE and add the `esp32` board package from Espressif.
2. Open `src/vision-AI/vison/CameraWebServer.ino`.
3. Copy `camera_index.h` from the Arduino CameraWebServer example into the same folder. It is deliberately kept out of version control because it is a large generated blob, and the sketch will not compile without it.
4. Put your network name and password at the top of the sketch.
5. Select board **AI Thinker ESP32 CAM**, set PSRAM to enabled, and set the partition scheme to **Custom** so the `partitions.csv` in the sketch folder is used.
6. Tie GPIO0 to ground, press reset, and upload. Remove the jumper and press reset again when it finishes.
7. Open the serial monitor at 115200 baud and read off the address the board prints.

Once it is up, put that address in a shell variable and pull a frame:

```bash
export BOARD_IP=192.168.1.50
```

```bash
curl http://$BOARD_IP/capture --output frame.jpg
```

The browser interface is at `http://$BOARD_IP/`, and the raw MJPEG stream the vision pipeline consumes is at `http://$BOARD_IP:81/stream`.

## Hardware

| Part | What we used | Count |
|---|---|---|
| Arm | Five joint design, SG90 class servos | 1 |
| Camera board | ESP32 CAM, AI Thinker layout with OV2640 | 1 |
| Arm controller | ESP32 development board | 1 |
| Servo supply | 5V at 2A or better, separate from the boards | 1 |
| USB serial adapter | For flashing the camera board | 1 |
| Workspace | Flat surface with even lighting and four fixed calibration markers | 1 |

One thing worth repeating because it has bitten us: do not run the servos off the ESP32 regulator. Give them their own supply and tie the grounds together. Wiring notes and the rest of the setup are in [docs/hardware.md](docs/hardware.md).

## Documentation

* [docs/README.md](docs/README.md) is the index.
* [docs/architecture.md](docs/architecture.md) covers the design and the reasoning behind it.
* [docs/hardware.md](docs/hardware.md) covers the build, wiring, and calibration.
* [docs/camera_server.md](docs/camera_server.md) covers the camera firmware and its HTTP interface.
* [docs/roadmap.md](docs/roadmap.md) covers what we are building next and in what order.
* [Plan, TODO, Idea, Execution overview](<docs/Plan - TODO - Idea -  Execution overview.md>) is the original working notes, kept as written.

## Team

Built by **IRobots**. The work is split across mechanical design, firmware, the vision and agent pipeline, and the demo itself, though at hackathon hours everyone ends up touching everything.

## License

Released under the MIT License. See [LICENSE](LICENSE).

The camera firmware under `src/vision-AI/vison/` derives from Espressif's CameraWebServer example and stays under Apache 2.0. Attribution is in [NOTICE](NOTICE).
