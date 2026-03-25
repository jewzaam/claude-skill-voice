# Manual Test Plan — Chunked Transcription

## Setup

```bash
PYTHON=~/.claude/venvs/voice/Scripts/python.exe  # Windows
PYTHON=~/.claude/venvs/voice/bin/python           # Linux/macOS

mkdir -p debug
```

All commands capture stdout and stderr to `debug/` with unique filenames.

**Button convention:** When a test says "press [Button]", click that button in the GUI window. "Do NOT press any buttons" means only interact via Done at the end.

---

## Test 1: Basic recording — no buttons except Done

**Purpose:** Verify short continuous speech works — no automatic chunking expected.

```bash
$PYTHON scripts/voice.py --debug >debug/t01-basic.stdout 2>debug/t01-basic.stderr
```

**Steps:**
1. Speak continuously for ~5 seconds — do NOT stop talking, no gaps
2. Press the **Done** button

**Expected:**
- `debug/t01-basic.stdout` — `[BEGIN]`, transcript, `[END]`
- `debug/t01-basic.stderr` — "Captured X.Xs", no "silence flush" messages (too short for 3s min chunk)

---

## Test 2: Pause button triggers chunk

**Purpose:** Verify pressing the Pause button flushes audio for background transcription.

```bash
$PYTHON scripts/voice.py --debug >debug/t02-pause.stdout 2>debug/t02-pause.stderr
```

**Steps:**
1. Speak for ~10 seconds
2. Press the **Pause** button
3. Wait ~10 seconds (watch stderr for transcription activity if tailing)
4. Press the **Resume** button
5. Speak for ~5 more seconds
6. Press the **Done** button

**Expected:**
- `debug/t02-pause.stderr` — chunk flush logged on pause, transcription starts in background
- `debug/t02-pause.stdout` — transcript contains text from both speech segments
- "Transcribing:" phase on Done should be shorter (first chunk already done)

---

## Test 3: Multiple Pause button presses

**Purpose:** Verify multiple chunks are transcribed and assembled in order.

```bash
$PYTHON scripts/voice.py --debug >debug/t03-multi-pause.stdout 2>debug/t03-multi-pause.stderr
```

**Steps:**
1. Say "First segment" → press **Pause** → wait 5s
2. Press **Resume** → say "Second segment" → press **Pause** → wait 5s
3. Press **Resume** → say "Third segment" → press **Done**

**Expected:**
- `debug/t03-multi-pause.stderr` — 3 chunk flushes logged
- `debug/t03-multi-pause.stdout` — all three segments in order

---

## Test 4: Silence detection — no buttons except Done

**Purpose:** Verify SilenceDetector triggers automatically during natural speech gaps. Do NOT press Pause.

```bash
$PYTHON scripts/voice.py --debug >debug/t04-silence.stdout 2>debug/t04-silence.stderr
```

**Steps:**
1. Speak a sentence (~5s)
2. Go completely silent for ~3 seconds — do NOT press any buttons
3. Speak another sentence (~5s)
4. Go completely silent for ~3 seconds — do NOT press any buttons
5. Speak a third sentence (~5s)
6. Press the **Done** button

**Expected:**
- `debug/t04-silence.stderr` — "silence flush" messages during the silent gaps
- `debug/t04-silence.stdout` — transcript contains all three sentences

---

## Test 5: Streaming output with Pause button

**Purpose:** Verify `--stream` prints chunks to stdout incrementally.

```bash
$PYTHON scripts/voice.py --stream --debug >debug/t05-stream.stdout 2>debug/t05-stream.stderr
```

**Steps:**
1. Speak for ~10 seconds
2. Press the **Pause** button (triggers flush)
3. Wait ~10 seconds for chunk to transcribe
4. Press the **Resume** button
5. Speak for ~5 more seconds
6. Press the **Done** button

**Expected:**
- `debug/t05-stream.stdout` — `[BEGIN]`, chunk text lines, `[END]` at the end
- `debug/t05-stream.stderr` — chunk transcription activity during pause

---

## Test 6: Cancel button during recording

**Purpose:** Verify cancel works normally.

```bash
$PYTHON scripts/voice.py --debug >debug/t06-cancel-record.stdout 2>debug/t06-cancel-record.stderr
```

**Steps:**
1. Speak for a few seconds
2. Press the **Cancel** button

**Expected:**
- `debug/t06-cancel-record.stdout` — `[CANCEL]` only (no `[BEGIN]`)
- `debug/t06-cancel-record.stderr` — no errors

---

## Test 7: Cancel button during transcription phase

**Purpose:** Verify cancel works after Done while transcription is finishing.

```bash
$PYTHON scripts/voice.py --debug >debug/t07-cancel-transcribe.stdout 2>debug/t07-cancel-transcribe.stderr
```

**Steps:**
1. Speak continuously for ~30 seconds — do NOT press Pause (forces all transcription after Done)
2. Press the **Done** button
3. Immediately press the **Cancel** button while "Transcribing:" label is shown

**Expected:**
- Process exits cleanly, no crash, no hang
- `debug/t07-cancel-transcribe.stdout` — `[CANCEL]` only (no `--stream`, so no `[BEGIN]` was printed)
- `debug/t07-cancel-transcribe.stderr` — no traceback

---

## Test 8: Very short recording — Done immediately

**Purpose:** Verify recordings shorter than min_chunk_s (3s) work.

```bash
$PYTHON scripts/voice.py --debug >debug/t08-short.stdout 2>debug/t08-short.stderr
```

**Steps:**
1. Say one word
2. Immediately press the **Done** button (~1-2 seconds total)

**Expected:**
- `debug/t08-short.stderr` — no "silence flush" messages
- `debug/t08-short.stdout` — `[BEGIN]`, the word, `[END]`

---

## Test 9: WAV output — Pause button once

**Purpose:** Verify `--wav-output` still works with chunked transcription.

```bash
$PYTHON scripts/voice.py --wav-output debug/t09-chunked.wav --debug >debug/t09-wav-output.stdout 2>debug/t09-wav-output.stderr
```

**Steps:**
1. Speak for ~10 seconds
2. Press the **Pause** button → wait 5s
3. Press the **Resume** button → speak for ~5 more seconds
4. Press the **Done** button

**Expected:**
- `debug/t09-chunked.wav` — saved, playable, contains full recording
- `debug/t09-wav-output.stdout` — `[BEGIN]`, transcript, `[END]`
- `debug/t09-wav-output.stderr` — "Saved audio to ..."

---

## Test 10: WAV input — no GUI

**Purpose:** Verify `--wav-input` still works (no chunking, direct transcription).

*(Requires Test 9 WAV file.)*

```bash
$PYTHON scripts/voice.py --wav-input debug/t09-chunked.wav --debug >debug/t10-wav-input.stdout 2>debug/t10-wav-input.stderr
```

**Steps:** None — runs automatically, no GUI.

**Expected:**
- No GUI appears
- `debug/t10-wav-input.stdout` — `[BEGIN]`, transcript, `[END]`
- `debug/t10-wav-input.stderr` — transcription time

---

## Test 11: Chunk vs single-pass quality comparison

**Purpose:** Compare chunk-transcribed output vs single-pass for quality.

```bash
$PYTHON scripts/voice.py --wav-output debug/t11-comparison.wav >debug/t11-chunked.stdout 2>debug/t11-chunked.stderr
$PYTHON scripts/voice.py --wav-input debug/t11-comparison.wav >debug/t11-single-pass.stdout 2>debug/t11-single-pass.stderr
diff debug/t11-chunked.stdout debug/t11-single-pass.stdout >debug/t11-diff.txt
```

**Steps for first command:**
1. Speak for ~20 seconds with a natural gap (~3s silence) in the middle — do NOT press Pause
2. Press the **Done** button

**Expected:**
- `debug/t11-diff.txt` — differences, if any, at chunk boundaries
- Single-pass may have slightly better continuity across sentence boundaries

---

## After testing

Review all captured output:
```bash
ls -la debug/
head debug/t*.stdout
```

Check for any stderr tracebacks:
```bash
grep -l "Traceback\|Error" debug/*.stderr
```
