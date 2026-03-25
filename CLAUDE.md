# CLAUDE.md

## Project Overview

Claude Code skill that provides voice-to-text input via `/voice`. Records audio through a tkinter GUI, transcribes locally with faster-whisper, and returns the transcript as the user's prompt.

## Tech Stack

- **Python 3.10+** with venv at `~/.claude/venvs/voice`
- **faster-whisper** for local Whisper transcription (CPU, int8)
- **sounddevice** for microphone capture
- **tkinter** for the recording GUI
- **pytest** for testing, **black** for formatting, **flake8** for linting, **mypy** for type checking

## Project Structure

- `scripts/voice.py` — GUI, recording, CLI entry point, config, locking
- `scripts/whisper.py` — Whisper model caching, preloading, transcription (WAV and raw audio)
- `scripts/chunks.py` — SilenceDetector and ChunkManager for background chunked transcription
- `tests/test_voice.py` — tests for GUI, main(), config, locking
- `tests/test_chunks.py` — tests for SilenceDetector and ChunkManager
- `tests/conftest.py` — mocks `sounddevice` since PortAudio is unavailable in CI
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

- **Single-instance lock**: PID-based lockfile at `~/.config/claude-skill-voice/voice.lock` (or `%APPDATA%` on Windows) prevents concurrent recording sessions. Uses `OpenProcess` on Windows (not `os.kill` which sends CTRL_C_EVENT).
- **Persistent config**: Window position and transcription time ratio stored in `~/.config/claude-skill-voice/settings.json`. Transcribe ratio is used to show an estimated transcription time in the GUI.
- **stdout is the transcript**: The script prints `[BEGIN version=X.Y.Z]` before transcript text and `[END]` after, or `[CANCEL]` if the user cancels. All progress/status goes to stderr via `logging`. Claude Code reads stdout as the user's input. With `--stream`, chunks are printed to stdout as they complete between `[BEGIN ...]` and `[END]`. If the user cancels after streaming started, `[CANCEL]` replaces `[END]`.
- **Chunked transcription**: Audio is transcribed in the background during recording. Two triggers: (1) Pause button flushes accumulated audio as a chunk. (2) SilenceDetector detects gaps in speech and triggers automatic flushes. ChunkManager coordinates a single worker thread that processes chunks sequentially via a queue, reusing the cached Whisper model.
- **Model caching**: `whisper._get_or_create_model()` loads the Whisper model once and reuses it across all chunk transcriptions. The model is preloaded in a background thread during recording startup.
- **Done transition**: When Done is pressed, the recording window transitions to "Transcribing:" mode — Pause/Done are grayed out, a progress bar shows remaining transcription, and the window closes when the worker finishes. No separate progress window.
- **Silence detection**: SilenceDetector tracks consecutive silent audio callbacks. When silence exceeds a duration threshold (default 2.0s) and minimum chunk size (15s) is met, it returns the chunk index at silence onset + 0.5s buffer. The audio callback sets a flag; the tkinter level timer checks it and calls flush from the main thread.
- **Silence trimming**: Before transcription, `trim_silence()` scans each chunk for the first and last speech-level samples, keeping 0.5s buffer on each side. Chunks that are entirely silence are skipped. This prevents Whisper from spending excessive time on ambient noise.
- **Transcript sanitization**: `_sanitize_transcript()` replaces characters that can't be encoded in the console's charset (e.g., Georgian script hallucinated from noise) to prevent UnicodeEncodeError crashes on Windows.
- **Direct audio transcription**: Chunked audio is passed directly to faster-whisper as float32 numpy arrays — no temp WAV files. The WAV file path is only used for `--wav-input` mode.
- **Versioning**: SemVer (`X.Y.Z`). Version is defined in `voice.py __version__` and `pyproject.toml`. Bump patch (Z) for tweaks, minor (Y) for most changes, major (X) reserved for 1.0.0. Version is logged in debug preamble and emitted in `[BEGIN version=X.Y.Z]` on stdout.

## Testing Notes

- `sounddevice` is mocked in `conftest.py` because it requires PortAudio system libraries not available in CI.
- Whisper model globals (`_whisper_preload_started`, `_cached_model`) must be reset between tests via the `_reset_whisper_globals` fixture to prevent cross-test contamination.
- SilenceDetector and ChunkManager use synthetic numpy arrays (not real audio) for unit tests. Real transcription quality is verified via manual testing.
- Tests cover: config persistence, path utilities, WAV save/load, lock file logic, transcription, CLI arg parsing, silence detection, chunk boundary tracking, worker thread lifecycle, streaming output.

## Deployment

Installed in `~/.claude/skills/voice`. The skill is globally available across all Claude Code sessions.
