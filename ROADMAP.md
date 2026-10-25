# linux-voice — Project Roadmap

## Phase 1: Build Infrastructure

**Goal:** Wire Kokoro in as the primary voice engine while keeping espeak-ng as a fallback.

### Files to modify

**1. `meson.build`** — Add sherpa-onnx as an optional dependency alongside espeak-ng

```meson
# Add near the top, after other dependencies:
espeak_dep = dependency('espeak-ng')  # required fallback
sherpa_onnx_dep = dependency('sherpa-onnx', required: false)  # optional preferred

# Add to dependencies array in executable():
dependencies: [alsa_dep, threads_dep, math_dep, espeak_dep, sherpa_onnx_dep]
```

If sherpa-onnx is not installed, the build still succeeds (espeak-ng fallback is always available).

**2. `src/espeakup.h`** — Add sherpa-onnx include alongside espeak-ng

```c
// Add alongside existing includes:
#include <espeak-ng/speak_lib.h>
#include "sherpa-onnx/c-api/c-api.h"
```

**3. `src/espeak.c`** — Add Kokoro API alongside espeak-ng calls

- Add `#include "sherpa-onnx/c-api/c-api.h"` alongside existing espeak-ng include
- Add Kokoro initialization (`SherpaOnnxCreateOfflineTts()`) alongside espeak-ng init
- Add Kokoro generation (`SherpaOnnxOfflineTtsGenerateWithConfig()`) alongside espeak-ng synth
- Add Kokoro cancellation alongside espeak-ng cancel
- Add Kokoro shutdown (`SherpaOnnxDestroyOfflineTts()`) alongside espeak-ng terminate

### Kokoro model setup

Download Kokoro model from sherpa-onnx releases:
```bash
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models/kokoro-en-v0_19.tar.bz2
tar xf kokoro-en-v0_19.tar.bz2
```
Extracts `kokoro-en-v0_19/` directory containing `model.onnx`, `voices.bin`, `tokens.txt`, `espeak-ng-data/`.

### Success criteria

- [ ] `meson setup . ./build` completes without error, with or without sherpa-onnx installed
- [ ] `ninja` builds with zero warnings and zero errors
- [ ] When sherpa-onnx is present: Kokoro can generate speech for a hardcoded string
- [ ] When sherpa-onnx is absent: espeak-ng handles all speech (full fallback)
- [ ] The kokoro thread can be initialized and shut down cleanly (no crashes, no leaks)
- [ ] Runtime Kokoro init failure logs a warning and falls back to espeak-ng seamlessly

### Verification

```bash
# Build
meson setup . ./build && cd ./build && ninja

# Test with a simple C program that loads Kokoro
# (Create a temporary test program in src/test_kokoro.c)
gcc -o test_kokoro test_kokoro.c $(pkg-config --cflags --libs sherpa-onnx)
./test_kokoro
# Should output: "Kokoro loaded successfully" and generate test.wav
```

---

## Phase 2: Core TTS Integration

**Goal:** Integrate Kokoro as the primary TTS engine in `src/espeak.c`, with espeak-ng as fallback.

### Files to modify

**1. `src/espeak.c`** — Full dual-engine integration

- Initialize Kokoro TTS with model path from config (attempt at startup)
- Implement dispatch logic: Kokoro first, espeak-ng fallback
- Map speakup text → Kokoro generation (preferred) or espeak-ng synth (fallback)
- Handle audio output (f32 PCM → ALSA s16)
- Implement speech cancellation for both engines
- Implement parameter setting (rate, volume) for both engines
- Handle error recovery (model reload, device restart, fallback activation)

### Implementation details

**Reference implementation:** https://github.com/k2-fsa/sherpa-onnx/blob/master/c-api-examples/kokoro-tts-en-c-api.c

**Kokoro initialization:**
```c
SherpaOnnxOfflineTtsConfig config;
memset(&config, 0, sizeof(config));
config.model.kokoro.model = "./kokoro-en-v0_19/model.onnx";
config.model.kokoro.voices = "./kokoro-en-v0_19/voices.bin";
config.model.kokoro.tokens = "./kokoro-en-v0_19/tokens.txt";
config.model.kokoro.data_dir = "./kokoro-en-v0_19/espeak-ng-data";
config.model.kokoro.num_threads = 2;
config.model.kokoro.speed = 1.0;
config.model.kokoro.device = "cpu";

tts = SherpaOnnxCreateOfflineTts(&config);
if (!tts) {
    fprintf(stderr, "Failed to create Kokoro TTS\n");
    return -1;
}
```

**Speech generation:**
```c
SherpaOnnxOfflineTtsGenerateConfig gen_config;
memset(&gen_config, 0, sizeof(gen_config));
gen_config.speaker_id = speaker_id;
gen_config.speed = speed;
gen_config.text = text_to_speak;

audio = SherpaOnnxOfflineTtsGenerateWithConfig(tts, &gen_config, NULL, NULL);
if (audio) {
    // Convert f32 to s16 and write to ALSA
    play_audio(audio->samples, audio->n, audio->sample_rate);
    SherpaOnnxDestroyOfflineTtsGeneratedAudio(audio);
}
```

### Success criteria

- [ ] speakup sends text, linux-voice speaks it through Kokoro with audible output (when Kokoro is available)
- [ ] When Kokoro is unavailable, espeak-ng produces audible output (fallback works)
- [ ] Voice changes work (speakup commands change voice in both engines)
- [ ] Rate changes work (speakup commands change speech speed in both engines)
- [ ] Flush works (0x18 byte stops current speech, clears buffer)
- [ ] No memory leaks (verify with valgrind if available)
- [ ] Kokoro init failure at startup does not crash — falls back to espeak-ng

---

## Phase 3: Parameter Mapping

**Goal:** Map espeakup's command parameters to Kokoro equivalents.

### Parameter mapping table

| espeakup param | speakup type | Kokoro equivalent | Notes |
|----------------|--------------|-------------------|-------|
| voice | voice name | speaker ID (0-10) | Implement lookup table |
| rate | `s` | `speed` (0.5-2.0) | 1.0 = normal |
| volume | `v` | post-processing gain | ALSA mixer or audio multiplier |
| pitch | `p` | N/A | VITS model limitation |
| frequency | `f` | N/A | VITS model limitation |
| range | `r` | N/A | VITS model limitation |
| punctuation | `b` | IPA tokens | Kokoro handles `.`, `!`, `,` |
| mark | `i` | N/A | Position marker, no audio output |
| pause | `P` | silence tokens | Stop current speech |

### Files to modify

**1. `src/espeak.c`** — Implement parameter handling

- Voice lookup table (name → ID)
- Speed parameter mapping
- Volume gain implementation
- Punctuation handling (verify Kokoro IPA tokens)
- Pause/flush implementation

### Success criteria

- [ ] All speakup commands produce expected behavior
- [ ] Voice switching is seamless (no restart needed)
- [ ] Rate changes are reflected in speech speed
- [ ] Volume changes are reflected in output volume
- [ ] Punctuation affects speech timing (periods cause pauses)

---

## Phase 4: Audio Output

**Goal:** Robust audio playback through ALSA.

### Files to modify

**1. `src/espeak.c`** — ALSA integration

- Create ALSA PCM handle
- Implement f32 → s16 conversion
- Handle ALSA buffer management
- Implement audio drain on speech completion
- Handle ALSA device failures

### Implementation details

**ALSA setup:**
```c
snd_pcm_t *pcm;
snd_pcm_open(&pcm, "default", SND_PCM_STREAM_PLAYBACK, 0);

// Set parameters for s16, mono, 24kHz (Kokoro output rate)
snd_pcm_set_params(pcm, SND_PCM_FORMAT_S16, ...);
```

**f32 → s16 conversion:**
```c
for (int i = 0; i < sample_count; i++) {
    float f = samples[i];
    // Clamp to [-1.0, 1.0]
    if (f > 1.0f) f = 1.0f;
    if (f < -1.0f) f = -1.0f;
    s16_buffer[i] = (int16_t)(f * 32767.0f);
}
```

**ALSA write loop:**
```c
int frames_written;
while (bytes_remaining > 0) {
    frames_written = snd_pcm_writei(pcm, s16_buffer, frames_to_write);
    if (frames_written == -EPIPE) {
        // Buffer underrun — recover
        snd_pcm_prepare(pcm);
        continue;
    }
    bytes_remaining -= frames_written * 2; // 2 bytes per s16 sample
}
```

### Success criteria

- [ ] Audio plays cleanly through all common ALSA configurations
- [ ] No glitches or underruns during speech
- [ ] Audio drains properly on speech completion
- [ ] Wedged device detection and auto-restart works
- [ ] Works with dmix, direct, and PulseAudio (via ALSA compat)

---

## Phase 5: CLI and Configuration

**Goal:** Complete command-line interface and configuration.

### Files to modify

**1. `src/espeak.c`** — Voice lookup table, model path config

**2. `src/cli.c`** — May need minor updates for new options

### Implementation details

**Voice name → ID mapping:**
```c
static const struct {
    const char *name;
    int id;
} voice_map[] = {
    {"af", 0},
    {"af_bella", 1},
    {"af_nicole", 2},
    {"af_sarah", 3},
    {"af_sky", 4},
    {"am_adam", 5},
    {"am_michael", 6},
    {"bf_emma", 7},
    {"bf_isabella", 8},
    {"bm_george", 9},
    {"bm_lewis", 10},
    {NULL, -1}
};

int voice_name_to_id(const char *name) {
    for (int i = 0; voice_map[i].name; i++) {
        if (strcmp(voice_map[i].name, name) == 0)
            return voice_map[i].id;
    }
    return 0; // Default to af
}
```

### Success criteria

- [ ] All CLI options work (`--default-voice`, `--alsa-volume`, etc.)
- [ ] Voice switching at runtime works seamlessly
- [ ] Model path configuration works (custom model locations)
- [ ] Debug output is informative and helpful

---

## Phase 6: Packaging and Distribution

**Goal:** Ready for end-user installation.

### Files to create/modify

- [ ] `scripts/download-models.sh` — Model download script
- [ ] `services/systemd/linux-voice.service.in` — Verify systemd service
- [ ] `doc/linux-voice.8.ronn` — Update man page
- [ ] `meson.build` — Install targets for model files
- [ ] CI configuration (GitHub Actions, etc.)

### Success criteria

- [ ] A user can install, download models, configure speakup, and have linux-voice working
- [ ] Systemd service starts automatically and respawns on crash
- [ ] Man page is complete and accurate
- [ ] Package recipes available (Arch PKGBUILD, Debian deb)

---

## Known Technical Considerations

### Kokoro's G2P pipeline
Kokoro requires text → IPA phonemes → token IDs → audio. The sherpa-onnx C API handles G2P internally — you pass plain text and get audio back. No manual G2P implementation needed.

### Parameter limitations
Kokoro's VITS architecture supports speed control but not direct pitch or frequency control. Pitch changes from speakup will need workarounds (e.g., speed modulation, voice selection) or may be acknowledged as limitations.

### Voice model size
Kokoro model files are ~100MB for the ONNX model + voice packs. This is a one-time download. Model caching should be implemented to avoid re-downloading.

### Thread safety
The sherpa-onnx Kokoro API may not be thread-safe for concurrent generation. The existing single-kokoro-thread model should be preserved.

### Memory management
Kokoro's C API returns `SherpaOnnxGeneratedAudio*` that must be freed with `SherpaOnnxDestroyOfflineTtsGeneratedAudio()`. All allocations must be properly freed to avoid leaks.
