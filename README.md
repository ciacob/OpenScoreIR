# OpenScoreIR

> **A compact, pragmatic score interchange representation**  
> Focused on clean separation between **Score IR** (authoring/notation) and downstream **Layout** / **Performance** IRs.

This repository hosts the JSON Schemas and validation tests for **OpenScoreIR v1** (draft‑07). The goal is to capture just enough structure to encode typical notated music without forcing layout or playback policies into the authoring layer.

- **Schema draft:** JSON Schema **draft‑07**
- **Status:** early working draft
- **Scope (v1):** parts, measures, events & event groups, percussion sets, attachments/midi models, music spans, ordering lists
- **Out of scope (v1):** full layout rules, performance timing map, polymetric notation (by design)

---

## Why OpenScoreIR?

- **Separation of concerns:** Score IR stays agnostic of layout and playback policy. Those are compiled later into dedicated IRs.
- **Composable:** Attachables (articulations, text marks, MIDI models, etc.) share one formal structure.
- **Deterministic ordering:** `$lists` expresses canonical order of parts and measures.
- **Pragmatic primitives:** Events, tuplets/beams, percussion sets, syllables, fermatas, repeats/voltas, text jumps, etc.
- **Explicit anchoring:** Spans reference specific part & measure instances by UUID; repetition systems use link identifiers instead of global addresses.

---

## Repository layout

```
schemas/
  v1/
    score.schema.json         # root schema (draft-07)
    common.schema.json        # common defs (UUIDs, fractions, parts, measures, percussion sets, etc.)
    events.schema.json        # events, event groups (tuplet/beam), music spans
    attachables.schema.json   # attachable definitions & instances (attachments + midi models)
validation-tests/
  v1/
    basic.jsonc
    ...                       # more samples
```

---

## Quick start: validate a score

Install the CLI validator (Ajv) globally or use `npx`.

```bash
# global install
npm i -g ajv-cli ajv-formats

# compile schemas (optional but useful)
ajv compile \
  -s schemas/v1/score.schema.json \
  -r schemas/v1/common.schema.json \
  -r schemas/v1/attachables.schema.json \
  -r schemas/v1/events.schema.json \
  --strict=true

# validate all test docs
ajv validate \
  -s schemas/v1/score.schema.json \
  -r schemas/v1/common.schema.json \
  -r schemas/v1/attachables.schema.json \
  -r schemas/v1/events.schema.json \
  -d validation-tests/v1/*.jsonc \
  --strict=true
```

> **Note:** We intentionally target **draft‑07** for authoring ergonomics. If you prefer 2020‑12, you can port the schemas, but draft‑07 keeps development friction low.

---

## Authoring in editors (VS Code)

To enable schema-aware validation of your test files without embedding `$schema` in every document, add this workspace setting:

```jsonc
// .vscode/settings.json
{
  "json.schemas": [
    {
      "fileMatch": ["validation-tests/v1/*.json", "validation-tests/v1/*.jsonc"],
      "url": "./schemas/v1/score.schema.json"
    }
  ]
}
```

You can still include a `$schema` pointer inside individual documents:

```json
{
   "$schema": "../../schemas/v1/score.schema.json", 
   ... 
}
```

or use an absolute URL if you prefer (we serve the schema via GitHub Pages):

```json
{
  "$schema": "https://ciacob.github.io/OpenScoreIR/schemas/v1/score.schema.json",
  ...
}
```

---

## Core modeling concepts

### 1) `$definitions` (library of types)
- **parts** → reusable part *definitions* (what names, clefs, staff type, patch, etc. a part uses).
- **attachments** → attachable *definitions* (articulations, slurs, repeats, text jumps, syllables, fermatas, etc. &mdash; pretty much everything except the notes themselves).  
- **midiModels** → same attachable structure, intended for playback augmentation (hairpin, glissando, programChange, etc.).
- **percussionSets** → named sets with combos, each mapping a shortcut (`"D"`, `"X"`, …) to one or more MIDI pitches, note-head types and a staff positions.

> Attachables and MIDI models are structurally identical and validated by `attachables.schema.json`. **N.B.**: we give you the framework for defining attachments, and plenty of samples (see `validation-tests/v1/large-misc-file.jsonc`), but **not** the actual attachments. **You** will define attachments &mdash; e.g., to represent key signatures, clefs changes, tempo changes, dynamics, etc.

### 2) `$instances` (concrete objects)
- **parts** → instantiated parts (by definition UUID), with index/layer info.
- **measures** → instantiated measures with `duration`, optional `timeSignature`, barline, number override, attachments.

### 3) `$lists` (ordering)
- **parts** → array of part instance UUIDs in display/play order.
- **measures** → array of measure instance UUIDs in score order.

### 4) `musicSpans`
- Keyed by span UUID.
- Each span targets **one part** and **one measure** (or a multi-rest range) and contains:
- **events** (regular/grace/percussion) and/or
- **event groups** (tuplet/beam).

> `multiRest` spans specify `fromMeasure` and `toMeasure` instead of `events`.

### 5) Events and groups
- **Event**
  - `regular`: `duration` + `pitches` (`notes` optional for spelling), optional `attachments`, optional `midiOuts`.
  - `grace`: same, plus optional `useSlash`, `useSlur`, `stealNext`.
  - `percussion`: `duration` + `setId` + `strikes` (forbids `pitches`/`notes`).
- **EventGroup**
  - `tuplet`: `numTupletBeats`, `numRegularBeats`, `tupletBeatDuration`, `regularBeatDuration`, and `groupEvents`.
  - `beam`: `groupEvents` only.

---

## Data conventions

- **UUIDs:** standard canonical text form (`xxxxxxxx-xxxx-4xxx-8xxx-xxxxxxxxxxxx`), validated via `format: "uuid"`.
- **Fractions:** as `"N/D"` strings (e.g. `"1/2"`, `"3/8"`), or tuple Arrays `[1,2]`, `[3,8]`, or objects `{ "num": 1, "den": 2 }`. Bare integers (e.g., `1` for a _whole_) are **not** permitted for durations (use, e.g., `"1/1"`).
- **MIDI pitch values:** integers `[0, 127]` inclusive.
- **Note spelling:** `{ step: "A".."G", alteration: -2..2, octave: -1..10 }` (optional unless you need explicit spelling).
- **Measure duration vs time signature:** `duration` is normative for timing math; `timeSignature` is for display (may differ, e.g., the `duration` of a **6/8** measure is, mathematically, a `3/4` fraction).
- **Attachments & MIDI models:**
  - Definition params accept primitives (`string`, `number`, `integer`, `boolean`) **or** an object with `{ kind, values? }` for enumerations.
  - You specify an instance via either its UUID (useful when it takes no arguments), an object `{ id, params }`, or an array of such values (you can mix them).

---

## Example: minimal score

```json
{
  "$schema": "../../schemas/v1/score.schema.json",
  "$$docInfo": { "version": "1.0.0" },
  "$definitions": {
    "parts": {
      "12e04ff5-b3b2-4ac2-83b6-598fb213bc43": { "partName": "Piano" }
    }
  },
  "$instances": {
    "parts": {
      "b6ad1d74-58d5-4c7a-b81f-f318a57f2324": { "type": "12e04ff5-b3b2-4ac2-83b6-598fb213bc43" }
    },
    "measures": { "3cd43737-1fea-41a4-8895-324b377f746a": { "duration": "1/1" } }
  },
  "$lists": {
    "parts":   ["b6ad1d74-58d5-4c7a-b81f-f318a57f2324"],
    "measures":["3cd43737-1fea-41a4-8895-324b377f746a"]
  },
  "scoreInfo": { "title": "Hello" },
  "musicSpans": {
    "dddddddd-dddd-4ddd-8ddd-dddddddddddd": {
      "part": "b6ad1d74-58d5-4c7a-b81f-f318a57f2324",
      "measure": "3cd43737-1fea-41a4-8895-324b377f746a",
      "events": [ { "duration": "1/1", "pitches": [60, 64, 67] } ]
    }
  }
}
```

You find more examples in the `validation-tests` folder.

---

## Validation philosophy

- **Draft‑07 strictness:** we keep schemas precise but avoid validator gymnastics that impede authoring.
- **Discriminated unions:** event/group/span kinds are modeled via **separate object definitions** combined through `oneOf`.
- **Order matters:** `$lists` is normative; `$instances` dictionaries aren’t inherently ordered.
- **No global address math in Score IR:** repeats/text-jumps use **linkage identifiers**; downstream IRs resolve exact jumps and time.

---

## Contributing

1. Open an issue with a concise use case (ideally referencing an engraving norm).
2. Propose schema diff + a **new** sample under `validation-tests/v1/` that exercises the change.
3. Keep score concerns separate from layout/performance policy. If you need rendering or playback hints, prefer attachables.
4. Run the validator locally (see Quick start) before PR.

---

## Roadmap

- **Import/export guides:** mapping notes for MusicXML/MEI/SMuFL when applicable.
- **Reference implementations:** small readers/writers in JS/Python for sanity tests.

---

## License

This project is licensed under the terms of the [MIT License](./LICENSE).

