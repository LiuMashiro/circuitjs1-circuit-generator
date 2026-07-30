# CircuitJS1 Circuit Generator Skill

**English** | [中文](./README.zh-CN.md)

Let AI generate complete, importable circuits for [circuitjs1](https://github.com/sharpie7/circuitjs1) (use it online [here](https://www.falstad.com/circuit/)). Text format — the user pastes into circuitjs1 via **File → Import From Text...**.



## Quick start

**Installation**

Download the entire repository and send it to your AI program as a skill. AI programs can usually recognize a .zip file that contains a [`SKILL.md`](./SKILL.md) in the root directory, or you can let the AI add this skill by natural language.

If using a web-based AI, you can also copy the contents of [`SKILL.md`](./SKILL.md) into the chat window.

**Usage**

1. **Ask the AI** in natural language:
   
   > "Generate a circuit: a 555 timer driving an LED at 2 Hz."
   
   Vision-capable AI can also accept images as input.
   
2. **The AI returns** a fenced `circuitjs` code block with the full circuit text, plus a brief explanation and expected behavior.

3. **Import into circuitjs1**:
   - Open the [online simulator](https://www.falstad.com/circuit/) or your local circuitjs1 build.
   - Menu **File → Import From Text...**
   - Paste the entire code block → **OK**.
   - The schematic appears on the canvas and starts simulating.

4. **Tweak as needed**: change resistor values, add sliders, attach scopes — either by editing the text and re-importing, or directly in the GUI.



## Files

[`SKILL.md`](./SKILL.md) includes:

- **Format overview** — line structure, token separators, entity dispatch table
- **Global options line** — the required `$` header
- **Connection rules** — the coordinate-matching mechanism, including the critical "wire intermediate points do NOT connect" warning
- **Escape rules** — the `CustomLogicModel.escape` scheme for strings containing spaces/special chars
- **Complete component reference** — every common component's exact field order (passive, sources, switches, measurement, logic, labels)
- **Model definitions** — DiodeModel (`34`), TransistorModel (`32`), CustomLogicModel (`!`), CustomCompositeModel (`.`), ideal and practical model selection guidance
- **Adjustable sliders** (`38`) — runtime-controllable component properties, when to use sliders guidance
- **Scopes** (`o`) — oscilloscope plots referencing components by index
- **Generation workflow** — a strict 9-step process with a 5-part self-validation checklist
- **6 complete examples** — LED, RC filter, transistor switch, voltage divider, custom diode model, adjustable resistor
- **Troubleshooting table** — common errors and fixes
- **Quick reference card** — one-line templates for every common component



## How it works

circuitjs1 serializes circuits via `CirSim.dumpCircuit()` and deserializes via `CirSim.readCircuit()` — both in [`src/com/lushprojects/circuitjs1/client/CirSim.java`](https://github.com/sharpie7/circuitjs1/blob/master/src/com/lushprojects/circuitjs1/client/CirSim.java). The text format is a line-oriented, space-separated token stream:

- Line 1: `$ <flags> <maxTimeStep> <speedParam> <currentBar> <voltageRange> ...`
- Each subsequent line: `<typeCode> <x1> <y1> <x2> <y2> <flags> [<subclass fields>...]`
- Connection = two endpoints with identical coordinates.
- Model definitions (`34`/`32`/`!`/`.`) must precede the components that reference them.



## License

[circuitjs1](https://github.com/sharpie7/circuitjs1) and the [Falstad simulator](https://www.falstad.com/circuit/) have their own licenses — please respect them when distributing bundled circuits or screenshots.

## Problem
In Section 9.0, the skill requires that AIs without multimodal capabilities avoid guessing image content and giving incorrect circuit results. This is a strict prompt that requires the AI to proactively reject requests beyond its capabilities. In a very small number of sensitive AI models, it may be mistakenly identified as a risk and cause the AI to refuse to respond. If you encounter this situation, as long as you confirm that you are currently using a multimodal model, you can simply remove the content of Section 9.0 yourself.
