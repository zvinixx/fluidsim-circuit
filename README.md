# FluidSIM Circuit Toolkit

<p align="center">
  <img src="https://img.shields.io/badge/FluidSIM-4.x%20Hydraulics-0a7ea4?style=for-the-badge" alt="FluidSIM 4.x">
  <img src="https://img.shields.io/badge/format-.ct-2ea44f?style=for-the-badge" alt=".ct">
  <img src="https://img.shields.io/badge/Agent%20Skills-compatible-6f42c1?style=for-the-badge" alt="Agent Skills">
  <img src="https://img.shields.io/badge/container-fully%20mapped-brightgreen?style=for-the-badge" alt="container mapped">
</p>

<p align="center">
  <img src="https://img.shields.io/github/license/zvinixx/fluidsim-circuit?style=flat-square" alt="License">
  <img src="https://img.shields.io/github/v/release/zvinixx/fluidsim-circuit?style=flat-square" alt="Release">
  <img src="https://img.shields.io/github/last-commit/zvinixx/fluidsim-circuit?style=flat-square" alt="Last commit">
</p>

<p align="center">
  <b>Reverse-engineered documentation and tooling for the FluidSIM 4 <code>.ct</code> circuit container — inspect circuits, rename labels without breaking them, and validate whether a file really opens.</b>
</p>

---

## What this is

A complete technical dossier on the `.ct` file format used by **FluidSIM 4 Hydraulics**
(Festo Didactic / Art Systems), plus a working codec and an automated validator.

The container is fully understood. The **payload** — how FluidSIM serialises a
circuit into that container — is understood structurally but **not yet
reproducible byte-for-byte**, which is why this project documents the boundary
instead of pretending to cross it.

| Capability | Status |
| :--- | :---: |
| Decode the container (obfuscation + LZ token stream) | ✅ complete |
| Read components, properties and labels from a `.ct` | ✅ works |
| Rename labels losslessly (same byte length) | ✅ works, verified visually |
| Detect whether a generated file actually opens | ✅ automated |
| Re-encode the container from literal tokens | ✅ accepted by the app |
| **Author a circuit from scratch** | ❌ **not yet possible** |
| **Insert a new component into an existing circuit** | ❌ **not yet possible** |

> This is deliberate. A tool that silently emits files FluidSIM rejects is worse
> than one that explains precisely where the wall is. Section
> [Where the boundary is](#where-the-boundary-is) has the full evidence.

---

## Install with one prompt

Paste this into any AI coding agent:

```text
Read https://raw.githubusercontent.com/zvinixx/fluidsim-circuit/main/SKILL.md
and install it as a skill named "fluidsim-circuit" in my global skills folder.
Then confirm it is installed and summarise what it can and cannot do.
```

<details>
<summary><b>Where the skill lands (per agent)</b></summary>

| Agent | Global folder | Project folder |
| :--- | :--- | :--- |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex / Cursor / Gemini CLI / any Agent Skills client | `~/.agents/skills/` | `.agents/skills/` |

Layout:

```text
<skills-folder>/
└── fluidsim-circuit/
    └── SKILL.md
```

Anything that reads a `SKILL.md` with YAML frontmatter works. No runtime
dependency on a specific vendor.

</details>

<details>
<summary><b>Manual install</b></summary>

```bash
git clone https://github.com/zvinixx/fluidsim-circuit.git
mkdir -p ~/.agents/skills
cp -r fluidsim-circuit ~/.agents/skills/fluidsim-circuit
```

Or paste the content of `SKILL.md` at the start of a chat — it works as a mode
with no file installation at all.

</details>

<details>
<summary><b>Verify the installation</b></summary>

```text
List your available skills. Is "fluidsim-circuit" there?
Then decode a .ct file and show me the component properties it contains.
```

</details>

---

## Requirements

| Item | Why |
| :--- | :--- |
| FluidSIM 4.x Hydraulics | the codec only needs the files; the validator opens the app |
| Python 3.8+ | runs the reference codec (only needed for the CLI workflow) |
| Windows (optional) | only for the automated open-validator, which drives window handles |

Adjust `D:\Hydraulic\` in `SKILL.md` if FluidSIM lives elsewhere — it is the
single machine-specific detail.

---

## Usage

### Inspect a circuit

```python
import ctcodec

print(ctcodec.describe(r'D:\Hydraulic\sym\hyd\actuator\a1.ct'))
for s in ctcodec.strings(r'D:\Hydraulic\sym\hyd\actuator\a1.ct', 5):
    print(s)
```

Output includes `BEGIN_FS`, `ENDCT`, `dxf_prim_list`, `Hyd.Actuator`,
`DAMPING_ADJ_L`, `only_top_rule`, `x_pos`, `stroke`, `gate1` and the component
description.

### Rename a component safely

```python
import ctcodec

hits = ctcodec.patch_text(
    'entrada.ct', 'saida.ct',
    [('Zylinder, doppeltwirkend', 'Cilindro principal     ')])
print('substituições:', hits)
```

The replacement is padded to the **exact same byte length**. Verified: the
resulting file opens in FluidSIM with the cylinder still drawn correctly.

### Validate

```powershell
powershell -File oracle.ps1 -CtPath saida.ct
# ABRIU | VAZIO | ERRO
```

Structural validation alone is not enough — a truncated file passes the token
check and still opens as an empty document. Only the application can arbitrate.

---

## Where the boundary is

### Proven facts

**1. Obfuscation layer.** Every byte is XOR'd with `0x9A`. Handler at
`0x41AEC0`, key stored at `0x5c544c`, initialised at `0x41A76C`.

**2. No header.** The token stream starts at byte 0. Proof by controlled
mutation: keeping the first 3 bytes intact and zeroing the rest produces
`Unerwartetes Dateiformat`; keeping 6 or 16 bytes and zeroing the rest also
fails. There is no field the app needs before the stream.

**3. Token grammar.** Verified exhaustively — the parser consumes 100% of the
bytes in all 201 sampled files, with zero desync.

```text
byte >= 0x80   →  literal, length = b - 0x80, followed by that many bytes
byte <  0x80   →  16-bit big-endian code
                  length = (code >> SHIFT) + 1
                  offset = code & (W - 1)

length == 1    →  RLE: value in next byte, repetitions = offset
length == 2    →  extended length: real length in next byte (3-byte token)
length >= 3    →  copy from ring buffer
```

Mode B (default): `W = 2048`, `SHIFT = 11`, `MAXLEN = 16`.
Mode A: `W = 512`, `SHIFT = 9`, `MAXLEN = 64`.

The ring is allocated with `0x7FF8` bytes and the write pointer flushes when
`lp + len >= 0x7FF8`.

The extended-length escape is the subtle one: treating `length == 2` as an
ordinary two-byte copy silently truncates the remainder of the file.

**4. Document model.** The payload is block-structured text:

```text
ENDPREVIEW
BEGIN_FS     … hydraulic sheet  … END_FS
ENDCT
BEGINDSD    … ladder / GRAFCET … ENDDSD
```

Property lines follow fixed formats confirmed against the format strings inside
the executable: `S %s B F`, `F %s x_pos I %ld`, `F %s stroke I %ld`,
`F %s MASS I %ld`, `DG %ld %ld %s …`, `POS %04ld %04ld %lf 1`.

### The remaining gap

Decoding recovers most of the payload — section headers, property names,
component descriptions, coordinates — but not byte-exactly: regions where the
ring buffer recycled come out corrupted. The practical consequence:

| Experiment | Result |
| :--- | :--- |
| Copy a real `.ct` | opens with content |
| Rename a label, same length | opens with content, drawing intact |
| Decode, re-encode using only literal tokens | opens **empty** |
| Assemble a document from scratch | opens **empty** |

Row three is the important one: it proves the **container encoding is correct**
(the app accepts it) while the **payload is incomplete**. Authoring a circuit
requires the exact plaintext, which in turn requires the compressor to be
reproduced or the decompressed buffer to be captured.

### How to close it

1. **Dynamic capture** — run `fl_sim_h.exe` under a debugger, break at
   `0x41B8FD` (immediately after decompression) and dump the output buffer at
   `[0x5d1db0]`. That yields the exact plaintext, with no inference.
2. **Reimplement the compressor** (`0x41A600` / `0x41A4E0`) and verify a
   byte-exact round-trip against that plaintext.
3. With a verified round-trip, authoring and component insertion become
   straightforward.

---

## How this was reached

**Stage 1 — Container identification.** 733 `.ct` files, no recognisable
header. A byte-frequency histogram over the corpus was flat but shifted by a
constant, which pointed to single-byte XOR. Applying `0x9A` to the whole file
produced structured data.

**Stage 2 — Grammar recovery.** Static analysis located the codec pair and its
global state: the key, ring size, shift packing length and offset, maximum
match length. The grammar was reconstructed from the decoder's control flow,
then validated statistically by requiring the parser to consume every byte of
every file. Four token classes emerged, two of them escapes keyed on the length
field.

**Stage 3 — Document model.** Decoded payloads turned out to be block-structured
text. Line formats were cross-checked field by field against the format strings
embedded in the executable.

**Stage 4 — The wrong turn.** An early validator read the window title and
reported success when it saw `FluidSIM-H`. That title appears even when the
document fails to load, so fabricated files looked valid. Correcting the
validator — enumerating MDI child windows instead of the frame — invalidated
several earlier conclusions and is the reason this document distinguishes
*accepted by the container parser* from *loaded as a circuit*.

**Stage 5 — Establishing the boundary.** Controlled mutation experiments
(truncate, zero ranges, shuffle, re-encode) mapped exactly which failures the
application tolerates and which it rejects. That is what the
`Where the boundary is` section reports.

---

## Technical reference

<details>
<summary><b>Executable map</b></summary>

| Address | Role |
| :--- | :--- |
| `0x41AEC0` | XOR apply / remove |
| `0x41A600` | compressor entry |
| `0x41A4E0` | LZ match search |
| `0x41AF80` | decompressor entry |
| `0x41B3A8` | decompressor loop |
| `0x41B5F7` | 16-bit code read |
| `0x41B740` | `length == 1` branch (RLE) |
| `0x41B786` | ring to output copy |
| `0x41B83D` | final flush |
| `0x4C8450` | decompression wrapper |

| Global | Meaning |
| :--- | :--- |
| `0x5c544c` | XOR key (`0x9A`) |
| `0x5c5450` | mode (0 = B/2048, ≠0 = A/512) |
| `0x5c5454` | ring size `W` |
| `0x5c5456` | `MAXLEN` (64 in A, 16 in B) |
| `0x5c5458` | shift |
| `0x5d1d94` | input buffer |
| `0x5d1d98` | ring buffer (`0x7FF8` bytes) |
| `0x5d1d9c` | write pointer `lp` |
| `0x5d1db0` | output buffer |

</details>

<details>
<summary><b>Property names</b></summary>

| Property | Meaning |
| :--- | :--- |
| `x_pos` / `y_pos` | symbol position on the sheet |
| `stroke` | cylinder stroke (mm) |
| `length` | drawn length |
| `MASS` | load mass (kg) |
| `DAMPING_ADJ_L` | damping |
| `only_top_rule` / `both_sides_rule` | drawing rules |
| `f_offset_x` / `f_offset_y` | label offset |
| `pos_l_<n>` / `pos_r_<n>` | limit-switch position |
| `sig_color` | signal colour |
| `PISTON_COUNT`, `PISTON_R`, `HUB` | cylinder geometry |

</details>

<details>
<summary><b>Symbol library layout</b></summary>

```text
sym/hyd/actuator/    cylinders  (a1.ct = double acting)
sym/hyd/valve/       valves
sym/hyd/supply/      pump, tank, filter, accumulator
sym/hyd/sensor/      manometer, pressure switch
sym/electr/switch/   switches (Manually, Proximit, General)
sym/electr/relay/    relays
sym/electr/ladder/   ladder contacts
sym/grafcet/         step, transition, action, sub, sync
```

</details>

---

## Repository structure

```text
fluidsim-circuit/
├── SKILL.md      agent skill: workflow, codec, format spec, validator
├── README.md     this document
└── LICENSE       MIT
```

---

## Contributing

The highest-value contribution is a captured plaintext of any `.ct` (stage 1 of
[How to close it](#how-to-close-it)). With one sample, the compressor can be
validated and the authoring limitation disappears.

Also welcome: additional property-name mappings, and reports of files whose
structure does not match the documented grammar.

Please open one pull request per feature.

## Credits

Reverse engineering, tooling and documentation: **[zvinixx](https://github.com/zvinixx)**.

FluidSIM is a product of Festo Didactic and Art Systems. This project is
independent and unaffiliated; it contains no vendor code or assets, only
documentation of an observed file format.

## License

[MIT](LICENSE) © zvinixx
