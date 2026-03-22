---
name: voice
description: "Trigger on `/voice` -- voice-to-text input for Claude Code. Records audio from the microphone, transcribes locally with faster-whisper, and returns the transcript as the user's prompt."
---

# Voice-to-Text Input

Record audio from the user's microphone, transcribe it locally, and treat the transcript as their message.

## Steps

1. **Run the voice script** via Bash with `run_in_background: true` (recording has no time limit — the user controls when to stop via the GUI):
   ```
   ~/.claude/venvs/voice/bin/python <skill_base_dir>/scripts/voice.py 2>/dev/null
   ```
   - On Windows, use `~/.claude/venvs/voice/Scripts/python.exe` instead of the Unix path above.
   - Use `run_in_background: true` on the Bash tool call. This avoids the 2-minute default timeout killing long recordings. You will be notified when the recording finishes.
   - `2>/dev/null` suppresses stderr because the Bash tool merges stderr into stdout, which would mix progress messages ("Captured 9.6s...", "Transcribing...") into the transcript.
   - Replace `<skill_base_dir>` with the directory containing this SKILL.md file.
   - Optional flags:
     - `--model tiny|base|small` (default: `base`). Use the model the user specifies, or `base` if unspecified.
     - `--device DEVICE_ID` to select a specific audio input device.
     - `--list-devices` to show available input devices and exit.

2. **Wait for completion** -- after launching in the background, tell the user that recording has started and they should press Done (or Enter) when finished. You will be automatically notified when the background task completes. Then read the output file to get the transcript.

3. **Parse the output** -- the first line of stdout is always a status indicator. Use it to determine what happened:
   - `[CANCEL]` — the user pressed Cancel or Escape. No transcript follows. **Do nothing** — do not ask them to try again, do not comment on it. Just stop.
   - `[DONE]` — the user pressed Done. If there is text on subsequent lines, that is the transcript. If there is no text after `[DONE]`, no speech was detected — tell the user and ask them to try again.

4. **Handle errors** (non-zero exit, no status line):
   - If stderr contains "already running", tell the user: "A voice recording session is already in progress. Please finish that session before starting a new one."
   - For other failures (e.g., `ModuleNotFoundError`, missing Python executable, or import errors), tell the user: "Voice capture failed. Make sure you've run `make install` in your skills/voice directory (e.g., `cd ~/.claude/skills/voice && make install`) to create the virtual environment and install dependencies." Also suggest checking microphone permissions if the error is not dependency-related.

5. **Respond to the transcript** -- treat the captured text as if the user had typed it directly. Do not quote or parrot it back; just act on it naturally.
