# Harmonica Delta Force

A desktop app for managing harmonica scores and playing them automatically in Delta Force.

It reads MIDI or JSON scores, arranges notes into the game's harmonica key
sequence, and uses Win32 `SendInput` scan codes to play them automatically.

**Current version: 1.1.10**

The app opens in compact mode. Switch to Advanced for the full interface. Version
1.1.10 adds a water-ripple transition when switching between light and dark themes.

---

## Download and run

Download the Windows executable from the
[GitHub Releases page](https://github.com/ngt-baor/Harmonica-Delta-Force/releases)
and run it. No Python installation or setup is required. On first launch, the app
creates sample scores and stores the user's settings and music library under
`%APPDATA%\GTIHarmonica`.

See `User Guide.txt` for the usage guide. In the app, open
`Cài đặt → Hướng dẫn sử dụng` for help and ways to add scores.

## Run from source

Windows and Python 3.10 or later are required. Administrator privileges are not
needed unless the game is running as administrator.

```bash
pip install PySide6-Essentials        # GUI

pip install numpy soundfile           # Required for audio-to-score conversion
                                      # miniaudio is an optional decoder fallback
```

If you only want to play scores and do not need audio-to-score conversion, you
can omit `numpy` and `soundfile`. The app hides the related entry points and
keeps the other features available.

## Quick start

```bash
# 1. Analyze without interacting with the game: check playability and compare strategies
python -m gtiharmonica analyze song.mid --verbose

# 2. Preview locally and confirm the melody and track
python -m gtiharmonica preview song.mid

# 3. Calibrate the shortest key press accepted by your game
python -m gtiharmonica calibrate --save cal.json

# 4. Play; switch back to the in-game harmonica during the 3-second countdown
python -m gtiharmonica play song.mid --strategy optimal

# 5. Drag in MIDI / JSON or use Nhập file; use Nhập bản số for numbered notation
```

In the score editor, press `Ctrl+S` to save changes.

During playback, `F8` pauses or resumes. `F9` or `Esc` stops playback and releases
all keys. Pressing `F9` or `Esc` during the 3-second countdown cancels playback.

Hotkeys use **global polling**, so they work while the game is in the foreground
without consuming the keys or interfering with the game's own `F8` and `F9`
bindings. The app polls every 8 ms, including during long notes and rests. Switching
back to this app automatically pauses playback; press `F8` after returning to the
game to resume. Switching to another app stops playback and releases all keys to
prevent unintended input.

---

## Song library location

**Packaged app:** `%APPDATA%\GTIHarmonica\songs`  
**Development version:** `songs\` in the project root

The library used to live beside the executable under `songs\`, inside `dist\`.
Rebuilding removed that directory and could delete user-created scores. The
library now lives in the user data directory, which builds and reinstalls leave
alone. To use portable mode from a USB drive, place an empty `portable.txt` beside
the executable; the app will keep its library there.

On first launch, songs from the previous location are **merged automatically**:
existing files are not overwritten or deleted, and the merge can be repeated.
The build script also compares the library before and after each build and fails
if any song is missing.

Automatic merging **skips songs you deleted**. Deleting a song from the library
records it in `.deleted.txt`, so startup migration and built-in song installation
will leave it alone. Without this record, a same-named file beside the executable
could be copied back on every launch. Importing or dragging that same-named file
into the library clears the record.

Favorites are stored in `.favorites.txt`, with one filename per line. Click the
star beside a song to add or remove it from favorites; the `Tất cả / Yêu thích`
filter shows the corresponding list. Both metadata files are plain text, travel
with the song library, and can be backed up or edited manually.

---

## Commands

| Command | Purpose |
|---|---|
| `play <file>` | Play a score |
| `preview <file>` | Preview a MIDI locally without interacting with the game |
| `analyze <file>` | Compare strategies and metrics; the main tuning reference |
| `transcribe <audio\|dir>` | Convert audio to a score with the built-in GAME model; directories are supported |
| `tracks <file>` | Show track details and playable coverage |
| `export <file>` | Export the arranged play plan as JSON |
| `import-legacy <dir>` | Batch-import a library from an older version |
| `calibrate` | Run guided calibration |
| `selftest` | Check the fingering table |
| `windows` | List visible windows and find target handles |
| `init` | Create the default configuration file |

---

## Finding scores

The in-game harmonica can play **only one note at a time**. For the best results,
look for scores in this order:

1. **Find an existing MIDI (recommended).** Download one from a score site such as
   midishow. Search for arrangements made for the original harmonica instrument or
   practice, or use the site's track-count filter to find a single-track MIDI.
    Drag the `.mid` file into the app or use **Nhập file** to add it directly.
2. **For a multi-track MIDI,** prefer a single-track or harmonica arrangement for
    a clearer result. The app imports MIDI files as-is without a conversion prompt.
3. **Audio transcription is command-line only** in this version; it is not shown
    as an app import option.

---

## Audio-to-score conversion (command line)

> Prefer a single-track MIDI whenever one is available. A melody transcribed from
> a recording is less precise in rhythm and detail than a score prepared by hand.

> The desktop interface currently focuses on MIDI / JSON and numbered-score
> imports. Audio transcription remains available through the command line only.
> Inference runs on the CPU; a four-minute song takes about 30 seconds. See
> `gtiharmonica/audio2score.py` for the GAME inference flow reimplementation,
> which does not depend on torch.

### Use the command line

```bash
# Transcribe one file
python -m gtiharmonica transcribe "song.mp3"

# Batch-transcribe a directory and save results to the library
python -m gtiharmonica transcribe "D:\Music\Album" --quantize 1/8

# Preview without saving
python -m gtiharmonica transcribe song.mp3 --dry-run

# Lower the prior for a low melody, such as cello or bass vocals
python -m gtiharmonica transcribe song.mp3 --prior 0.3

# Use a triplet grid
python -m gtiharmonica transcribe song.mp3 --quantize 1/8t
```

Measured results, described in [Verification status](#verification-status): on the
same vocal sample, the built-in result matches 94% of the notes from the official
`GameInfer.exe` within ±100 ms, with 92% pitch agreement. When slicing fails, the
built-in tool falls back to forced chunks instead of exiting with
`Slice duration exceeds 60 seconds`, as the official CLI does.

### Processing pipeline

```text
Audio file
   ↓  Decode (soundfile → miniaudio → ffmpeg fallbacks)
   ↓  Short-time Fourier transform per frame + spectral whitening
   ↓  Log-domain weighted harmonic sum → pitch-salience panel
   ↓  Viterbi pitch tracking (continuity + octave penalty)
   ↓  Onset detection and note segmentation
   ↓  BPM estimate + beat-grid quantization
   ↓  Monophonic conversion (the harmonica plays one note at a time)
   ↓  Scale snapping + octave folding into the harmonica's range
Playable score
```

### Why this can be simpler than general transcription

General audio-to-score transcription is difficult and often needs deep-learning
models. This app does not target general-purpose notation: the instrument has only
8 keys and 38 playable pitches, so the converter needs the pitch contour of a
dominant monophonic melody rather than polyphonic transcription. That makes a
NumPy-only DSP approach practical without gigabytes of model dependencies.

Evaluation follows the instrument's needs: a note is correct when its **pitch
class** is correct. An octave shift that folds to the same key does not count as an
error.

### Measured accuracy

| Scenario | Pitch-class accuracy |
|---|---:|
| Eight synthetic melodies (scales, wide leaps, dense chromatic notes, high and low ranges) | **100%** (70/70) |
| Melody with bass line, chords, and light/medium/heavy percussion | **100%** |
| Real MP3 (293 seconds, 545 notes) | **2.2 seconds** to transcribe (1% of real time) |
| Additive noise at 40 dB | **100%** |

### Tuning parameters

The sliders and dropdowns in the interface map to these transcription parameters:

| Parameter | Default | When to change it |
|---|---:|---|
| Sensitivity | 0.5 | Raise it when notes are missed; lower it when noise creates extra notes |
| Melody prior | 0.8 | Raise it when the bass is mistaken for the melody; lower it when the melody is low |
| Quantization | `1/8` | Use `1/4` if the rhythm does not align; use `1/16` if rhythmic detail is lost |
| Scale | Major | Choose `minor` for a minor song, `penta` for many accidentals, or `off` to preserve pitches |
| Key | C | Change this when the song is not in C, or the whole result will sound out of tune |
| Meter | Automatic | Enter a value manually if automatic detection is wrong; start by estimating 4/4 |

The command line also supports these advanced parameters:

| Parameter | Default | Description |
|---|---:|---|
| `--sensitivity` | 0.5 | Same as above |
| `--prior` | 0.8 | Same as above |
| `--min-note-ms` | 70 | Discard notes shorter than this; increase it if the game misses notes |
| `--bpm` | 0 | `0` means automatic detection |

### Quality and limitations

- **Transcription is an estimate, not an exact reconstruction.** Results may be
  weaker with accompaniment, key changes, or vocal harmonies. Preview a saved score
  and adjust the settings if needed.
- **Monophonic output:** the harmonica can play one note, so chords are reduced to
  a single melody, keeping the higher note.
- **Pure percussion or music without a clear melody is unsupported.** The app will
  report that few notes were recognized.
- **BPM is normalized to 72–156** to avoid half- or double-tempo estimates. Those
  tempos are musically equivalent, but the beat-grid resolution differs by a factor
  of two.
- **Each transcription runs independently.** It does not use the network or upload
  any audio.

---

## Score editor

Transcription is an estimate and may split notes incorrectly or recognize a wrong
pitch. Instead of repeatedly tuning parameters, switch from the melody preview to
the score editor using the segmented control in the preview title row.

The editor changes the **score itself**, not just the key sequence. Each note has a
pitch, start time, and duration; bar lines are calculated from BPM and time
signature. After an edit, `arrange_from_notes()` recomputes the fingering, so the
score being edited is the one that will be played.

| Action | Result |
|---|---|
| Click / Shift-click | Select / add to selection |
| Drag a note body | Change pitch and position |
| Drag either note edge | Change duration; dragging the left edge keeps the end fixed |
| Drag across empty space | Select a region |
| Double-click empty space / a note | Insert a note / split at the clicked point |
| `M` | Merge selected fragments |
| Delete | Delete the selection |
| Arrow keys | Change pitch or time; hold Shift to move by an octave |
| Ctrl+Z / Ctrl+Shift+Z | Undo / redo |
| Ctrl+A | Select all |
| Wheel / Ctrl+wheel | Scroll horizontally / zoom |
| Middle-button drag | Pan the view |

The toolbar also offers quantization (snap to the beat grid), scale snapping (fix
drift by ±1 semitone), and overlap cleanup (the game can press only one key at a
time). The information row reports mergeable fragment groups and lets you change
time signature, BPM, key, and scale; bar lines update immediately.

**Merge long notes** is the most direct fix for fragmented sustains. Select the
fragments and use the merge action, or use it with nothing selected to merge
fragments across the entire song. Transcribed fragments are often scattered across
the score, so selecting them individually is impractical. The merge groups notes
with the same pitch and adjacent times, avoiding merges across real gaps. Undo it
with Ctrl+Z if the result is not right.

Pitches in the editor are **already the final pitches to play**. Applying edits
does not transpose or fold octaves a second time; doing so would change pitches
that are already correct. In editor mode, the transpose slider recomputes
fingering without moving the edited pitches.

---

## Core concepts

### Instrument model

The in-game harmonica has 8 scale keys and 3 mouse modifier keys (held while
playing):

```text
Scale keys  z   x   c   v   b   n   m   ,
Scale       1   2   3   4   5   6   7   high 1
MIDI        60  62  64  65  67  69  71  72      (when base = 60)

Modifiers   Left mouse button  = down one octave
            Middle mouse button = up one semitone
            Right mouse button = up one octave
```

Modifiers can be combined, so one pitch may have several fingerings. For example,
MIDI 60 can use:

```text
z                         (no modifier)
, + left mouse button     (high 1, down one octave)
m + left + middle buttons (7, down an octave, then up one semitone)
```

The default configuration can play **38 pitches, MIDI 48 (C3) through 85 (C#6)**.

Choosing the fingering is the main optimization problem in this project.

### Processing pipeline

```text
MIDI / JSON score
   ↓  Filter tracks
   ↓  Convert chords to single notes (keep the highest simultaneous note by default)
   ↓  Transpose
   ↓  Fold octaves into the playable range (by multiples of 12 semitones)
   ↓  Choose fingering ★ strategy selection happens here
   ↓  Add breathing gaps / legato / speed
Play plan (key, start time, and duration for each note)
   ↓  Schedule against the perf_counter absolute clock
SendInput scan codes → game audio
```

### Why scan codes are required

Games using DirectInput or Raw Input read hardware **scan codes**. `SendInput` or
`keybd_event` calls that set only a virtual-key code are usually ignored; this is
why many home-made scripts do not work in games.

Punctuation keys also need the OEM virtual-key code (`ord(',')` is 44; the correct
value is 188).

---

## Targeted tuning guide

**All parameters that affect feel and success rate are configurable** and can be
measured and tuned individually.

### 1. Measure before tuning

Do not tune by feel alone. Run calibration first:

```bash
python -m gtiharmonica calibrate --save cal.json
```

Calibration measures:

| Measurement | Use |
|---|---|
| Shortest recognized note | Set `min_note` so the game does not miss notes |
| SendInput p95 latency | High latency may indicate antivirus interception; tune `spin_window` |
| Key acceptance rate | Find keys or modifiers the game does not receive |

### 2. Choose a fingering strategy

```bash
python -m gtiharmonica analyze song.mid
```

Example output for a wide-range, large-leap test score:

| Strategy | Presses | Modifier changes | Key changes | Movement | Cost |
|---|---:|---:|---:|---:|---:|
| min-modifiers | **42** | 19 | 18 | **87.0** | 76.4 |
| greedy | 46 | 19 | 16 | 51.0 | 70.9 |
| **optimal** | 45 | 19 | **15** | **51.0** | **70.4** |
| stable-key | 46 | 19 | 16 | 51.0 | 70.9 |

Choose a strategy based on your needs:

- **Frequent fingering errors or modifier problems:** use `optimal` for the fewest
  modifier changes and the least movement.
- **The game misses key presses:** use `min-modifiers` for the fewest total presses.
- **Limited hand speed:** use `stable-key` to reduce key changes.
- If unsure, use `optimal`; it finds the global optimum under the cost model.

`analyze` also checks the dynamic-programming result and warns if another strategy
has a lower cost than `optimal`.

### 3. Tune cost weights

The cost model in the `cost` section of the configuration determines which
strategy is considered best:

```json
"cost": {
  "modifier_switch": 1.00,     // Modifier changes; most error-prone, highest default cost
  "key_move": 0.35,            // Keyboard movement, in key widths
  "extra_press": 0.30,         // Cost per additional modifier key pressed
  "key_repeat_bonus": -0.20,   // Reuse the same scale key (negative value is a bonus)
  "octave_jump": 0.15          // Extra cost for switching between octave modifiers
}
```

Tuning suggestions:

- If modifiers are often not held or released correctly, increase
  `modifier_switch` and `octave_jump`.
- If your fingers cannot comfortably span the keys, increase `key_move`.
- If the game handles simultaneous keys poorly, increase `extra_press`.

### 4. Tune timing parameters

| Parameter | Default | When to change it |
|---|---:|---|
| `speed` | 1.0 | Lower to 0.8 if your hands cannot keep up; range 0.25–2.0 |
| `gate` | 0.9 | Lower to 0.7 if notes run together; raise to 0.95 if they sound too detached |
| `breath_ms` | 90 | Raise to 150 for long phrases; lower to 50 if the rhythm feels sluggish |
| `min_note` | 0.02 | Use the calibration result; increase it if the game misses notes |
| `min_gap` | 0.012 | Increase it when repeated notes of the same pitch run together |

### 5. Tune pitch handling

| Parameter | Default | Description |
|---|---:|---|
| `transpose` | 0 | Transpose the entire score by up to ±24 semitones to fit the range |
| `fold_octaves` | true | Fold out-of-range pitches; disable it to drop them instead |
| `fold_prefer` | nearest | Choose `down` for a lower melody or `up` for a higher one |
| `chord_policy` | highest | Choose which note survives chord reduction; `lowest` keeps the bass note |
| `track` | null | Select a track after checking its coverage with `tracks` |

### 6. Scheduling precision

```json
"scheduler": {
  "spin_window": 0.002,   // Busy-wait window; smaller is more precise but uses more CPU
  "poll_interval": 0.02,  // Pause and hotkey polling interval
  "tight": true           // Skip a sleep round trip when notes have no gap
}
```

At the end of each `play`, the app prints a timing report with average, p95, and
maximum latency. Use it to decide whether `spin_window` needs adjustment.

---

## Module architecture

```text
gtiharmonica/
├── instrument.py    Instrument model, fingering candidates, and key coordinates
├── fingering.py     Four fingering strategies, cost model, and dynamic programming
├── arrange.py       Monophonic conversion, octave folding, transposition, and breathing
├── audio2score.py   Audio-to-score conversion using built-in GAME ONNX inference
├── melody.py        Melody conversion: sixteenth-note grid and lead-track selection
├── edit.py          Score-edit commands: delete, change, insert, and cut time
├── jianpu.py        Key-based number notation parser
├── score.py         MIDI / JSON parsing and tempo maps
├── library_state.py Song-library metadata: favorites and deleted-song list
├── backend.py       SendInput scan codes, preview, and dry-run mode
├── player.py        Absolute-clock scheduler, state machine, and hotkeys
├── calibrate.py     Calibration tool
├── config.py        Persistent configuration
├── gui/             PySide6 interface: main window, panels, dialogs, editor, overlay, theme
└── cli.py           Command-line interface
```

The main entry points for targeted optimization are `fingering.py`, `arrange.py`,
`audio2score.py`, `melody.py`, `library_state.py`, and `player.py`.

To add a strategy, subclass `FingeringStrategy` in `fingering.py`, implement
`plan()`, and register it in the `STRATEGIES` dictionary. `analyze` will include it
automatically.

---

## Known limitations

No separate known-issues document is currently included in this repository.

- **Windows only:** input injection depends on `SendInput`.
- **In-game key bindings must match the configuration.** The default configuration
  matches the game's defaults. If you changed them, update `gtiharmonica.json` or
  use the older `instrument.json`; the app reads it automatically.
- **Monophonic playback:** the harmonica plays one note at a time, so multiple
  voices are reduced to a single melody.
- **Borderless window mode is recommended.** Keep the game in the foreground during
  playback.
- **Administrator privileges:** if the game runs as administrator, this app must
  be elevated too, or UIPI will block `SendInput`.
- **Song deletion is remembered.** Deleted songs are recorded in `.deleted.txt`;
  startup migration and built-in song installation skip them. To include a song in
  migration again, import a file with the same name.
- **The editor does not seek by clicking the score.** Clicks select notes; use the
  timeline slider below the card to seek.

## Verification status

| Item | Status |
|---|---|
| Fingering table (38 pitches / MIDI 48–85) | ✅ Verified note by note in the game |
| Score parsing (MIDI + tempo) | ✅ Multi-track cases pass |
| Keep the highest note during monophonic conversion | ✅ |
| Octave folding (100 → 76 + right mouse button) | ✅ |
| Breathing formula (0.5 s → 0.450 s / final note 0.410 s) | ✅ |
| Four strategy comparison + global DP optimum | ✅ Cost 70.4 < 70.9 < 76.4 |
| Arrangement performance for 240 notes | ✅ 194 ms |
| Dry-run sequence output | ✅ |
| Historical audio-to-score feature | ⛔ Removed from the app in v1.2; old code deleted |
| MIDI import | ✅ Added directly to the library without a conversion dialog |
| Song favorites: star, all/favorites filter, persistence after restart | ✅ Dark and light themes checked |
| Deleted-song list: deleted songs stay deleted after restart | ✅ End-to-end packaged-app check |
| Hotkeys during long notes / 12 ms presses / cancel countdown / foreground ownership / auto-pause on return | ✅ 21 checks |
| Real F8/F9 input through the GUI (pause → resume → stop) | ✅ 6 end-to-end checks |
| Built-in audio inference vs. official GameInfer.exe on the same vocal sample | ✅ 94% note alignment; 92% pitch agreement |
| Audio conversion for the song `Like Fireworks in My Heart`: MIDI → rendered audio → built-in transcription → comparison | ✅ 100% time alignment; 93.8% pitch agreement |
| Audio slicing fallback / cancellation / GUI workflow / packaged build | ✅ 19 checks plus packaged CLI run |
| Score editor commands / undo / snapping / fragment merging | ✅ 88 checks |
| Score editor interaction and real-pixel rendering | ✅ 51 checks |
| Editor ↔ main window: switch / recompute / JSON save / key-row sync | ✅ 41 checks |
| Edited scores are not transposed twice (`transpose=7`, pitch unchanged) | ✅ |
| Library safety: migration / idempotency / build-manifest check | ✅ 21 checks |
| On-device library migration (launch → 46 songs moved → source retained → restart is idempotent) | ✅ 10 checks |
| Pitch accuracy: scan 20 pitches | ⚠️ G3 and higher all match; see known limitation below |
| **Actual in-game playback** | ⚠️ Not verified; requires a game environment and local calibration |
| **Sound and feel of in-game playback** | ⚠️ Not verified; arrangement cost and fingering are measured, but try it in the game |

## Compliance note

Simulated keyboard and mouse input is a gray area in game automation. Harmonica
playback does not affect competitive fairness, but check the game's terms of
service and avoid using it in ranked play.
