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

## 1A. 根据 `BBBSchematic.jpg` 精确确定硬件架构

这张图不是只用来看元件名，而是用来决定 transceiver 的切入点。先按下面顺序把原板分成 6 个功能块。

### 1A.1 原接收机 RF 路径

从 `BBBSchematic.jpg` 可以读出原接收链：

```text
U6 SMA connector
 -> BPFIN
 -> Band Pass Filter
    R20, C20-C32, J4/J5/J6 external inductors, R21
 -> BPFOUT / RFIN
 -> C2 AC coupling
 -> U1 THS4304 RF amplifier
 -> RFOUT
 -> U3 TS3A5017 Tayloe / quadrature mixer
 -> MIXI+, MIXI-, MIXQ+, MIXQ-
 -> U4 TL972 audio amplifier
 -> J3 audio jack, Iout/Qout
```

这说明原板真正的 RF input/output 关系是：

| 位置 | 图中标号 | 硬件含义 |
|---|---|---|
| RF connector | `U6` | SMA 接口，当前只作为接收输入。 |
| BPF input | `BPFIN` | SMA 后进入 bandpass filter 的一端。 |
| BPF output | `BPFOUT` | bandpass filter 输出端。 |
| RX amplifier input | `RFIN` / `RFIN_TP7` | 与 `BPFOUT` 同一输入区域，经 `C2` 进入 U1。 |
| RX amplifier output | `RFOUT` / `RFOUT_TP8` | U1 输出，送入 mixer。 |
| Mixer input | `MIXIN` | U3 TS3A5017 的 RF mixer input。 |

结论：Part B 不应该把 PA 接到 `RFOUT`。`RFOUT` 是接收机 RF amplifier 的输出，不是天线输出。TX power path 应围绕 `CLK2`、`BPFOUT/BPFIN` 和 SMA 重新设计。

### 1A.2 原时钟和可用 TX 源

图中 `U2 Si5351A-B-GT` 有 3 个输出：

| Si5351 输出 | 原用途 | 新设计动作 |
|---|---|---|
| `CLK0` | 送 mixer 的 0 度 LO，标成 `MIXCLK0` | 保留，不改。 |
| `CLK1` | 送 mixer 的 90 度 LO，标成 `MIXCLK90` | 保留，不改。 |
| `CLK2` | 已引出但未用于 RX 主链 | 作为 WSPR TX RF source。 |

结论：TX 链从 `CLK2` 开始，不能占用 `CLK0/CLK1`，否则会破坏接收机 mixer。

### 1A.3 最推荐的 Part B 架构

为了最少改动原接收机，同时满足“发射机与接收机共用 bandpass filter”，推荐架构如下：

```text
RX mode:
SMA U6
 -> BPFIN
 -> shared BPF
 -> BPFOUT
 -> T/R switch RX throw
 -> RFIN / C2
 -> U1 THS4304
 -> RFOUT
 -> U3 mixer
 -> I/Q audio

TX mode:
Si5351 CLK2
 -> DC block / attenuator
 -> TX driver
 -> PA final
 -> PA output matching / optional LPF
 -> T/R switch TX throw
 -> BPFOUT
 -> shared BPF used in reverse direction
 -> BPFIN
 -> SMA U6
 -> 50 ohm load
```

这个方案的核心切入点是 `BPFOUT/RFIN`：

- `BPFOUT` 是 BPF 和 receiver amplifier 之间的天然分界点。
- 在 RX 模式，`BPFOUT` 接回 `RFIN/C2/U1`，原接收机照常工作。
- 在 TX 模式，`BPFOUT` 从 receiver input 断开，改接 PA output，让 TX 信号反向通过同一个 BPF 到 SMA。
- BPF 是 LC 被动网络，理论上双向可用；但原板的 50 ohm 电阻和实际插损必须测量/计算。

### 1A.4 为什么不从其他位置接入

不要选择这些错误切入点：

| 错误切入点 | 问题 |
|---|---|
| `RFOUT` | 这是 U1 接收放大器输出，会把 TX 功率打进 mixer 区域，不能作为天线输出。 |
| `MIXIN` | 这是 mixer input，不经过 BPF，也不能输出 0.1 W。 |
| `CLK0` / `CLK1` | 已用于 quadrature mixer LO，占用后 RX 不工作。 |
| SMA 直接并联 PA output | RX input 和 PA output 会互相加载，TX 时可能损坏 receiver。 |
| 另加独立 BPF | 不满足“发射器应与接收器共用 bandpass filter”的要求。 |

### 1A.5 必须复核的 R20 / R21

图中 BPF 两端有两个关键 50 ohm 电阻：

- `R20 = 50 ohm`，在 `BPFIN` 进入 BPF 的位置。
- `R21 = 50 ohm`，在 `BPFOUT` 端附近。

这两个元件在接收机里可能用于滤波器匹配/终端，但在 100 mW TX 路径里会影响功率。

你必须做以下判断：

1. 在 TX 模式，`R21` 是否会变成 PA output 的额外 50 ohm shunt load？
2. 在 TX 模式，`R20` 是否会成为 SMA 前的 50 ohm series loss？
3. 如果保留 R20/R21，PA 需要输出多少功率才能保证 SMA 外接 50 ohm load 上得到 0.1 W？
4. 如果 R21 会明显吃掉 TX 功率，是否应把 R21 改成：
   - DNP footprint；
   - 只在 RX 模式接入的 switchable termination；
   - 或移到 T/R switch 的 RX throw 后面。
5. 如果 R20 造成明显插损，是否应改成：
   - 0 ohm / small matching resistor；
   - RF-rated resistor；
   - 或重新计算 BPF matching。

报告中不要只写“使用原 BPF”。必须写清楚 R20/R21 对 TX 输出功率的影响，并说明最终是保留、改值、DNP，还是切换接入。

### 1A.6 精确的 T/R switch 定义

建议把 T/R switch 放在 `BPFOUT` 与 `RFIN/C2` 之间。

推荐网络命名：

```text
BPFOUT_SHARED      BPF 的 BPFOUT 端
RX_RFIN_SWITCHED   接回 C2/U1 的 RX input
TX_PA_FILTERED     PA output/matching 后的 TX 信号
TR_CTRL            GPIO 控制信号
PA_EN              PA 使能信号
```

逻辑关系：

| `TR_CTRL` | `PA_EN` | 模式 | Switch 状态 | RF path |
|---|---|---|---|---|
| 0 | 0 | RX | `BPFOUT_SHARED -> RX_RFIN_SWITCHED` | SMA -> BPFIN -> BPF -> U1/U3 |
| 1 | 1 | TX | `TX_PA_FILTERED -> BPFOUT_SHARED` | CLK2 -> PA -> BPF reverse -> SMA |

默认安全状态应为 RX：

- 上电时 `TR_CTRL = 0`。
- 上电时 `PA_EN = 0`。
- 软件只有在准备 TX 时才拉高 `TR_CTRL` 和 `PA_EN`。

### 1A.7 你在 KiCad 中要做的精确修改

在 `wspr_transceiver.kicad_sch` 中按这个顺序改：

1. 保留 `U6 -> BPFIN -> BPF -> BPFOUT`。
2. 断开 `BPFOUT/RFIN` 直接进入 `C2/U1` 的连接。
3. 在断开点加入 RF SPDT / PIN diode switch。
4. RX throw 连接到原 `RFIN -> C2 -> U1`。
5. TX throw 连接到 `TX_PA_FILTERED`。
6. `TX_PA_FILTERED` 来自：

```text
CLK2 -> DC block -> attenuator -> driver -> PA final -> matching/LPF
```

7. 增加 `PA_EN` 控制 PA 偏置或供电。
8. 增加 `TR_CTRL` 控制 T/R switch。
9. 在这些点加 test points：

```text
TP_CLK2_TX
TP_TX_PA_IN
TP_TX_PA_OUT
TP_TX_FILTERED
TP_BPFOUT_SHARED
TP_TR_CTRL
TP_PA_EN
TP_ANT_50R
```

### 1A.8 需要量出来或算出来的数据

根据 `BBBSchematic.jpg`，你需要为硬件报告准备这些数据：

| 数据 | 测量/计算位置 | 目的 |
|---|---|---|
| BPF passband loss | `BPFIN` 到 `BPFOUT` | 判断 TX 反向通过 BPF 后还剩多少功率。 |
| R20/R21 power dissipation | BPF 两端 50 ohm 元件 | 确认 0805 resistor 不过热且不吞掉太多功率。 |
| PA output before BPF | `TX_PA_OUT` 或 `TX_FILTERED` | 确认 PA 本身能达到目标。 |
| Final output power | SMA 外接 50 ohm load | 证明符合 `0.1 W into 50 ohms`。 |
| RX isolation during TX | `RFIN/C2` 处 | 确认 TX 不会打进 U1。 |
| RX path insertion loss | RX mode 下 SMA 到 `RFIN` | 确认新增 switch 没明显损坏接收性能。 |

最终验收时以 SMA 外接 50 ohm dummy load 上的功率为准：

```text
0.1 W into 50 ohms = 2.236 Vrms = 6.324 Vpp sine-equivalent
```

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

### 6.1 接收机过载不是 5 V / 3.3 V 逻辑不匹配

Part A 中说的 receiver overload，主要不是普通数字电路里的 `5 V -> 3.3 V` 逻辑电平不匹配，而是 **RF 输入幅度太大，超过接收机前端线性范围**。

示波器证据见：

```text
report_images/partA_clk_overload_scope.jpg
```

该截图显示 Si5351 相关时钟输出约：

```text
Frequency ~= 6.993 MHz
CH1 Vpp ~= 4.28 V
CH2 Vpp ~= 4.45 V
Phase ~= 89 deg
```

这类信号是 CMOS 方波时钟，幅度是“几伏 Vpp”等级；而 receiver RF input 期望的是弱 RF 信号，通常是 microvolt 到 millivolt 等级。把几伏方波直接接到 `RFIN` 或 `BPFIN`，会让后级饱和：

```text
RFIN / C2
 -> U1 THS4304 RF amplifier saturates
 -> RFOUT clips or becomes nonlinear
 -> U3 TS3A5017 mixer is overdriven
 -> I/Q audio distortion
 -> WSPR decoder cannot decode reliably
```

如果把 `4.3 Vpp` 粗略按正弦等效估算：

```text
Vrms = Vpp / (2 * sqrt(2))
     = 4.3 / 2.828
     ~= 1.52 Vrms

P50 = Vrms^2 / 50
    ~= 1.52^2 / 50
    ~= 0.046 W
    ~= +16.6 dBm
```

`+16.6 dBm` 对接收机输入来说非常强。WSPR receiver 的 RF input 应从很小电平开始测试；Part A loopback 只需要 decoder 能解码，不需要大功率。

### 6.2 电压不匹配和 RF 过载的区别

| 问题 | 发生位置 | 结果 | Part A 关注点 |
|---|---|---|---|
| 5 V / 3.3 V 逻辑不匹配 | 数字 GPIO / digital input | 可能损坏 3.3 V 逻辑输入 | 不要把 5 V digital signal 直接接 3.3 V pin。 |
| RF receiver overload | `RFIN`, `BPFIN`, U1/U3 analog RF path | 放大器/mixer 饱和，解码失败，严重时可能损坏输入 | `CLK2` 必须先隔直和大幅衰减。 |

本项目 Part A 的核心问题是第二类：`CLK2` 是 RF 方波源，但 receiver input 是敏感模拟 RF 输入。即使 `CLK2` 是 3.3 V CMOS，也仍然远大于接收机需要的 RF 电平。

### 6.3 安全的 Part A 衰减起步值

第一次 RF loopback 不要直接连接：

```text
CLK2 -> RFIN
```

推荐从大衰减开始：

```text
CLK2
 -> 100 pF to 1 nF DC blocking capacitor
 -> 10 kΩ series resistor
 -> injection point: RFIN_TP7 or BPFIN
        |
       50 Ω or 100 Ω
        |
       GND
```

若使用 `10 kΩ + 50 Ω` 分压：

```text
Voltage ratio = 50 / (10000 + 50)
              ~= 0.005
Attenuation   ~= -46 dB

4.3 Vpp -> about 21 mVpp
```

若使用 `10 kΩ + 100 Ω` 分压：

```text
Voltage ratio = 100 / (10000 + 100)
              ~= 0.0099
Attenuation   ~= -40 dB

4.3 Vpp -> about 43 mVpp
```

调试顺序：

```text
10 kΩ series first
 -> if too weak, try 4.7 kΩ
 -> then 2.2 kΩ
 -> then 1 kΩ
```

每次只减少一级衰减，并观察 receiver 是否出现饱和、削顶、audio distortion 或 decoder 失败。

### 6.4 已验证参考输入：30 mVpp

之前实验中接收机使用过的参考 RF 输入为：

```text
report_images/partA_30mVpp_reference_input.jpg

Function generator:
frequency = 7.007 MHz
amplitude = 30.000 mVpp
offset    = 0 V
load mode = High-Z
waveform  = sine
```

把这个值作为 Part A RF loopback 的首选目标注入电平：

```text
Target receiver injection level ~= 30 mVpp
```

注意 `High-Z` 设置的含义：函数发生器屏幕上的 `30 mVpp` 是按高阻负载显示的。如果把函数发生器直接接到 50 ohm termination，实际电压可能约减半。Part A loopback 中应以示波器在 `RFIN_TP7` 或 `BPFIN` 处实际测到的 Vpp 为准。

### 6.5 从 Si5351/CLK2 反推所需衰减

示波器测得未衰减 Si5351 clock 约：

```text
CLK Vpp ~= 4.3 Vpp
```

目标接收机输入：

```text
Vtarget = 30 mVpp = 0.030 Vpp
```

所需电压比例：

```text
ratio = Vtarget / Vclk
      = 0.030 / 4.3
      ~= 0.00698
      ~= 1 / 143
```

换算为 dB：

```text
attenuation_dB = 20 * log10(0.00698)
               ~= -43.1 dB
```

所以从 `CLK2` 到 receiver input 需要约 `43 dB` 的电压衰减，才能把几伏方波压到约 `30 mVpp`。

如果按 50 ohm 等效输入计算 `30 mVpp` 对应功率：

```text
Vrms = Vpp / (2 * sqrt(2))
     = 0.030 / 2.828
     ~= 10.6 mVrms

P50 = Vrms^2 / 50
    ~= 2.25 uW
    ~= -26.5 dBm
```

这个电平仍然比真实空中 WSPR 弱信号大很多，但作为硬线 loopback/debug 输入是合理的起点，因为你们已经用函数发生器验证过接收链可处理 `30 mVpp`。

### 6.6 用电阻实现 30 mVpp 目标

使用一个隔直电容、一个串联电阻和一个明确的 shunt resistor，可以让输入电平可预测：

```text
CLK2
 -> 100 pF to 1 nF DC blocking capacitor
 -> Rs
 -> injection point: RFIN_TP7 or BPFIN
        |
       Rsh
        |
       GND
```

公式：

```text
Vout = Vin * Rsh / (Rs + Rsh)

Rs = Rsh * (Vin / Vout - 1)
```

若 `Vin = 4.3 Vpp`、`Vout = 30 mVpp`：

| Shunt resistor `Rsh` | Calculated `Rs` | Practical choice | Expected Vout |
|---:|---:|---:|---:|
| 50 ohm | 7.1 kΩ | 6.8 kΩ | about 31 mVpp |
| 50 ohm | 7.1 kΩ | 7.5 kΩ | about 28 mVpp |
| 100 ohm | 14.2 kΩ | 15 kΩ | about 28 mVpp |
| 235 ohm approximate receiver load | 33.4 kΩ | 33 kΩ | about 30 mVpp |

推荐实际起步：

```text
CLK2 -> 1 nF -> 6.8 kΩ -> RFIN_TP7
                         |
                        50 Ω
                         |
                        GND
```

这会把 `4.3 Vpp` 降到约 `31 mVpp`，非常接近之前实验的 `30 mVpp`。如果担心过载，先用 `10 kΩ + 50 Ω` 得到约 `21 mVpp`，确认接收链没有削顶后再换成 `6.8 kΩ`。

若只用串联电阻、不加明确 shunt resistor，输出电平会依赖接收机输入阻抗。图中 `RFIN` 经 `C2` 后看到的偏置网络大约包含 `R1 || R3 = 470 Ω || 470 Ω ~= 235 Ω`，但实际还会受 BPF、测试点、探头和布线影响。因此报告里更推荐写明使用了外部 `50 Ω` 或 `100 Ω` shunt 来固定衰减比。

### 6.7 推荐测量点和判断标准

示波器测量时：

- 优先使用 `10x probe`，减少探头对 RF 节点的负载。
- 先测 `CLK2` 原始输出，再测衰减后的注入点。
- 注入点可以先选 `RFIN_TP7`，成功后再换到 `BPFIN` 或 SMA，让信号经过 BPF。
- 保持短地线或用短同轴，避免测到过多 ringing。

记录表：

| 测量点 | 记录内容 | 判断 |
|---|---|---|
| `CLK2` | frequency, Vpp, waveform | 证明 TX source 存在。 |
| attenuator output | Vpp after attenuation | 确认不是几伏直接进 receiver。 |
| `RFIN_TP7` | injected RF level | 先从 tens of mVpp 或更低开始。 |
| `RFOUT_TP8` | U1 output waveform | 不应明显削顶/贴 rail。 |
| audio I/Q output | Iout/Qout waveform | 不应严重削顶，decoder 应能看到 tone。 |

报告写法：

```text
Receiver overload was avoided by treating the Si5351 CLK2 output as a large
CMOS RF waveform, not as a small antenna-level signal. A DC blocking capacitor
and resistive attenuator were inserted before RFIN/BPFIN. The unattenuated
clock was about 4.3 Vpp at 6.993 MHz, so the first loopback tests used about
40-46 dB attenuation before injecting the signal into the receiver input.
```

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
