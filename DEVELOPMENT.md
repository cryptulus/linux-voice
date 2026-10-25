# Development Guide

This guide covers setup, build configuration, and development workflow for linux-voice.

## Prerequisites

### System packages

**Debian/Ubuntu:**
```bash
sudo apt install meson ninja-build ronn libasound2-dev libespeak-ng-dev
# sherpa-onnx is optional — see below
```

**Arch Linux:**
```bash
sudo pacman -S meson ninja ronn alsa-lib espeak-ng
# sherpa-onnx: `yay -S sherpa-onnx` (AUR) or build from source
```

**Fedora:**
```bash
sudo dnf install meson ninja-build ronn alsa-lib-devel espeak-ng-devel
# sherpa-onnx: build from source
```

### espeak-ng (required fallback)

espeak-ng is the fallback TTS engine. It must always be installed.

### sherpa-onnx (optional — preferred engine)

sherpa-onnx provides the Kokoro TTS C API. When installed, linux-voice uses Kokoro as the preferred voice engine. When absent, it falls back to espeak-ng automatically.

#### Option 1: Pre-built wheel (Python, easiest)

```bash
pip install sherpa-onnx
```

This installs Python bindings and the C library. The C library location is typically:
```
~/.local/lib/python*/site-packages/sherpa_onnx/lib/
```

For meson, you'll need to point it to this location or install the C library system-wide.

#### Option 2: Meson subproject (recommended for development)

Add sherpa-onnx as a meson subproject:

```bash
mkdir -p subprojects
git clone https://github.com/k2-fsa/sherpa-onnx.git subprojects/sherpa-onnx
```

Then add to `meson.build`:
```meson
sherpa_onnx_dep = subproject('sherpa-onnx').get_variable('sherpa_onnx_dep')
```

#### Option 3: System install from source

```bash
git clone https://github.com/k2-fsa/sherpa-onnx.git
cd sherpa-onnx
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
sudo make install
sudo ldconfig
```

This installs:
- `/usr/local/include/sherpa-onnx/c-api/c-api.h`
- `/usr/local/lib/libsherpa-onnx.so`

### Kokoro model files

Download the Kokoro model from sherpa-onnx releases (approx 100MB):

```bash
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models/kokoro-en-v0_19.tar.bz2
tar xf kokoro-en-v0_19.tar.bz2
rm kokoro-en-v0_19.tar.bz2
```

This extracts a `kokoro-en-v0_19/` directory containing:
- `model.onnx` — the Kokoro neural model
- `voices.bin` — voice embeddings for all 11 speakers
- `tokens.txt` — token vocabulary
- `espeak-ng-data/` — G2P data (required by Kokoro)

The model directory path goes in the Kokoro config:
```c
config.model.kokoro.model = "./kokoro-en-v0_19/model.onnx";
config.model.kokoro.voices = "./kokoro-en-v0_19/voices.bin";
config.model.kokoro.tokens = "./kokoro-en-v0_19/tokens.txt";
config.model.kokoro.data_dir = "./kokoro-en-v0_19/espeak-ng-data";
```

## Build Configuration

### meson setup

The project uses meson for building. Standard workflow:

```bash
# Configure
meson setup . ./build

# Build
cd ./build
ninja

# Install (requires root)
sudo ninja install

# Uninstall
sudo ninja uninstall
```

### Build options

From `meson_options.txt`:

| Option | Default | Description |
|--------|---------|-------------|
| `man` | auto | Build man page with ronn |
| `systemd` | auto | Enable systemd service installation |

Enable/disable:
```bash
meson setup . ./build -Dman=enabled -Dsystemd=disabled
```

### Adding sherpa-onnx to meson.build

The meson dependency declaration depends on how you installed sherpa-onnx:

**System package:**
```meson
sherpa_onnx_dep = dependency('sherpa-onnx', required: false)  # optional
```

**Custom install location:**
```meson
sherpa_onnx_dep = dependency('sherpa-onnx',
    required: false,
    dirs: ['/path/to/sherpa-onnx/lib']
)
```

**Meson subproject:**
```meson
# In subprojects/sherpa-onnx/meson.build, the subproject exports:
sherpa_onnx_dep = subproject('sherpa-onnx').get_variable('sherpa-onnx_dep')
```

## Development Workflow

### 1. Make changes

Edit source files in `src/`. The key file for dual-engine dispatch is `src/espeak.c`.

### 2. Build

```bash
cd ./build
ninja
```

Build must succeed with zero warnings and zero errors, with or without sherpa-onnx installed.

### 3. Test manually

```bash
# Start in debug mode (foreground)
sudo linux-voice --debug --default-voice=af_bella

# In another terminal, send a test command
echo -ne '\x01s5s' | sudo tee /dev/softsynth  # Set rate
echo -ne 'hello world' | sudo tee /dev/softsynth  # Speak text
```

### 4. Test with speakup

```bash
# Load speakup kernel module
sudo modprobe speakup_soft

# Configure speakup to use software synthesizer
echo 1 | sudo tee /proc/sys/speakup/synth

# Start linux-voice
sudo linux-voice --debug

# Speakup should now use linux-voice for speech output
# With Kokoro available: speech via Kokoro (preferred)
# Without Kokoro: speech via espeak-ng (fallback)
```

## Debugging

### Enable debug output

```bash
linux-voice --debug
```

Debug mode runs in the foreground and prints diagnostic messages to stderr.

### Check speakup communication

```bash
# Verify /dev/softsynth exists
ls -l /dev/softsynth

# Check if speakup is configured
cat /proc/sys/speakup/synth

# Monitor speakup commands (raw binary)
sudo hexdump -C /dev/softsynth
```

### Common issues

**Build fails: "sherpa-onnx not found"**
- sherpa-onnx is optional — the build should succeed without it (espeak-ng fallback)
- If you want Kokoro, verify it is installed: `pkg-config --libs sherpa-onnx`
- Check include path: `pkg-config --cflags sherpa-onnx`
- If using subproject, verify `subprojects/sherpa-onnx/` exists

**Build fails: "c-api.h not found"**
- sherpa-onnx C API header location varies by install method
- Check: `find /usr -name "c-api.h" 2>/dev/null`
- Add include path to meson.build if needed

**No audio output:**
- Check ALSA device: `aplay -l`
- Test ALSA: `speaker-test -t wav`
- Check kokoro model loads: verify model file paths
- Check sample rate mismatch (Kokoro 24kHz vs ALSA 48kHz)

**Kokoro synthesis fails:**
- Verify model files exist and are valid ONNX
- Check text is plain ASCII (not IPA)
- Check Kokoro version compatibility with sherpa-onnx version
- If Kokoro fails at runtime, the system falls back to espeak-ng automatically

## Adding New Voices

Kokoro ships with 11+ voices. To add support for new voices:

1. Obtain voice pack (`.bin` file with speaker embeddings)
2. Add voice name → ID mapping in `espeak.c`
3. Update README.md voice table
4. Update `--default-voice` CLI option handling

## Contributing

1. Fork this repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Make changes, ensure `ninja` builds with no warnings
4. Test with speakup or manual `/dev/softsynth` commands
5. Commit with descriptive message
6. Push and open a pull request

## License

GPL-3.0+. Based on the original espeakup project.
