# FluidSIM Circuit Skill

<p align="center">
  <img src="https://img.shields.io/badge/FluidSIM-4.x%20Hydraulics-0a7ea4?style=for-the-badge" alt="FluidSIM 4.x">
  <img src="https://img.shields.io/badge/format-.ct-2ea44f?style=for-the-badge" alt=".ct">
  <img src="https://img.shields.io/badge/Agent%20Skills-compatible-6f42c1?style=for-the-badge" alt="Agent Skills">
  <img src="https://img.shields.io/badge/bases%20validated-201%2F201-brightgreen?style=for-the-badge" alt="201/201">
</p>

<p align="center">
  <img src="https://img.shields.io/github/license/zvinixx/fluidsim-circuit?style=flat-square" alt="License">
  <img src="https://img.shields.io/github/v/release/zvinixx/fluidsim-circuit?style=flat-square" alt="Release">
  <img src="https://img.shields.io/github/last-commit/zvinixx/fluidsim-circuit?style=flat-square" alt="Last commit">
  <img src="https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat-square" alt="PRs welcome">
</p>

<p align="center">
  <b>Generate FluidSIM hydraulic and electro-hydraulic circuits as ready-to-open <code>.ct</code> files — straight from a natural-language request.</b>
</p>

---

## What it does

Ask for a circuit. Get a `.ct` file you can double-click.

```
"Mount a hydraulic circuit with a double-acting cylinder, a directional valve and a pump."
```

```diff
+ Projeto.ct   generated
+ double-click  →  circuit opens in FluidSIM, fully editable and simulatable
```

| Capability | Status |
| :--- | :---: |
| Cylinder, valve, pump, tank, filter | ✅ |
| Cylinder with **limit switches / position sensors** | ✅ |
| Hydraulic motor | ✅ |
| Electro-hydraulic with solenoids and buttons | ✅ |
| Ladder logic, relays, indicator lamps | ✅ |
| GRAFCET / step sequences | ✅ |
| Rename components and labels | ✅ |
| Adjust physical values (stroke, mass) | ⚠️ validate afterwards |
| Draw a circuit from scratch | ❌ not possible today |

> **Read this first.** The `.ct` format is a compressed, obfuscated binary. Lossless
> re-encoding from scratch is not supported by this skill — it builds on **real
> base circuits**, which is what guarantees the file actually opens. Details in
> [Limits & why](#limits--why).

---

## Install with one prompt

Open your AI coding agent and paste this:

```text
Read https://raw.githubusercontent.com/zvinixx/fluidsim-circuit/main/SKILL.md
and install it as a skill named "fluidsim-circuit" in my global skills folder.
Then confirm it is installed and show me the three most useful prompts to use it.
```

That's it. The agent fetches the skill, drops it in the right folder for itself,
and reports back.

<details>
<summary><b>Where the skill lands (per agent)</b></summary>

| Agent | Global folder | Project folder |
| :--- | :--- | :--- |
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex / Cursor / Gemini CLI / others following the Agent Skills standard | `~/.agents/skills/` | `.agents/skills/` |

The layout is always the same:

```text
<skills-folder>/
└── fluidsim-circuit/
    └── SKILL.md
```

Anything that reads a `SKILL.md` with YAML frontmatter works — the skill has no
runtime dependency on a specific product.

</details>

<details>
<summary><b>Manual install</b></summary>

```bash
git clone https://github.com/zvinixx/fluidsim-circuit.git
mkdir -p ~/.agents/skills
cp -r fluidsim-circuit ~/.agents/skills/fluidsim-circuit
```

Or simply paste the content of `SKILL.md` at the start of a chat session — it
works as a mode, with no file installation at all.

</details>

<details>
<summary><b>Verify the installation</b></summary>

Ask the agent:

```text
List your available skills. Is "fluidsim-circuit" there?
```

Then, as a smoke test:

```text
Which circuits in the catalog have a cylinder and a limit switch?
```

A correct answer lists bases such as `ct/tp601/TP601_11b.ct` and
`ct/Shows/dida_173d.ct`.

</details>

---

## Requirements

| Item | Why |
| :--- | :--- |
| **FluidSIM 4.x Hydraulics** installed | the skill references the real component library in `D:\Hydraulic\` |
| Python 3.8+ *(optional)* | only to run the reference codec; the skill itself is plain Markdown |
| An AI agent that reads `SKILL.md` | any CLI or IDE extension following the Agent Skills standard |

> Different installation path? Adjust `D:\Hydraulic\` in `SKILL.md` — it is the
> only machine-specific detail.

---

## Usage

### Recipe — piston + pump + filter

```text
Build a hydraulic circuit with a double-acting cylinder, a pump, a filter and a
directional valve.
```

The skill picks `ct/Shows/dida_042.ct`, copies it and delivers `Projeto.ct`.

### Recipe — piston with limit switches

```text
I need a cylinder with limit switches to detect the end of stroke.
```

The skill picks `ct/tp601/TP601_11b.ct` — one cylinder, two limit switches
(`IS1`/`IS2`) wired into the ladder — and validates the output.

### Recipe — electro-hydraulic

```text
Make an electro-hydraulic circuit: cylinder driven by a solenoid with push buttons.
```

Base: `ct/tp601/TP601_08b.ct` → hydraulics + ladder + state table in a single file.

### Recipe — rename components

```text
Use TP601_11b as base and rename the cylinder to "Press actuator".
```

Renames are padded or truncated to the **same byte length**, which is what keeps
internal offsets valid.

### Phrases that work well

| Goal | Prompt |
| :--- | :--- |
| Create | *"Create a circuit with two cylinders working in sequence."* |
| Adjust | *"Use circuit X as base and translate the labels to English."* |
| Verify | *"Validate the file and tell me whether it opens."* |
| Analyse | *"Show the five bases closest to 'piston with motor'."* |

---

## How to place a piston (manual reference)

The skill does this for you, but here is what it looks like inside FluidSIM:

1. **Library → Hydraulics → Actuators** → drag *Double-acting cylinder*.
2. Right-click → **Properties**:

   | Field | Meaning |
   | :--- | :--- |
   | `x_pos` / `y_pos` | position on the sheet |
   | `stroke` | stroke in mm (default 200) |
   | `length` | drawn length |
   | `MASS` | load mass (kg) |
   | `DAMPING_ADJ_L` | damping |

3. **Library → Valves → Directional valve** → connect A/B to the cylinder, P/T
   to pump and tank.

In the `.ct` file each of these becomes a property line:

```text
 F x_pos I 120
 F y_pos I 240
 F stroke I 200
 F MASS I 9
```

## How to place a limit switch

1. Open the circuit that already contains the cylinder.
2. **Library → Electrical → Switches → Proximity switch**.
3. Drag the sensor **onto the cylinder rod** — it snaps and starts sensing.
4. Wire contact **13/14** to the relay coil or the logic input.
5. Repeat on the opposite end for the second limit switch.

In the file it appears as:

```text
 S ProximitySwitch B F
 F pos_l_<n> I <coordinate>
 F pos_r_<n> I <coordinate>
```

and, in the ladder, as contact `IS1` / `IS2` on terminals 13/14.

---

## Catalog

201 bases validated by opening each generated file in FluidSIM. Paths relative to
`D:\Hydraulic\`.

| Base | Components |
| :--- | :--- |
| `ct/tp601/TP601_11b.ct` | cylinder, directional valve, **limit switch**, solenoid, pump |
| `ct/Shows/dida_173d.ct` | 2 cylinders, **limit switch**, pump, check valve, pressure valve |
| `ct/tp501/TP501_20.ct` | cylinder, **limit switch**, pump, check valve |
| `ct/tp701/TP701_09.ct` | cylinder, **limit switch**, filter |
| `ct/tp602/TP602_01.ct` | cylinder, **limit switch**, tank, lamp |
| `ct/Grafcet18.ct` | GRAFCET, cylinder, **limit switch** |
| `ct/Shows/dida_042.ct` | cylinder, directional valve, pump, filter |
| `ct/Shows/dida_037.ct` | cylinder, directional valve, pump, tank |
| `ct/tp601/TP601_08b.ct` | cylinder, solenoid, pump, ladder + state table |
| `ct/tp502/TP502_01.ct` | hydraulic motor, directional valve, pump |
| `ct/tp502/TP502_05a.ct` | cylinder, throttle (speed control) |
| `ct/tp502/TP502_10.ct` | two cylinders + state diagram |
| `ct/demo_02.ct` | basic cylinder with manual valve |

The full 201-entry table lives in [`SKILL.md`](SKILL.md) §7.

---

## Limits & why

The `.ct` container is a **block-oriented LZ77 stream under a single-byte XOR**.
There is no magic header and no uncompressed fallback, so writing a circuit from
scratch would require reproducing the vendor's compressor, including its ring-buffer
resynchronisation at every block flush.

What that means in practice:

| Scenario | Result |
| :--- | :--- |
| Copy a real base and rename labels | ✅ opens and simulates |
| Rebuild the full text with literal tokens only | ❌ `Unknown file format` |
| Add a component the base does not have | ❌ not implemented |

The skill is therefore honest by design: it selects the closest real base and
tells you what is missing instead of emitting a file that will be rejected.

**Path to close the gap** — the remaining work is documented in `SKILL.md` §8:

1. Dynamic analysis: breakpoint on the decompressor epilogue, dump the output buffer.
2. Reimplement the compressor and validate a byte-exact round-trip.
3. Automate the host application UI (paste symbols, save).

---

## How this skill was built

The project started from a single question — *can a description become a finished,
double-clickable circuit file?* — and proceeded in five stages.

### 1. Container identification

A sample of 733 `.ct` files showed no recognisable header. Frequency analysis over
the full corpus revealed a flat single-byte distribution shifted by a constant:
every file is XOR-obfuscated with `0x9A`. Applied to the whole file, the payload
turned into a structured byte stream.

### 2. Grammar recovery

Static analysis of the host executable located the codec pair and its global state:
the XOR key, the ring size, the shift used to pack length and offset, and the
maximum match length. The token grammar was reconstructed from the decoder control
flow and then **verified statistically**: the reconstructed parser consumes every
byte of all sampled files without desynchronising.

Three token classes emerged: literal runs, ring-buffer copies, and two escapes keyed
on the length field — a run-length form and an extended-length form. The second
escape was the subtle one: treating it as an ordinary copy silently truncates the
tail of the file, and fixing it moved validation from 195/201 to **201/201**.

### 3. Document model

With the stream decoded, the payload proved to be block-structured text: an
overview block, a hydraulic sheet, and an electrical/GRAFCET sheet, each with
`S`/`F`/`D` property lines. Line formats were cross-checked against the format
strings embedded in the executable, which matched field for field.

### 4. Base-circuit strategy

Because lossless re-encoding was not reproducible, the approach shifted to
**composition from real circuits**: index every shipped circuit by the components
it contains, rank candidates against the request, copy the winner, and apply
same-length label substitutions so internal offsets remain valid.

### 5. Validation loop

Every candidate was checked by actually opening the generated file in FluidSIM,
reading the window title, and comparing against the expected circuit name. This
loop is what turned a plausible theory into a verified pipeline — and what exposed
the extended-length bug.

---

## Technical reference

<details>
<summary><b>Format at a glance</b></summary>

```text
file
 └─ XOR 0x9A  (whole file)
     └─ token stream
         ├─ literal      b >= 0x80          → b - 0x80 bytes follow
         ├─ copy         b <  0x80, len >= 3 → 2 bytes, big-endian
         ├─ RLE          len == 1           → 3 bytes (value follows)
         └─ extended     len == 2           → 3 bytes (real length follows)

 code      = (b0 << 8) | b1
 length    = (code >> SHIFT) + 1
 offset    = code & (W - 1)

 Mode B (default): W = 2048, SHIFT = 11
 Mode A:           W =  512, SHIFT =  9
```

Document sections:

```text
ENDPREVIEW
BEGIN_FS   … hydraulic sheet … END_FS
ENDCT
BEGINDSD  … ladder / GRAFCET … ENDDSD
```

</details>

<details>
<summary><b>Executable map</b></summary>

| Address | Role |
| :--- | :--- |
| `0x41AEC0` | XOR apply/remove |
| `0x41A600` | compressor entry |
| `0x41A4E0` | LZ match search |
| `0x41B3A8` | decompressor loop |
| `0x41B5F7` | 16-bit code read |
| `0x41B740` | `length == 1` branch (RLE) |
| `0x41B786` | ring → output copy |
| `0x41B83D` | final flush |

| Global | Meaning |
| :--- | :--- |
| `0x5c544c` | XOR key (`0x9A`) |
| `0x5c5454` | ring size `W` |
| `0x5c5458` | shift |
| `0x5d1d98` | ring buffer |
| `0x5d1db0` | output buffer |

</details>

<details>
<summary><b>Symbol library layout</b></summary>

```text
sym/hyd/actuator/    cylinders   (a1 = double acting)
sym/hyd/valve/       valves
sym/hyd/supply/      pump, tank, filter, accumulator
sym/hyd/sensor/      manometer, pressure switch
sym/electr/switch/   switches  (Manually, Proximit, General)
sym/electr/relay/    relays
sym/electr/ladder/   ladder contacts
sym/grafcet/         step, transition, action, sub, sync
```

</details>

---

## Repository structure

```text
fluidsim-circuit/
├── SKILL.md      complete skill: workflow, catalog, reference codec, format spec
├── README.md     this document
└── LICENSE       MIT
```

Two files, no prototypes, no fixtures.

---

## Contributing

Contributions are welcome, especially:

- new **validated** base circuits (with the components you verified visually);
- support for adding components to an existing circuit;
- a byte-exact compressor, which would remove the base-circuit limitation entirely.

Please open one pull request per feature so each change can be reviewed on its own.

## Credits

Reverse engineering, skill design and implementation: **[zvinixx](https://github.com/zvinixx)**.

FluidSIM is a product of Festo Didactic and Art Systems. This project is
independent and unaffiliated; it contains no vendor code or assets, only
documentation of an observed file format.

## License

[MIT](LICENSE) © zvinixx
