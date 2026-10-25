# linux-voice

linux-voice is a connector for the Linux kernel [speakup](http://linux-speakup.org) screen reader that uses the [Kokoro](https://github.com/hexgrad/kokoro) neural TTS model as its preferred voice engine, with [espeak-ng](https://github.com/espeak-ng/espeak-ng) as a fallback.

It provides the same speakup interface as the original espeakup connector, delivering high-quality neural speech through Kokoro when available, and falling back to espeak-ng when Kokoro dependencies aren't installed or a language has no Kokoro voice.

## What it does

linux-voice bridges the gap between speakup (the kernel-level screen reader) and Kokoro (a high-quality neural TTS engine). When speakup needs to speak text — whether it's reading the cursor position, announcing menus, or reading back typed characters — linux-voice receives those commands and produces speech audio through your sound card.

## How it works

```
speakup (kernel) → /dev/softsynth → linux-voice → Kokoro (ONNX model) → ALSA → speakers
                                            ↘ espeak-ng (fallback) → ALSA → speakers
```

linux-voice reads speakup's binary command protocol from `/dev/softsynth`, interprets commands for voice, rate, volume, pitch, and text-to-speak, then dispatches to **Kokoro** (via sherpa-onnx) as the preferred engine. If Kokoro is unavailable or doesn't support the requested language, it falls back to **espeak-ng** seamlessly — the user doesn't need to change anything.

## Features

- **Full speakup compatibility** — works with any speakup configuration and voice
- **Dual-engine TTS** — Kokoro for preferred quality, espeak-ng as reliable fallback
- **Kokoro voice library** — access to 11+ high-quality English voices (af_bella, af_nicole, af_sarah, am_adam, bf_emma, and more)
- **espeak-ng fallback** — wide language support when Kokoro isn't available
- **Real-time synthesis** — low-latency speech generation suitable for screen reader use
- **Configurable parameters** — rate, volume, pitch, voice selection via speakup commands or CLI
- **ALSA volume control** — optional direct ALSA mixer integration for volume changes
- **Daemon mode** — runs as a background service, respawning on crash
- **Debug mode** — foreground operation for development and troubleshooting

## Requirements

- Linux kernel with speakup support (`CONFIG_SPEAKUP`)
- speakup configured with software synthesizer mode
- **espeak-ng** — required fallback TTS engine
- **ALSA library** — audio playback
- **sherpa-onnx** (optional) — provides Kokoro TTS via C API for preferred quality
- **Kokoro model files** (optional, downloaded at runtime) — needed only if using Kokoro

## Installation

### From source

```bash
# Install system dependencies
sudo apt install meson ninja-build libasound2-dev ronn  # Debian/Ubuntu
# or: sudo pacman -S meson ninja-build alsa-lib ronn     # Arch Linux

# Clone and build
git clone https://github.com/v5y5mhb6c2-pixel/linux-voice.git
cd linux-voice
meson setup . ./build
cd ./build
ninja
sudo ninja install

# (Optional) Download Kokoro model for preferred quality
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models/kokoro-en-v0_19.tar.bz2
tar xf kokoro-en-v0_19.tar.bz2
```

### From package

(Packages are being prepared for major distributions. In the meantime, build from source.)

## Usage

```bash
# Start in foreground (debug mode)
linux-voice --debug

# Start as daemon with default voice
linux-voice --default-voice=af_bella

# Start with ALSA volume control
linux-voice --alsa-volume --default-voice=af_nicole
```

### Command-line options

| Option | Description |
|--------|-------------|
| `--pid-path=path` | Path for the PID file (default: `/run/linux-voice.pid`) |
| `--default-voice=name` | Kokoro voice name (e.g., `af_bella`, `am_adam`, `bf_emma`) |
| `--alsa-volume` | Enable direct ALSA mixer control for volume changes |
| `--debug` | Run in foreground instead of daemonizing |
| `--help` | Show help and exit |
| `--version` | Show version and exit |

### Voice selection

Kokoro ships with multiple voices identified by speaker ID:

| ID | Voice | Description |
|----|-------|-------------|
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

Set the default voice with `--default-voice=af_bella` or change it at runtime through speakup commands.

## Starting Up

linux-voice should be started after speakup is configured for software synthesizer mode and `/dev/softsynth` exists. The systemd service handles this automatically:

```bash
sudo systemctl enable --now linux-voice
```

## Getting Started for Developers

1. Read `AGENTS.md` — architecture, speakup protocol, coding standards
2. Read `DEVELOPMENT.md` — dependency setup, build configuration, debugging
3. Read `ROADMAP.md` — phased task list
4. Start with Phase 1: wire Kokoro in as the primary voice engine

## Acknowledgements

- Marc Mulcahy — original speakup to TTSynth connector
- Kirk Reiser — speakup screen reader author
- Jonathan Duddington — espeak-ng author
- William Hubbs — espeakup connector author
- hexgrad — Kokoro model authors
- Xiaomi — sherpa-onnx authors

## License

GNU General Public License v3 or later. Based on the original espeakup project, also GPL-3.0+.
