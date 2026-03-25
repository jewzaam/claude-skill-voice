---
name: voice
description: "Trigger on `/voice` -- voice-to-text input for Claude Code. Records audio from the microphone, transcribes locally with faster-whisper, and returns the transcript as the user's prompt."
---

# Voice-to-Text Input

Record audio from the user's microphone, transcribe it locally, and treat the transcript as their message.

## Steps

1. **Run the voice script** via Bash with `run_in_background: true` (recording has no time limit — the user controls when to stop via the GUI):
   ```
   ~/.claude/venvs/voice/bin/python <skill_base_dir>/scripts/voice.py --stream --debug 2>~/.config/claude-skill-voice/debug.log
   ```
   - On Windows, use `~/.claude/venvs/voice/Scripts/python.exe` instead of the Unix path above.
   - Use `run_in_background: true` on the Bash tool call. This avoids the 2-minute default timeout killing long recordings. You will be notified when the recording finishes.
   - The `2>` redirect is required because the Bash tool merges stderr into stdout, which would mix log messages into the transcript.
   - Replace `<skill_base_dir>` with the directory containing this SKILL.md file.
   - `--stream` and `--debug` are always included in the command above. Do not remove them.
     - `--stream` prints transcript chunks to stdout as they complete during recording. This gives the user visible feedback that transcription is working, and preserves partial transcript if the tool crashes mid-session.
     - `--debug` enables debug-level logging to stderr. The `2>` redirect captures it to `~/.config/claude-skill-voice/debug.log` (overwritten each session) for troubleshooting. This tool is a work in progress with sharp edges — the debug log is essential for triaging failures.
   - Optional flags:
     - `--model tiny|base|small` (default: `base`). Use the model the user specifies, or `base` if unspecified.
     - `--device DEVICE_ID` to select a specific audio input device.
     - `--list-devices` to show available input devices and exit.

2. **Wait for completion** -- after launching in the background, tell the user that recording has started and they should press Done (or Enter) when finished. You will be automatically notified when the background task completes. Then read the output file to get the transcript.

3. **Parse the output** -- stdout uses marker lines to delimit the transcript:
   - `[CANCEL]` — the user cancelled. This can appear alone (cancelled before speaking) or after `[BEGIN]` and partial transcript lines (cancelled during transcription). In either case, **do nothing** — do not ask them to try again, do not comment on it. Discard any partial transcript. Just stop.
   - `[BEGIN]` — marks the start of transcript output. All lines between `[BEGIN]` and `[END]` are transcript text. Line breaks within the transcript indicate the user paused (via the Pause button or a natural gap in speech).
   - `[END]` — marks the end of transcript output. If there is no text between `[BEGIN]` and `[END]`, no speech was detected — tell the user and ask them to try again.

4. **Handle errors** (non-zero exit, no `[BEGIN]` line):
   - If stderr contains "already running", tell the user: "A voice recording session is already in progress. Please finish that session before starting a new one."
   - For other failures (e.g., `ModuleNotFoundError`, missing Python executable, or import errors), tell the user: "Voice capture failed. Make sure you've run `make install` in your skills/voice directory (e.g., `cd ~/.claude/skills/voice && make install`) to create the virtual environment and install dependencies." Also suggest checking microphone permissions if the error is not dependency-related.

5. **Respond to the transcript** -- treat the captured text as if the user had typed it directly. Do not quote or parrot it back; just act on it naturally.
