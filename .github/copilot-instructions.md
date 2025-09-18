# Copilot instructions for Adaptive-Traffic-Signal-Timer

Purpose: Help AI coding agents be productive quickly in this repo by capturing the architecture, critical workflows, conventions, and safe extension points.

## Big picture
- Two runnable modules under `Code/YOLO/darkflow`:
  - `vehicle_detection.py`: Batch object detection on images using Darkflow (YOLOv2 via TensorFlow 1.x). Reads from `./test_images/`, writes annotated images to `./output_images/`.
  - `simulation.py`: Pygame-based traffic intersection simulator with an adaptive signal timing algorithm driven by vehicle counts.
- Current integration is loose: detection runs on static images; the simulator uses its own in-memory vehicle queues and computes green times from those queues. Hooks to call detection are present but commented.

## Key files and roles
- `Code/YOLO/darkflow/vehicle_detection.py`
  - Builds `TFNet` with:
    - `model: ./cfg/yolo.cfg`
    - `load: ./bin/yolov2.weights`
    - `threshold: 0.3`
  - Example of label filtering and drawing boxes for classes: `car`, `bus`, `bike`, `truck`, `rickshaw`.
- `Code/YOLO/darkflow/simulation.py`
  - Core globals: `signals`, `vehicles`, `currentGreen`, `currentYellow`, `nextGreen`, min/max timings.
  - Threads: `initialize()` (signal schedule), `generateVehicles()` (random traffic), `simulationTime()` (wall clock), and the control loop `repeat()`.
  - Adaptive timing formula (clamped to min/max):
    - `greenTime = ceil((cars*carTime + rickshaws*rickshawTime + buses*busTime + trucks*truckTime + bikes*bikeTime)/(noOfLanes+1))`
  - Detection hook: `setTime()` is triggered when the next signal’s red time reaches `detectionTime` (5s). It currently counts vehicles from the `vehicles` queues; the YOLO-based call is commented.
  - UI: draws signals, timers, and per-lane crossed counts from sprites in `images/`.
- Build scaffolding: `Code/YOLO/darkflow/requirements.txt` (TensorFlow 1.13.1, pygame, OpenCV, etc.), `Code/YOLO/darkflow/setup.py` (Cython extensions for Darkflow).

## Developer workflows (Windows PowerShell)
- Work from `Code/YOLO/darkflow` so relative paths resolve.

```powershell
# Create and activate a Python 3.7 env (TensorFlow 1.13.x compatible)
py -3.7 -m venv .venv
.\.venv\Scripts\Activate.ps1

# Install deps and build cython extensions
pip install -r requirements.txt
python setup.py build_ext --inplace

# Ensure YOLO weights exist at ./bin/yolov2.weights

# Run vehicle detection (annotates all files under ./test_images)
python .\vehicle_detection.py

# Run simulator
python .\simulation.py
```

Notes:
- Requires MSVC Build Tools on Windows (per `readme.md`).
- `simulation.py` calls `os.system("say ...")` (macOS). On Windows this is a no-op/error and can be safely removed when editing.

## Conventions and patterns
- CWD-sensitive paths: cfg/weights/images are all relative to `Code/YOLO/darkflow`.
- Asset layout used by the simulator:
  - `images/signals/{red,yellow,green}.png`
  - Vehicle sprites: `images/<direction>/{car,bus,truck,rickshaw,bike}.png`
  - Background: `images/mod_int.png`
- Global mutable state + threads: `signals`, `vehicles`, and timing variables are shared across threads; keep detection non-blocking (use a thread like `setTime()` does).
- No test suite. Use scripts above for smoke checks.

## Safe extension points for agents
- Integrate real detection into the simulator:
  - Implement a non-blocking `detection()` that returns counts per class for the upcoming `nextGreen` direction, then set `signals[nextGreen].green` using the existing formula. Call it from inside `setTime()` (already threaded).
- Adjust detection thresholds/models:
  - Change `options['threshold']`, `options['model']`, or `options['load']` in `vehicle_detection.py`. Keep paths relative to the darkflow folder.
- Simulator knobs:
  - Tuning: `defaultMinimum`, `defaultMaximum`, `carTime/bikeTime/...`, `noOfLanes`, `detectionTime`.

## Examples (codebase-aligned)
- Building TFNet like detection does:
```python
from darkflow.net.build import TFNet
options = {'model':'./cfg/yolo.cfg','load':'./bin/yolov2.weights','threshold':0.3}
tfnet = TFNet(options)
```
- Inside `setTime()` replace queue counting with YOLO counts for `directionNumbers[nextGreen]`; keep it in the existing thread.

## Gotchas
- TensorFlow 1.13.x requires Python 3.7; newer Python versions will fail to install pins in `requirements.txt`.
- Always run from `Code/YOLO/darkflow` to satisfy relative paths to `cfg/`, `bin/`, `images/`, `test_images/`, and `output_images/`.
