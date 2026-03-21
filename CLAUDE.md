# CLAUDE.md

## Project Overview

Claude Code skill that provides voice-to-text input via `/voice`. Records audio through a tkinter GUI, transcribes locally with faster-whisper, and returns the transcript as the user's prompt.

## Tech Stack

- **Python 3.10+** with venv at `.venv/`
- **faster-whisper** for local Whisper transcription (CPU, int8)
- **sounddevice** for microphone capture
- **tkinter** for the recording GUI
- **pytest** for testing, **black** for formatting, **flake8** for linting, **mypy** for type checking

## Project Structure

- `scripts/voice.py` — single main module: recording GUI, audio capture, transcription, config, locking
- `tests/` — unit tests; `conftest.py` mocks `sounddevice` since PortAudio is unavailable in CI
- `SKILL.md` — skill definition that Claude Code reads when `/voice` is invoked
- `Makefile` — all dev commands; venv-aware (`VENV_DIR`, auto-detects Windows vs Unix Python path)

## Key Commands

```bash
make check          # format + lint + typecheck + test + coverage (default target)
make install        # create venv and install runtime deps
make install-dev    # install with dev deps (pytest, black, flake8, mypy)
make format         # black
make lint           # flake8
make typecheck      # mypy
make test           # pytest
make coverage       # pytest --cov
```

## Architecture Notes

- **Single-instance lock**: PID-based lockfile at `~/.config/claude-skill-voice/voice.lock` (or `%APPDATA%` on Windows) prevents concurrent recording sessions.
- **Persistent config**: Window position and transcription time ratio stored in `~/.config/claude-skill-voice/settings.json`. Transcribe ratio is used to show an estimated transcription time in the GUI.
- **stdout is the transcript**: The script prints only the final transcript to stdout. All progress/status goes to stderr via `logging`. Claude Code reads stdout as the user's input.
- **Temp WAV cleanup**: Audio is saved to a temp WAV, transcribed, then deleted in a `finally` block.

## Testing Notes

- `sounddevice` is mocked in `conftest.py` because it requires PortAudio system libraries not available in CI.
- Tests cover: config persistence, path utilities, WAV save/load, lock file logic, transcription, CLI arg parsing.

## Deployment

Installed as a git submodule in `~/.claude/skills/voice`. The skill is globally available across all Claude Code sessions.
