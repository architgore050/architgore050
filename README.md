# IRobots-Devansh_Gaur
Here's a quick and crisp README file for your hackathon project. It's designed to be immediately understandable, showcase the core innovation, and be demo-ready.

---

# 🏭 FactoryReconfig AI: Prompt-Driven Robot Arm

> **Zero-shot factory reconfiguration through natural language prompting**

[![Hackathon](https://img.shields.io/badge/Hackathon-48%20Hours-blue)](https://example.com)
[![Python](https://img.shields.io/badge/Python-3.10+-green)](https://python.org)
[![License](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)

---

## 🎯 What This Is

An **agentic robotic arm** that completely changes its behavior just by changing the system prompt. No code changes. No fine-tuning. Just natural language.

**The demo**: "Sort red blocks to the left, blue to the right." → Robot sorts.  
**Switch prompt**: "Stack objects in a tower, smallest on top." → Robot stacks.

Same robot. Same code. Different factories. This is the future of manufacturing flexibility.

---

## 🧠 The Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR LAPTOP (AI)                        │
│                                                                 │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐   │
│   │  PERCEPTION  │    │  REASONING  │    │    EXECUTION    │   │
│   │             │    │             │    │                 │   │
│   │  YOLOv8    ──┼───→│  LLM (12B)  │    │  Coordinate    │   │
│   │  OpenCV     │    │   + Vision  │    │   Mapping      │   │
│   │             │    │             │    │                 │   │
│   └─────────────┘    └─────────────┘    └────────┬────────┘   │
│         ↑                   ↑                     │            │
│         │                   │                     │            │
└─────────┼───────────────────┼─────────────────────┼────────────┘
          │                   │                     │
          │                   │                     │ JSON over WiFi
          │                   │                     │
     ┌────┴────┐         ┌────┴────┐          ┌────┴────┐
     │         │         │         │          │         │
     │ Camera  │         │ Prompt  │          │  ESP32  │
     │ (Top-   │         │ (User)  │          │  + 5×   │
     │  down)  │         │         │          │ Servos  │
     │         │         │         │          │         │
     └─────────┘         └─────────┘          └─────────┘
```

**The magic**: The LLM never guesses coordinates. YOLO+OpenCV provides exact math. The LLM only reasons about *what* to do based on the prompt.

---

## 📦 Tech Stack

| Layer | Technology | Why |
|-------|------------|-----|
| **Perception** | YOLOv8 + OpenCV | Fast, accurate object detection with pixel-perfect coordinates |
| **Spatial Mapping** | OpenCV Homography | Maps camera pixels to real-world centimeters |
| **Reasoning** | Gemma 4 12B (or Gemini 1.5 Flash via API) | SOTA VLM for visual reasoning and tool calling |
| **Control** | Inverse Kinematics (Custom) | Converts (X,Y,Z) to 5 servo angles |
| **Communication** | WebSocket over Wi-Fi | Real-time, reliable JSON command transport |
| **Hardware** | ESP32 + 5× SG90 Servos | Cheap, accessible, and works out of the box |

---

## 🚀 Quick Start

### 1. Clone & Install

```bash
git clone https://github.com/your-team/factory-reconfig-ai.git
cd factory-reconfig-ai
pip install -r requirements.txt
```

### 2. Camera Calibration (5 minutes)

```bash
python calibrate.py
# Place 4 markers on the table
# Measure their physical positions (cm)
# Enter pixel coordinates when prompted
# → Calibration complete!
```

### 3. Run the AI

```bash
# Option A: Local model (requires 16GB VRAM)
ollama pull gemma-4-vl:12b
python run.py --model gemma

# Option B: Cloud API (Gemini 1.5 Flash)
export GEMINI_API_KEY="your-key"
python run.py --model gemini
```

### 4. Control the Arm

```bash
python run.py --prompt "Sort red blocks to the left bin"
# The robot starts executing autonomously
```

---

## 🎮 How It Works (In 30 Seconds)

1. **Camera captures** the workspace
2. **YOLOv8 detects** objects and outputs pixel coordinates
3. **OpenCV converts** pixels → centimeters relative to robot base
4. **LLM receives**: "Detected red block at X:15.2, Y:22.1. System prompt: [your factory instruction]"
5. **LLM reasons** and outputs structured JSON tool call
6. **WebSocket** sends JSON to ESP32
7. **Inverse Kinematics** calculates 5 servo angles
8. **Servos move**, completing the task

---

## 🧪 Demo Scenarios (Switch Prompts. Watch It Adapt.)

### Scenario 1: Sorting Factory
```
System Prompt: "Sort red objects to the left bin and blue objects to the right bin."
```
Robot identifies colors and sorts accordingly.

### Scenario 2: Stacking Factory
```
System Prompt: "Stack all objects in a tower, smallest on top."
```
Robot changes behavior immediately. No code changes.

### Scenario 3: Inspection Factory
```
System Prompt: "Push any cylindrical objects off the table."
```
Robot now acts as a QC inspector.

---

## 🔧 Configuration

### Environment Variables

```bash
# For cloud-based reasoning
GEMINI_API_KEY="your-api-key"
OPENAI_API_KEY="your-api-key"

# For Wi-Fi communication
ESP32_IP="192.168.1.100"
ESP32_PORT="8765"

# For camera
CAMERA_ID="0"
CAMERA_WIDTH="640"
CAMERA_HEIGHT="480"
```

### Calibration (Critical!)

The 4-point calibration maps camera pixels to physical space:

```python
# Your calibration data (example)
MARKERS = {
    "top-left": (10.0, 10.0),    # cm from robot base
    "top-right": (10.0, 30.0),
    "bottom-left": (30.0, 10.0),
    "bottom-right": (30.0, 30.0)
}
```

**Why this matters**: Without calibration, the robot will miss objects by 5-15cm.

---

## 📁 Project Structure

```
factory-reconfig-ai/
├── src/
│   ├── perception/
│   │   ├── detector.py      # YOLOv8 object detection
│   │   └── calibrate.py     # OpenCV homography
│   ├── reasoning/
│   │   ├── llm.py           # LLM interface (local/cloud)
│   │   └── tools.py         # Tool definitions for function calling
│   ├── execution/
│   │   ├── websocket.py     # ESP32 communication
│   │   └── ik.py            # Inverse Kinematics (if not on ESP32)
│   └── run.py               # Main pipeline orchestrator
├── hardware/
│   └── esp32_firmware/
│       ├── webSocketServer.ino
│       └── ik_solver.ino
├── config/
│   └── calibration.json     # Saved camera calibration
├── prompts/
│   └── system_prompts.txt   # Pre-made factory prompts
├── requirements.txt
├── README.md
└── demo.sh                  # One-command demo script
```

---

## 🛠️ Hardware Requirements

| Component | Specification | Quantity |
|-----------|--------------|----------|
| Robot Arm | 5-DOF with SG90 servos | 1 |
| ESP32 | Any variant with WiFi | 1 |
| Camera | USB webcam (1080p) | 1 |
| Power Supply | 5V 2A (for servos) | 1 |
| Laptop | 16GB+ VRAM (for local) | 1 |
| Markers | 4 distinct colored objects | 4 |

---

## ⚠️ Critical Gotchas (Read This Before Demo)

### 1. **Power the servos separately!**
- ESP32 → USB power
- Servos → Separate 5V 2A supply
- Connect GNDs together

### 2. **Calibrate before every demo session**
- If the camera moves 1mm, the calibration breaks
- Run `calibrate.py` first thing

### 3. **Wi-Fi can fail**
- Implemented watchdog timer on ESP32
- If no command for 250ms, arm goes to safe state

### 4. **Coordinate hallucination**
- LLM never guesses coordinates
- YOLO+OpenCV provides exact math
- LLM only reasons about *which* object to target

### 5. **Servos can strip gears**
- Software limits prevent over-rotation
- ESP32 code clamps servo angles

---

## 🏆 Presenting the Demo

### The Story Arc

1. **"Look at this factory"** - Show the workspace, objects, arm
2. **"Here's the system prompt"** - Show Prompt 1 on screen
3. **"Watch it work"** - Robot executes task
4. **"Now the factory redesigns"** - Change system prompt (visibly!)
5. **"Watch it work differently"** - Robot executes new task
6. **"No code changed. No fine-tuning. Just a new prompt."** - The mic drop

### Presentation Tips

- **Show the prompt** on a large screen during the demo
- **Type the new prompt** live so judges see the change
- **Use a timer** to show the prompt-to-action latency
- **Have a backup video** of the simulation if Wi-Fi fails

---

## 🔬 Technical Terms (For Your Presentation)

| When to Use | What to Say |
|-------------|-------------|
| Describing the approach | "Decoupled Vision-Language-Action pipeline" |
| Explaining the AI | "Tool-augmented LLM with structured output generation" |
| Highlighting the innovation | "Zero-shot task generalization via prompt engineering" |
| Describing the control | "Geometric inverse kinematics on a 5-DOF serial manipulator" |
| Discussing the system | "Distributed cyber-physical system with networked control" |

---

## 🚧 Troubleshooting

### "The arm doesn't move"
- Check power: Are servos getting 5V?
- Check WiFi: Is ESP32 connected?
- Check JSON: Is it valid? `{"x": 15.2, "y": 22.1, "z": 0.5}`

### "The arm jitters"
- Servo power supply is insufficient (use 2A+)
- WiFi latency (increase interpolation on ESP32)

### "The arm misses objects"
- Calibration is off (re-run `calibrate.py`)
- Camera moved (fix the mount)

### "YOLO doesn't detect objects"
- Object is too small (move camera closer)
- Lighting is poor (add more light)

---

## 📚 Further Reading

- [YOLOv8 Documentation](https://docs.ultralytics.com/)
- [OpenCV Perspective Transform](https://docs.opencv.org/4.x/da/d54/group__imgproc__transform.html)
- [Gemma 4 Model Card](https://ai.google.dev/gemma)
- [ESP32 WebSocket Server](https://randomnerdtutorials.com/esp32-websocket-server-arduino/)

---

## 🤝 Team Roles

| Role | Responsibility |
|------|---------------|
| **AI Lead (You)** | Pipeline orchestration, LLM integration, YOLO + OpenCV |
| **Hardware Lead** | ESP32 firmware, Inverse Kinematics, servo control |
| **Integration Lead** | Camera calibration, WebSocket comms, testing |
| **Demo Lead** | Prompt engineering, presentation, backup planning |

---

## 📞 Quick Commands

```bash
# Run the full pipeline with a specific prompt
python run.py --prompt "Sort red to the left"

# Calibrate the camera
python calibrate.py --save calibration.json

# Run just the AI (simulate without robot)
python run.py --simulate --prompt "Stack objects"

# Test WebSocket connection
python test_websocket.py --ip 192.168.1.100 --port 8765

# Record a demo video
python run.py --record demo.mp4 --prompt "Sort red to the left"
```

---

## 🙏 Acknowledgments

Built in 48 hours for [Hackathon Name] by [Team Name].

---

**Made with ❤️ and way too much caffeine**