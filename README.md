# CUETAG: Time-Anchored Metadata for Audio

*A lightweight scheme for embedding ****sample-accurate**** cues in existing tag systems.*

Version: 0.1-draft

Author: Patrick Hayes - patrick.d.hayes@gmail.com

Date: 2025

## High-Level Overview

**CUETAG** stores **point** or **range** markers inside common audio metadata (Vorbis Comments in FLAC/Ogg/Opus, ID3 TXXX in MP3/AAC, MP4 freeform atoms, APEv2). Each cue is a repeatable tag whose value is in the form:

```
CUETAG=<sample_spec>|<kind>|<value>
```

- `<sample_spec>` pinpoints a **single sample** (`N`) or an **inclusive range** (`START..END`).
- `<kind>` names what the cue represents (e.g., `ARTIST`, `TITLE`, `MOVEMENT`, `INITIALKEY`).\
  Use established tag names when they fit, or define your own.
- `<value>` is the payload of the cue tag. It can be arbitrary data, restricted only by the underlying format (e.g., Vorbis Comments require Unicode text, ID3 `TXXX` requires text encoding, etc.).

CUETAG also defines the `CUETAG_ORIGIN` tag (e.g. `CUETAG_ORIGIN=sr=44100; len=12345678;`), which provides information to allow cue tags to be recovered after a transcode.

CUETAG is container-agnostic, order-insensitive, and complements existing file-level tags by adding **when** those tags apply within a track.

### Quick Examples

Here are some introductory examples to illustrate CUETAG usage for different kinds of users:

**For DJs (notes on tracks):**

```
CUETAG=88200|COMMENT|Big drop here
CUETAG=176400..220500|COMMENT|Loop this section
```

**For DJ software (Cues, Loops, BPM changes):**

```
CUETAG=0..88200|BPM|120
CUETAG=88201..176400|BPM|128
CUETAG=45823|CUE|1
CUETAG=4567854|CUE|2
CUETAG=176400..220500|LOOP|3
```

**For players (lyrics display):**

```
CUETAG=0..44100|LYRICS|Hello darkness, my old friend
CUETAG=44101..88200|LYRICS|I've come to talk with you again
```

**For ensembles (performer markers):**

```
CUETAG=0..132300|PERFORMER|Piano – Alicia Keys
CUETAG=132300..264600|PERFORMER|Guitar – John Mayer
CUETAG=264600..396900|PERFORMER|Drums – Questlove
```

---

# CUETAG Specification

## 1. Scope and Goals

- CUETAG defines a **value syntax** and **semantics** for repeatable tags that annotate audio by **absolute sample positions**.
- Designed to work in: Vorbis Comments, ID3 (via `TXXX`), MP4 freeform atoms, and APEv2.
- CUETAG does **not** define a new binary block or file structure.

## 2. Field Name

- Implementations **MUST** use the tag/atom/frame name `CUETAG` for each cue entry.
- Tag systems that separate a key and a freeform description (e.g., ID3 `TXXX`) **MUST** set the description/key to `CUETAG`.

## 3. Value Syntax

### 3.1 Overview

Each CUETAG entry is structured as:

```
<sample_spec>|<kind>|<value>
```

- The first two `|` **MUST** be treated as structural delimiters.
- `<sample_spec>|<kind>|` **MUST** be encoded in ASCII.
- `<value>` may be any arbitrary data permitted by the host tag format (e.g., UTF-8 strings for Vorbis, UTF-16/UTF-8 text for ID3, binary-safe text in MP4 freeform atoms, etc.).
- `<value>` may contain additional `|` characters, so when parsing consumers **MUST** only process the first two `|` characters.
- When parsing non-text binary representations, `<sample_spec>|<kind>|` remains the same, so consumers **MUST** split on the first two `0x7C` bytes (`|` character) and interpret `<sample_spec>` and `<kind>` as ASCII bytes.

### 3.2 `<sample_spec>`

- Encodes absolute sample positions at the file’s sample rate (same units/range as the host format).
- Allowed forms (decimal, unsigned):
  - **Point:** `N` — exactly sample `N` (0-based).
  - **Inclusive range:** `START..END` — both bounds present; **START ≤ END**.
- **MUSTs/SHOULDs:**
  - Values **MUST** be base-10 digits only (`0–9`), length 1–20.
  - **MUST** be representable within the host format’s valid sample index range.
  - Consumers **MAY** normalize internally: (e.g. Point `N` → interval `[N, N]`)

### 3.3 `<kind>`

- **MUST** match `^[A-Z0-9_]+$` (ASCII; case-insensitive).
- Producers **SHOULD** emit uppercase.
- **SHOULD** reuse recognized tag names (e.g., `ARTIST`, `TITLE`, `MOVEMENT`, `INITIALKEY`).
- **MAY** introduce custom kinds where needed.
- Keep kinds short (≤16 chars) and stable.

### 3.4 `<value>`

- The `<value>` portion is format-dependent:
  - Vorbis Comments: must be valid UTF-8 text.
  - ID3 `TXXX`: must be valid text using the declared frame encoding.
  - MP4 freeform atoms: may use UTF-8 text or binary-safe payloads.
  - APEv2: arbitrary UTF-8 text.
- `<value>` **MAY** be empty. If `<value>` is empty, CUETAG does not prescribe semantics; interpretation is application-specific (e.g., marker with no payload).
- `<value>` **MAY** contain arbitrary data supported by the format.

## 4. CUETAG\_ORIGIN

### Introduction

`CUETAG_ORIGIN` allows sample-accurate portability across different audio formats and processing pipelines. When audio files undergo transcoding, resampling, or editing operations, the absolute sample positions of cues can become invalid or misaligned. `CUETAG_ORIGIN` preserved the original authoring context, enabling software to automatically remap cue positions.

Consider a DJ's loop points marked at samples 88200-176400 in a 44.1 kHz file. When transcoded to 48 kHz, these samples would naively become invalid. However, with `CUETAG_ORIGIN=sr=44100; len=1940400;`, consuming software can precisely calculate the new positions (96000-192000) that maintain the exact same temporal boundaries. This ensures that beat-matched loops, vocal cues, and other timing-critical markers remain sample-accurate across format conversions.

### Purpose

- Declares the **authoring timebase** for all CUETAG sample indices.
- Enables remapping after resample/transcode operations.
- Preserves sample-accurate timing across format conversions and audio processing.
- Not itself a cue, but metadata about the cue coordinate system.

### Field/Key

- `CUETAG_ORIGIN`

### Value Syntax (Canonical)

```
sr=<uint>; len=<uint>; [ext_key=<ext_value>; ...]
```

- Use `=` as key/value delimiter; fields end with `;`.
- Optional spaces around `=`/`;` are ignored.
- Keys are ASCII lowercase `[a-z][a-z0-9_]*`.
- **Required keys:** `sr` (Hz), `len` (total decoded samples at that sr).
- Unknown `ext_*` pairs are allowed and **MUST** be preserved verbatim.

### Reader Behavior

- Let `sr_file`, `len_file` be the current file's effective decoded rate/length (after priming/padding).
- If valid `CUETAG_ORIGIN` is present:
  - For point `N₀`: `N₁ = round(N₀ * sr_file / sr)`
  - For range `[S₀..E₀]`: `S₁ = round(S₀ * sr_file / sr)`; `E₁ = max(S₁, round(E₀ * sr_file / sr))`
  - Apply global timeline shifts (encoder delay/padding) consistently.
  - Clip to `[0, len_file-1]`; drop cues that become empty.
- If missing/malformed: assume cues authored at current file (`sr_origin = sr_file; len_origin = len_file`).

### Writer/Transcoder Behavior

- If timeline changes (resample/trim/pad):
  - Remap/shift all CUETAG indices.
  - Write/update `CUETAG_ORIGIN=sr=<sr_file>; len=<len_file>;`.
- If unchanged: MAY omit CUETAG\_ORIGIN; if present, must match file.
- Escaping: `=` inside values is safe. `CUETAG_ORIGIN` key MUST NOT contain `=`.

### Example

```
CUETAG_ORIGIN=sr=44100; len=12345678;
```

### Worked Example: Resampling 44.1 kHz → 48 kHz

- Input file: `sr_in = 44100`, cue `CUETAG=88200|COMMENT|Big drop here` (2.0 s).
- CUETAG\_ORIGIN: `sr=44100; len=200000;`.
- Output file: resampled to `sr_out = 48000`.

**Remap:**

```
N₀ = 88200
N₁ = round(88200 * 48000 / 44100) = round(96000)
```

- New cue: `CUETAG=96000|COMMENT|Big drop here` (still 2.0 s).

**CUETAG\_ORIGIN:**

```
CUETAG_ORIGIN=sr=48000; len=<new_length>;
```

### Compliant Transcoder Behavior

**Goal**: Preserve cue timing exactly while changing codec/container and possibly sample rate.

#### 1. Read

- Parse all CUETAG entries and optional CUETAG_ORIGIN.
- Let `sr_origin`, `len_origin` = from CUETAG_ORIGIN if valid; otherwise `sr_origin = sr_in`, `len_origin = len_in`.
- Determine output timeline: `sr_out`, `len_out`, and any global shift applied by your pipeline (e.g., removing encoder priming adds a negative leading shift; adding padding/silence adds a positive shift).

#### 2. Remap (if needed)

**Sample-rate change:**
- Point `N₀` → `N₁ = round(N₀ * sr_out / sr_origin)`
- Range `[S₀..E₀]` →
  - `S₁ = round(S₀ * sr_out / sr_origin)`
  - `E₁ = max(S₁, round(E₀ * sr_out / sr_origin))`

**Global shift (priming/padding/trim/insert):**
- Apply the same leading/trailing sample offset to all mapped indices.

**Bounds:** Clip to `[0, len_out-1]`; drop cues that become empty after clipping.

**Order:** Sort by `(START, END)`.

#### 3. Write

- Re-emit all cues as `CUETAG=<sample_spec>|<kind>|<value>`.
- **Format-specific serialization:**
  - Vorbis/FLAC/Ogg/Opus: `KEY=VALUE` with key `CUETAG`.
  - ID3: `TXXX` with description `CUETAG`.
  - MP4 freeform: `----:com.apple.iTunes:CUETAG`.
  - APEv2: key `CUETAG`.
- Update or add `CUETAG_ORIGIN=sr=<sr_out>; len=<len_out>;`.
- Preserve unknown kinds and payloads verbatim (no rewriting inside `<value>`).
- If timeline truly unchanged (same sr, no shifts), you **MAY** leave cues untouched; CUETAG_ORIGIN is optional but recommended as a hint.

### Compliant Audio Editor Behavior

**Goal**: Keep cues sample-locked to the audio through edits.

#### On Import/Open

- Parse cues; if CUETAG_ORIGIN present and `sr_origin != sr_file`, remap as in the transcoder. Otherwise, use indices as-is.
- Display cues; treat points as `[N..N]` internally if convenient.

#### During Edits (must keep cues glued to audio)

**Trim/delete at head:** shift all later cues by `-removed_samples`.

**Insert silence/audio at head:** shift all later cues by `+inserted_samples`.

**Cut/delete inside track:**
- Points inside the cut → drop.
- Ranges overlapping the cut → truncate to the remaining portion; if split in the middle, either split into two ranges or truncate (document your policy).

**Move/drag regions:** move any cues wholly inside the region by the same delta; preserve their relative offsets.

**Resample (change sr with duration preserved):** scale indices by `sr_new / sr_old` (then round), as in the transcoder.

**Time-stretch/compress (tempo change without resampling):** scale indices by the stretch factor for the affected region(s); apply piecewise if only part of the track is stretched.

**Per-region warps (elastic editing):** apply the editor's time-warp mapping to cue indices inside the warped region so they land at the warped positions.

**Bounds & order:** clip to valid range; drop empty ranges; keep cues sorted `(START, END)`.

#### On Export/Bounce/Save

- Serialize cues in host format as above.
- Write/update `CUETAG_ORIGIN=sr=<sr_file_after_edit>; len=<len_file_after_edit>;`.
- Keep structural ASCII: `<sample_spec>|<kind>|` must be ASCII; `<value>` stays opaque (UTF-8/declared encoding).

## 5. Semantics

- Positions are **absolute, 0-based samples** at the stream’s sample rate.
- Consumers **MUST** map to seconds as `t = sample_index / sample_rate`.
- Multiple CUETAG entries **MAY** exist. Only a single CUETAG_ORIGIN entry **SHOULD** exist.
- Order not guaranteed; consumers **SHOULD** sort by `(START, END)`.
- Overlapping ranges: application-specific handling.

## 6. Validation & Error Handling

- A cue is **VALID** if `<sample_spec>` parses as a point or range (START ≤ END).
- Invalid entries (bad delimiters, malformed ranges, non-digits) **MUST** be rejected.
- Unknown `<kind>` values **MUST NOT** invalidate a cue.
- `<value>` is opaque to parser.

## 7. Interoperability Profiles

### 7.1 Vorbis Comments (FLAC/Ogg/Opus)

```
CUETAG=<sample_spec>|<kind>|<value>
```

- `<value>` must be Unicode text.
- Producers **MUST NOT** embed raw binary.

### 7.2 ID3 (MP3/AAC)

- Use `TXXX` frames with description `CUETAG`; frame text = CUETAG string.

Example:\
`TXXX (desc="CUETAG"): 88200..176400|ARTIST|John Coltrane (tenor sax solo)`

### 7.3 MP4/M4A

- Use freeform atom: `----:com.apple.iTunes:CUETAG` with UTF-8 or binary-safe value.

### 7.4 APEv2

- Key `CUETAG`, value = CUETAG string (UTF-8).

## 8. Examples

**Key changes (using Vorbis ****\`KEY\`**** codes):**

```
CUETAG=0..132300|KEY|G
CUETAG=132301..264600|KEY|Em
CUETAG=264601..396900|KEY|G
```

**Classical movements + performers:**

```
CUETAG=0..132300|MOVEMENT|I. Allegro con brio
CUETAG=132301..264600|MOVEMENT|II. Andante con moto
CUETAG=220501..330750|PERFORMER|Anne-Sophie Mutter
```

**Audiobook chapters:**

```
CUETAG=0..441000|TITLE|An Unexpected Party
CUETAG=441001..882000|TITLE|Roast Mutton
CUETAG=882001..1323000|TITLE|A Short Rest
```

**Production notes (custom kinds):**

```
CUETAG=352800|TAKE|Comp from takes 3+5
CUETAG=176400..264600|FX|Tape echo engaged
```

**Tempo changes (BPM markers):**

```
CUETAG=0..88200|BPM|120
CUETAG=88201..176400|BPM|128
CUETAG=176401..264600|BPM|135
```

**Lyrics segments:**

```
CUETAG=0..44100|LYRICS|Hello darkness, my old friend
CUETAG=44101..88200|LYRICS|I've come to talk with you again
CUETAG=88201..132300|LYRICS|Because a vision softly creeping
```

## 9. Parsing Algorithms (Non-Normative)

### 9.1 CUETAG Parsing

1. **Split structure**: Split at first two `|` → `spec`, `kind`, `value`.
2. **Parse sample specification**:
   - `^\d+$` → point `N`.
   - `^(\d+)..(\d+)$` → range `[START, END]`.
3. **Validate range**: ensure `START ≤ END`.
4. **Validate kind**: match `^[A-Z0-9_]+$` (case-insensitive, emit uppercase).
5. **Validate bounds**: ensure all sample indices are within `[0, stream_length-1]`:
   - For points: `0 ≤ N < stream_length`
   - For ranges: `0 ≤ START ≤ END < stream_length`
   - **MUST** reject cues with out-of-bounds indices to prevent buffer overruns.
6. **Value handling**: `<value>` is opaque; only format-level validation applies.
7. **Normalize internally**: optionally convert points to ranges `[N..N]` for uniform processing.

**CUETAG Regex:**

```regex
^(?:
    (?P<point>\d+)
  |
    (?P<start>\d+)\.\.(?P<end>\d+)
)
\|
(?P<kind>[A-Z0-9_]+)
\|
(?P<value>.*)$
```

### 9.2 CUETAG_ORIGIN Parsing

1. **Split key-value pairs**: Split on `;` to get individual assignments.
2. **Parse each assignment**: Split on `=` → `key`, `value`.
3. **Trim whitespace**: Remove optional spaces around `=` and `;`.
4. **Validate keys**: ensure lowercase ASCII `[a-z][a-z0-9_]*`.
5. **Required keys**: verify presence of `sr` (sample rate) and `len` (length).
6. **Parse values**: 
   - `sr`: positive integer (Hz)
   - `len`: positive integer (total samples)
   - Unknown `ext_*` keys: preserve verbatim

**CUETAG_ORIGIN Regex:**

```regex
^(?:\s*(?P<key>[a-z][a-z0-9_]*)\s*=\s*(?P<value>[^;]*?)\s*;\s*)+$
```

**Example parsing**:
- Input: `sr=44100; len=1940400; ext_tool=audacity;`
- Output: `{sr: 44100, len: 1940400, ext_tool: "audacity"}`

## 10. Security & Robustness Considerations

- Very large tag blocks can impact memory.
- Consumers **MUST** validate and ignore malformed CUETAGs.
- Consumers **MUST** reject sample indices outside media length to prevent buffer overruns in dependent software implementations. 

