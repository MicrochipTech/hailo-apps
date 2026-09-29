# Skill: Set Up Meal Monitoring Demo (Windows + Thunderbolt)

> Step-by-step guide to run the meal monitoring dashboard on a Windows machine with a Hailo-8 in a Thunderbolt PCIe enclosure.

## When This Skill Is Loaded

- Coworker or collaborator needs to run the meal monitoring demo on their machine
- Setting up a new Windows machine with Hailo-8 over Thunderbolt
- User mentions: setup, install, meal monitoring, onboarding, demo handoff

## Prerequisites

Before starting, confirm the following are available:

| Requirement | Details |
|---|---|
| **OS** | Windows 10 or 11 with Thunderbolt 3 or 4 port |
| **Hailo-8** | M.2 M-Key module in a Thunderbolt PCIe enclosure, connected and powered on |
| **Webcam** | USB webcam (integrated webcam also works) |
| **Python** | Python 3.10 or newer — download from https://python.org |
| **Git** | Git for Windows — download from https://git-scm.com |
| **Hailo account** | Developer Zone account at https://hailo.ai/developer-zone/ (for driver + SDK downloads) |

## Step 1: Install HailoRT for Windows

Download these from the Hailo Developer Zone (Software Downloads > HailoRT > Windows):

1. **HailoRT Windows installer** (`HailoRT-X.XX.X-win64.exe`) — installs the PCIe driver and runtime library
2. **HailoRT Python wheel** (`hailort-X.XX.X-cpXXX-win_amd64.whl`) — Python bindings for the runtime

Run the Windows installer first. It registers the Hailo-8 as a PCIe device over Thunderbolt.

**Verify the device is detected:**

```cmd
hailortcli fw-control identify
```

Expected output includes the board name, firmware version, and serial number. If this fails:
- Check that the Thunderbolt enclosure is powered on
- Authorize the Thunderbolt device in Windows Settings > Thunderbolt
- Try unplugging and reconnecting the Thunderbolt cable

## Step 2: Clone and Set Up the Repository

```cmd
git clone https://github.com/hailo-ai/hailo-apps.git
cd hailo-apps
```

Create and activate a Python virtual environment:

```cmd
python -m venv --system-site-packages hailo_venv
hailo_venv\Scripts\activate
```

The `--system-site-packages` flag is required so the venv can access the HailoRT bindings installed system-wide.

Install the HailoRT Python wheel (adjust the filename to match your downloaded version):

```cmd
pip install path\to\hailort-4.23.0-cp310-cp310-win_amd64.whl
```

Install the hailo-apps package in editable mode:

```cmd
pip install -e .
```

**Verify Python imports work:**

```cmd
python -c "from hailo_platform import VDevice; print('HailoRT OK')"
python -c "from hailo_apps.python.core.common.hailo_inference import HailoInfer; print('hailo-apps OK')"
```

## Step 3: Set Environment Variables

Set these in your terminal session (or add to system environment variables for persistence):

```cmd
set HAILO_ARCH=hailo8
set PYTHONPATH=%cd%
```

`PYTHONPATH` must point to the repo root so absolute imports (`from hailo_apps.python...`) resolve correctly.

For persistent setup, add `HAILO_ARCH=hailo8` and the repo path to `PYTHONPATH` in Windows System Environment Variables.

## Step 4: Download AI Models

Models auto-download on first run, but you can pre-download them:

```cmd
hailo-download-resources --group detection --arch hailo8
hailo-download-resources --group pose_estimation --arch hailo8
```

This downloads:
- **yolov8m.hef** (~25 MB) — object detection model for food, utensils, drinks
- **yolov8s_pose.hef** (~15 MB) — pose estimation model for 17 body keypoints

Models are stored in the resources directory (typically `C:\usr\local\hailo\resources\models\hailo8\`).

If `hailo-download-resources` is not available, the app will download models automatically on first launch (requires internet).

## Step 5: Run the Meal Monitoring App

```cmd
cd hailo_apps\python\standalone_apps\meal_monitoring
python meal_monitoring.py -n yolov8m -i usb --show-fps
```

### Command flags

| Flag | Purpose |
|---|---|
| `-n yolov8m` | Object detection model (medium YOLOv8) |
| `-i usb` | Use USB webcam as input |
| `--show-fps` | Display frame rate |
| `--pose-model yolov8s_pose` | Pose model (default, can omit) |
| `--no-gesture` | Disable pose model for higher FPS (~25 vs ~12) |
| `--dashboard-width 400` | Dashboard panel width in pixels |

### What you should see

The app opens a window with two panels side by side:

**Left panel (camera feed):**
- Live webcam feed
- Bounding boxes around food items, utensils, and drinks (color-coded)
- Skeleton overlay on detected person (magenta lines, yellow dots)

**Right panel (dashboard):**
- MEAL MONITOR header with duration timer
- Food Items section — detected foods with confidence scores
- Utensils section — detected utensils/drinks with confidence scores
- Gesture section — current gesture (Eating / Drinking / Reaching / Resting) with confidence and `[camera]` source
- Event Log — timestamped arrivals and departures of items

**Controls:** Press **Q** or **Esc** to quit.

## Step 6: Verify Everything Works

Run through this checklist:

- [ ] `hailortcli fw-control identify` shows the Hailo-8 device
- [ ] App starts without import errors
- [ ] Log shows: `Found HEF in resources: ...yolov8m.hef`
- [ ] Log shows: `Loading pose model: ...yolov8s_pose.hef`
- [ ] Camera feed appears with bounding boxes on food/utensils
- [ ] Skeleton is drawn on detected person
- [ ] Place a bottle/cup in frame — dashboard shows it under Utensils
- [ ] Raise hand to mouth — gesture changes from "Resting" to "Eating" or "Drinking"
- [ ] Both models run at ~10-15 FPS combined (shown in terminal if `--show-fps` is set)

## Troubleshooting

| Problem | Solution |
|---|---|
| `No Hailo device found` or `Failed to create VDevice` | Check enclosure power, authorize Thunderbolt device in Windows Settings, reconnect cable, re-run HailoRT installer |
| Camera shows black / doesn't open | Close other apps using the webcam (Teams, Zoom, browser). Try `-i 1` if you have multiple cameras |
| `ModuleNotFoundError: hailo_platform` | Activate the venv (`hailo_venv\Scripts\activate`), then reinstall the `.whl` |
| `ModuleNotFoundError: hailo_apps` | Set `PYTHONPATH` to the repo root, or run `pip install -e .` again |
| Models not downloading | Check internet connectivity. Manually download HEF files from Hailo Model Zoo and place in `C:\usr\local\hailo\resources\models\hailo8\` |
| `Gesture stuck on "Resting (40%)"` | Ensure the pose model loaded (check for `Loading pose model` in logs). Move your hand clearly to your face — the classifier needs 2+ frames of raised wrist with bent elbow |
| Low FPS (< 5) | Close other apps, ensure Thunderbolt is connected (not USB fallback). Try `--no-gesture` for single-model mode |

## Architecture Overview

The app runs two neural networks on the Hailo-8 simultaneously using round-robin scheduling:

```
USB Camera → preprocess → [YOLOv8m (OD)] → food/utensil detections
                        → [YOLOv8s_pose]  → 17 body keypoints
                                           → gesture classifier (heuristic)
                                           → dashboard renderer → display
```

Both models share the Hailo-8 via a single virtual device (`group_id = "SHARED"`). No GStreamer or TAPPAS required — this is a pure standalone app using HailoInfer + OpenCV.
