---
name: voice
description: "Trigger on `/voice` -- voice-to-text input for Claude Code. Records audio from the microphone, transcribes locally with faster-whisper, and returns the transcript as the user's prompt."
---

# Voice-to-Text Input

Record audio from the user's microphone, transcribe it locally, and treat the transcript as their message.

## Steps

1. **Run the voice script** via Bash:
   ```
   python3 <skill_base_dir>/scripts/voice.py
   ```
   - Replace `<skill_base_dir>` with the directory containing this SKILL.md file.
   - Optional flags:
     - `--model tiny|base|small` (default: `base`). Use the model the user specifies, or `base` if unspecified.
     - `--device DEVICE_ID` to select a specific audio input device.
     - `--list-devices` to show available input devices and exit.

2. **Capture stdout** -- that is the transcript. Ignore stderr (it contains progress/status messages).

3. **Handle edge cases**:
   - If the script exits with a non-zero status, inform the user that voice capture failed and suggest they check microphone permissions or dependencies (`sounddevice`, `faster-whisper`, `numpy`).
   - If stdout is empty or whitespace-only, tell the user no speech was detected and ask them to try again.

4. **Respond to the transcript** -- treat the captured text as if the user had typed it directly. Do not quote or parrot it back; just act on it naturally.
