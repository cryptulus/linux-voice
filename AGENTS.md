# linux-voice — Agent Guide

> **TL;DR:** linux-voice is a speakup connector that uses Kokoro TTS as the preferred voice output, with espeak-ng as a fallback when Kokoro dependencies aren't installed or a language isn't supported. Read this file first, then ROADMAP.md for the task list.

## Project Overview

linux-voice is a connector for the Linux kernel [speakup](http://linux-speakup.org) screen reader. It receives commands from speakup (via `/dev/softsynth`) and produces speech audio using the [Kokoro](https://github.com/hexgrad/kokoro) neural TTS model (82M parameters) via the [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) C library.

**Preferred path:** Kokoro via sherpa-onnx — higher quality, natural-sounding speech.
**Fallback path:** espeak-ng — used when Kokoro is unavailable, dependencies aren't installed, or the requested language has no Kokoro voice.
**What stays the same:** speakup protocol, threading model, CLI options, systemd service

## Quick Start for New Agents

1. Read this file (AGENTS.md) — understand the dual-engine architecture and what needs to change
2. Read ROADMAP.md — see the phased task list
3. Read `src/espeakup.h` — understand the data structures and thread interfaces
4. Read `src/softsynth.c` — understand how speakup commands are parsed
5. Read `src/espeak.c` — this is the main work file (dual-engine TTS dispatch)
6. Start with Phase 1 of ROADMAP.md: wire Kokoro in as the primary voice engine

## Build & Test

```bash
# Setup
meson setup . ./build
cd ./build

# Build
ninja

# Install (requires root)
sudo ninja install

# Run in debug mode (foreground, no daemonize)
linux-voice --debug
```

**Dependencies:**
- `meson >= 0.51.0`, `ninja` — build system
- `sherpa-onnx` — Kokoro TTS via C API (see DEVELOPMENT.md for install)
- `alsa-lib` — audio playback
- `pthread` — threading
- `libm` — math library

## Project Structure

```
meson.build              # Top-level meson config (project name, dependencies)
meson_options.txt        # Build options (man page, systemd)
src/
  espeakup.c             # main(), daemon setup, thread creation
  espeakup.h             # Public header: data structures, extern declarations
  espeak.c               # TTS engine interaction — THIS IS THE FILE TO MODIFY
  softsynth.c            # Reads /dev/softsynth, parses speakup protocol, queues commands
  signal.c               # SIGINT/SIGTERM handler thread
  cli.c                  # Command-line argument parsing
  queue.c, queue.h       # Thread-safe queue for commands
  stringhandling.c, .h   # Memory helpers: allocMem(), dupeString(), initString()
  version.h.in           # Version string template
doc/
  linux-voice.8.ronn     # Man page source
services/systemd/
  linux-voice.service.in # Systemd service template
```

## Architecture

linux-voice has three threads communicating via a shared queue:

### Thread 1: softsynth (softsynth.c)
- Opens `/dev/softsynth` (or reads stdin in `--acsint` mode)
- Reads binary speakup commands
- Parses commands into structured entries
- Adds entries to `synth_queue` (protected by `queue_guard` mutex)
- Signals `runner_awake` condition variable to wake the kokoro thread

### Thread 2: kokoro (espeak.c, previously "espeak thread")
- Waits on `runner_awake` condition variable
- When signaled, processes entries from `synth_queue`
- Dispatches to **Kokoro** first (via sherpa-onnx) for preferred quality
- Falls back to **espeak-ng** if Kokoro is unavailable, a language has no Kokoro voice, or Kokoro fails
- Handles audio playback through ALSA
- Manages error recovery (wedged audio device detection, auto-restart)

### Thread 3: signal (signal.c)
- Waits for SIGINT or SIGTERM via `sigwait()`
- Sets `should_run = 0` and broadcasts to all condition variables
- Triggers shutdown sequence

### Data flow
```
speakup → /dev/softsynth → softsynth thread → synth_queue → kokoro thread → Kokoro → ALSA
```

### Key shared state (from espeakup.h)
```c
struct queue_t *synth_queue;        // Shared command queue
pthread_mutex_t queue_guard;        // Protects synth_queue
pthread_cond_t runner_awake;        // Wakes kokoro thread when queue has work
volatile int should_run;            // Shutdown flag
volatile int stop_requested;        // Stop current speech flag
```

## Speakup Protocol

speakup sends a binary protocol through `/dev/softsynth`. The softsynth thread parses it in `process_command()` (softsynth.c).

### Command format

**Parameter commands:** `0x01` + modifier + decimal value + type letter

| Byte | Content |
|------|---------|
| 1 | `0x01` (command prefix) |
| 2 | Modifier: `+` (increment), `-` (decrement), or digit (set directly) |
| 3+ | Decimal number (e.g., `5`, `100`) |
| Last | Type letter: `b`, `f`, `i`, `p`, `r`, `s`, `v`, `P` |

**Type letters:**
| Letter | Meaning | espeakup command | Kokoro equivalent |
|--------|---------|-------------------|-------------------|
| `b` | Punctuation level | CMD_SET_PUNCTUATION | N/A (Kokoro uses IPA tokens) |
| `f` | Frequency | CMD_SET_FREQUENCY | No direct equivalent |
| `i` | Mark | CMD_SET_MARK | N/A (position marker) |
| `p` | Pitch | CMD_SET_PITCH | No direct equivalent |
| `r` | Range | CMD_SET_RANGE | No direct equivalent |
| `s` | Rate | CMD_SET_RATE | `speed` parameter (1.0 = normal) |
| `v` | Volume | CMD_SET_VOLUME | Post-processing gain |
| `P` | Pause | CMD_PAUSE | Stop current speech |

**Text commands:** Plain printable ASCII bytes are queued as text to speak.

**Flush:** Byte `0x18` (FS / File Separator) stops current speech and clears buffer.

**acsint mode:** Newline (`\n`) or carriage return (`\r`) flushes pending text.

### Example speakup command sequence

When speakup wants to say "cursor at line 5":
```
0x01 + 's' → set rate (or similar param)
"cursor at line " → text bytes
0x01 + 's' → another param
"5" → text bytes for the number
```

The softsynth thread accumulates text and parameter commands into queue entries. The kokoro thread processes them in order.

## Audio Output

Kokoro outputs 24,000 Hz mono f32 PCM samples. Audio must be played through ALSA.

### Conversion requirements
1. **Sample format:** f32 → s16 (Kokoro outputs float, ALSA expects signed 16-bit)
2. **Sample rate:** Kokoro is 24kHz. ALSA may be configured at 48kHz, 44.1kHz, etc. Resampling may be needed.
3. **Channels:** Mono (1 channel) — no interleaving needed.

### ALSA integration approach
```c
// Create PCM handle
snd_pcm_t *pcm;
snd_pcm_open(&pcm, "default", SND_PCM_STREAM_PLAYBACK, 0);

// Set format
snd_pcm_set_params(pcm, SND_PCM_FORMAT_S16, ...);

// Write audio (convert f32 → s16 in a loop)
for each chunk of Kokoro output:
    convert f32 samples to s16
    snd_pcm_writei(pcm, s16_buffer, sample_count)
```

## Code Instructions

### Language and Standards
- C11 / gnu11 (`-std=gnu11`)
- Build with `-Wall -Wextra -Werror` (no warnings, no errors)
- Follow existing code style (project has `.indent.pro`)
- GPL-3.0+ licensed — all new files need proper headers

### Memory Management
- Use the project's helpers from `stringhandling.c`:
  - `allocMem(size)` — malloc wrapper
  - `dupeString(s)` — strdup wrapper
  - `initString(len)` — calloc wrapper for string buffers
- All Kokoro API allocations must be freed:
  - `SherpaOnnxDestroyOfflineTtsGeneratedAudio(audio)` — after each generation
  - `SherpaOnnxDestroyOfflineTts(tts)` — on shutdown

### Thread Safety
- `queue_guard` mutex protects all queue operations
- `runner_awake` condition variable wakes the kokoro thread
- `should_run` is `volatile int` — read without lock
- `stop_requested` is `volatile int` — read without lock
- The kokoro API call should be made **outside** the lock (like the original espeak calls)
- After kokoro API returns, re-acquire `queue_guard` before continuing

### Error Handling
- Audio device failures: detect stalled engine (retries without progress), auto-restart
- On unrecoverable failure: `_exit(3)` (not `exit()`, to avoid hanging in destructors)
- Queue allocation failures: log error, skip entry (don't crash)
- Kokoro model load failure: log error, report to user, allow retry

### Implementation Rules
1. **Do NOT modify:** `softsynth.c`, `signal.c`, `queue.c`, `cli.c` — these interface with speakup and the OS. Only modify `espeak.c` and `espeakup.h`.
2. **Do NOT change:** the threading model, queue structure, or condition variable usage.
3. **DO implement dual-engine dispatch in `espeak.c`:**
   - Kokoro is the **preferred** engine — use it when available and the language is supported.
   - espeak-ng is the **fallback** — use it when Kokoro dependencies aren't installed, Kokoro doesn't support the language, or Kokoro generation fails.
   - Both engines must coexist in the same codebase; don't remove espeak-ng support.
4. **DO keep:** the CLI options (`--default-voice`, `--alsa-volume`, `--debug`, etc.) — map them to Kokoro equivalents where applicable, keep espeak-ng equivalents for fallback.
5. **DO keep:** the systemd service, PID file, daemon mode behavior.

## Phase 1 Implementation Details

Phase 1 is about wiring Kokoro in as the primary voice engine while keeping espeak-ng as a fallback. Here's what that means:

### Files to modify
1. **`meson.build`** — add `sherpa-onnx` as an optional dependency alongside `espeak-ng`
2. **`src/espeakup.h`** — add `#include <sherpa-onnx/c-api/c-api.h>` alongside the existing espeak-ng include
3. **`src/espeak.c`** — add Kokoro initialization, dual-engine dispatch logic, and fallback chain

### Before/after for `meson.build`

**Before:**
```meson
espeak_dep = dependency('espeak-ng')
```

**After:**
```meson
espeak_dep = dependency('espeak-ng')
sherpa_onnx_dep = dependency('sherpa-onnx', required: false)  # optional
```

Both dependencies appear in the `executable()` call's dependencies array. If sherpa-onnx is missing, the build still succeeds (espeak-ng fallback is always available).

### Before/after for `src/espeakup.h`

**Before:**
```c
#include <espeak-ng/speak_lib.h>
```

**After:**
```c
#include <espeak-ng/speak_lib.h>
#include <sherpa-onnx/c-api/c-api.h>
```

Keep both includes. Update function declarations:
- `initialize_espeak()` → `initialize_engines()` (initializes both Kokoro and espeak-ng)
- `espeak_thread()` → `kokoro_thread()` (the main processing loop, now dispatching to either engine)

### Before/after for `src/espeak.c`

**Before:**
```c
#include <espeak-ng/speak_lib.h>
```

**After:**
```c
#include <espeak-ng/speak_lib.h>
#include <sherpa-onnx/c-api/c-api.h>
```

Keep both espeak-ng calls **and** add Kokoro calls. The dispatch logic in the thread function should:

1. Check if Kokoro is initialized and the language is supported → use Kokoro
2. If Kokoro is unavailable or language unsupported → fall back to espeak-ng

Kokoro API calls to add:
| Kokoro (sherpa-onnx) | Purpose |
|----------------------|---------|
| `SherpaOnnxCreateOfflineTts()` | Initialize Kokoro engine |
| `SherpaOnnxOfflineTtsGenerateWithConfig()` | Generate speech for text |
| `SherpaOnnxDestroyOfflineTts()` | Shutdown Kokoro engine |
| `SherpaOnnxDestroyOfflineTtsGeneratedAudio()` | Free generated audio buffer |

espeak-ng calls to keep (fallback path):
| espeak-ng | Purpose |
|-----------|---------|
| `espeak_Initialize()` | Already initialized in fallback path |
| `espeak_Synth()` | Already used for fallback |
| `espeak_Cancel()` | Already used for cancellation |
| `espeak_Terminate()` | Already called on shutdown |
| `espeak_SetParameter()` | Already used for parameter changes |
| `espeak_Synchronize()` | Already used for drain |

### Dependency setup
sherpa-onnx can be obtained as:
- System package: `apt install libsherpa-onnx-dev` (Debian/Ubuntu) or equivalent
- Meson subproject: add sherpa-onnx as a git subproject in `subprojects/`
- Manual install: build sherpa-onnx from source, point meson to it

espeak-ng remains a **required** dependency (fallback path).

### Dependency detection at runtime

At startup, attempt to initialize Kokoro. If it fails (model not found, library missing, etc.), log a warning and operate in espeak-ng-only mode. The fallback is seamless — the user doesn't need to change anything.

### Success criteria for Phase 1
- `ninja` builds with zero warnings and zero errors, with or without sherpa-onnx installed
- When sherpa-onnx is present: Kokoro generates speech for supported languages, espeak-ng handles the rest
- When sherpa-onnx is absent: espeak-ng handles all speech (full fallback)
- The kokoro thread can be initialized and shut down cleanly in both modes

## Kokoro API Reference

**Full reference implementation:**
https://github.com/k2-fsa/sherpa-onnx/blob/master/c-api-examples/kokoro-tts-en-c-api.c

**Model download:**
```bash
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models/kokoro-en-v0_19.tar.bz2
tar xf kokoro-en-v0_19.tar.bz2
```
Extracts `kokoro-en-v0_19/` containing `model.onnx`, `voices.bin`, `tokens.txt`, `espeak-ng-data/`.

**Initialization:**
```c
#include "sherpa-onnx/c-api/c-api.h"

SherpaOnnxOfflineTtsConfig config;
memset(&config, 0, sizeof(config));
config.model.kokoro.model = "./kokoro-en-v0_19/model.onnx";
config.model.kokoro.voices = "./kokoro-en-v0_19/voices.bin";
config.model.kokoro.tokens = "./kokoro-en-v0_19/tokens.txt";
config.model.kokoro.data_dir = "./kokoro-en-v0_19/espeak-ng-data";
config.model.num_threads = 2;

const SherpaOnnxOfflineTts *tts = SherpaOnnxCreateOfflineTts(&config);
if (!tts) { /* handle error */ }
```

**Generate speech:**
```c
SherpaOnnxGenerationConfig cfg = {0};
cfg.silence_scale = 0.2f;
cfg.sid = 0;        // speaker ID (0-10)
cfg.speed = 1.0;    // 1.0 = normal, >1.0 = faster

const SherpaOnnxGeneratedAudio *audio =
    SherpaOnnxOfflineTtsGenerateWithConfig(tts, text, &cfg, NULL, NULL);

// audio->samples: float[] (f32 PCM, 24kHz mono)
// audio->n: sample count
// audio->sample_rate: 24000

// Cleanup (MUST call this):
SherpaOnnxDestroyOfflineTtsGeneratedAudio(audio);
```

**Shutdown:**
```c
SherpaOnnxDestroyOfflineTts(tts);
```

**Voice ID mapping:**
```
0=af, 1=af_bella, 2=af_nicole, 3=af_sarah, 4=af_sky,
5=am_adam, 6=am_michael, 7=bf_emma, 8=bf_isabella,
9=bm_george, 10=bm_lewis
```

## Kokoro Voice Reference

| ID | Name | Description |
|----|------|-------------|
| 0 | af | Default female (neutral) |
| 1 | af_bella | Female, clear and expressive |
| 2 | af_nicole | Female, warm tone |
| 3 | af_sarah | Female, professional |
| 4 | af_sky | Female, energetic |
| 5 | am_adam | Male, deep |
| 6 | am_michael | Male, standard |
| 7 | bf_emma | Female, British |
| 8 | bf_isabella | Female, British |
| 9 | bm_george | Male, British |
| 10 | bm_lewis | Male, British |

Voice name → ID mapping should be implemented in `espeak.c` as a lookup table.

## Key Files to Read (in order)

1. `AGENTS.md` — this file (project overview and architecture)
2. `ROADMAP.md` — phased task list
3. `src/espeakup.h` — data structures and thread interfaces
4. `src/softsynth.c` — speakup protocol parsing
5. `src/espeak.c` — dual-engine TTS dispatch (your main work file)
6. `src/espeakup.c` — main(), thread creation, lifecycle
7. **sherpa-onnx C API example:** https://github.com/k2-fsa/sherpa-onnx/blob/master/c-api-examples/kokoro-tts-en-c-api.c — full working C program

## Common Pitfalls

1. **Forgetting to free Kokoro audio:** Every `SherpaOnnxOfflineTtsGenerateWithConfig()` returns an audio pointer that must be freed with `SherpaOnnxDestroyOfflineTtsGeneratedAudio()`. Leaks will accumulate.
2. **Holding the lock during Kokoro calls:** The original code releases `queue_guard` before calling espeak functions. Do the same for Kokoro calls to avoid blocking the softsynth thread.
3. **Ignoring ALSA sample rate mismatch:** Kokoro outputs 24kHz. If ALSA is configured for 48kHz, audio will play at half speed. Resample or configure ALSA for 24kHz.
4. **Not handling Kokoro's G2P:** sherpa-onnx handles G2P internally. Pass plain text, not IPA. Don't try to phonemize yourself.
5. **Thread safety on Kokoro API:** The sherpa-onnx Kokoro API may not be thread-safe. Use a single kokoro thread (don't parallelize generation).
6. **Removing espeak-ng entirely:** espeak-ng is the fallback, not the thing being replaced. Both engines must coexist.
7. **Language support detection:** Kokoro only supports a limited set of languages (primarily English and British English via the 11 voice IDs). For unsupported languages, the fallback to espeak-ng must be seamless.

## References

- **sherpa-onnx repo:** https://github.com/k2-fsa/sherpa-onnx
  - C API examples: `c-api-examples/kokoro-tts-en-c-api.c` (the reference implementation)
  - Kokoro model download: https://github.com/k2-fsa/sherpa-onnx/releases/tag/tts-models
- **Kokoro model:** https://github.com/hexgrad/kokoro (model weights)
- **Original espeakup (upstream):** https://github.com/linux-speakup/espeakup
- **linux-voice (this project):** https://github.com/v5y5mhb6c2-pixel/linux-voice
- **speakup kernel module:** built into Linux kernel (`CONFIG_SPEAKUP`)
- **Kokoro architecture:** https://github.com/pguso/kokoro/blob/main/ARCHITECTURE.md
