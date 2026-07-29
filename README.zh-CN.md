# CircuitJS1 电路生成器 Skill

[English](./README.md) | **中文**

让 AI 生成可直接导入 [circuitjs1](https://github.com/sharpie7/circuitjs1)（在[此处](https://www.falstad.com/circuit/)在线使用）电路模拟器的完整电路。文本格式，用户通过 **文件 → 从文本导入** 粘贴到 circuitjs1 即可。



## 快速使用

**安装**

下载库，发送至你的 AI 程序。

如果使用网页端 AI，也可以将skill.md中的内容复制至 AI 对话窗口。

**使用**

1. **用自然语言向 AI 提问**：
   
   > "生成一个电路：555 定时器驱动 LED 以 2 Hz 闪烁。"
   
   支持视觉的 AI 也可以输入图片。
   
2. **AI 返回**一个 `circuitjs` 代码块，内含完整电路文本，附带简要说明和预期行为。

3. **导入 circuitjs1**：
   - 打开[在线模拟器](https://www.falstad.com/circuit/)或本地 circuitjs1 构建。
   - 菜单 **文件 → 从文本导入...**
   - 粘贴整个代码块 → **确定**。
   - 电路图出现在画布上并开始仿真。

4. **按需调整**：修改电阻值、添加滑块、连接示波器 —— 既可以编辑文本后重新导入，也可以直接在 GUI 中操作。



## 文件

[`examples/`](./examples) 目录下包含六个完整的电路示例：

| 文件 | 电路 | 演示内容 |
|------|------|---------|
| `01_led_circuit.txt` | LED + 220Ω 电阻接 5V 直流 | 基础无源元件、LED、闭合回路 |
| `02_rc_lowpass.txt` | RC 低通滤波器 + 示波器 | 交流电源、电容、示波器 |
| `03_transistor_switch.txt` | NPN 晶体管开关 LED | 三极管、双电源、基极驱动 |
| `04_voltage_divider.txt` | 12V 分压器（8kΩ/4kΩ）+ 示波器 | 分压、示波器显示中点电压 |
| `05_custom_diode_model.txt` | 使用自定义 1N4148 模型的二极管 | DiodeModel 定义、FLAG_MODEL |
| `06_slider_resistor.txt` | 可调电阻 + 示波器 | 可调滑块（`38`）、运行时控制 |

[`SKILL.md`](./SKILL.md) 包含：

- **格式总览** —— 行结构、token 分隔符、实体类型分派表
- **全局选项行** —— 必需的 `$` 头部
- **连接规则** —— 坐标匹配机制，包括关键的"导线中间点不连接"警告
- **转义规则** —— `CustomLogicModel.escape` 方案，用于含空格/特殊字符的字符串
- **完整元件参考** —— 每个常用元件的精确字段顺序（无源、电源、开关、测量、逻辑、标签）
- **模型定义** —— DiodeModel（`34`）、TransistorModel（`32`）、CustomLogicModel（`!`）、CustomCompositeModel（`.`）
- **可调滑块**（`38`）—— 运行时可控制的元件属性
- **示波器**（`o`）—— 按索引引用元件的示波器波形
- **生成工作流** —— 严格的 9 步流程，附 5 部分自检清单
- **故障排查表** —— 常见错误与修复方法
- **快速参考卡** —— 每个常用元件的单行模板



## 技术原理

circuitjs1 通过 `CirSim.dumpCircuit()` 序列化电路，通过 `CirSim.readCircuit()` 反序列化 —— 两者均位于 [`src/com/lushprojects/circuitjs1/client/CirSim.java`](https://github.com/sharpie7/circuitjs1/blob/master/src/com/lushprojects/circuitjs1/client/CirSim.java)。文本格式是面向行、空格分隔的 token 流：

- 第 1 行：`$ <flags> <maxTimeStep> <speedParam> <currentBar> <voltageRange> ...`
- 后续每行：`<类型码> <x1> <y1> <x2> <y2> <flags> [<子类字段>...]`
- 连接 = 两个端点坐标完全相同。
- 模型定义行（`34`/`32`/`!`/`.`）必须出现在引用它的元件行之前。



## 许可证

circuitjs1 项目和 Falstad 模拟器有各自的许可证 —— 分发打包电路或截图时请遵守相应许可。
