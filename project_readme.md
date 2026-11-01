# StareOff — Two-Player Computer Vision Staring Contest

A real-time, two-player computer vision game written in modern C++ and OpenCV. Two competitors face the webcam in an arcade-style stare-off: the game continuously tracks both players' faces, isolates eye regions, approximates an Eye Aspect Ratio (EAR), and detects blinks to determine round and match winners automatically.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
  - [Detection Pipeline](#detection-pipeline)
  - [Eye Aspect Ratio (EAR) Heuristic](#eye-aspect-ratio-ear-heuristic)
  - [Game State Machine](#game-state-machine)
- [Prerequisites](#prerequisites)
- [Directory Structure](#directory-structure)
- [Building and Running](#building-and-running)
  - [Linux / macOS](#linux--macos)
  - [Windows (MSVC / MinGW)](#windows-msvc--mingw)
  - [CMake Build (Cross-Platform)](#cmake-build-cross-platform)
- [Controls](#controls)
- [Configuration & Tuning](#configuration--tuning)
- [Technical Trade-offs & Limitations](#technical-trade-offs--limitations)
- [Future Enhancements](#future-enhancements)
- [License](#license)

---

## Overview

StareOff is designed as an interactive graduation/capstone demonstration combining classical computer vision techniques with low-latency game state management. Operating at standard webcam frame rates (30+ FPS) on commodity CPU hardware, it bypasses the compute overhead of heavy neural networks by pairing optimized Haar Cascade Classifiers with a bounding-box EAR approximation.

---

## Key Features

- **Multi-Player Face Tracking:** Simultaneously identifies and segments Player 1 (left) and Player 2 (right) within the webcam frame.
- **Eye Region of Interest (ROI) Slicing:** Dynamically extracts facial sub-regions before invoking eye classifiers to maintain high performance.
- **Custom Alpha-Blended HUD:** Native OpenCV rendering featuring rounded semi-transparent panels, real-time EAR visualizer meters, threshold tick markers, and animated countdowns.
- **Robust Blink Filtration:** Employs consecutive frame thresholds, cooldown timers, and missing-frame tolerances to eliminate false positives caused by webcam noise or partial occlusions.
- **Deterministic State Machine:** Built around 5 distinct game phases (`MENU`, `COUNTDOWN`, `GAMEPLAY`, `ROUND_END`, `GAME_OVER`) timed via monotonic high-resolution clocks (`std::chrono::steady_clock`).

---

## System Architecture

### Detection Pipeline

```
 [ Webcam Frame ]
        │
        ▼
 [ Grayscale Conversion & Histogram Equalization (equalizeHist) ]
        │
        ▼
 [ Haar Cascade Face Detection (Top 2 Faces by Area, Sorted by X) ]
        │
   ┌────┴────────────────────────┐
   ▼                             ▼
[ Player 1 ROI ]          [ Player 2 ROI ]
   │                             │
   ▼                             ▼
[ Haar Eye Cascade ]      [ Haar Eye Cascade ]
   │                             │
   ▼                             ▼
[ 6-Point EAR Calculation ] [ 6-Point EAR Calculation ]
   │                             │
   ▼                             ▼
[ EMA Smoothing (α = 0.28) ] [ EMA Smoothing (α = 0.28) ]
   │                             │
   └──────────────┬──────────────┘
                  ▼
         [ Blink Logic & Game State ]
                  │
                  ▼
         [ Rendered HUD Overlay ]
```

### Eye Aspect Ratio (EAR) Heuristic

The game adopts the landmark EAR formulation originally developed by Soukupová and Čech:

$$\text{EAR} = \frac{\|p_2 - p_6\| + \|p_3 - p_5\|}{2 \, \|p_1 - p_4\|}$$

Where:
- $p_1, p_4$ designate the lateral eye corners.
- $p_2, p_3$ designate the upper eyelid boundary points.
- $p_5, p_6$ designate the lower eyelid boundary points.

Because Haar Cascades return bounding boxes rather than discrete 68-point facial landmarks, the application synthesizes a 6-point relative coordinate mesh based on empirical proportions of the detected eye rectangle ($w \times h$):
- **Corners:** $10\%$ inward from bounding box left and right edges at mid-height ($0.5h$).
- **Upper Lid:** $20\%$ and $80\%$ horizontal width at $0.25h$.
- **Lower Lid:** $20\%$ and $80\%$ horizontal width at $0.75h$.

Blinks are confirmed either when the calculated EAR dips below the configurable threshold ($0.23$ default) for $N$ consecutive frames, or when the eye cascade completely loses tracking while the face bounding box remains present.

### Game State Machine

```
   ┌─────────┐
   │  MENU   │ <──────────┐
   └────┬────┘            │
        │ [Space]         │ [R] / Auto-timeout
        ▼                 │
 ┌─────────────┐          │
 │  COUNTDOWN  │          │
 └──────┬──────┘          │
        │ 3 Seconds       │
        ▼                 │
  ┌───────────┐           │
  │ GAMEPLAY  │           │
  └─────┬─────┘           │
        │ Blink Detected  │
        ▼                 │
 ┌─────────────┐          │
 │  ROUND_END  │          │
 └──────┬──────┘          │
        ├─ Round < 3 ─────┘ (loops to COUNTDOWN)
        │
        └─ Round >= 3
                │
                ▼
         ┌─────────────┐
         │  GAME_OVER  │
         └─────────────┘
```

---

## Prerequisites

- **C++ Compiler:** C++14 or newer (`gcc` 7+, `clang` 6+, or MSVC 2017+).
- **OpenCV:** OpenCV 4.x (Development headers and shared libraries).
- **Webcam:** Any standard USB or integrated camera (720p/1080p recommended).
- **Haar Cascade Models:**
  - `haarcascade_frontalface_default.xml`
  - `haarcascade_eye.xml`
  - *(Optional)* `haarcascade_eye_tree_eyeglasses.xml`

---

## Directory Structure

```
stare-off/
├── CMakeLists.txt                      # Build script
├── main.cpp                            # Complete game source code
├── haarcascade_frontalface_default.xml # Face classifier
├── haarcascade_eye.xml                 # Primary eye classifier
├── haarcascade_eye_tree_eyeglasses.xml # Fallback glasses classifier
└── README.md                           # Documentation
```

---

## Building and Running

### Linux / macOS

Install OpenCV dependencies (Ubuntu/Debian example):
```bash
sudo apt update
sudo apt install build-essential libopencv-dev
```

Compile directly using `g++`:
```bash
g++ -std=c++14 -O3 main.cpp -o stare_off `pkg-config --cflags --libs opencv4`
./stare_off
```

### Windows (MSVC / MinGW)

Ensure OpenCV `include` and `lib` directories are configured in your system environment path.

Using MSVC developer prompt:
```cmd
cl /EHsc /std:c++14 /O2 main.cpp /I "C:\opencv\build\include" /link /LIBPATH:"C:\opencv\build\x64\vc16\lib" opencv_world4xx.lib
stare_off.exe
```

### CMake Build (Cross-Platform)

```cmake
cmake_minimum_required(VERSION 3.10)
project(StareOff)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})

add_executable(stare_off main.cpp)
target_link_libraries(stare_off ${OpenCV_LIBS})
```

Build commands:
```bash
mkdir build && cd build
cmake ..
cmake --build . --config Release
./stare_off
```

> **Note:** Ensure XML cascade files are placed in the working directory from which the executable is launched.

---

## Controls

| Key | Action |
| :---: | :--- |
| <kbd>SPACE</kbd> | Start match from main menu |
| <kbd>R</kbd> / <kbd>r</kbd> | Reset game to main menu at any time |
| <kbd>+</kbd> / <kbd>=</kbd> | Increase EAR blink detection sensitivity |
| <kbd>-</kbd> / <kbd>_</kbd> | Decrease EAR blink detection sensitivity |
| <kbd>T</kbd> / <kbd>t</kbd> | Output diagnostic status & telemetry to terminal |
| <kbd>ESC</kbd> | Exit application |

---

## Configuration & Tuning

Key detection parameters are defined at the top of `main.cpp`:

```cpp
static const float DEFAULT_EAR_THRESHOLD   = 0.23f; // Base EAR threshold for closed eyes
static const int   DEFAULT_CONSECUTIVE_FRAMES = 3;  // Frames below threshold required to confirm blink
static const int   MAX_MISSING_FRAMES        = 5;   // Frame tolerance for temporary detector loss
static const int   BLINK_COOLDOWN_FRAMES     = 15;  // Buffer period preventing double-triggering
```

- **In High Ambient Light / Direct Sunlight:** Slightly increase `DEFAULT_EAR_THRESHOLD` (e.g., `0.25f`) using the on-screen controls (`+`).
- **If Players Wear Eyeglasses:** Add `haarcascade_eye_tree_eyeglasses.xml` to the project root directory; the loading routine checks for this fallback automatically.

---

## Technical Trade-offs & Limitations

1. **Haar Cascades vs. Facial Landmarks:**
   - *Trade-off:* Haar Cascades are computationally lightweight and require no external dependencies (like Dlib, ONNX, or MediaPipe), but they predict bounding boxes rather than sub-pixel eyelid contours.
   - *Mitigation:* The system combines bounding-box aspect ratio with classifier absence detection (`eyesVisible == false`) to create a stable hybrid indicator.
2. **Player Identity Assignment:**
   - Players are differentiated by their horizontal $x$-coordinate index (`getTopTwoFaces`). If players cross positions or move rapidly out of view, identity assignment may shift.
3. **Lighting Sensitivity:**
   - Addressed using `cv::equalizeHist` on input grayscale frames, though drastic shadows can still impede cascade edge feature matching.

---

## Future Enhancements

- [ ] Incorporate lightweight ONNX landmark models (e.g., LDM68 / MediaPipe Iris) via OpenCV's `dnn` module for true sub-pixel contour extraction.
- [ ] Implement a centroid tracker (Kalman Filter or Euclidean Distance tracker) to maintain player identities through head rotation and occlusion.
- [ ] Add sound effects using SDL2_mixer or native platform audio APIs.
- [ ] Add networked multi-device gameplay via WebSockets.

---

## License

This project is open-source and available under the [MIT License](LICENSE).