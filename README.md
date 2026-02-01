# whisperer

Linux push-to-talk dictation: hold a key to record, transcribe with OpenAI, and inject text into the focused app.

This repo starts with a minimal CLI. Main flow is push-to-talk, with extra modes for:

- one-shot dictation (fixed duration)
- looped dictation in chunks (record N seconds, transcribe, inject, repeat)
- injection via uinput (ydotool) with clipboard+paste fallback for Unicode

## Requirements

System packages (Ubuntu LTS example):

```sh
sudo apt update
sudo apt install pipewire-bin wl-clipboard ydotool curl python3 python3-evdev
```

Note: on some Ubuntu LTS versions, `ydotoold` is missing from the packaged `ydotool`. If so, build from source:

```sh
sudo apt install git build-essential cmake scdoc libevdev-dev libudev-dev
git clone https://github.com/ReimuNotMoe/ydotool.git
cd ydotool
mkdir -p build
cd build
cmake ..
make -j"$(nproc)"
sudo make install
```

`python3-evdev` is required for push-to-talk.

You also need access to `/dev/uinput` for ydotool. A common setup:

```sh
sudo modprobe uinput
echo uinput | sudo tee /etc/modules-load.d/uinput.conf
echo 'KERNEL=="uinput", MODE="0660", GROUP="input", OPTIONS+="static_node=uinput"' | sudo tee /etc/udev/rules.d/99-uinput.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo usermod -aG input "$USER"
```

Log out and back in after adding the group.

## Install

Option A (recommended): keep the repo and add a symlink into your PATH.

```sh
git clone https://github.com/Poor-Plebs/whisperer.git
cd whisperer
mkdir -p ~/.local/bin
ln -sf "$(pwd)/bin/whisperer" ~/.local/bin/whisperer
```

Option B: add the repo `bin/` folder to your PATH in your shell profile.

```shell
export PATH="$HOME/path/to/whisperer/bin:$PATH"
```

Install systemd user service (manages `ydotoold`) and config placement:

```sh
whisperer --install-service
```

After running --install-service, add your OpenAI API key in:

```sh
~/.config/whisperer/config
```

## Quick start

Run one-shot dictation (default 8s):

```sh
whisperer --duration 8
```

Run chunked dictation (keeps recording 4s chunks until Ctrl+C):

```sh
whisperer --loop --chunk-seconds 4
```

Push-to-talk (hold key to record, default: Right Alt):

```sh
whisperer --ptt
```

If Right Alt is used for AltGr on your layout, pick another key:

```sh
whisperer --ptt --ptt-key KEY_F9
```

Some layouts report Right Alt as `KEY_ALTGR` instead of `KEY_RIGHTALT`:

```sh
whisperer --ptt --ptt-key KEY_ALTGR
```

List keyboard devices (for --ptt-device):

```sh
whisperer --ptt-list-devices
```

Prefer a specific device by name fragment:

```sh
whisperer --ptt --ptt-device-match SONiX
```

Show status and key events:

```sh
whisperer --ptt --debug
```

## Notes

- Injection strategy:
  - If transcript is plain ASCII, type it via ydotool.
  - Otherwise, copy to clipboard and paste via Ctrl+V.

## Configuration

- Config file: `~/.config/whisperer/config`
- CLI flags override config file values.
- The script will auto-start `ydotoold` unless `WHISPERER_START_DAEMON=0`.
  - Set `WHISPERER_START_DAEMON=0` if you manage `ydotoold` via systemd or run it manually.

## Advanced

See all options:

```sh
whisperer --help-advanced
```

Options are grouped by context (install/maintenance, dictation, audio/gating/debug).

## Troubleshooting

Quick diagnostics:

```sh
whisperer --doctor
```

Auto-fix common issues:

```sh
whisperer --doctor --fix
```

Paste not working in terminals?

- Many terminals use Ctrl+Shift+V for paste. Set:
  - `WHISPERER_PASTE_KEYS=ctrl+shift+v`

Debug output:

- `WHISPERER_DEBUG=1` or `--debug` shows status and key events.

Cleanup:

```sh
whisperer --uninstall
```

## RMS notes

RMS is a simple average loudness measure (normalized 0.0–1.0). The RMS gate only runs when `--rms-threshold` is set > 0 (otherwise no sampling happens).

Typical values:

- ~0.000–0.005: near silence
- ~0.005–0.02: quiet speech / ambient noise
- ~0.02–0.08: normal speech
- ~0.08–0.2+: loud speech / music

Environment variables (also supported in config file):

- `OPENAI_API_KEY` (required)
- `WHISPERER_MODEL` (default: `whisper-1`). Other options: `gpt-4o-mini-transcribe`, `gpt-4o-transcribe`.
- `WHISPERER_RATE` (default: `16000`)
- `WHISPERER_CHANNELS` (default: `1`)
- `WHISPERER_MODE` (`auto` | `type` | `paste`, default: `auto`)
- `WHISPERER_START_DAEMON` (`1` to auto-start ydotoold, default: `1`)
- `WHISPERER_SOCKET` (default: `/tmp/.ydotool_socket`)
- `WHISPERER_PASTE_KEYS` (`ctrl+v` or `ctrl+shift+v`, default: `ctrl+v`)
- `WHISPERER_NO_INJECT` (`1` to disable injection, default: `0`)
- `WHISPERER_RMS_THRESHOLD` (default: `0`, disabled)
- `WHISPERER_PRINT_RMS` (`1` to print RMS, default: `0`)
- `WHISPERER_RMS_SAMPLE_SECONDS` (default: `0.5`)
- `WHISPERER_DEBUG` (`1` to enable debug output, default: `0`)
- `WHISPERER_PTT` (`1` to enable push-to-talk, default: `0`)
- `WHISPERER_PTT_KEY` (default: `KEY_RIGHTALT`)
- `WHISPERER_PTT_DEVICE` (default: auto-detect)
- `WHISPERER_PTT_DEVICE_MATCH` (default: empty; substring match)

Run `./bin/whisperer --help` for common options and `whisperer --help-advanced` for all options.
