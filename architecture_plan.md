# ESP32 Atomic Audio Streamer - Architecture Plan

This document outlines the architectural plan for creating a minimalist, highly optimized "atomic repository" for the ESP32. The goal is to distill the core bidirectional real-time audio streaming capabilities found in projects like Xiaozhi, completely stripping away UI, display drivers, and vendor bloat.

The resulting firmware will serve as a headless, robust, real-time audio pipeline and network bridge connecting the ESP32 to a locally hosted or cloud-based AI backend.

## Core Technical Stack

> [!NOTE]
> **Framework Choice:** This repository will be built exclusively using **ESP-IDF** with the **ESP-ADF (Audio Development Framework)**. This provides the most robust, professional-grade primitives for audio pipelines, VAD, and Opus encoding without the overhead of the Arduino core.

## Hardware-Specific Considerations

> [!WARNING]
> **M5Stack Atom Voice (Atom Echo):**
> *   **Reserved Pins (CRITICAL):** GPIOs **G19, G22, G23, and G33** are hard-wired to the internal I2S audio components. Reassigning or misusing these pins in the atomic repo WILL damage the device.
> *   **Hardware Components:** It uses an **SPM1423** I2S microphone and an **NS4168** I2S amplifier. (It also includes an SK6812 RGB LED on G27 and a Button on G39 for status/control).
> *   **Playback Safety:** The tiny 0.5W speaker is fragile. The atomic repo must NEVER output a DC signal, long periods of white noise, or full-scale square-wave audio through the I2S channel, as this will blow the speaker coil.
> *   **Amplifier Enable Pin:** The NS4168 requires its specific enable pin to be pulled HIGH before it will output audio.

> [!WARNING]
> **Waveshare ESP32-S3 1.8inch AMOLED Touch Display:**
> *   **Onboard ES8311 Codec:** Unlike simpler boards, this unit features a dedicated **ES8311** low-power mono audio codec to drive its onboard speaker and microphone. Your atomic repo cannot just send raw I2S data; it MUST first configure the ES8311 via an I2C control bus before the I2S audio lines will function.
> *   **Pin Conflicts:** The ESP32-S3 on this board is heavily loaded. It drives the AMOLED via QSPI, the touch controller via I2C, and the ES8311 via both I2C (control) and I2S (data). The `hw_config.h` must perfectly map these internal traces to avoid bus collisions.

## Finalized Design Decisions (Xiaozhi-Compatible)

> [!NOTE]
> Per the requirement to follow the known-good Xiaozhi architecture, the following protocols are locked in:
> 
> 1. **Backend Protocol:** The repo will implement **JSON-RPC 2.0 over WebSockets** (Model Context Protocol). This ensures strict compatibility with existing open-source Xiaozhi backend servers and allows the AI to execute local hardware tool calls (like toggling the Atom Echo's RGB LED or reading the button).
> 2. **Audio Codec:** The pipeline will strictly enforce **Opus Compression** (20ms/40ms framing) from day one to match Xiaozhi's ultra-low latency, jitter resistance, and network efficiency.

---

## Proposed Architecture

The architecture relies on the **ESP-ADF Pipeline** model. Data flows strictly from the hardware layer, through DSP and encoding layers, and out to the network layer (and vice versa).

### 1. The Uplink Pipeline (Capture to Network)
Responsible for capturing user voice, detecting speech, encoding it, and transmitting it to the server.

*   **`[I2S Reader]`**: Interfaces directly with the I2S microphone (e.g., INMP441) via DMA. Captures raw 16kHz, 16-bit mono PCM data.
*   **`[Software AEC (Optional)]`**: Intercepts the raw mic data and subtracts any audio currently being played by the speaker to prevent echo loops.
*   **`[VAD Filter]`**: Analyzes the PCM stream. If silence is detected, it drops the frames to save bandwidth and battery. If speech is detected, it passes the frames forward and signals the WebSocket client to ensure the connection is open.
*   **`[Opus Encoder]`**: Compresses the raw PCM data into 20ms or 40ms Opus frames (~24 kbps) for ultra-low latency transmission.
*   **`[WebSocket Writer]`**: Transmits the binary Opus frames to the backend server.

### 2. The Downlink Pipeline (Network to Playback)
Responsible for receiving AI responses, buffering them to prevent stuttering, decoding, and playing the audio.

*   **`[WebSocket Reader]`**: Listens for incoming binary frames from the server.
*   **`[Jitter Buffer (RingBuffer)]`**: A strict FreeRTOS Ring Buffer. It accumulates ~50-100ms of incoming audio before releasing it to the decoder. This absorbs Wi-Fi jitter and prevents audio stuttering.
*   **`[Opus Decoder]`**: Decompresses the incoming Opus frames back into raw PCM data.
*   **`[I2S Writer]`**: Pushes the raw PCM data to the I2S DAC/Amplifier (e.g., MAX98357A) via DMA for speaker output.

---

## Core Optimizations (Extracted from Xiaozhi)

To ensure professional-grade performance in a minimal codebase, the following optimizations will be strictly enforced:

> [!TIP]
> **1. Strict Core Pinning:** 
> *   **Core 0 (PRO_CPU):** Dedicated exclusively to the Wi-Fi stack, TCP/IP, and WebSocket connection handling.
> *   **Core 1 (APP_CPU):** Pinned for all I2S DMA tasks, VAD, and Opus encoding/decoding.

> [!TIP]
> **2. Task Priorities:**
> I2S read/write tasks will be assigned near-maximum FreeRTOS priorities (`configMAX_PRIORITIES - 1`) to ensure they are never preempted by network interrupts.

> [!TIP]
> **3. Intelligent VAD Uplink:**
> The WebSocket uplink will only stream data when the VAD filter detects active human speech. A configurable "hang-time" (e.g., 500ms of silence) will trigger an End-Of-Speech frame.

> [!TIP]
> **4. Opus Frame Alignment:**
> Audio frames will be tightly aligned to the Wi-Fi MTU sweet spot (20ms/40ms) to minimize TCP/IP overhead and avoid fragmentation latency.

---

## Proposed Directory Structure

The repository will be kept entirely flat and modular, focusing only on the audio bridge.

```text
atomic-esp32-audio/
├── CMakeLists.txt                # ESP-IDF build configuration
├── sdkconfig.defaults            # Pre-configured optimizations (Core pinning, RingBuffer sizes)
├── main/
│   ├── main.c                    # Application entry point & pipeline orchestration
│   ├── CMakeLists.txt
│   ├── network/
│   │   ├── wifi_manager.c/.h     # Auto-reconnecting Wi-Fi state machine
│   │   └── websocket_client.c/.h # WebSocket read/write handlers
│   ├── audio/
│   │   ├── i2s_hardware.c/.h     # I2S Mic/Speaker init and DMA config
│   │   ├── pipeline_uplink.c/.h  # VAD and Opus Encoder orchestration
│   │   └── pipeline_downlink.c/.h# Jitter Buffer and Opus Decoder orchestration
│   └── config/
│       └── hw_config.h           # Centralized GPIO pin mappings and buffer sizes
└── README.md
```

## Verification Plan

### Automated/Build Verification
- Run `idf.py build` to ensure the project compiles cleanly under the latest ESP-IDF framework without any of the Xiaozhi UI/Display bloat.

### Manual Verification
1. **Network Connectivity:** Flash the ESP32 and verify it connects to Wi-Fi and establishes a WebSocket connection with a dummy local server.
2. **Uplink Test (Mic):** Speak into the microphone. Verify the local server receives binary WebSocket packets ONLY when speaking (verifying VAD).
3. **Downlink Test (Speaker):** Stream a raw PCM or Opus file from the server to the ESP32. Verify clean, stutter-free playback out of the speaker (verifying Jitter Buffer and I2S output).
