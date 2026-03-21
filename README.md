# claude-skill-voice

Voice-to-text input skill for Claude Code. Records audio from your microphone, transcribes locally with [faster-whisper](https://github.com/SYSTRAN/faster-whisper), and feeds the transcript back as your prompt.

## Prerequisites

- Python 3.10+
- `tkinter` (used for the recording GUI)
  - **Fedora**: `sudo dnf install python3-tkinter`
  - **Ubuntu/Debian**: `sudo apt install python3-tk`
  - **Windows/macOS**: included with standard Python
- A working microphone

## Installation

Clone (or add as a git submodule) into your Claude Code skills directory:

```bash
# As a submodule in your global skills directory
cd ~/.claude/skills
git submodule add https://github.com/jewzaam/claude-skill-voice.git voice
```

Then install the Python dependencies:

```bash
cd ~/.claude/skills/voice
make install
```

This creates a local `.venv` virtual environment and installs `faster-whisper`, `sounddevice`, and `numpy`.

To verify everything works:

```bash
make check
```

This runs formatting, linting, type checking, and tests.

## Usage

In a Claude Code session, type:

```
/voice
```

A GUI window appears with **Cancel**, **Pause**, and **Done** buttons. Speak into your microphone, then press **Done** (or hit Enter). The audio is transcribed locally and the resulting text is treated as if you had typed it.

### Keyboard shortcuts

| Key     | Action         |
|---------|----------------|
| Enter   | Done recording |
| Space   | Pause/Resume   |
| Escape  | Cancel         |

### Whisper model selection

The skill uses the `base` model by default. You can request a different model when invoking `/voice` by telling Claude which model to use:

- `tiny` -- fastest, least accurate
- `base` -- default balance of speed and accuracy
- `small` -- slower, more accurate

The first run downloads the selected model automatically (requires internet).

### Listing audio devices

To see available input devices, run the script directly:

```bash
~/.claude/skills/voice/.venv/bin/python ~/.claude/skills/voice/scripts/voice.py --list-devices
```

## How it works

1. `scripts/voice.py` opens a tkinter GUI and records audio via `sounddevice`
2. Audio is saved to a temporary WAV file
3. `faster-whisper` transcribes the WAV locally on CPU (int8 quantization)
4. The transcript is printed to stdout, which Claude Code reads as your input
5. The temp WAV file is cleaned up

A PID-based lockfile prevents multiple simultaneous recording sessions. Window position and transcription time estimates are persisted in `~/.config/claude-skill-voice/settings.json`.

## Development

```bash
make install-dev    # install with dev dependencies (pytest, black, flake8, mypy)
make format         # run black formatter
make lint           # run flake8
make typecheck      # run mypy
make test           # run pytest
make coverage       # run pytest with coverage report
make check          # all of the above
```

## License

Apache License 2.0. See [LICENSE](LICENSE).
