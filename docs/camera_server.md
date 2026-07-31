# Camera node

The camera node is an ESP32 with an OV2640 sensor that sits above the workspace and serves frames over Wi Fi. It is the piece of this project that is finished and running, so it is also the easiest place to start if you are reproducing the build.

The firmware lives in `src/vision-AI/vison/` and began as the CameraWebServer example from the Arduino core for ESP32. We changed the clock, the resolution defaults, and the brownout behaviour, and left the rest close to the original because it works and it is well tested.

## What you need

* An ESP32 CAM board with the AI Thinker pinout and PSRAM. Boards without PSRAM will run but cap out at a lower resolution.
* A USB serial adapter. The AI Thinker board has no USB port of its own.
* Arduino IDE with the `esp32` board package from Espressif installed through the board manager.

## The file the repository does not contain

`app_httpd.cpp` includes `camera_index.h`, which holds the gzipped browser interface as a byte array. It is a large generated file, so it is listed in `.gitignore` and is not in this repository.

Copy it from the Arduino ESP32 core's CameraWebServer example into `src/vision-AI/vison/` before you build. Without it the compile fails immediately with a missing header. This trips up everyone once, including us.

## Flashing

1. Open `src/vision-AI/vison/CameraWebServer.ino`.
2. Set your network credentials near the top:

   ```cpp
   const char *ssid = "your network";
   const char *password = "your password";
   ```

3. In the board menu select **AI Thinker ESP32 CAM**.
4. Set PSRAM to **enabled**.
5. Set the partition scheme to **Custom**. The sketch folder carries its own `partitions.csv` giving the application close to 4 MB, which the stock schemes do not.
6. Connect GPIO0 to ground and press reset. The board is now in flash mode.
7. Upload.
8. Remove the GPIO0 jumper and press reset again.
9. Open the serial monitor at 115200 baud. The board prints its address once it joins the network.

If the upload fails partway, it is almost always the GPIO0 jumper or an underpowered USB port. The camera draws real current during a Wi Fi transmit.

## HTTP interface

Two servers run. The main one is on port 80 and the video stream gets port 81 to itself, so a client pulling frames continuously cannot block a control request.

### Port 80

| Endpoint | What it does |
|---|---|
| `GET /` | Browser interface. The firmware picks the page variant that matches the detected sensor |
| `GET /capture` | One JPEG frame. This is what you want for detection |
| `GET /bmp` | One uncompressed BMP frame. Large and slow, useful when JPEG artefacts are a problem |
| `GET /status` | Current sensor settings as JSON |
| `GET /control?var=NAME&val=VALUE` | Change one sensor setting |
| `GET /xclk?xclk=MHZ` | Change the sensor clock at runtime |
| `GET /resolution?...` | Set the sensor window directly |
| `GET /reg`, `GET /greg`, `GET /pll` | Low level sensor register access. You will not need these unless something is badly wrong |

Settings accepted by `/control` include `framesize`, `quality`, `brightness`, `contrast`, `saturation`, `hmirror`, `vflip`, `awb`, `agc`, `aec`, `aec_value`, `agc_gain`, `gainceiling`, `special_effect`, `wb_mode`, `ae_level`, and `led_intensity`.

### Port 81

| Endpoint | What it does |
|---|---|
| `GET /stream` | Continuous MJPEG as `multipart/x-mixed-replace`. Each part carries a timestamp header |

All responses send `Access-Control-Allow-Origin: *`, so a page served from anywhere can read them.

## Using it

The examples below assume you have put the address the board printed into a shell variable:

```bash
export BOARD_IP=192.168.1.50
```

Grab a single frame:

```bash
curl http://$BOARD_IP/capture --output frame.jpg
```

Read the current sensor state:

```bash
curl http://$BOARD_IP/status
```

Set a resolution that suits detection, VGA in this case:

```bash
curl "http://$BOARD_IP/control?var=framesize&val=8"
```

Open the continuous stream:

```bash
curl http://$BOARD_IP:81/stream --output stream.mjpeg
```

From Python, the stream reads like any other video source:

```python
import os
import cv2

cap = cv2.VideoCapture(f"http://{os.environ['BOARD_IP']}:81/stream")
ok, frame = cap.read()
```

## Settings we changed and why

**Sensor clock lowered to 10 MHz.** The stock example runs at 20 MHz. On several AI Thinker boards that produces horizontal banding, corrupted frames, or an outright init failure, because the cheap clones do not hold signal integrity at the higher rate. Halving it costs frame rate and buys stability, which is the right trade when a detector is reading the output.

**Brownout detector disabled at boot.** The camera draws a current spike when it initialises, and on marginal supplies the ESP32 reads that dip as a brownout and resets in a loop. Disabling the detector stops the loop. This is a workaround, not a fix. The real fix is a supply that holds 5V under load, and if the board still misbehaves the supply is where to look first.

**Initial frame size dropped to QVGA.** The sensor is configured for UXGA so the buffers are allocated large, then immediately set to QVGA so the first frames arrive fast. Raise it once the node is confirmed running.

## When it does not work

**Camera init fails with an error code.** Wrong board selected, or the ribbon connector is not seated. Reseat it. The connector is fragile and easy to get slightly wrong.

**Boot loops.** Power. Use a supply that holds 5V under load, and avoid long thin USB cables.

**Connects to Wi Fi but the stream stalls.** Usually channel congestion. `WiFi.setSleep(false)` is already set, which handles the common power save stall. If it persists, move to 5 GHz for the host and give the board a quieter 2.4 GHz network.

**Frames are dark or washed out.** Automatic exposure is fighting the workspace lighting. Fix the lighting rather than the exposure. The detector cares more about consistency than brightness, and an auto exposure that keeps adjusting will keep changing what detection sees.
