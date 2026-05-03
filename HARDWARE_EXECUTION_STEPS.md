# WSPR Transceiver 硬件负责人执行步骤

负责人范围：Part A 的 RF loopback 硬件改造 + Part B 的 transceiver PCB 新硬件设计。  
起始日期：2026-05-03。  
目标：尽快冻结硬件方案、采购关键器件、完成 KiCad 原理图和 PCB，并为报告准备可解释的设计证据。

## 0. 你要交付什么

你的硬件部分最终必须能回答这几个问题：

1. `CLK2` 发射信号如何进入 RF 发射链路？
2. 发射输出如何达到 `0.1 W into 50 ohms`？
3. 发射器和接收器如何共用原来的 bandpass filter？
4. GPIO 如何切换 TX/RX？
5. TX 时如何保护 RX 输入，RX 时如何关闭/隔离 PA？
6. PCB 尺寸如何保持与原板一致？
7. 新增器件为什么这样选，是否来自 Element14 Australia？

## 1. 第一天：先锁定原板 RF 路径

打开 KiCad 工程：

- `bbb.kicad_pro`
- `bbb.kicad_sch`
- `bbb.kicad_pcb`

在 schematic 和 PCB 中标记这些网络：

| 网名 | 你要确认的内容 |
|---|---|
| `CLK2` | Si5351 剩余输出，作为 TX RF source。 |
| `BPFIN` | 原 bandpass filter 输入。 |
| `BPFOUT` | 原 bandpass filter 输出。 |
| `RFIN` | 原 receiver RF input path。 |
| `RFOUT` | 原 RF amplifier/mixer 相关网络，注意不要误认为最终 TX output。 |
| `U6 SMA` | 原 RF 接口，优先复用为收发共用 50 ohm 端口。 |

执行动作：

1. 在原理图截图上用箭头画出原 RX 路径：

```text
SMA / RFIN -> BPFIN -> BPF -> BPFOUT -> receiver amplifier/mixer
```

2. 在 PCB 截图上圈出：
   - Si5351 `U2`
   - `CLK2`
   - BPF 电容/电感区域
   - SMA `U6`
   - receiver input/mixer 区域

3. 保存两张截图给报告使用：
   - `report_images/original_rf_path_schematic.png`
   - `report_images/original_rf_path_pcb.png`

完成标准：

- 你能指着图解释 RX 信号从 SMA 到 mixer 的路径。
- 你能说出 TX 新电路要插入在哪里。

## 2. 第二步：建立新的 KiCad 工程副本

不要直接破坏原始 `bbb` 工程。复制出新工程：

```text
wspr_transceiver.kicad_pro
wspr_transceiver.kicad_sch
wspr_transceiver.kicad_pcb
```

建议做法：

1. 在文件管理器中复制 `bbb.kicad_pro`、`bbb.kicad_sch`、`bbb.kicad_pcb`。
2. 改名为 `wspr_transceiver.*`。
3. 用 KiCad 打开新 `.kicad_pro`。
4. 确认原 schematic 和 PCB 能正常打开。

完成标准：

- 原 `bbb.*` 保留不动。
- 新工程可以编辑。
- PCB outline 和 mounting holes 与原板一致。

## 3. 第三步：确定硬件架构

推荐你采用这个主方案：

```text
TX mode:
Si5351 CLK2
 -> input attenuator / DC block
 -> driver stage
 -> PA final stage
 -> harmonic cleanup / matching
 -> GPIO-controlled T/R switch
 -> shared BPF
 -> SMA 50 ohm output

RX mode:
SMA 50 ohm input
 -> GPIO-controlled T/R switch
 -> shared BPF
 -> original receiver chain
```

优先方案：

- PA：分立 transistor/MOSFET PA，目标输出 20 dBm。
- T/R switch：PIN diode switch 或合适的 RF switch。
- GPIO：新增 `TR_CTRL` 控制 TX/RX。
- PA enable：新增 `PA_EN`，保证 RX 时 PA off。

备用方案：

- 如果 PIN diode switch 调不顺，可使用小继电器或确认频率范围合适的 RF switch IC。
- 如果分立 PA 达不到 20 dBm，可使用 driver + final 两级，或换 RF transistor。

你需要在 report 中明确：

```text
TR_CTRL = 0: RX mode, PA off, SMA -> BPF -> RX
TR_CTRL = 1: TX mode, PA on, PA -> BPF -> SMA
```

## 4. 第四步：先做功率计算

作业要求：

```text
Pout = 0.1 W into 50 ohms
```

报告中必须写出：

```text
Pout = 0.1 W = 20 dBm
Rload = 50 ohms
Vrms = sqrt(P * R) = sqrt(0.1 * 50) = 2.236 V
Vpk = sqrt(2) * Vrms = 3.162 V
Vpp = 2 * Vpk = 6.324 Vpp
Irms = Vrms / R = 44.7 mA
```

设计目标建议：

- PA 设计能力留到约 `23 dBm`。
- 实际输出通过 bias、matching 或 attenuator 调到约 `20 dBm`。
- PA 后必须接 50 ohm dummy load 测试，不要直接接接收机。

## 5. 第五步：当天完成采购清单

采购必须来自 Element14 Australia。下单前当天重新确认库存、交期和 order code。

第一批优先买这些，不等 PCB 完成：

| 类别 | 数量建议 | 用途 |
|---|---:|---|
| PA final MOSFET / RF transistor | 每种 5-10 | 100 mW PA 原型和备件。 |
| RF driver transistor | 5-10 | 驱动 PA final。 |
| PIN diode | 每种 10-20 | 做 T/R switch。 |
| C0G/NP0 RF capacitor | 每值 10-20 | matching、filter、DC block。 |
| RF inductor | 每值 5-10 | PA 输出匹配和 LPF。 |
| 50 ohm / attenuator resistor | 每值 20 | 输入/输出衰减、dummy load、匹配。 |
| SMA connector / test point | 2-5 | RF 输入输出和测量点。 |

采购表必须记录：

```text
Part name
Manufacturer part number
Element14 order code
Package
Quantity
Unit price
Stock status
Estimated delivery date
Datasheet URL
Why selected
Backup part
Checked date
```

保存为：

```text
hardware_bom_element14.md
```

不要只买一个型号。PA final、PIN diode、RF switch 至少各保留一个备选。

## 6. 第六步：画 Part A RF loopback 修改

Part A 是现有板子的物理改造，用来让软件发射先能被接收机解码。

推荐电路：

```text
Si5351 CLK2
 -> 100 pF to 1 nF DC blocking capacitor
 -> resistive attenuator
 -> short coax / short wire
 -> receiver RF input before mixer/BPF chain
```

执行步骤：

1. 找到 `CLK2` test point 或 Si5351 `CLK2` 引脚附近可焊点。
2. 找到 `RFIN` 或 receiver RF input 可焊点。
3. 先设计较大衰减，不要直接连接。
4. 在纸上或 KiCad 中画出修改电路。
5. 焊接后拍照。
6. 用示波器测：
   - `CLK2` 直接输出幅度
   - 衰减后进入 receiver 的幅度
7. 把测量表放进报告。

完成标准：

- 有修改电路图。
- 有实物照片。
- 有测量数据。
- 能解释为什么不会过载 receiver input。

## 7. 第七步：画 Part B 新原理图

在 `wspr_transceiver.kicad_sch` 中新增或修改这些模块。

### 7.1 TX input from Si5351

新增网络：

```text
CLK2 -> TX_RF_RAW -> C_DC_BLOCK -> R_ATTENUATOR -> TX_PA_IN
```

要求：

- `CLK2` 后加 DC blocking capacitor。
- 加一个可调/可改值 attenuator，避免 PA input 过驱。
- 加 test point：`TP_CLK2_TX`、`TP_PA_IN`。

### 7.2 PA driver + final

新增网络：

```text
TX_PA_IN -> driver -> PA_FINAL -> TX_PA_OUT
```

要求：

- 明确 bias resistor。
- 电源旁边放 `100 nF + 1 uF/10 uF` 去耦。
- `PA_EN` 控制 PA 供电或偏置。
- 输出端预留 matching network：

```text
series L / shunt C / pi network / DNP footprints
```

### 7.3 Harmonic cleanup

在 PA 后加：

```text
TX_PA_OUT -> LPF or matching/filter network -> TX_FILTERED
```

如果你认为共享 BPF 已足够，也要在报告中解释。但 PCB 上建议预留 DNP 的 LPF 位，方便调试。

### 7.4 T/R switch

新增网络：

```text
TX_FILTERED -> T/R switch -> shared BPF -> SMA
SMA -> T/R switch -> shared BPF -> RX_CHAIN
```

要求：

- `TR_CTRL` 控制 TX/RX。
- 默认状态建议 RX。
- TX 时 RX input 必须隔离或保护。
- RX 时 PA 必须关闭。

### 7.5 Test points

至少加：

```text
TP_CLK2_TX
TP_PA_IN
TP_PA_OUT
TP_TR_CTRL
TP_PA_EN
TP_ANT_50R
TP_RX_IN
```

## 8. 第八步：PCB layout 执行顺序

layout 不要从漂亮开始，从 RF 关键路径开始。

执行顺序：

1. 锁定原板 outline、mounting holes、SMA 位置。
2. 先放 T/R switch，靠近 BPF 和 SMA。
3. 再放 PA final，靠近 T/R switch，但和 RX input 保持距离。
4. 放 PA driver，靠近 PA final。
5. 从 Si5351 `CLK2` 到 `PA_IN` 走线尽量短。
6. PA 输出到 switch/BPF/SMA 走线尽量短、少过孔、少 stub。
7. PA 电源线加宽，去耦电容贴近 PA 管脚。
8. RF path 下方尽量保持连续 ground。
9. 增加 ground vias，尤其是 PA、switch、SMA 附近。
10. 最后再放 test points 和丝印。

PCB 检查：

- 板框尺寸不变。
- 新增器件 footprint 与 datasheet land pattern 一致。
- TX PA output 不要长距离贴着 RX input 走。
- `TR_CTRL` 和 `PA_EN` 不要从 RF 线旁边长距离平行走。
- 50 ohm RF 线尽量短；如果不能精确控阻，也要避免长线和突变。

## 9. 第九步：ERC/DRC 和制造文件

KiCad 中执行：

1. Schematic ERC。
2. Update PCB from schematic。
3. PCB DRC。
4. Plot Gerbers。
5. Generate drill files。
6. Export BOM。
7. Export placement/position file。

需要保存：

```text
fabrication/gerbers/
fabrication/drill/
fabrication/bom/
fabrication/position/
report_images/pcb_top.png
report_images/pcb_bottom.png
report_images/pcb_3d.png
```

DRC 如果有错误：

- 能修就修。
- 不能修的写进 `hardware_known_issues.md`，说明原因。

## 10. 第十步：硬件 bring-up 测试

不要第一次上电就直接 TX 100 mW。

顺序：

1. 不焊 PA final，测电源。
2. 测 `TR_CTRL` 和 `PA_EN` GPIO 电平。
3. 焊 T/R switch，测 RX path 是否通。
4. 焊 driver，测 `TX_PA_IN` 和 driver output。
5. 焊 PA final，限流供电。
6. PA 输出先接：

```text
20 dB attenuator -> 50 ohm dummy load
```

7. 测输出电压，换算功率。
8. 达到接近 20 dBm 后，再接 shared BPF。
9. 测 TX/RX 切换：
   - RX mode：PA off，SMA -> BPF -> RX
   - TX mode：PA on，PA -> BPF -> SMA
10. 最后再和软件发射/decoder 联调。

输出功率换算：

```text
P = Vrms^2 / 50
dBm = 10 * log10(P / 1 mW)
```

如果测的是 Vpp：

```text
Vrms = Vpp / (2 * sqrt(2))
```

## 11. 报告中你负责放的内容

硬件部分报告必须有：

- Part A RF loopback 修改电路图。
- Part A 修改后 PCB 照片。
- `CLK2` 输出和 receiver input 的实测电平。
- Part B 完整 TX/RX signal path 图。
- PA 输出 `0.1 W into 50 ohms` 计算。
- PA 原理图和关键元件值。
- T/R switch 原理图。
- GPIO truth table。
- PCB top/bottom layout 截图。
- Element14 BOM 表。
- ERC/DRC 状态说明。

GPIO truth table 可直接用：

| `TR_CTRL` | `PA_EN` | 模式 | RF path |
|---|---|---|---|
| 0 | 0 | RX | SMA -> T/R switch -> shared BPF -> receiver |
| 1 | 1 | TX | CLK2 -> PA -> T/R switch -> shared BPF -> SMA |

## 12. 你每天结束前要提交的东西

每天至少更新一次：

```text
hardware_decision_log.md
hardware_bom_element14.md
hardware_known_issues.md
```

每天 Git 提交：

```powershell
git add .
git commit -m "Update hardware design progress"
git push
```

建议每次 commit message 写具体一点，例如：

```powershell
git commit -m "Add PA and TR switch schematic draft"
git commit -m "Add Element14 hardware BOM"
git commit -m "Route transmitter RF path"
git commit -m "Add PCB screenshots for report"
```

## 13. 最容易扣分的点

- 只画了 PA，没有说明 TX/RX 如何共用 BPF。
- 只写了 0.1 W，没有给 `50 ohm` 电压/电流计算。
- TX 时没有保护 RX input。
- GPIO 只写“控制”，没有 truth table。
- 新器件没有 Element14 order code。
- PCB 尺寸改变了。
- 报告解释了太多旧 WSPR-SDR，反而没解释你的新增设计。

## 14. 最小可交付版本

如果时间不够，最低也要完成：

1. 新 KiCad 工程。
2. `CLK2 -> PA -> T/R switch -> shared BPF -> SMA` 原理图。
3. `SMA -> T/R switch -> shared BPF -> RX` 原理图。
4. 0.1 W / 50 ohm 计算。
5. Element14 BOM。
6. PCB 保持原尺寸，并放置/连接所有新增模块。
7. ERC/DRC 记录。
8. Part A RF loopback 修改图和照片。

