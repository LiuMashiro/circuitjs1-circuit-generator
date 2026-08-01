---
name: "circuitjs1-circuit-generator"
description: "Generates circuitjs1-importable plain-text circuits from natural language. Invoke when user asks to create/design/build a circuit, or wants AI to generate a circuit for the circuitjs1/Falstad simulator."
---

# CircuitJS1 Circuit Text Generator

This skill enables the AI to generate complete, valid circuitjs1 circuits as plain text (code). The output can be imported into the circuitjs1 (Falstad) circuit simulator via **File → Import From Text...** (Chinese UI: 文件 → 从文本导入...) without any GUI operations.

## When to Invoke

Invoke this skill when:
- User asks to "create / design / build / generate a circuit"
- User describes a circuit in natural language (e.g., "an LED with a 220 ohm resistor on 5V")
- User wants the AI to produce a circuit for the circuitjs1 / Falstad simulator
- User asks for a schematic-like output that can be simulated
- User mentions "circuitjs1", "Falstad circuit", or "import from text"

## How the User Imports the Output

1. **Import From Text**: Open circuitjs1 → menu `File → Import From Text...` (Chinese UI: 文件 → 从文本导入...) → paste the entire text block → click OK.
2. **URL parameter**: Replace every `$` with `%24` and every newline with `%0A`, then append to the simulator URL as `?cct=<encoded>`. (www.falstad.com/circuit/circuitjs.html?cct=...) **You do NOT need to access this URL.** The AI should provide this URL as a deliverable alongside the text block (see §14).
3. **Local file**: Save as `.txt`, then `File → Import From File...` (Chinese UI: 文件 → 打开文件...).

The AI should always present the circuit inside a fenced code block so the user can copy it easily.

### URL construction rule (for the AI to generate deliverable URLs)

To build a clickable URL that opens the circuit directly in the Falstad online simulator:

1. Take the complete circuit text (starting with `$`, including all lines).
2. **First**, if the circuit text contains any literal `%` characters (e.g. from legacy `%2B` encoding of `+` in sliderText), replace `%` with `%25` BEFORE any other replacement. This prevents the URL decoder from misinterpreting `%24`/`%2B` sequences. (Best practice: use the `\p` escape from §4 instead of `%2B` to avoid this issue entirely.)
3. Replace every `$` character with `%24`.
4. Replace every newline (`\n`) with `%0A`.
5. Replace spaces with `+` (or `%20`). Do NOT leave raw spaces in the URL — use `+` for robustness.
6. Prepend `https://www.falstad.com/circuit/circuitjs.html?cct=` to the encoded string.

**Example** — circuit text:
```
$ 1 5.0E-6 10 50 5.0 50
v 96 256 96 64 0 0 40.0 5.0 0.0 0.0 0.5
g 96 256 96 288 0
```

Becomes URL:
```
https://www.falstad.com/circuit/circuitjs.html?cct=%24+1+5.0E-6+10+50+5.0+50%0Av+96+256+96+64+0+0+40.0+5.0+0.0+0.0+0.5%0Ag+96+256+96+288+0
```

**Length warning**: URLs over ~2000 characters may be truncated by some browsers/servers. For long circuits, prefer the text-block import method. The URL is best for short circuits (≤30 lines) or for sharing.

## Output Language

circuitjs1 itself supports Chinese localization.

- **If the user asks in Chinese**, respond in Chinese for all explanations, expected behavior, and import instructions. The circuit text itself is language-neutral (format is identical regardless of language).
  - String fields inside the circuit text (sliderText, text labels, model names, scope labels) MAY contain Chinese characters — the escape rules in §4 handle them correctly. When the user uses Chinese, prefer Chinese labels (e.g. sliderText `亮度` for "brightness").

- **If the user asks in English**, respond in English.

---

## 1. Format Overview

The circuit is **line-oriented plain text**:

> **draft before you write.** draw an ASCII-art schematic to lock the topology, then assign exact coordinates (node list + component list + connectivity trace, see §9 Step 2) and verify it passes the closed-loop, ground-attachment, wire-splitting, and no-floating-node checks. Most simulator errors are preventable at the draft stage.

- **Line 1** must be the global options line starting with `$`.
- **Subsequent lines** each represent one entity: a component, a model definition, a scope, or a slider.
- Tokens within a line are separated by **space**, **`+`**, **tab**, or any whitespace. (`+` is a separator — never use it raw in string fields; use the escape rules in §4.)
- **Empty lines are silently skipped.**
- **Comments are NOT supported.** Every non-empty line is parsed.
- Lines are parsed independently; a malformed line is skipped (caught by try/catch) but does NOT abort the whole load.

### Entity type dispatch (by first token)

| First token | Entity type | Notes |
|-------------|-------------|-------|
| `$` | Global options | Must be line 1 |
| `o` (lowercase letter o) | Scope (oscilloscope) | Panel object, references a component by index |
| `h` | Hint | Optional, rarely needed |
| `38` | Adjustable slider | References a component by 0-based index |
| `32` | TransistorModel definition | Referenced by TransistorElm via modelName |
| `34` | DiodeModel definition | Referenced by DiodeElm/LEDElm/ZenerElm via modelName |
| `!` | CustomLogicModel definition | Referenced by CustomLogicElm (208) |
| `.` | CustomCompositeModel definition | Referenced by CustomCompositeElm (410); contains nested sub-circuit |
| `%` `?` `B` | Ignored (legacy afilter) | Safe to omit |
| Any letter (ASCII < 127) | Component, char-typed | e.g. `r`, `c`, `w`, `v`, `g` |
| Any number ≥ 128 | Component, number-typed | e.g. `162` (LED), `172` (VarRail) |

**Critical ordering rule**: A model definition line (34/32/!/.) MUST appear BEFORE the component line that references it. (This mirrors `dumpCircuit`, which calls `dumpModel()` before `dump()`.)

After reading and executing this step, you must explicitly mark **"Step 1 completed"** in the reasoning process.

---

## 2. Global Options Line (Required, Line 1)

```
$ <flags> <maxTimeStep> <speedParam> <currentBar> <voltageRange> [<powerBar> <minTimeStep>]
```

| Field | Type | Meaning | Recommended |
|-------|------|---------|-------------|
| flags | int | Bitmask: bit0=show current dots, bit1=small grid, bit2=hide voltage, bit3=show power, bit4=hide values | `1` |
| maxTimeStep | double | Max simulation time step in seconds | `5.0E-6` |
| speedParam | double | Speed (mapped via `log(10*sp)*24+61.5`) | `10` |
| currentBar | int | Current display bar | `50` |
| voltageRange | double | Voltage display range (V) | `5.0` |
| powerBar | int | Power bar (optional) | `50` |
| minTimeStep | double | Min time step (optional, can omit) | omit |

**Always use this exact line unless the user requests otherwise:**
```
$ 1 5.0E-6 10 50 5.0 50
```

---

## 3. Component Line Format & Connection Rules

### 3.1 Universal component prefix

Every component line begins with:
```
<dumpType> <x1> <y1> <x2> <y2> <flags> [<subclass fields>...]
```

- `dumpType`: a single ASCII char (e.g. `r`) when < 127, or an integer (e.g. `162`) when ≥ 127.
- `(x1,y1)` and `(x2,y2)` are the two endpoint coordinates in pixels.
- `flags`: bitmask, usually `0` for beginners (see §5 for per-component flag bits).

### 3.2 Connection rule (THE key mechanism)

**Two endpoints with identical coordinates are electrically connected (same node).**

```
Component A endpoint (192, 64)  ==  Component B endpoint (192, 64)  →  connected
Component A endpoint (192, 64)  !=  Component B endpoint (192, 65)  →  NOT connected
```

- Use **wires (`w`)** to bridge distant endpoints. A wire is itself a component whose two endpoints are its connection points.
- GroundElm has ONE terminal at point1 (point2 is only for drawing the ground symbol). All GroundElm point1 coordinates share a single global "ground node" (node 0).
- Coordinates are in pixels. Use multiples of 8 (the default grid snap) to keep things tidy. Recommended range: 16–1000.

**WARNING — Wire intermediate points do NOT connect**: A wire only connects at its TWO endpoints (point1 and point2). If another component's terminal happens to lie on the wire's path (e.g. a wire from (64,288) to (384,288) passes through (96,288), and a voltage source negative terminal is at (96,288)), they are **NOT connected** unless (96,288) is an endpoint of that wire. **Fix**: split the wire into two segments at the junction: `w 64 288 96 288 0` + `w 96 288 384 288 0`, so (96,288) becomes a wire endpoint.

### 3.3 Coordinate layout advice

- Plan the layout mentally before writing lines.
- Assign coordinates so that intended connections share exact coordinates.
- Power rails often run horizontally at fixed y-values (e.g. VCC at y=64, GND at y=256).
- Components are typically vertical (different y, same x) or horizontal (different x, same y).

After reading and executing this step, you must explicitly mark **"Step 3 completed"** in the reasoning process.

---

## 4. Escape Rules (CustomLogicModel.escape)

Some string fields (modelName, sliderText, text labels, rules, etc.) may contain characters that would break tokenization. Use this escape scheme:

| Raw character | Escaped as |
|---------------|------------|
| `\` (backslash) | `\\` |
| space ` ` | `\s` |
| `+` | `\p` |
| `=` | `\q` |
| `#` | `\h` |
| `&` | `\a` |
| newline `\n` | `\n` (backslash + n) |
| carriage return `\r` | `\r` (backslash + r) |
| empty string | `\0` |

**When to escape**: Always escape string fields that may contain spaces or special chars. Numeric fields never need escaping.

**Components that use escape**:
- DiodeElm / LEDElm / ZenerElm / TransistorElm: `modelName` (when flags has FLAG_MODEL)
- VarRailElm: `sliderText` (also `+` → `%2B` legacy; prefer escape)
- LabeledNodeElm (207): `text` (set flags bit2 = FLAG_ESCAPE = 4)
- TextElm (`x`): `text` (set flags bit2 = FLAG_ESCAPE = 4)
- Adjustable (38): `sliderText`
- CustomLogicModel (`!`): name, inputs, outputs, infoText, rules
- Scope (`o`): optional text label

---

## 5. Complete Component Reference

All fields are listed in exact dump order. "State vars" are simulation state; initialize to `0` unless stated.

### 5.1 Basic Passive Components

| Type | Class | Full field format |
|------|-------|-------------------|
| `w` | WireElm | `w x1 y1 x2 y2 flags` |
| `r` | ResistorElm | `r x1 y1 x2 y2 flags resistance` |
| `c` | CapacitorElm | `c x1 y1 x2 y2 flags capacitance voltdiff initialVoltage` |
| `l` | InductorElm | `l x1 y1 x2 y2 flags inductance current` |
| `g` | GroundElm | `g x1 y1 x2 y2 flags symbolType` |
| `d` | DiodeElm | `d x1 y1 x2 y2 0` (flags=0 → default model) |
| `z` | ZenerElm | `z x1 y1 x2 y2 0` (flags=0 → default model) |
| `t` | TransistorElm | `t x1 y1 x2 y2 flags pnp Vbc Vbe [beta] [modelName]` (beta defaults to 100, modelName defaults to "default") |
| `f` | MosfetElm | `f x1 y1 x2 y2 flags vt beta` |
| `209` | PolarCapacitorElm | `209 x1 y1 x2 y2 flags capacitance voltdiff initialVoltage maxNegativeVoltage` |

**Field meanings**:
- `resistance` (Ω), `capacitance` (F), `inductance` (H): physical values.
- `voltdiff`, `current`, `Vbc`, `Vbe`: simulation state vars → fill `0`.
- `initialVoltage` (capacitor): fill `0`.
- `symbolType` (ground): `0` = earth symbol.
- `pnp` (transistor): `1` = NPN, `-1` = PNP.
- `beta` (transistor): current gain, typical `100`.
- `vt` (MOSFET): threshold voltage. Source default `1.5` V; typical logic-level MOSFET `0.75`–`1.0` V.
- `beta` (MOSFET): transconductance param. Source default `0.02`; typical `0.03`. The model is Shichman-Hodges (no channel-length modulation by default).
- **MOSFET body diode**: the GUI auto-sets `FLAG_BODY_DIODE=32` when creating a MOSFET, but **text import does NOT** — a `flags=0` MOSFET has NO body diode. To enable the body diode (recommended for real circuits with inductive loads), set `flags=32` (or `33` = 32+1 if also PNP). The body diode uses the default diode model (Vf≈0.806V).
- `modelName` (transistor): optional; omit to use default model.
- `maxNegativeVoltage` (polar cap): max reverse voltage before sim stops, default `1`.

**TransistorElm pinout** (3 terminals): node 0 = base (point1), node 1 = collector, node 2 = emitter. Collector/emitter positions are geometry-derived from (x2,y2) with offset 16px — you only specify two points.

**Transistor orientation** (NPN, pnp=1, horizontal layout where point1 is left, point2 is right):
- **flags=0** (no flip): collector at (x2, y2−16) = ABOVE, emitter at (x2, y2+16) = BELOW. Use this for standard NPN switch (collector→load→VCC, emitter→GND).
- **flags=1** (FLAG_FLIP): collector BELOW, emitter ABOVE. Use for PNP-style or inverted layouts.
- For PNP (pnp=−1), the positions are swapped relative to NPN.

**MOSFET pinout** (3 terminals): node 0 = gate (point1), node 1 = drain, node 2 = source. Same geometry derivation as transistor. **flags=1 (FLAG_PNP)** selects P-channel MOSFET (pnp=−1); **flags=8 (FLAG_FLIP)** swaps drain/source geometric positions.

### 5.2 Sources

| Type | Class | Full field format |
|------|-------|-------------------|
| `v` | VoltageElm | `v x1 y1 x2 y2 flags waveform frequency maxVoltage bias phaseShift dutyCycle` |
| `R` | RailElm | `R x1 y1 x2 y2 flags waveform frequency maxVoltage bias phaseShift dutyCycle` |
| `172` | VarRailElm | `172 x1 y1 x2 y2 flags waveform frequency maxVoltage bias phaseShift dutyCycle sliderText` |
| `i` | CurrentElm | `i x1 y1 x2 y2 flags currentValue` |

**waveform values** (from VoltageElm constants):

| Value | Meaning |
|-------|---------|
| 0 | DC |
| 1 | AC (sine) |
| 2 | Square |
| 3 | Triangle |
| 4 | Sawtooth |
| 5 | Pulse |
| 6 | Noise |
| 7 | Variable (VarRailElm only, pairs with slider) |

**DC 5V source example**: `v 96 256 96 64 0 0 40.0 5.0 0.0 0.0 0.5`
- flags=0, waveform=0(DC), frequency=40, maxVoltage=5.0, bias=0.0, phaseShift=0.0, dutyCycle=0.5

**RailElm vs VoltageElm**: RailElm draws as a circle "rail" with ONE external terminal at point1 (the other end is internal ground; point2 is only for drawing direction). VoltageElm draws as a battery with TWO external terminals (point1 and point2). **For closed loops, prefer `v` (VoltageElm)** since it has explicit + and − terminals.

**LogicInputElm** (`L`): has ONE external terminal at point1 (output). point2 is only for drawing direction. The output voltage is hiV (default 5V) or loV (default 0V) depending on `position`.

**VarRailElm**: has a built-in slider; `sliderText` is the label (escape spaces, or use `%2B` for `+`).

### 5.3 Switches

| Type | Class | Full field format |
|------|-------|-------------------|
| `s` | SwitchElm (SPST) | `s x1 y1 x2 y2 flags position momentary` |
| `S` | Switch2Elm (SPDT) | `S x1 y1 x2 y2 flags position momentary link throwCount` |

- `position`: `0` = closed, `1` = open (SwitchElm); in Switch2Elm it selects which throw is connected.
- `momentary`: `false` or `true` (literal string).
- `link`: linkage group ID, `0` = none. Switches sharing the same link toggle together.
- `throwCount`: number of throws, default `2`.

### 5.4 Measurement & Display

| Type | Class | Full field format |
|------|-------|-------------------|
| `p` | ProbeElm | `p x1 y1 x2 y2 flags meter scale` |

**WARNING — ProbeElm is a high-impedance voltmeter**: ProbeElm does NOT conduct current (`getConnection()=false`, `stamp()` is empty). It measures `volts[0]-volts[1]` (point1 minus point2). It must be used in **parallel** (like a voltmeter), never in series. If placed in series, it breaks the circuit and the downstream side becomes grey/floating. Both endpoints MUST connect to nodes that already have other driving components (sources, resistors, ground). **For simple voltage measurement, prefer a scope (`o` line) instead** — scopes don't occupy circuit topology and never cause floating nodes.
| `O` | OutputElm | `O x1 y1 x2 y2 flags scale` |
| `370` | AmmeterElm | `370 x1 y1 x2 y2 flags` |
| `162` | LEDElm | `162 x1 y1 x2 y2 0 colorR colorG colorB` (flags=0 → LED default model) |
| `181` | LampElm | `181 x1 y1 x2 y2 flags temp nom_pow nom_v warmTime coolTime` |

**LED fields** (recommended flags=0 format):
- `colorR`/`colorG`/`colorB`: RGB 0.0–1.0. Red = `1.0 0.0 0.0`, Green = `0.0 1.0 0.0`, Blue = `0.0 0.0 1.0`, White = `1.0 1.0 1.0`, Yellow = `1.0 1.0 0.0`.
- A 4th field `maxBrightnessCurrent` (default 0.01) may follow but is optional.

**ProbeElm meter values**:

| Value | Meaning |
|-------|---------|
| 0 | Voltage |
| 1 | RMS |
| 2 | Max |
| 3 | Min |
| 4 | Peak-to-Peak |
| 5 | Binary |
| 6 | Frequency |
| 7 | Period |
| 8 | Pulse Width |
| 9 | Duty Cycle |

**LampElm fields**: `temp` (filament temp, init 300), `nom_pow` (rated power W), `nom_v` (rated voltage V), `warmTime`/`coolTime` (time constants, default 0.4).

### 5.5 Logic Components

| Type | Class | Full field format |
|------|-------|-------------------|
| `L` | LogicInputElm | `L x1 y1 x2 y2 flags position momentary hiV loV` |
| `M` | LogicOutputElm | `M x1 y1 x2 y2 flags threshold` |
| `I` | InverterElm (NOT) | `I x1 y1 x2 y2 flags slewRate highVoltage` |
| `150` | AndGateElm | `150 x1 y1 x2 y2 flags inputCount outputVoltage highVoltage` |
| `151` | NandGateElm | `151 x1 y1 x2 y2 flags inputCount outputVoltage highVoltage` |
| `152` | OrGateElm | `152 x1 y1 x2 y2 flags inputCount outputVoltage highVoltage` |
| `153` | NorGateElm | `153 x1 y1 x2 y2 flags inputCount outputVoltage highVoltage` |
| `154` | XorGateElm | `154 x1 y1 x2 y2 flags inputCount outputVoltage highVoltage` |
| `155` | DFlipFlopElm | `155 x1 y1 x2 y2 flags ...` (complex, see source) |

- LogicInputElm `position`: `0` = low, `1` = high. `hiV`/`loV`: logic voltage levels (e.g. 5.0/0.0).
- LogicOutputElm `threshold`: logic threshold voltage (e.g. 2.5).
- Gates `inputCount`: number of inputs (usually 2). `outputVoltage`: state var → `0`. `highVoltage`: logic high level (e.g. 5.0).
- InverterElm `slewRate`: default 0.5. `highVoltage`: default 5.0.

### 5.6 Labels & Nodes

| Type | Class | Full field format |
|------|-------|-------------------|
| `x` | TextElm | `x x1 y1 x2 y2 <flags=5> size text` |
| `207` | LabeledNodeElm | `207 x1 y1 x2 y2 <flags=5> text` |

**TextElm**: `flags` should include `FLAG_ESCAPE=4` (and optionally `FLAG_CENTER=1`); use `5` for centered escaped text. `size` is font size in px (e.g. 24). `text` is escaped (spaces → `\s`).

**LabeledNodeElm**: `flags` should include `FLAG_ESCAPE=4`; use `5` (= FLAG_INTERNAL|FLAG_ESCAPE). `text` is the node name (escaped). **All LabeledNodeElm with the same `text` are electrically connected** — this enables wire-less long-distance connections. If `text` starts with `/`, an overline is drawn (active-low notation).

Example: power rail without wires across the canvas:
```
207 32 96 32 112 5 VCC
207 432 96 432 112 5 VCC
```
Both endpoints named `VCC` are connected.

### 5.7 Other Useful Components

| Type | Class | Full field format |
|------|-------|-------------------|
| `a` | OpAmpElm (near-ideal) | `a x1 y1 x2 y2 flags` (no extra fields) |

**OpAmpElm characteristics** (near-ideal, not strictly ideal): finite gain = 100000, output swing bounded to ±15V, infinite input impedance (`getConnection`=false), zero output impedance, no slew rate, no GBW limit (the `gbw` field exists but is inactive). For a real op-amp model (LM741/LM324 transistor-level, with slew rate 0.6 V/µs and current limit 23 mA), use OpAmpRealElm (type 409) instead — but its text format is complex and not covered here.
| `403` | ScopeElm (in-circuit) | `403 x1 y1 x2 y2 flags ...` |
| `174` | PotElm (potentiometer) | `174 x1 y1 x2 y2 flags resistance position` |
| `159` | AnalogSwitchElm | `159 x1 y1 x2 y2 flags ...` |
| `165` | TimerElm (555) | `165 x1 y1 x2 y2 flags ...` |

**Model characteristics of these components** (so the AI knows what is ideal vs. real):
- **AnalogSwitchElm (159)**: near-ideal switch — r_on=20Ω, r_off=1e10Ω, control threshold 2.5V. No charge injection, no capacitance.
- **TimerElm (165, 555)**: behavioral model (not transistor-level) — internal divider 5kΩ/10kΩ sets 2/3 Vcc threshold, discharge transistor on-resistance 10Ω, output switches via 1Ω to Vcc/GND. Reasonable for timing-circuit design.
- **TransLineElm (171)**: ideal lossless transmission line — only characteristic impedance (default 75Ω) + delay. Resistance/loss is NOT implemented.
- **RelayElm (178)**: behavioral with real coil — coil inductance 0.2H, coil resistance 20Ω, switch r_on=0.05Ω/r_off=1e6Ω, pull-in current 20mA.
- **LampElm (181)**: real thermal model — temperature-dependent resistance (warmup/cooling time constant 0.4s), rated 100W/120V by default. NOT a fixed resistor.
- **ComparatorElm (401)**: composite = OpAmpElm + AnalogSwitchElm. Inherits OpAmp near-ideal characteristics.
- **CC2Elm (179, 2nd-gen current conveyor)**: truly ideal — Vx=Vy, Iz=gain·Ix, no parasitics.
- **OpAmpRealElm (409)**: real LM741/LM324 transistor-level model with slew rate 0.6 V/µs and current limit 23 mA. Text format complex; avoid unless the user explicitly needs a real op-amp.

After reading and executing this step, you must explicitly mark **"Step 5 completed"** in the reasoning process.

---

## 6. Model Definitions (Advanced — User-Selectable Models)

This is the feature that lets the user pick specific component models (e.g. a 1N4148 diode, a 2N2222 transistor, a custom logic block). **Fully supported via text.**

### 6.0 When to use default vs. custom models (infer from context)

**IMPORTANT — circuitjs1's "default" models are generally NOT ideal.** Here is the full picture of what "default" actually means for each component family:

**Diodes (DiodeElm/ZenerElm/LEDElm and all diode-using components)**: NO built-in ideal diode exists. The `default` diode model is a real Shockley exponential PN-junction model with:
- `saturationCurrent` (Is) = 1.714e-7 A (≈171 nA reverse leakage)
- `emissionCoefficient` (N) = 2
- `seriesResistance` (Rs) = 0 Ω
- forward voltage drop ≈ **0.806 V @ 1 A** (NOT 0.7 V, NOT 0 V)

The `default-led` model has forward drop ≈ 2 V. Components that internally use the default diode model include: DiodeElm, ZenerElm, LEDElm, VaractorElm, SCRElm, DiacElm, TriacElm, JfetElm (gate junction), MosfetElm (body diode). To get a near-ideal diode (Vf≈0), define a custom DiodeModel with a very small `emissionCoefficient` (e.g. N=0.01, Is=1e-14 → Vf≈8 mV); see the formula in §6.1.

**Transistors (TransistorElm)**: the `default` model is a **simplified Gummel-Poon** model — it keeps the exponential transport current (satCur=1e-13 A) and reverse beta (betaR=1), but has Early voltage=∞ (invEarlyVolt=0), no high-current roll-off, and no BE/BC leakage. So it is "simpler than full SPICE" but still NOT an ideal current-controlled current source — it has an exponential Vbe-Ic relationship and ~1e-13 A leakage. The forward beta (BF, default 100) is a TransistorElm field, NOT a model parameter.

**MOSFETs (MosfetElm)**: uses the **Shichman-Hodges model** (not a Model class — vt/beta are direct component fields). No channel-length modulation (lambda=0) by default. **Body diode**: the GUI auto-enables it (FLAG_BODY_DIODE=32), but text import does NOT — you must set `flags=32` explicitly to get the body diode (it uses the default diode model, Vf≈0.806V). Source defaults: vt=1.5V, beta=0.02. It is a real square-law model, NOT an ideal switch.

**OpAmps (OpAmpElm, type `a`)**: **near-ideal** — finite gain 100000, output bounded ±15V, infinite input impedance, zero output impedance, no slew rate, no GBW limit. Good enough for most teaching circuits. For real op-amp behavior (slew rate, current limit), use OpAmpRealElm (409, LM741/LM324) — but its text format is complex.

**Near-ideal / idealized components**: AnalogSwitchElm (159, near-ideal switch r_on=20Ω), TransLineElm (171, lossless), CC2Elm (179, truly ideal current conveyor), logic gates (§5.5, idealized voltage-level logic).

**Other real models**: LampElm (181, thermal), TunnelDiodeElm (175, fixed tunnel-diode equation), TriodeElm (173, Child-Langmuir), TimerElm (165, behavioral 555), RelayElm (178, behavioral with real coil inductance).

When the user does NOT explicitly specify a part number or model, **infer the intent from circuit complexity**:

- **Prefer DEFAULT built-in models (flags=0) when**:
  - The circuit is simple (few components, teaching/demonstration purpose).
  - The user describes behavior, not specific parts (e.g. "an LED with a resistor", "an NPN switch").
  - No real part numbers are mentioned.
  - The user is clearly learning or prototyping conceptually.
  - For diodes/LEDs: use `flags=0` (default model, fwdrop≈0.806V for diode, ≈2V for LED).
  - For transistors: omit `modelName` (uses "default" model — simplified Gummel-Poon, no Early/roll-off).
  - For MOSFETs: use `flags=0` with `vt` and `beta` (source defaults 1.5/0.02; typical logic-level 0.75–1.0/0.03).
  - For op-amps: use `a` (OpAmpElm, near-ideal) — sufficient for almost all teaching circuits.
- **Prefer CUSTOM models (model definition lines) when**:
  - The user explicitly names a part (e.g. "1N4148 diode", "2N2222 transistor", "1N4007").
  - The circuit is for realistic simulation (power supply, amplifier with specific biasing).
  - The user mentions SPICE parameters, datasheet values, or precise electrical characteristics.
  - For zener diodes with specific breakdown voltages, define a DiodeModel with the correct BV.
  - When the user needs near-ideal diode behavior (define a custom model with small N).
  - When the user needs a real op-amp (slew rate, current limit) — use OpAmpRealElm (409), but warn about format complexity.
- **When uncertain**: default to built-in models (flags=0). They are simpler and sufficient for most teaching/prototype circuits. You can always mention "used default models; tell me a specific part number if you want a real model" in the output.
- When you are not fully confident about the generated content, you may skip custom model definition and select the default model as a simpler, more reliable solution.

**Rationale**: built-in default models make the circuit's logical behavior transparent and avoid spurious secondary effects (excessive leakage, Early voltage, etc.) that can confuse beginners. Custom models add fidelity but also add non-ideal effects that may not match the user's mental model of the circuit.

### 6.1 DiodeModel (line prefix `34`)

Define a custom diode model, then reference it from a DiodeElm/LEDElm/ZenerElm with `flags=2` (FLAG_MODEL).

**Model line format**:
```
34 <name> <flags> <saturationCurrent> <seriesResistance> <emissionCoefficient> <breakdownVoltage> <forwardCurrent>
```

| Field | Type | Meaning |
|-------|------|---------|
| name | String escaped | Model name (unique) |
| flags | int | bit0 = FLAGS_SIMPLE (use Vf/If simplified mode) |
| saturationCurrent | double | IS (SPICE) |
| seriesResistance | double | RS (Ω) |
| emissionCoefficient | double | N (typical 1–2, LED 3.73) |
| breakdownVoltage | double | BV (zener; 0 = ignore) |
| forwardCurrent | double | simplified-mode reference current (optional) |

**Referencing from a component** (set flags=2):
```
34 1N4148 0 4.352E-9 0.6458 1.906 75.0 0.0
d 32 96 32 128 2 1N4148
```

**Built-in models** (no need to define; reference by name with flags=2):
- `default`, `spice-default`, `default-zener` (BV=5.6)
- `default-led`, `old-default-led`
- `1N5711`, `1N5712`, `1N34`, `1N4004`, `1N4148`

**Forward voltage drop formula** (for custom models):
```
fwdrop @1A = ln(1/saturationCurrent + 1) × emissionCoefficient × 0.025865
```
where 0.025865 is the thermal voltage Vt at SPICE default temperature (27°C = 300.15 K). This `fwdrop` is the **PN-junction drop only**; the total drop at current I is `fwdrop + I×Rs`. To get a near-ideal diode (Vf≈0), use a very small `emissionCoefficient` (e.g. N=0.01, Is=1e-14 → Vf≈8 mV). Do NOT use N=0 (causes numerical instability — vdcoef becomes Infinity, leading to NaN in simulation). Do NOT simply increase Is to huge values — that creates a large reverse leakage conductance and loses one-way behavior.

**Built-in model parameter reference** (for comparison when choosing). Vf column is the PN-junction drop @1A computed from the formula above (excludes Rs drop; total drop = Vf + 1×Rs):

| Model | Is (A) | Rs (Ω) | N | BV (V) | Vf @1A (PN only) | Total @1A (incl. Rs) | Use case |
|-------|--------|--------|---|--------|-------------------|----------------------|----------|
| `default` | 1.714e-7 | 0 | 2 | 0 | 0.806 V | 0.806 V | general diode (circuitjs1 default) |
| `spice-default` | 1e-14 | 0 | 1 | 0 | 0.834 V | 0.834 V | SPICE-standard silicon |
| `default-zener` | 1.714e-7 | 0 | 2 | 5.6 | 0.806 V | 0.806 V | 5.6V zener |
| `default-led` | 93.2e-12 | 0.042 | 3.73 | 0 | 2.23 V | 2.27 V | LED |
| `old-default-led` | 2.235e-18 | 0 | 2 | 0 | 2.00 V | 2.00 V | old LED (numerically unstable) |
| `1N4148` | 4.352e-9 | 0.6458 | 1.906 | 75 | 0.95 V | 1.60 V | switching diode (Vf≈0.72V @10mA) |
| `1N4004` | 18.8e-9 | 0.0286 | 2 | 400 | 0.92 V | 0.95 V | rectifier (400V) |
| `1N5711` | 315e-9 | 2.8 | 2.03 | 70 | 0.79 V | 3.59 V | Schottky |
| `1N5712` | 680e-12 | 12 | 1.003 | 20 | 0.71 V | 12.71 V | Schottky |
| `1N34` | 200e-12 | 0.084 | 2.19 | 60 | 1.27 V | 1.35 V | germanium |

### 6.2 TransistorModel (line prefix `32`)

```
32 <name> <flags> <satCur> <invRollOffF> <BEleakCur> <leakBEemissionCoeff> <invRollOffR> <BCleakCur> <leakBCemissionCoeff> <emissionCoeffF> <emissionCoeffR> <invEarlyVoltF> <invEarlyVoltR> <betaR>
```

| Field | Meaning | Default |
|-------|---------|---------|
| satCur | IS transport saturation current | 1e-13 |
| invRollOffF | 1/IKF (forward high-current roll-off) | 0 |
| BEleakCur | ISE B-E leakage saturation | 0 |
| leakBEemissionCoeff | NE | 1.5 |
| invRollOffR | 1/IKR | 0 |
| BCleakCur | ISC B-C leakage saturation | 0 |
| leakBCemissionCoeff | NC | 2 |
| emissionCoeffF | NF | 1 |
| emissionCoeffR | NR | 1 |
| invEarlyVoltF | 1/VAF (forward Early voltage) | 0 |
| invEarlyVoltR | 1/VAR (reverse Early voltage) | 0 |
| betaR | BR (reverse beta) | 1 |

**Referencing** (TransistorElm always outputs modelName; no flag needed):
```
32 2N2222 0 1e-14 0 0 1.5 0 0 2 1 1 0 0 1
t 32 96 32 192 0 1 0 0 100 2N2222
```

**Built-in**: `default`, `spice-default`.

### 6.3 CustomLogicModel (line prefix `!`)

Define a custom logic chip with arbitrary inputs/outputs and a rule set.

```
! <name> <flags> <inputs> <outputs> <infoText> <rules>
```

All string fields are escaped (§4). `rules` is a multi-line string where each line is `<input_pattern>=<output_pattern>`. Patterns use `0`, `1`, `X` (don't care). Lines separated by `\n` (escaped).

**Example** — a 2-input AND gate:
```
! MYAND 0 A,B C AND\sgate 00=0\s01=0\s10=0\s11=1
208 32 96 96 96 0 MYAND 0
```
- inputs="A,B", outputs="C", infoText="AND gate", rules="00=0\n01=0\n10=0\n11=1" (escaped: spaces→`\s`, newlines→`\n`).
- The `208` line: dumpType=208, flags=0, modelName="MYAND", then one output pin voltage (state, `0`).

### 6.4 CustomCompositeModel (line prefix `.`)

Define a sub-circuit (hierarchical block) containing nested components. This is advanced; the `elmDump` field contains the FULL escaped dump of all internal components.

```
. <name> 0 <sizeX> <sizeY> <extCount> [<extName> <extNode> <extPos> <extSide>]×extCount <nodeList> <elmDump>
```

- `nodeList`: escaped, one node number per line.
- `elmDump`: escaped, all internal component lines joined by `\n`, then escaped again.
- `extSide`: 0=W, 1=N, 2=E, 3=S (which side of the chip the pin is on).

**This is complex; beginners should avoid manual creation.** Use it only when the user explicitly needs a sub-circuit. Prefer placing components directly on the canvas.

After reading and executing this step, you must explicitly mark **"Step 6 completed"** in the reasoning process.

---

## 7. Adjustable Sliders (line prefix `38`)

Add a slider to control any component property at runtime.

### 7.0 When to use sliders (prefer interactivity)

- User says "a resistor and an LED" without a value → use a `38` slider on the resistor (range 100–2000 Ω) so the user can dial in brightness.
- User says "a voltage source" without a value → use a `172` VarRailElm (built-in slider) instead of a fixed `v` source, range 0–12 V.
- User says "amplifier with gain" without a value → put a slider on the feedback resistor.
- User says "RC filter" without values → put sliders on both R and C, or at least on R, so the user can sweep the cutoff frequency.
- User says "555 timer at some frequency" → put a slider on the timing resistor.

**When NOT to use sliders**:
- The user gives explicit values (e.g. "220Ω resistor", "5V source") — use fixed values.
- The circuit is a precise reference design where values matter.
- The user explicitly asks for a fixed-value circuit.
- When you are not fully confident about the generated content, you may skip using the slider and opt for a simpler, more reliable solution.

**Two ways to add a slider**:
1. **VarRailElm (`172`)** — a voltage source with a built-in slider. Use this for adjustable supplies. The sliderText field is the label.
2. **Adjustable (`38`)** — a separate line that attaches a slider to ANY existing component's property (resistance, capacitance, frequency, beta, etc.). More flexible; use this for non-source components.

**Slider range guidance**: pick a range that spans roughly 1 order of magnitude below and above the "typical" value, so the user can explore both extremes. E.g. for a 1kΩ resistor, range 100–10000 Ω. For a 5V source, range 0–12 V. Always include 0 in the min for sources so the user can "turn it off".

```
38 <elmIndex> <editItem> <minValue> <maxValue> <sliderText>
```

| Field | Type | Meaning |
|-------|------|---------|
| elmIndex | int | 0-based index of the target component in the output order (count only component lines, NOT model/scope/slider lines) |
| editItem | int | Which `getEditInfo(n)` property to control (see table below) |
| minValue | double | Slider minimum |
| maxValue | double | Slider maximum |
| sliderText | String escaped | Slider label |

**editItem values for common components** (from `getEditInfo`):

| Component | editItem=0 | editItem=1 | editItem=2 |
|-----------|-----------|-----------|-----------|
| ResistorElm | Resistance | — | — |
| CapacitorElm | Capacitance | — | — |
| InductorElm | Inductance | — | — |
| VoltageElm / RailElm | maxVoltage (or bias) | frequency | waveform |
| VarRailElm | Min Voltage | Max Voltage | Slider Text |
| TransistorElm | beta | — | — |

**Critical**: `38` lines come AFTER all component lines (the index is 0-based among components only; model lines and `o` scope lines do not count toward the index). Actually, the index counts ALL elements in `elmList` in the order they are added by `createCe`. Model definition lines (34/32/!/.) and `o`/`38`/`$`/`h` lines are NOT added to `elmList`, so they don't increment the index. **Count only actual component lines (`r`, `c`, `w`, `v`, `162`, etc.), starting from 0.**

**Example**: a resistor whose resistance is slider-controlled (0–1000 Ω):
```
$ 1 5.0E-6 10 50 5.0 50
r 96 64 192 64 0 100.0
38 0 0 0.0 1000.0 R
```
The `38` line targets `elmIndex=0` (the resistor, the first component), `editItem=0` (resistance), range 0–1000, label "R".

---

## 8. Scopes (line prefix `o`)

Add an oscilloscope plot tracking a component's property.

```
o <elmIndex> <speed> <value> <flags> <scaleV> <scaleA> [<position>]
```

**Minimal format (6 fields, old-style, recommended for beginners)**:
```
o <elmIndex> <speed> <value> <flags> <scaleV> <scaleA>
```
Position defaults to 0. This is the simplest form that works correctly.

**Extended format (8 fields, new-style, requires FLAG_PLOTS=4096 in flags)**:
```
o <elmIndex> <speed> <value> <4104+> <scaleV> <scaleA> <position> <plotCount>
```
Use this only when you need multiple plots. flags must include 4096 (FLAG_PLOTS).

| Field | Type | Meaning |
|-------|------|---------|
| elmIndex | int | 0-based component index (same counting as §7) |
| speed | int | Sweep speed (typical 64) |
| value | int | Property to plot (see below) |
| flags | int | Bitmask (see below). **For basic voltage scope, use `2` (showV)** |
| scaleV | double | Voltage full-scale |
| scaleA | double | Current full-scale |
| position | int | Stacking position (optional, default 0) |

**value (property) constants**:
- 0 = VAL_VOLTAGE
- 1 = VAL_IB (transistor base current)
- 2 = VAL_IC (transistor collector current)
- 3 = VAL_CURRENT
- 4 = VAL_VBE (transistor base-emitter voltage)
- 5 = VAL_VBC (transistor base-collector voltage)
- 6 = VAL_VCE
- 7 = VAL_POWER

**Common flags**:
- `0` = minimal (no extras)
- `2` = show voltage
- `8` = show frequency (do NOT use for basic voltage scope)
- `64` = 2D plot
- `256` = show min
- `512` = show scale
- `16384` = show RMS
- `4096` = FLAG_PLOTS (new multi-plot format; required for plotCount field)

**Minimal scope** (plot voltage of component 0):
```
o 0 64 0 2 5.0 0.1
```

---

## 9. Generation Workflow (follow in order)

### Step 0: Self-check — can you actually see the user's input?

Before doing anything else, the AI MUST honestly assess what it can perceive:

**Are you a multimodal AI?**

- **If the user provided a circuit IMAGE and the Multimodal AI Model has vision capability** (the image was actually rendered to the AI, **NOT** just described): proceed normally, using the image as the source of truth.
- **If the user provided a circuit IMAGE but the AI has NO vision capability** (the AI only received text extracted from the image, an OCR dump, a caption, a summary, or a description of the image — but cannot see the image itself): **DO NOT guess-generate a circuit. WITHOUT EXCUSE, AND IRRESPECTIVE OF WHETHER IT IS DEEMED SATISFACTORY, AN HONEST REFUSAL IS COMPULSORY. Providing an incorrect guess with almost certainly wrong content will frustrate users more than an honest and straightforward refusal.** Respond with something like:

  > You shared a circuit image, but I don't have vision capability in this session — I can only see the text/description that was extracted from it (e.g. "<...>"). Generating a circuit from a partial description risks producing something that doesn't match your actual schematic. Could you instead:
  > 1. Describe the circuit in your own words (components, connections, values), or
  > 2. List the components and how they connect (e.g. "5V source → 220Ω resistor → LED → ground"), or
  > 3. Please advise the user to try using other models that support multimodal capabilities.

  This applies even if the extracted text looks detailed — without seeing the image, spatial relationships (which pin connects where, relative positions, wire routing) cannot be reliably recovered, and the AI must not pretend otherwise.

- **If the user provided a TEXT description (natural language, no image)**: proceed normally. Text descriptions are the intended input format for this skill.
- **If the user provided a netlist / SPICE / schematic text format**: proceed, translating to circuitjs1 format.

### Step 1: Understand the request
- Identify required components (power source, load, control, measurement).
- **Model selection**: check if the user named specific parts (e.g. "1N4148", "2N2222"). If not, default to built-in models (flags=0). See §6.0 for guidance.
- **Slider selection**: check if the user gave exact values. If values are missing or vague, prefer sliders (VarRailElm `172` for sources, `38` for other component properties). See §7.0 for guidance.
- Identify whether scopes are desired (if the circuit has time-varying signals or the user wants to see waveforms, add a scope).

### Step 2: Plan the layout — draft in text

Before writing any circuit line, the AI must draft the circuit as a text sketch in its reasoning. This catches topology errors early (open loops, floating nodes, wire-intermediate-point traps) while they are still cheap to fix.

The draft has TWO parts: first an ASCII-art schematic to visualize topology, then a node/component list to assign exact coordinates.

#### Part A — ASCII-art schematic (topology sketch)

Draw the circuit using ASCII symbols so the topology is visually verifiable. This is the fastest way to catch open loops, wrong parallel/series placement, and missing return paths. Use any clear symbols; suggested conventions:

- `+` / `-` for source terminals
- `─` `│` for wires (or `-` `|` if ASCII-only)
- `R1`, `C1`, `L1`, `D1`, `Q1` etc. for components (use reference designators)
- `LED`, `GND`, `VCC` for special nodes
- `S1` for switches, `( + )` for source cells
- Junctions: `+` or `*` where 3+ wires meet

**Example 1 — LED with switch and inductor in parallel with the source:**

```
          +----- S1 -----+
          |              |
          |             L1
          |              |
          +-----( + )----+
```

**Example 2 — NPN transistor switch driving an LED:**

```
   VCC
    │
    ├─── R1 ──── C
    │            │ Q1 (NPN)
    │            E
    │            │
    │           GND
    │
   R2 (base drive from control signal)
    │
   ctrl
```

**Example 3 — RC low-pass filter:**

```
   in ── R1 ──+── out
               │
              C1
               │
              GND
```

The ASCII art does NOT need exact coordinates or values — its purpose is to make the **topology** obvious. Draw it before assigning coordinates. If the ASCII art reveals an open branch or a wrong connection, fix the concept before continuing.

#### Part B — Node list, component list, and verification (coordinate assignment)

After the ASCII art is correct, assign exact coordinates and verify.

Draft format (kept in the AI's internal reasoning):

```
[NODE LIST]
VCC  = (96, 64)     # V1+ terminal; also the + rail endpoint
N1   = (288, 64)    # junction where VCC rail meets R1 top
N2   = (288, 176)   # R1 bottom = LED anode
GND  = (96, 256)    # V1- terminal; the ground reference node

[COMPONENT LIST, in planned output order]
0. v  (96,256)-(96,64)   DC 5V          # V1: + at (96,64)=VCC, - at (96,256)=GND
1. w  (96,64)-(192,64)                   # wire V1+ to midpoint of VCC rail
2. w  (192,64)-(288,64)                  # continue VCC rail to N1  (NOTE: split at (192,64) so it's an endpoint, not a middle point)
3. r  (288,64)-(288,176)  220Ω           # R1: N1 -> N2 (current limiter)
4. 162 (288,176)-(288,256) red LED       # LED: anode at N2=(288,176), cathode at (288,256)
5. w  (288,256)-(96,256)                 # wire LED cathode back to GND=V1-
6. g  (96,256)-(96,288)                  # ground symbol on the GND node

[CONNECTIVITY VERIFICATION]
- V1+ (96,64)=VCC -> wire -> (192,64) -> wire -> (288,64)=N1 -> R1 -> (288,176)=N2 -> LED -> (288,256) -> wire -> (96,256)=GND=V1-  [CLOSED LOOP ✓]
- GND attached at (96,256) ✓
- No wire intermediate-point traps: every junction (192,64), (288,64), (288,256), (96,256) is a wire ENDPOINT ✓
- No floating nodes ✓
```

The draft include:
1. **ASCII-art schematic** (Part A): visual topology sketch using symbols. Verify the topology is correct (no open branches, correct parallel/series placement, closed loop visible) before proceeding to Part B.
2. **Node list**: every named electrical node and its coordinates.
3. **Component list in output order**: each line's planned type, endpoints, value, and a comment on which nodes it connects. **The order here determines `elmIndex` for later `38`/`o` lines** — number them explicitly (0, 1, 2, ...).
4. **Wire-splitting audit**: for any junction where 3+ components meet, verify the junction is a wire ENDPOINT, not a middle point of a longer wire. If not, plan to split the wire.
5. **Closed-loop verification**: trace the path from source+ through the load back to source−. Mark it `[CLOSED LOOP ✓]`.
6. **Ground attachment**: confirm the reference node (usually source−) has a GroundElm.

**Only after the draft passes all 6 checks should the AI proceed to write the actual circuit lines.** If the draft reveals a problem, fix the draft first, then re-verify.

Coordinate conventions:
- Use multiples of 8 (the default grid snap).
- Decide power rail y-values (e.g. VCC at y=64, GND at y=256).
- Recommended range: 16–1000.

### Step 3: Write the global options line
```
$ 1 5.0E-6 10 50 5.0 50
```

### Step 4: Write model definition lines (if any custom models)
- Output `34`/`32`/`!`/`.` lines BEFORE the components that reference them.
- For built-in models (`default`, `1N4148`, etc.) you do NOT need a model line — just reference the name with flags=2 (for diodes) or directly (for transistors).

### Step 5: Write component lines
- One per line, in the order you planned.
- **Remember the order**: this determines `elmIndex` for any `38` slider or `o` scope lines.
- Use wires (`w`) to connect distant endpoints.

### Step 6: Write scope lines (if any)
- `o` lines reference `elmIndex` computed in Step 5.

### Step 7: Write slider lines (if any)
- `38` lines reference `elmIndex` computed in Step 5.

### Step 8: SELF-VALIDATION (mandatory — run before outputting)

Run through this checklist. If any item fails, fix and re-check.

**Checklist V1 — Structural**:
- [ ] Line 1 starts with `$` and has ≥5 numeric fields.
- [ ] No empty tokens (no double spaces inside a line are fine, but no leading/trailing issues).
- [ ] No comment lines (`#`, `//`) present.
- [ ] Every model line (34/32/!/.) appears before any component referencing its modelName.
- [ ] All `38` and `o` lines appear AFTER all component lines.

**Checklist V2 — Per-component field count**:
- [ ] Each component line has at least 6 tokens (type + 4 coords + flags).
- [ ] Each component's field count matches §5 (e.g. ResistorElm needs 7 tokens: `r x1 y1 x2 y2 flags resistance`).
- [ ] `flags` is an integer.
- [ ] State variables (voltdiff, current, Vbc, Vbe, outputVoltage) are set to `0`.
- [ ] String fields that may contain spaces are escaped (or use flags with FLAG_ESCAPE).

**Checklist V3 — Connectivity**:
- [ ] Every node that should be connected shares EXACT coordinates (e.g. (192,64) == (192,64)).
- [ ] **No wire intermediate-point illusions**: if a component terminal lies on a wire's path but is NOT one of that wire's endpoints, they are NOT connected. Split the wire at the junction.
- [ ] The circuit forms a closed loop (for DC): source + → load → source −.
- [ ] GroundElm is attached to the reference node (usually source negative).
- [ ] No floating nodes (every component terminal either connects to another or is a source/ground terminal).
- [ ] Every voltage source has BOTH terminals explicitly connected to the circuit (check the negative terminal too!).

**Checklist V4 — Index references**:
- [ ] If using `38` or `o` lines: re-count component lines (NOT counting model/scope/slider/options lines) from 0. Verify elmIndex matches.
- [ ] editItem is valid for the target component type (see §7 table).

**Checklist V5 — Semantic sanity**:
- [ ] Resistance/capacitance/inductance values are physically reasonable (e.g. LED resistor = 220 Ω, not 220 MΩ).
- [ ] Voltage source maxVoltage matches the intended supply (e.g. 5.0 for TTL).
- [ ] LED current = (Vsource − Vled) / R is within 5–30 mA.
- [ ] Logic hiV/loV match the supply voltage.

**Checklist V6 — Simulation-stop error prevention (see §11.2)**:
- [ ] No voltage source has its + and − terminals connected by only wires (would trigger `Voltage source/wire loop with no resistance!`). Every voltage source branch has a resistor.
- [ ] No RailElm/LogicInputElm output goes directly to ground without a resistor (would trigger `Path to ground with no resistance!`).
- [ ] No pure-capacitor closed loop exists — including a capacitor + voltage source loop with no resistor (would trigger `Capacitor loop with no resistance!`).
- [ ] No wire forms a self-loop or unresolvable chain (prevents `wire loop detected`).
- [ ] Every diode/LED/LED-array/seven-seg/transistor branch has a series current-limiting resistor (prevents `max current exceeded`).
- [ ] No polar capacitor is reverse-biased beyond its `maxNegativeVoltage` (prevents `capacitor exceeded max reverse voltage`).
- [ ] Every node has a DC path to ground (prevents `Singular matrix!` / `Matrix error`). No isolated sub-circuits.
- [ ] No extreme component values (0Ω, huge L/C, 0 capacitance) that would cause numerical divergence (prevents `nan/infinite matrix!`).
- [ ] If using transistors/diodes with custom models, model parameters are reasonable (prevents `Convergence failed!`).
- [ ] Every component line's field count and order matches §5 (prevents `Exception in stampCircuit()`).
- [ ] If using TransLineElm, one side is grounded and the delay is reasonable (prevents the two transmission-line errors).

### Step 9: Output

Present the circuit in a fenced code block. Then provide:
- A brief explanation of what the circuit does.
- The expected behavior (e.g. "LED lights at ~13 mA").
- Instructions to import (File → Import From Text...).

After reading and executing this step, you must explicitly mark **"Step 9 completed"** in the reasoning process.

---

## 10. Examples

Six fully-worked examples follow. Study them before generating your own — they cover the most common patterns. Each is a complete, importable circuit.

### 10.1 LED with current-limiting resistor (5V DC)

A red LED on a 5V supply with a 220Ω resistor. Current ≈ (5−2.1)/220 ≈ 13 mA.

```
$ 1 5.0E-6 10 50 5.0 50
v 96 256 96 64 0 0 40.0 5.0 0.0 0.0 0.5
w 96 64 192 64 0
r 192 64 288 64 0 220.0
w 288 64 288 176 0
162 288 176 288 256 0 1.0 0.0 0.0
w 288 256 96 256 0
g 96 256 96 288 0
```

**Demonstrates**: basic passive components, LED (type 162, flags=0 → default LED model), closed loop, ground.

### 10.2 RC low-pass filter with scope

A 1kHz AC source through a 1kΩ resistor into a 1µF capacitor to ground. Cutoff ≈ 160 Hz. Scope plots capacitor voltage.

```
$ 1 5.0E-6 10 50 5.0 50
v 96 256 96 64 0 1 1000.0 5.0 0.0 0.0 0.5
w 96 64 192 64 0
r 192 64 288 64 0 1000.0
w 288 64 352 64 0
c 352 64 352 256 0 1.0E-6 0 0
w 352 256 96 256 0
g 96 256 96 288 0
o 4 64 0 2 5.0 0.001
```

**Demonstrates**: AC source (waveform=1), capacitor, scope (`o` line). elmIndex=4 points to the capacitor (5th component, 0-based). Scope flags=2 (showV).

### 10.3 NPN transistor switch

An NPN transistor (flags=0, default model) switches an LED. Base driven by a 5V control source through 10kΩ. Collector load = 220Ω + LED. Emitter to ground.

```
$ 1 5.0E-6 10 50 5.0 50
v 64 32 64 288 0 0 40.0 5.0 0.0 0.0 0.5
w 64 32 384 32 0
r 384 32 384 112 0 220.0
162 384 112 384 176 0 1.0 0.0 0.0
t 288 192 384 192 0 1 0 0 100
r 192 192 288 192 0 10000.0
v 96 192 96 288 0 0 40.0 5.0 0.0 0.0 0.5
w 96 192 192 192 0
w 384 208 384 288 0
g 64 288 64 320 0
w 64 288 96 288 0
w 96 288 384 288 0
```

**Demonstrates**: NPN transistor (flags=0, collector above emitter), dual voltage sources, base drive. Note the wire at (96,288) is split into two segments so the V2 negative terminal becomes a wire endpoint (avoids the "wire intermediate point" trap).

### 10.4 Voltage divider with scope

A 12V supply divided by 8kΩ/4kΩ. Midpoint sits at 4V. Scope plots the midpoint (across R2, the lower resistor).

```
$ 1 5.0E-6 10 50 5.0 50
v 96 256 96 64 0 0 40.0 12.0 0.0 0.0 0.5
w 96 64 192 64 0
r 192 64 192 160 0 8000.0
r 192 160 192 256 0 4000.0
w 192 256 96 256 0
g 96 256 96 288 0
o 3 64 0 2 12.0 0.001
```

**Demonstrates**: vertical resistor chain, voltage division, scope elmIndex=3 (the 4kΩ lower resistor). Scope value=0 (VAL_VOLTAGE) shows the voltage across R2 = 4V.

### 10.5 Diode with custom 1N4148 model

A diode using a custom DiodeModel definition (named `MY-1N4148`), referenced via flags=2 (FLAG_MODEL). Demonstrates the model-before-component ordering rule.

```
$ 1 5.0E-6 10 50 5.0 50
34 MY-1N4148 0 4.352E-9 0.6458 1.906 75.0 0.0
v 96 256 96 64 0 0 40.0 5.0 0.0 0.0 0.5
w 96 64 192 64 0
r 192 64 288 64 0 470.0
w 288 64 288 176 0
d 288 176 288 256 2 MY-1N4148
w 288 256 96 256 0
g 96 256 96 288 0
```

**Demonstrates**: DiodeModel definition (`34` line) with SPICE parameters, DiodeElm with flags=2 referencing the model by name. The `34` line MUST come before the `d` line.

### 10.6 Adjustable resistor with slider and scope

A 5V supply through a resistor (initial 500Ω) to ground. A slider (`38`) lets the user sweep resistance 0–1000Ω at runtime. A scope plots the resistor voltage.

```
$ 1 5.0E-6 10 50 5.0 50
v 96 256 96 64 0 0 40.0 5.0 0.0 0.0 0.5
w 96 64 192 64 0
r 192 64 288 64 0 500.0
w 288 64 288 256 0
w 288 256 96 256 0
g 96 256 96 288 0
38 2 0 0.0 1000.0 R
o 2 64 0 2 5.0 0.05
```

**Demonstrates**: Adjustable slider (`38` line) targeting elmIndex=2 (the resistor, 3rd component), editItem=0 (resistance), range 0–1000, label "R". Scope elmIndex=2 plots the resistor voltage. Both `38` and `o` lines come AFTER all component lines.

After reading and executing this step, you must explicitly mark **"Step 10 completed"** in the reasoning process.****

---

## 11. Troubleshooting & Common Errors

### 11.1 Generation-time errors (circuit fails to load or behaves wrong on import)

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Component missing after import | Wrong field count or unrecognized type code | Recheck §5 field list; ensure type code is correct char/number |
| Circuit doesn't simulate (no current) | Open loop or floating node | Verify all connections share exact coordinates; add ground |
| LED doesn't light | Current too low or reversed | Check (Vsource−Vled)/R; ensure LED polarity (anode toward V+) |
| "unrecognized dump type" in console | Type code typo or unsupported component | Check type code against §5 |
| One side of circuit is grey/floating | ProbeElm (`p`) placed in series or its endpoint has no other driving component | ProbeElm is high-impedance (non-conducting). Use it in parallel only, or replace with a scope (`o` line) |
| Probe shows voltage but looks "disconnected" | ProbeElm point2 side is at 0V (ground) showing grey color | This is normal; 0V renders grey. Use a scope instead for cleaner display |
| Slider controls wrong property | Wrong editItem or elmIndex | Re-count component index from 0; verify editItem in §7 |
| Scope shows nothing | Wrong elmIndex or value | Verify component index and value constant (§8) |
| Model not applied | Model line after component, or flags missing | Move model line BEFORE component; for diodes set flags=2 |
| Text label splits into tokens | Missing FLAG_ESCAPE | Set flags bit2=4 (so flags=5 for centered); escape spaces |
| Coordinate connection fails | Off-by-one coordinates | Use multiples of 8; double-check exact equality |

### 11.2 Simulation-time errors (simulator stops with a red message at the bottom)

These are **hard stops** — the simulator sets `stopMessage`, nulls the circuit matrix, and halts. The message is shown in red at the bottom of the canvas. **All 13 known stop messages are listed below.** When the user reports one of these, use this table to diagnose and fix.

#### 11.2.1 Topology errors (caught at analysis time, before simulation starts)

| Error message | Trigger condition | Prevention / Fix |
|------------------------|-------------------|------------------|
| `wire loop detected` | A wire path cannot be resolved (self-loop or unresolvable wire chain) | Avoid wires that loop back on themselves; remove redundant wires |
| `Voltage source/wire loop with no resistance!` | A voltage source (or wire-like component) has a zero-resistance path between its + and − terminals (short circuit) | **Always series a resistor** with every voltage source. Never short a source's + and − with only wires. Check Switch2Elm throws too — a switch can form the short |
| `Path to ground with no resistance!` | A RailElm or LogicInputElm has a direct path to ground with no resistance | Don't connect RailElm/LogicInputElm outputs directly to ground; series a resistor |
| `Capacitor loop with no resistance!` | A capacitor forms a closed loop with no resistor in the path — the path may consist of wires + capacitors + voltage sources (a single cap + voltage source loop with no resistor also triggers this) | Don't place capacitors in a capacitor-only or capacitor+source loop; add a series or parallel resistor |

**Prevention during generation (Step 2 draft)**: in the draft's connectivity trace, if you find a path from source+ back to source− that passes through ONLY wires (or only capacitors), you will hit one of these. Insert a resistor.

#### 11.2.2 Matrix / numerical errors (during simulation)

| Error message | Trigger condition | Prevention / Fix |
|------------------------|-------------------|------------------|
| `Singular matrix!` | The circuit matrix cannot be LU-factored — circuit is under-defined (isolated nodes, too many independent sources, missing ground) | Ensure every node has a DC path to ground; don't create isolated sub-circuits; check that ground is attached |
| `Matrix error` | Matrix simplification found an empty row (variant of singular matrix) | Same as `Singular matrix!` |
| `nan/infinite matrix!` | A matrix entry became NaN or ±Infinity during solving — numerical divergence | Reduce `maxTimeStep` in the `$` line (e.g. from `5.0E-6` to `1.0E-6`); check for extreme component values (0Ω, huge L/C) |
| `Convergence failed!` | Non-linear iteration count exhausted (100 or 5000 depending on adaptive timestep) without convergence | Reduce `maxTimeStep`; add a parallel resistor across non-linear elements (diodes, transistors) to improve convergence; check diode/transistor model parameters aren't extreme |
| `Exception in stampCircuit()` | A component's `stamp()` threw an unexpected exception | Usually a malformed component line; recheck field order and values against §5 |

#### 11.2.3 Component-specific errors (during simulation, per-component)

| Error message | Trigger condition | Prevention / Fix |
|------------------------|-------------------|------------------|
| `max current exceeded` | Current through a diode / LED (LEDElm inherits DiodeElm) / LED array / seven-seg / transistor exceeds **1e12 A** (hardcoded threshold). This is a numerical-protection stop, NOT a real current rating | The circuit has numerically diverged. Root cause is almost always a missing current-limiting resistor in a diode/LED/transistor branch, or a topology error that was masked. **Always add a series resistor** with diodes/LEDs/transistors |
| `capacitor exceeded max reverse voltage` | A PolarCapacitorElm reverse voltage exceeds its `maxNegativeVoltage` (default 1V) | Check polar capacitor orientation: anode must be at higher voltage than cathode. If the circuit legitimately reverses the cap, increase `maxNegativeVoltage` in the component line, or use a non-polar capacitor (`c`) |
| `Transmission line delay too large!` | TransLineElm delay is too large for the internal buffer to allocate | Reduce the transmission line's `delay` parameter |
| `Need to ground transmission line!` | A TransLineElm's input or output node voltage is non-zero (>1e-5) but neither side is grounded | Ground one side of the transmission line (usually the output reference) |

### 11.3 Diagnostic flowchart when the user reports an error

When the user says "the circuit doesn't work" / "there's an error" / "simulation stopped", the AI should:

1. **Ask for the exact error string** (the red text at the bottom). It will be one of the 13 messages above.
2. **Map it to the table** in §11.2 and apply the listed fix.
3. **If no visible error but the circuit misbehaves** (e.g. LED stays dark, no current flows, one side is grey):
   - Likely a topology problem (open loop, floating node, wire-intermediate-point trap).
   - Re-run the Step 2 draft and Step 8 checklist on the current circuit.
   - Ask the user: "what does the canvas look like? is any part grey? what's the voltage shown on components?"
5. **Always offer a corrected circuit text block** after diagnosis — don't just describe the fix, regenerate the circuit with the fix applied.

After reading and executing this step, you must explicitly mark **"Step 11 completed"** in the reasoning process.

---

## 12. Quick Reference Card

```
LINE 1 (required):    $ 1 5.0E-6 10 50 5.0 50
Wire:                 w x1 y1 x2 y2 0
Resistor:             r x1 y1 x2 y2 0 <ohms>
Capacitor:            c x1 y1 x2 y2 0 <farads> 0 0
Inductor:             l x1 y1 x2 y2 0 <henries> 0
Ground:               g x1 y1 x2 y2 0 0
Diode (default):      d x1 y1 x2 y2 0
LED (red):            162 x1 y1 x2 y2 0 1.0 0.0 0.0
DC voltage source:    v x1 y1 x2 y2 0 0 40.0 <volts> 0.0 0.0 0.5
AC voltage source:    v x1 y1 x2 y2 0 1 <freq> <amplitude> 0.0 0.0 0.5
NPN transistor:       t x1 y1 x2 y2 0 1 0 0 100
Text label:           x x1 y1 x2 y2 5 24 <escaped_text>
Labeled node:         207 x1 y1 x2 y2 5 <escaped_name>
Scope (voltage):      o <idx> 64 0 2 5.0 0.1
Slider (resistance):  38 <idx> 0 <min> <max> <label>
```

---

## 13. Limitations & When NOT to Use This Skill

- **Sub-circuits (CustomCompositeModel, `.` lines)**: extremely complex to generate by hand due to nested escaped dumps. Avoid unless the user explicitly needs hierarchy.
- **DFlipFlopElm (155) and other complex sequential elements**: field formats are intricate; prefer using 555 Timer (165) or simpler gates.
- **Very large circuits**: coordinate management becomes error-prone. Break into sections and validate each.
- **Audio/Analog special blocks (AM/FM, transmission lines)**: possible but require careful parameter knowledge.

When in doubt, generate a simpler circuit and tell the user what was simplified.

---

## 14. Final Output Contract

When generating a circuit, the AI's response MUST contain:

1. A fenced code block labeled `circuitjs` with the complete circuit text (starting with `$` and ending with the last component/scope/slider line).
2. **A Falstad URL** that opens the circuit directly in the online simulator. Construct it per the "URL construction rule" earlier in this file (replace `$`→`%24`, newline→`%0A`, prepend `https://www.falstad.com/circuit/circuitjs.html?cct=`). Present the URL in a fenced code block labeled `url` (or inline code if short enough). If the circuit exceeds ~30 lines, note that the URL may be truncated and recommend the text-block import instead, but still provide the URL.
3. A one-paragraph explanation of the circuit function.
4. The expected simulation behavior (currents, voltages, LED states).
5. Import instructions: "Open circuitjs1 → File → Import From Text... (文件 → 从文本导入...) → paste the block → OK." Also mention: "Or click the URL above to open it directly in the Falstad online simulator."
6. (Optional) Suggestions for the user to tweak (e.g. "change 220.0 to 330.0 to dim the LED").
7. **Language**: respond in the same language the user used (Chinese → Chinese, English → English). The circuit text itself is language-neutral.

Do NOT include any commentary inside the code block — only valid circuit text.
