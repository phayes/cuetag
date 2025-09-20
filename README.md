# CUETAG: Time-Anchored Metadata for Audio

*A lightweight scheme for embedding ****sample-accurate**** cues in existing tag systems.*

## High-Level Overview

**CUETAG** stores **point** or **range** markers inside common audio metadata (Vorbis Comments in FLAC/Ogg/Opus, ID3 TXXX in MP3/AAC, MP4 freeform atoms, APEv2). Each cue is a repeatable tag whose value is in the form:

```
CUETAG=<sample_spec>|<kind>|<value>
```

- `<sample_spec>` pinpoints a **single sample** (`N`) or an **inclusive range** (`START..END`).
- `<kind>` names what the cue represents (e.g., `ARTIST`, `TITLE`, `MOVEMENT`, `INITIALKEY`).\
  Use established tag names when they fit, or define your own.
- `<value>` is the payload of the cue tag. It can be arbitrary data, restricted only by the underlying format (e.g., Vorbis Comments require Unicode text, ID3 `TXXX` requires text encoding, etc.).

The first part of the tag (`<sample_spec>|<kind>|`) **MUST** be ASCII so that it can be parsed reliably as raw bytes, ASCII, or UTF-8. The `<value>` portion follows and may use the broader capabilities of the host metadata format.

CUETAG is container-agnostic, order-insensitive, and complements existing file-level tags by adding **when** those tags apply within a track.

### Quick Examples

Here are some introductory examples to illustrate CUETAG usage for different kinds of users:

**For DJs (notes on tracks):**

```
CUETAG=88200|COMMENT|Big drop here
CUETAG=176400..220500|COMMENT|Loop this section
```

**For performers (tempo/BPM markers):**

```
CUETAG=0..88200|BPM|120
CUETAG=88201..176400|BPM|128
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
  - Consumers **MAY** normalize internally: (eg Point `N` → interval `[N, N]`)

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
- `<value>` **MAY** be empty. If <value> is empty, the cue’s semantics are application-defined (e.g., a marker with no payload). 
- `<value>` **MAY** contain arbitrary data supported by the format.

## 4. Semantics

- Positions are **absolute, 0-based samples** at the stream’s sample rate.
- Consumers **MUST** map to seconds as `t = sample_index / sample_rate`.
- Multiple CUETAG entries **MAY** exist.
- Order not guaranteed; consumers **SHOULD** sort by `(START, END)`.
- CUETAG does not prescribe behavior for overlapping ranges. Applications are free to interpret overlaps as appropriate for their domain (e.g., multiple active performers, layered annotations).

## 5. Validation & Error Handling

- A cue is **VALID** if `<sample_spec>` parses as a point or range (START ≤ END).
- Invalid entries (bad delimiters, malformed ranges, non-digits) **MUST** be rejected.
- Unknown `<kind>` values **MUST NOT** invalidate a cue.
- `<value>` content is opaque to the parser and should be passed through as-is.

## 6. Interoperability Profiles

### 6.1 Vorbis Comments (FLAC/Ogg/Opus)

```
CUETAG=<sample_spec>|<kind>|<value>
```

- `<value>` must be Unicode text.
- Producers **MUST NOT** embed raw binary.

### 6.2 ID3 (MP3/AAC)

- Use `TXXX` frames with description `CUETAG`; frame text = CUETAG string.

Example:\
`TXXX (desc="CUETAG"): 88200..176400|ARTIST|John Coltrane (tenor sax solo)`

### 6.3 MP4/M4A

- Use freeform atom: `----:com.apple.iTunes:CUETAG` with UTF-8 or binary-safe value.

### 6.4 APEv2

- Key `CUETAG`, value = CUETAG string (UTF-8).

## 7. Examples

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

## 8. Parsing Algorithm (Non-Normative)

1. Split at first two `|` → `spec`, `kind`, `value`.
2. Parse `spec`:
   - `^\d+$` → point `N`.
   - `^(\d+)..(\d+)$` → range `[START, END]`.
3. Validate: for ranges ensure `START ≤ END`.
4. Validate `kind` (`^[A-Z0-9_]+$`).
5. `<value>` is opaque to the parser, only format-level validation applies.
6. Normalize internally if desired.

**Regex:**

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

## 10. Security & Robustness Considerations

- Very large tag blocks can impact memory; avoid excessive volumes.
- Consumers **MUST** validate and ignore malformed CUETAGs without affecting playback.

