# AGENTS.md

This file provides guidance to coding agents when working with code in this repository.

## What this is

A proof of concept: GNSS navigation message formats (ICDs) are described in language-agnostic YAML files (`GPS/`, `Galileo/`, `GLONASS/`, `BeiDou/`), and a Python tool (`icd-manager`, package `src/gnss_parser`) reads them to generate documentation (Markdown, Typst PDF) and parser code (Zig), or to parse a u-blox `RXM-SFRBX` stream directly in Python.

## Commands

Tools: `uv` (Python ≥ 3.11), `zig` (≥ 0.14, for the binary), `typst` (for the PDF).

```bash
make markdown          # default target: regenerate <ICD>.md next to each <ICD>.yaml
make pdf               # typst compile ephemerides.typ -> ephemerides.pdf
make zig               # generate zig/generated.zig, then `zig build` -> zig/zig-out/bin/ephemerides
make clean

# Python parser on a recorded stream (ARGS defaults to --serial, so override it for files)
make parse PORT=test/three_GPS_ephemerides.ubx ARGS=--dump
# Zig parser on a recorded stream (modes: subframe, raw_ephemeris, ephemeris)
./zig/zig-out/bin/ephemerides test/one_galileo.ubx -m subframe
./zig/zig-out/bin/ephemerides /dev/ttyACM0 -b 115200

# Direct CLI: -I/--icd is a global option (repeatable) and must precede the subcommand
uv run icd-manager -I Galileo/INAV.yaml translate md
uv run icd-manager -I GPS/LNAV-L.yaml -I BeiDou/D1.yaml translate zig | zig fmt --stdin
uv run icd-manager -I GPS/LNAV-L.yaml parse test/three_GPS_ephemerides.ubx --dump
```

There is no test suite or linter. Verification is done by regenerating outputs and parsing the sample captures in `test/*.ubx`.

## Generated files

- `*/<ICD>.md` are generated from the YAML and **committed**. CI (`.github/workflows/generate.yaml`) runs `make clean markdown` on push/PR and auto-commits the result, so after editing a YAML or the Python sources, regenerate with `make markdown` and commit the `.md` alongside.
- `zig/generated.zig` is gitignored; it is rebuilt from all YAMLs by `make zig`. Never hand-edit it. Change `src/gnss_parser/generators/zig.py` instead.

## Architecture

### YAML ICD schema (`kind: GNSS_format`)

- `metadata`: `constellation` (must match a `Constellation` enum name in `constellations.py`, whose values are the u-blox `gnssId`s), `message`, optional `description`.
- `ublox`: `signal` (u-blox `sigId`s mapping to this message) and `layout`, which lists the data bits of each 32-bit word u-blox delivers (`words`, `discard_msb`, `keep`). An optional `subframe_id` says where u-blox puts the subframe ID when it isn't in the message header.
- `header`: fields read at the start of every subframe; must contain `subframe_id`.
- `page_header`: optional, read after the header for subframes whose formats have `pages`; must contain `page_id`.
- `formats`: list of `{subframe, pages?, description?, fields}`.
- `valid_subframes`: optional, every subframe ID the ICD defines, implemented or not. It must include every implemented subframe; the ones without a format become `error.UnimplementedSubframe` in Zig instead of `error.InvalidSubframe`, and are listed in the generated Markdown. Defaults to the implemented subframes, so only messages with known gaps (currently Galileo) need it.
- Field keys: `bits` (required), `name` (omit for reserved/padding), `value` (expected constant), `signed` (two's complement), `shift` (×2^shift) **or** `factor` (never both), `unit` (astropy unit string), `latex`, `half: msb|lsb` (a field split across subframes, recombined later by name).
- `subframe`, `pages`, `words` and `signal` accept "range lists": an int, or a list mixing ints and `[first, last]` inclusive ranges (`yaml.RangeList`). YAML anchors and aliases (`&x` / `*x`) are used to repeat fields.

The same schema is consumed in **four places**, so a schema change usually has to be made in all of them:
1. `src/gnss_parser/format.py` + `field.py` + `ublox.py`: parsing the YAML into `GnssFormatHandler` / `GnssFormat` / `FieldArray` / `Field`.
2. `src/gnss_parser/generators/markdown.py`
3. `src/gnss_parser/generators/zig.py`
4. `ephemerides.typ`: Typst reads the YAMLs itself with `yaml()` and reimplements range-list handling (`rangelist`). The list of YAML files and the per-constellation notes (time rollovers, etc.) are hardcoded there.

### Python runtime path (`parse` subcommand)

`__main__.py` reads `RXM-SFRBX` messages via `pyubx2`, maps `(gnssId, sigId)` to a message through `GnssFormatHandler.ublox_mapping`, and turns the payload words into a bit stream with a **hand-written, per-message extractor in `ublox.reader_from_ublox`** (parity checking and stripping for GPS LNAV, BeiDou D1, Galileo FNAV, GLONASS). The Python path does not use the YAML `ublox.layout`, so a message without an entry there (e.g. Galileo INAV) can't be parsed in Python. `accumulate.py` then stores subframes per satellite and fires subscribers (currently just printing ephemerides) once the needed subframes have arrived, merging `msb`/`lsb` halves.

### Zig path

`generators/zig.py` emits one namespace struct per message. Each contains a `Reader` (`SkippingBitReader` built from `ublox.layout`, which comes from the external `utils` module), `Header`/`PageHeader`/`Subframe<N>[Page<M>]` structs with `read_*` functions, a tagged `Data` union, and `Subframe.from_ublox`. It also emits the top-level `Constellation`, `GnssMessageType`, `GnssSubframe` and `Subframe` types. `zig/main.zig` is hand-written: it reads the u-blox stream through the `orbital_elements` dependency (modules `utils` and `o2s`, which wrap a C u-blox library), accumulates subframes per satellite, and outputs JSON. Ephemeris assembly in `main.zig` is hardcoded per message (currently LNAV-L and D1).

### Adding a new message

1. Write `<Constellation>/<MSG>.yaml`.
2. Add it to the hardcoded list in `ephemerides.typ` if it should appear in the PDF.
3. For Python parsing, add a word extractor to `ublox.reader_from_ublox` (and optionally a subscriber in `accumulate.py`).
4. For ephemeris output from the Zig binary, extend `receive_ublox_subframe` in `zig/main.zig`.
5. Run `make markdown zig`, then check against a capture.
