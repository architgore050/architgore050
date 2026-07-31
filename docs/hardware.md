# Hardware

What the physical build looks like, how it is wired, and the things that went wrong so you do not have to rediscover them.

## Bill of materials

| Part | What we used | Count | Notes |
|---|---|---|---|
| Arm | Five joint design, our own assembly | 1 | SolidWorks files in `robotic arm schematics/` |
| Servos | SG90 class hobby servos | 5 | Four joints plus the gripper |
| Camera board | ESP32 CAM, AI Thinker layout, OV2640 | 1 | Get one with PSRAM |
| Arm controller | ESP32 development board | 1 | Any variant with Wi Fi |
| Servo supply | 5V, 2A minimum | 1 | Must be separate from the boards |
| USB serial adapter | 3.3V logic | 1 | For flashing the camera board |
| Calibration markers | Four distinct coloured objects | 4 | Fixed positions, measured once |
| Frame | Rigid mount holding the camera above the workspace | 1 | Rigidity matters more than it sounds |

## The mechanical build

The arm assembly is in `robotic arm schematics/Assembly.SLDASM`. Five joints: base rotation, shoulder, elbow, wrist, and gripper. That is enough to reach any point on a flat workspace with a usable approach angle, and it keeps the kinematics tractable enough to solve on a microcontroller.

SG90 class servos are the compromise everyone makes here. They are cheap, they are everywhere, and their gears are plastic. They will strip if you drive a joint past its mechanical limit. Software limits in the controller are the only thing standing between an enthusiastic model and a rebuild, so treat those limits as load bearing code.

## Power

This is the part that eats hackathon time, so it goes near the top.

**Give the servos their own supply.** Five servos moving together pull well over an amp, with spikes above that on direction changes. An ESP32 regulator cannot deliver it. What happens if you try is not a clean failure: the board browns out mid motion, resets, and you spend an hour debugging firmware that was fine.

The arrangement that works:

```
  USB  ---->  ESP32 boards        (logic)

  5V 2A supply  ---->  servo power rail

  supply ground  ----  ESP32 ground     (tied together, not optional)
```

The common ground is what makes the servo control signal meaningful. Without it the servos see a reference voltage that is not related to the one the ESP32 is driving against, and behaviour ranges from jitter to nothing at all.

A capacitor of a few hundred microfarads across the servo rail smooths the current spikes and is worth the two minutes it takes to add.

## Camera mounting

The camera looks straight down at the workspace from a fixed position. Two requirements, both absolute:

**It must not move.** Calibration maps camera pixels to table centimetres. Move the camera a millimetre and every position the system reports is wrong, with no error and no warning. Mount it rigidly. Do not mount it to anything the arm can touch, and do not mount it to a table someone will lean on.

**The whole workspace must be in frame, including the arm base.** Objects outside the frame do not exist as far as the system is concerned.

Height is a trade. Higher covers more area with less resolution per object. Lower gives detection more to work with but risks the arm occluding the scene during a reach. We settled by putting the arm through its full range with the camera in place and checking what it hid.

## Calibration

Calibration builds the homography that turns pixels into centimetres. It takes about five minutes and it is the difference between an arm that picks things up and an arm that gestures near them.

1. Place four markers at the corners of the workspace, well separated and all visible.
2. Measure each marker's real position in centimetres from the arm base. Measure carefully. Errors here propagate to every action.
3. Capture a frame and record each marker's pixel position.
4. Save the four pairs to the configuration under `src/vision-AI/config/`.

Then verify before you trust it. Put an object at a known position, ask the system where it is, and compare. If the answer is off by more than roughly a centimetre, something in the measurement is wrong and it will not fix itself.

Recalibrate at the start of every session. It costs five minutes. Discovering mid demo that the camera got bumped costs the demo.

## Workspace

* **Even, constant lighting.** Automatic exposure adapting to changing light means detection sees a different image minute to minute. Consistent and slightly dull beats bright and variable.
* **A matte surface.** Glossy tabletops throw specular highlights that detection reads as objects.
* **Contrast against the objects.** A dark surface with light objects, or the reverse. Do not put grey blocks on a grey table.
* **Boundaries.** A physical box or frame around the workspace keeps objects inside the arm's reach and inside the camera frame.

## Things that went wrong

Collected honestly, because every one of these cost real time.

**Servos powered from the board.** Covered above. It presents as random resets and looks exactly like a firmware bug.

**Camera moved between calibration and demo.** Everything runs, nothing errors, the arm just misses by a consistent offset. Now the first check when the arm starts missing is always the camera mount, not the code.

**Joint limits found the hard way.** One stripped servo before the software limits went in. The limits are in the controller for a reason.

**Ribbon connector not fully seated.** Presents as a camera init failure with an error code, which reads like a software problem. It is not. Reseat it.

**Wi Fi in a crowded room.** Fine on a home network, unusable in a hall full of hotspots. Test on a network as bad as the one you will demo on, and bring a phone hotspot as a fallback.
