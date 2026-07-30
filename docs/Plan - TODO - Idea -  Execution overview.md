
# Todo: 

- Get robotics arm
- Get esp32
- Make basic web socket based camera + execution pipeline for esp (no ROS ) (reverse kinematics + drift accountability + state memorizations ) 
- Help in AI work (take coordinate data from open cv and yolo --> feed in and take data from LLM regarding location of object, arm state and actions it took --> get coordinates and apply reverse kinematics to get coordinates regarding how much agent should move  )
- multi agent setup
- Web UI for agent 


# Ambitious TODO list:

* cam module on gripper 
* 2 phones, parpendicuar to each-other for all 3 axis view
* Covert this code to ROS jazzy
* Add ultra sonic sensor on top of gripper 
* Add ability to remotely control the location (add car module via Arduino)
* add face module + good presentation --> Led matrix for loading 
* make it control with joystick 
* simulation of training data to use AVL model 
* Make UI for interacting with AI
* Add data for RAG related to our system 
* Add SST for intracting via voice
* Make the agent smart ( give agentic abilities + MCP server + Learing schema + Skills.md and more ) 
* Add OWL-ViT --> tool call
* Add normal AVL model with fine tuneing

# Idea :
We plan to make a generalize system for any Robotic arm controller with agentic abilities of executing a task (just tell it what task to do and agent 1 will come up with plan of execution step by step and feed it to worker agent for taking action) .
This can help in industry for adaptive assemblies and custom assembly lines . It can also be used in making humanoid robots . Can be used for controlling and doing autonomous tasks in unreachable conditions (space, hazardous conditions)

unique: 
--> decentralized model architecture (2 models types) for more complex tasks
--> Easily scalable to multiple robotic arms + Multiple embodiments (Adapts to a diverse array of robot forms)
--> Thinking ability (dynamic changes according to the needs and learns from experience)
--> Agentic (Assess complex challenges, natively call tools – like Google Search)
--> All this without ALV models

vision :
--> tell it what to do and it will do it ; learn from mistakes , optimize movements and finally give output for your custom chain . or u can just make a constantly varying assembly line .  
--> used for outer space, deep sea exploration and construction + performing any action .

# Working pipeline during task :

```
Camera Frame
    ↓
YOLOv8/OpenCV/World model (5ms) → Exact pixel coordinates (x: 450, y: 380)
    ↓
OpenCV Perspective Transform (1ms) → Real-world cm (x: 15.2, y: 22.1)
    ↓
LLM (Gemma 4 12B with vision) → Receives: "Detected red block at X:15.2cm, Y:22.1cm. System prompt: Sort red blocks to left bin. What should I do?"
    ↓
LLM Tool Call → {"action": "pick", "target_x": 15.2, "target_y": 22.1, "z_mode": "surface"}
    ↓
Python maps z_mode: "surface" → z: 0.5cm (actual height above table)
    ↓
ESP32 receives (15.2, 22.1, 0.5) → Inverse Kinematics → 5 servo angles → Arm moves
```

# Versions of project :
## V1 ( ideally it should be done before entering hackathon) :
- Arm controlling pipeline : ESP32 based + IK + motor control + WebSocket server for camera feed  
- Arm fully functional 
- basic TUI 
- impressive task execution (any 1)
- fully integrated system all working fine together
- 2-3 practices of speaking skills 
- Box contraption --> work environment
- dual AI sys

## V2 (ideally done in 12 - 15 Hrs ) :  
- The full pipeline in ROS2 jazzy
- Good TUI + good visual indicators 
- multiple task execution + variable task
- Web UI for agent 
- Really good practice of speaking skills
- security

## V3 (do as mush as you can) :
- addition of things in ambitious list
- Highly visually pleasing (both WEB UI + TUI)
- cool additions to robotic arm (loading screen and shit)