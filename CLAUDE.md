# Cordial Progressions — Authoring Guide

This repo houses `default.json`, the canonical chord-progression preset library
consumed by the Cordial VST suite. The file is read at plugin load by
`shared/musical/Progressions.cpp` in the parent repo. This document is the full
spec needed to author or edit presets without reading the C++.

## File shape

`default.json` is a **single JSON array** of preset objects. Order in the file
is the order shown in the plugin UI. Presets with duplicate `name` values are
**replaced in-place** (last wins), so use unique names.

```json
[
  { "cat": "Pop/Folk", "name": "I V vi IV", "mode": "major",
    "degrees": [1, 5, 6, 4] }
]
```

## Preset fields

| Field           | Type                  | Req | Default      | Purpose                                          |
|-----------------|-----------------------|-----|--------------|--------------------------------------------------|
| `cat`           | string                | yes | —            | UI grouping label                                |
| `name`          | string                | yes | —            | UI display name; must be unique                  |
| `mode`          | string (enum)         | yes | `"major"`    | Scale mode — roots resolve through it            |
| `degrees`       | int[] (1..7)          | yes | —            | Scale degree of each chord                       |
| `qualities`     | (string\|null)[]      | no  | mode default | Chord-quality override per chord; null = default |
| `beats`         | number[]              | no  | 4.0 each     | Duration of each chord in quarter-note beats     |
| `inversions`    | int[] (0..3)          | no  | 0 each       | Inversion (rotation) per chord                   |
| `bass`          | string[]              | no  | "" each      | Slash-bass spec per chord; "" = none             |
| `rootPcOffsets` | int[] (0..11, -1=off) | no  | -1 each      | Override root pitch-class; bypasses mode/degree  |

All array fields are parallel to `degrees`. Shorter arrays are padded with the
default; longer arrays are truncated.

### `mode` — accepted values (exact strings)

`major`, `lydian`, `lydian_dom`, `mixolydian`, `minor`, `dorian`, `phrygian`,
`locrian`, `harmonic_minor`.

Each mode also fixes a default diatonic triad-quality per degree, used when the
corresponding `qualities[i]` entry is `null` or missing.

### `qualities` — accepted strings (exact)

- Triads: `maj` (0,4,7), `min` (0,3,7), `dim` (0,3,6), `aug` (0,4,8)
- Power: `5` (0,7,12)
- Suspended: `sus2` (0,2,7), `sus4` (0,5,7), `7sus4` (0,5,7,10)
- Sevenths: `maj7`, `min7`, `dom7` (alias `7`), `dim7`, `m7b5` (half-diminished), `minMaj7` (0,3,7,11)
- Altered dominants: `7b5` (0,4,6,10), `7#5` / `aug7` (0,4,8,10), `7#9` (0,4,7,10,15), `7#9no5` (0,4,10,15), `7b9` (0,4,7,10,13)
- Sixths: `6` (0,4,7,9), `min6` (0,3,7,9), `6/9` (0,4,7,9,14)
- Extended: `add9`, `madd9`, `maj9`, `min9`, `dom9` (alias `9`), `min11`, `maj13`
- Lydian colour / upper-structure dominants: `maj7#11` (0,4,7,11,18), `maj7#5` (0,4,8,11), `7#11` (0,4,7,10,18), `11` (0,7,10,14,17 — 3rd omitted), `13` (0,4,7,10,14,21 — 11th omitted), `min13` (0,3,7,10,14,21), `min7b9` (0,3,7,10,13)

Anything outside this set is silently dropped to the mode default.

### `degrees` and `rootPcOffsets`

Normal flow: `root_pc = (key + scaleIntervals[mode][degree-1]) % 12`. The
`degrees` entry also drives the default chord quality (via `modeChords[mode]`).

`rootPcOffsets[i]` (0..11) **overrides** the root calculation chromatically:
`root_pc = (key + offset) % 12`. Use this for borrowed chords whose root is
not in the mode (e.g. Coltrane changes, tritone subs, parallel-mode borrows
the current `mode` cannot express). The `degrees[i]` entry is still consulted
for the default quality lookup, so always pair `rootPcOffsets` with an
explicit `qualities[i]`. Use `-1` (or omit the entry) to keep mode-derived
behaviour for that chord.

### `inversions` — rotation, not revoicing

`0` = root position. `N` = rotate the bottom N chord tones up an octave and
re-sort. Inversions affect only the chord-tone stack; the bass note remains
the chord root unless `bass[i]` is set.

### `bass` — slash-bass spec (scale-degree string)

Format: optional accidental (`b` or `#`) followed by a single digit 1..7,
interpreted as a **scale degree of the current `mode`** in the current key.
Examples: `"5"`, `"b7"`, `"#4"`, `"1"`. Empty string means no slash bass.

The slash bass is placed *below* the lowest chord tone (clamped to MIDI >= 28)
and is independent of `inversions[i]` — both may be set on the same chord.
The chord label becomes `"<root><quality>/<bass>"`.

Slash bass is **not** a chord tone selector; it is an added low pitch. Use it
for pedals, descending bass lines, and inversion-by-bass-note (e.g. `I/3`
written as `degree=1, bass="3"`).

### `beats` — duration

Beats are quarter notes (canonical time unit). The host time signature is
read live; `beats` is independent of it. Use fractional values for tresillo /
3+3+2 (`[1.5, 1.5, 1.0]`), waltz feels, etc.

## Mode / borrowed-chord invariant

A name like `"bVII - IV - I"` only renders at the actual flat-seventh root if
the chosen `mode` has +10 semitones at degree 7 (i.e. minor, dorian, phrygian,
mixolydian, lydian_dom, locrian). Authoring a `bVII` preset in `mode: "major"`
will silently retune it to the natural seventh (+11). Either pick a mode whose
diatonic 7th is flat, or use `rootPcOffsets` with an explicit `qualities`.

The same warning applies to `bII`, `bIII`, `bVI`, `#IV`, `#V`.

## Canonical example

```json
{
  "cat": "Jazz",
  "name": "Coltrane changes (Giant Steps core)",
  "mode": "major",
  "degrees":       [1,      1,      1,      1,      1,      1,      1,      1     ],
  "qualities":     ["maj7", "dom7", "maj7", "dom7", "maj7", "min7", "dom7", "maj7"],
  "rootPcOffsets": [0,      3,      8,      11,     4,      10,     3,      8     ],
  "beats":         [2,      2,      2,      2,      2,      2,      2,      4     ],
  "inversions":    [0,      0,      0,      0,      2,      0,      2,      1     ]
}
```

## Recommended `cat` labels

Diatonic, Pop/Folk, Ambient, Neo-Soul, Jazz, Fusion, Gospel, Blues, Funk,
Disco, Rock, Metal, EDM/Synth, Latin, Reggae, Cinematic, Game/JRPG, Modal,
Classical. New categories are free-form strings; the UI groups by exact match.

## Assumptions baked into the format

1. Quarter note = 1 beat everywhere. No tempo, swing, or time-signature
   metadata travels with a preset.
2. Octave, key, and seed are owned by the plugin instance, not the preset.
3. A preset is a flat one-cycle sequence. Repeats and loop counts are handled
   by the host transport, not encoded here.
4. Chord-tone landings, voicing smoothness, arp/melody phrasing, etc. are all
   generator concerns. A preset describes only the harmonic skeleton.
5. Slash-bass is restricted to scale-degree-of-mode; arbitrary chromatic bass
   pitches must be expressed as a chord with `rootPcOffsets` plus a slash-bass
   naming the original root.
6. The final chord is auto-flagged as a cadence only when its `degrees[i]` is
   `1` **and** no `rootPcOffsets` override is set on that slot.

## Known gaps (not currently representable)

Real-world harmonic situations the schema cannot express cleanly. Listed so an
author knows when to stop fighting the format.

- **Per-chord cadence override.** The C++ `CustomChordEntry` has
  `isCadenceOverride` (force / suppress / auto); JSON presets only get the
  auto path.
- **Per-chord octave shift.** One global octave; no way to push a pivot chord
  up or drop a Picardy-third resolution down.
- **Per-chord dynamics / velocity / accent.**
- **Sections.** No verse/chorus/bridge grouping, no `repeat: 2`, no
  first/second endings.
- **Polychords / upper-structure triads.** No way to stack two chord qualities
  (e.g. `D/C7`).
- **Chord qualities outside the fixed table.** Anything not in the list above
  (e.g. `7alt`, `13b9`, `min/maj7`, quartal voicings) silently falls back to
  the mode default — author a custom voicing via `degrees` + `rootPcOffsets` +
  the nearest supported quality if it matters.
- **Non-scale-degree slash bass.** Bass is parsed as a mode scale degree with
  one accidental; truly chromatic bass requires expressing the chord itself
  with `rootPcOffsets`.
- **Voicing register / inversion-by-revoicing.** Inversions are rotations
  only; you cannot specify "drop-2" or "spread closed voicing" from JSON.
- **Comments / authoring notes.** JSON has no comment syntax; long preset
  names are the only place to leave a hint.
- **Time-signature-aware notation.** `beats` is a raw number; there is no
  `"1 bar"` or `"half-bar"` shorthand, so a 3/4 preset must be written
  `[3, 3, 3, 3]`, not `[1, 1, 1, 1]`.
- **Tempo / feel metadata.** Swing, rubato, ritardando — none of these travel
  with a preset.

## Validation checklist before committing a preset

- [ ] `name` is unique across `default.json`.
- [ ] `mode` is one of the nine accepted strings.
- [ ] Every entry in `qualities` is either `null` or a string from the
      accepted list.
- [ ] Lengths of `qualities` / `beats` / `inversions` / `bass` /
      `rootPcOffsets` are 0 or equal to `degrees.length`.
- [ ] Borrowed-chord labels in `name` are consistent with the chosen `mode`
      (see invariant above).
- [ ] `default.json` parses as valid JSON (no trailing commas, no comments).
