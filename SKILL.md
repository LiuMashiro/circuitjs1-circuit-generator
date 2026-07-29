---
name: "circuitjs1-circuit-generator"
description: "Generates circuitjs1-importable plain-text circuits from natural language. Invoke when user asks to create/design/build a circuit, or wants AI to generate a circuit for the circuitjs1/Falstad simulator."
---

# CircuitJS1 Circuit Text Generator

This skill enables the AI to generate complete, valid circuitjs1 circuits as plain text. The output can be imported into the circuitjs1 (Falstad) circuit simulator via **File → Import From Text...** without any GUI操作.

## When to Invoke

Invoke this skill when:
- User asks to "create / design / build / generate a circuit"
- User describes a circuit in natural language (e.g., "an LED with a 220 ohm resistor on 5V")
- User wants the AI to produce a circuit for the circuitjs1 / Falstad simulator
- User asks for a schematic-like output that can be simulated
- User mentions "circuitjs1", "Falstad circuit", or "import from text"

## How the User Imports the Output

After the AI generates the circuit text, the user does ONE of:

1. **Import From Text (recommended)**: Open circuitjs1 → menu `File → Import From Text...` → paste the entire text block → click OK.
2. **URL parameter**: Replace every `$` with `%24` and every newline with `%0A`, then append to the simulator URL as `?cct=<encoded>`.
3. **Local file**: Save as `.txt`, then `File → Import From File...`.

The AI should always present the circuit inside a fenced code block labeled `circuitjs` so the user can copy it easily.

---

## 1. Format Overview

The circuit is **line-oriented plain text**:

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
| `t` | TransistorElm | `t x1 y1 x2 y2 flags pnp Vbc Vbe [beta] [modelName]`（beta 缺省=100，modelName 缺省=default） |
| `f` | MosfetElm | `f x1 y1 x2 y2 flags vt beta` |
| `209` | PolarCapacitorElm | `209 x1 y1 x2 y2 flags capacitance voltdiff initialVoltage maxNegativeVoltage` |

**Field meanings**:
- `resistance` (Ω), `capacitance` (F), `inductance` (H): physical values.
- `voltdiff`, `current`, `Vbc`, `Vbe`: simulation state vars → fill `0`.
- `initialVoltage` (capacitor): fill `0`.
- `symbolType` (ground): `0` = earth symbol.
- `pnp` (transistor): `1` = NPN, `-1` = PNP.
- `beta` (transistor): current gain, typical `100`.
- `vt` (MOSFET): threshold voltage, typical `0.75`.
- `beta` (MOSFET): transconductance param, typical `0.03`.
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
| `a` | OpAmpElm (ideal) | `a x1 y1 x2 y2 flags` (no extra fields) |
| `403` | ScopeElm (in-circuit) | `403 x1 y1 x2 y2 flags ...` |
| `174` | PotElm (potentiometer) | `174 x1 y1 x2 y2 flags resistance position` |
| `159` | AnalogSwitchElm | `159 x1 y1 x2 y2 flags ...` |
| `165` | TimerElm (555) | `165 x1 y1 x2 y2 flags ...` |

---

## 6. Model Definitions (Advanced — User-Selectable Models)

This is the feature that lets the user pick specific component models (e.g. a 1N4148 diode, a 2N2222 transistor, a custom logic block). **Fully supported via text.**

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

---

## 7. Adjustable Sliders (line prefix `38`)

Add a slider to control any component property at runtime.

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

## 9. Generation Workflow (STRICT — follow in order)

### Step 1: Understand the request
- Identify required components (power source, load, control, measurement).
- Identify whether the user needs custom models (specific diode/transistor part numbers).
- Identify whether sliders or scopes are desired.

### Step 2: Plan the layout
- Sketch coordinates mentally. Use multiples of 8.
- Decide power rail y-values (e.g. VCC at y=64, GND at y=256).
- List components in order and assign each a planned (x1,y1,x2,y2).
- Verify every intended connection has matching coordinates.

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

### Step 9: Output

Present the circuit in a fenced code block. Then provide:
- A brief explanation of what the circuit does.
- The expected behavior (e.g. "LED lights at ~13 mA").
- Instructions to import (File → Import From Text...).

---

## 10. Examples

Six fully-worked examples are in the `examples/` directory of this skill:

1. `01_led_circuit.txt` — LED with current-limiting resistor on 5V DC.
2. `02_rc_lowpass.txt` — RC low-pass filter driven by AC source, with scope on capacitor.
3. `03_transistor_switch.txt` — NPN transistor switching an LED, base driven by control voltage.
4. `04_voltage_divider.txt` — 12V divider (8kΩ/4kΩ) with scope showing midpoint voltage (4V).
5. `05_custom_diode_model.txt` — Diode using a custom 1N4148 model definition.
6. `06_slider_resistor.txt` — Resistor with an adjustable slider (0–1000 Ω) + scope.

Study these before generating your own. They cover the most common patterns.

---

## 11. Troubleshooting & Common Errors

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
2. A one-paragraph explanation of the circuit function.
3. The expected simulation behavior (currents, voltages, LED states).
4. Import instructions: "Open circuitjs1 → File → Import From Text... → paste the block → OK."
5. (Optional) Suggestions for the user to tweak (e.g. "change 220.0 to 330.0 to dim the LED").

Do NOT include any commentary inside the code block — only valid circuit text.
