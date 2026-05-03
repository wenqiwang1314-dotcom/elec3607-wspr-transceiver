# ELEC3607 Assignment 1: WSPR Transceiver 任务设计

版本依据：Assignment 1 - WSPR Transceiver Version 0.4, 15 Apr 2026  
工程目录：`pcb_2026`  
原始 KiCad 工程：`bbb.kicad_pro`, `bbb.kicad_sch`, `bbb.kicad_pcb`  
最终提交：单个 `.zip`，包含 KiCad schematic、PCB design files、IEEE conference format 报告。

## 1. 项目目标

本项目在实验课 WSPR-SDR 接收机基础上完成两个修改：

1. Part A：用 Si5351 剩余的 `CLK2` 输出实现 WSPR 发射软件，并通过硬连线 RF loopback 让已有接收机或教师 decoder board 解码。
2. Part B：从现有 PCB 出发，重新设计成收发一体 transceiver PCB。发射机输出需达到 `0.1 W into 50 ohms`，收发共用接收机 bandpass filter，收发切换由 GPIO 控制，新增器件来自 Element14 Australia。

评分重点不是解释旧 WSPR-SDR，而是清楚说明新增发射功能、RF loopback、功率放大、收发切换和 PCB 修改如何工作。

## 2. 本地 Resources 使用清单

| 资料 | 用途 |
|---|---|
| `Resources/wspr_coding_process.pdf` | Part A WSPR message 编码、symbol 生成、tone mapping 的主要依据。 |
| `Resources/Si5351-B.pdf` | Si5351 `CLK2` 输出频率、驱动强度、寄存器/I2C 控制和硬件限制。 |
| `Resources/QS11-2010-Taylor.pdf` | WSPR 协议背景、4-FSK、tone spacing、解码约束和报告引用。 |
| `Resources/Tayloe_mixer_x3a.pdf` | 原接收机 quadrature sampling/Tayloe mixer 信号路径说明。 |
| `Resources/A Comparison of Affordable, Self-Assembled Software-Defined Radio Receivers Using Quadrature Sampling Down-Conversion.pdf` | 报告风格和 SDR 接收架构描述参考。 |
| `Resources/ABCs of Probes 60W-6053-15.pdf` | 示波器探头、RF/低电平测量误差说明。 |
| `Resources/emc_and_signal_integrity_ce_magazine_march-april_1999.pdf` | PCB layout、RF grounding、return path 和串扰风险说明。 |
| `Resources/website.txt` | WSPR/SDR 相关网页入口。 |

## 3. 团队分工建议

| 角色 | 主要负责 | 输出物 |
|---|---|---|
| Software lead | `wspr_transmit` C 程序、WSPR 编码、Si5351 `CLK2` 控制、CLI 参数验证 | C 源码、运行说明、软件测试记录 |
| Hardware modification lead | Part A 现有 PCB 飞线/RF loopback/衰减网络/照片/测量 | 修改原理图、PCB 照片、测试数据 |
| RF/PCB lead | Part B PA、T/R switch、共享 BPF、KiCad schematic/PCB layout | 更新后的 `.kicad_sch`, `.kicad_pcb`, BOM |
| Report/integration lead | IEEE 报告、图表、参考文献、提交 zip 检查 | 4 页以内 PDF 报告、最终 zip |

每个人都需要能在 oral discussion 中解释自己部分之外的完整信号路径。

## 4. Part A 任务：WSPR 发射与 RF Loopback

### A1. 明确 WSPR 发射需求

- 输入格式：`./wspr_transmit CALLSIGN LOCATOR POWER`
- 示例：`./wspr_transmit VK2AAX QF56 3`
- CLI 参数检查：
  - callsign：合法 WSPR callsign 格式，报告中说明支持范围。
  - locator：4 字符 Maidenhead grid，例如 `QF56`。
  - power：WSPR dBm power code，示例 `3` 代表作业要求输入格式，实际含义需在报告中解释。
- 输出信号：
  - WSPR uses 4-FSK.
  - 每个 symbol 选择 4 个 tone 之一。
  - tone spacing 约 `1.4648 Hz`。
  - 一帧 WSPR 标准消息共 `162` symbols，持续约 `110.6 s`。
- 软件必须使用 Si5351 `CLK2` 产生 RF tone，不使用空中发射。

完成标准：

- 程序能接受示例命令并生成确定的 WSPR symbol/tone 序列。
- 参数错误时给出 usage。
- 报告中给出 symbol generation 流程图或伪代码。

### A2. 编写 `wspr_transmit` C 程序

建议文件：

- `software/wspr_transmit.c`
- `software/wspr_encode.c`
- `software/wspr_encode.h`
- `software/si5351_control.c`
- `software/si5351_control.h`
- `software/Makefile`

软件模块任务：

1. `main()` 解析 `CALLSIGN LOCATOR POWER`。
2. WSPR message packing：将 callsign、locator、power 打包成 WSPR message bits。
3. FEC/interleaving/sync：按 `Resources/wspr_coding_process.pdf` 生成 162 个 symbols。
4. Tone scheduler：将 symbol 映射到 `f0 + symbol * 1.4648 Hz`。
5. Si5351 driver：通过 I2C 更新 `CLK2` 输出频率。
6. Timing：每个 symbol 周期约 `110.592 / 162 = 0.6827 s`，尽量减少切换抖动。
7. Logging：启动时打印 callsign、locator、power、base frequency、frame start time。

建议测试：

- Unit test：固定输入 `VK2AAX QF56 3` 时 symbol 序列可重复。
- Dry run：不控制 Si5351，只打印 tone frequency 和 symbol index。
- Hardware run：连接示波器/频率计确认 `CLK2` 有 4 个细小频偏。

完成标准：

- `make` 后生成 `wspr_transmit`。
- `./wspr_transmit VK2AAX QF56 3` 能运行完整帧。
- 报告附核心 C 代码片段，不必粘贴全部文件。

### A3. 音频 loopback 小步验证

用途：在 RF loopback 之前先验证编码和解码链路。

任务：

- 生成等效 audio baseband 4-FSK tone 序列。
- 将 audio 输出接入现有接收/解码链路。
- 用实验室 receiver/decoder 确认 callsign、locator、power 可解出。

完成标准：

- 记录一次成功 decode 的截图或串口/终端输出。
- 在报告中说明 audio loopback 只是中间验证，不是最终 RF 方案。

### A4. RF loopback 硬件修改

目标：将 Si5351 `CLK2` RF 输出硬连线接回 receiver RF input，不经过天线空中传播。

建议电路：

```text
Si5351 CLK2
    |
    |-- DC blocking capacitor
    |
  resistive attenuator / level-setting network
    |
    |-- optional test point
    |
receiver RF input before existing bandpass/mixer chain
```

关键设计点：

- 不应直接把过大的 `CLK2` 方波灌入 receiver input。
- 通过电容隔直和电阻衰减控制接收机输入电平。
- 尽量保持短接线、公共地可靠、避免 RF pickup。
- 报告需要包含：
  - 修改电路图。
  - 修改后 PCB 照片。
  - 衰减网络计算。
  - 实测 `CLK2` 输出幅度、receiver input 幅度、decode 结果。

完成标准：

- 教师 decoder board 或本组 WSPR receiver 能从 RF loopback 解码。
- 测量记录包含仪器型号/探头设置/测量点。

### A5. Part A 测量清单

| 测量项 | 测量点 | 期望记录 |
|---|---|---|
| `CLK2` RF 输出 | Si5351 `CLK2` 或飞线起点 | 频率、幅度、波形 |
| 衰减后 RF | receiver input | 幅度，确认不会过载 |
| Tone spacing | `CLK2` 或 receiver output | 约 `1.4648 Hz` 间隔，或解释仪器分辨率限制 |
| Decode 结果 | WSPR receiver/decoder | callsign、locator、power、SNR/time offset/frequency offset |
| 电源电流 | 板级输入 | RX/TX 状态对比 |

## 5. Part B 任务：Transceiver PCB 设计

### B1. 建立 KiCad 修改分支

任务：

- 复制原始工程，避免破坏 baseline。
- 建议新文件命名：
  - `wspr_transceiver.kicad_pro`
  - `wspr_transceiver.kicad_sch`
  - `wspr_transceiver.kicad_pcb`
- 保持 PCB 外形尺寸与原始 `bbb.kicad_pcb` 一致。
- 保留原接收机必要链路，只修改收发相关电路。

完成标准：

- KiCad ERC/DRC 主要错误已处理或在报告中解释。
- 原始板框尺寸未改变。

### B2. Part B RF 信号路径定义

建议完整信号路径：

```text
TX mode:
Si5351 CLK2
 -> output shaping / attenuator
 -> driver / PA
 -> low-pass or harmonic filtering if required
 -> GPIO-controlled T/R switch
 -> shared bandpass filter
 -> 50 ohm RF output / loopback/test connector

RX mode:
50 ohm RF input
 -> GPIO-controlled T/R switch
 -> shared bandpass filter
 -> existing receiver mixer / audio chain
```

报告必须说明：

- GPIO 为 RX 时，PA 输出不能反灌 receiver。
- GPIO 为 TX 时，receiver input 被隔离或保护。
- Bandpass filter 被 TX 和 RX 共用，符合题目要求。
- 50 ohm 系统如何匹配。

### B3. 0.1 W into 50 ohms 设计计算

设计目标：

- 输出功率：`Pout = 0.1 W = 20 dBm`
- 负载：`R = 50 ohms`
- 正弦波等效：
  - `Vrms = sqrt(P * R) = sqrt(0.1 * 50) = 2.236 V`
  - `Vpk = 3.162 V`
  - `Vpp = 6.324 V`
  - `Irms = Vrms / R = 44.7 mA`

任务：

- 选择 PA 架构：小信号 driver + RF transistor/MOSFET，或可买到的 RF gain block/PA 器件。
- 计算所需 gain：由 Si5351 `CLK2` 可用输出电平推算到 `20 dBm`。
- 设计输出匹配网络和 bias。
- 检查功耗和热耗散。
- 检查 harmonic suppression，必要时加入 LPF 或利用共享 BPF 后的选择性。

完成标准：

- 报告给出 PA 电路图、关键元件值、偏置点、预期 gain、预期输出功率。
- BOM 中新增器件来自 Element14 Australia，并记录订购编号、封装、库存检查日期。

### B4. GPIO 控制收发切换

可选方案：

1. RF switch IC：GPIO 直接/经电平转换控制，优点是隔离度和插损明确。
2. 二极管/模拟开关方案：成本低，但需要认真计算 RF 隔离、插损和偏置。
3. 小继电器：隔离好，但体积、电流、切换速度和 PCB 尺寸可能不利。

任务：

- 确定 GPIO 电压标准和可用 pin。
- 设计默认安全状态：上电默认 RX 或 TX disabled。
- 增加 PA enable，避免 RX 时 PA 自激或泄漏。
- 在原理图中清楚标注 `TX_EN` 或 `TR_CTRL`。

完成标准：

- 原理图中可从 GPIO 追踪到 RF switch/PA enable。
- 报告给出 TX/RX 两种状态的开关真值表。

示例真值表：

| GPIO `TR_CTRL` | 模式 | PA | RF path |
|---|---|---|---|
| `0` | RX | Off | RF input -> BPF -> receiver |
| `1` | TX | On | PA -> BPF -> RF output |

### B5. PCB layout 修改任务

任务：

- 保持原 PCB 大小和安装孔。
- PA 到 RF connector 的路径短、宽度接近 50 ohm 微带线要求。
- PA 输出和 receiver input 保持物理隔离。
- RF switch 靠近 BPF 或 RF connector，减少 stub。
- PA 电源就近放置 decoupling capacitors。
- 高电流/高频回流路径下方保持连续地。
- 增加测试点：
  - `TP_CLK2`
  - `TP_PA_IN`
  - `TP_PA_OUT`
  - `TP_TR_CTRL`
  - `TP_RX_IN`

完成标准：

- KiCad DRC 通过，或每个剩余 issue 都有理由。
- 关键 RF 网络可在 PCB 上清楚追踪。
- 生成 Gerber/Drill/BOM/position files 供提交或制造参考。

### B6. Element14 器件选择任务

需要记录每个新增器件：

- Manufacturer part number
- Element14 order code
- Package/footprint
- Datasheet URL
- 关键参数：频率范围、gain、P1dB/output power、supply voltage、insertion loss/isolation/current rating
- 查询日期

候选类别：

- RF amplifier / transistor / MOSFET
- RF switch 或 PIN diode
- Matching inductors/capacitors
- Attenuator resistors
- RF connector / test point
- Power decoupling capacitors

注意：库存和价格会变化，最终报告中要写明查询日期。

## 6. 报告结构建议（IEEE Conference Format，最多 4 页）

建议页数分配：

| 部分 | 建议长度 | 内容 |
|---|---:|---|
| Title/Abstract | 0.25 页 | 一句话说明实现了 RF loopback transmitter 和 transceiver PCB redesign。 |
| I. Introduction | 0.25 页 | 项目目标、频段、WSPR-SDR 到 transceiver 的变化。 |
| II. Part A Transmitter Implementation | 1.25 页 | WSPR 编码、`CLK2` 发射、CLI、RF loopback、衰减计算、软件/硬件框图。 |
| III. Part A Measurements | 0.5 页 | 波形、频率、电平、decode 结果、照片。 |
| IV. Part B Transceiver Hardware | 1.25 页 | PA、0.1 W 计算、T/R switch、共享 BPF、GPIO 真值表、完整信号路径。 |
| V. PCB Layout and Expected Performance | 0.5 页 | PCB 截图、RF layout 决策、预期功率/隔离/风险。 |
| References | 0.25 页 | WSPR、Si5351、器件 datasheet、Resources 中论文。 |

报告中必须放在正文的证据：

- Part A 修改电路图。
- Part A 修改后 PCB 照片。
- `wspr_transmit` 核心 C 代码。
- Part A 测量数据和 decode 结果。
- Part B 完整 schematic。
- Part B PA 和 50 ohm 功率计算。
- Part B PCB layout 截图。
- 新增器件 BOM。

## 7. 最终提交检查表

- [ ] `wspr_transmit` C 源码和 Makefile 已包含在 zip 中。
- [ ] `./wspr_transmit VK2AAX QF56 3` 已测试。
- [ ] RF loopback 修改电路图已画出。
- [ ] 修改后 PCB 照片已放入报告。
- [ ] 至少一次 WSPR decode 成功结果已记录。
- [ ] Part B KiCad schematic 已更新。
- [ ] Part B PCB layout 已更新且板框尺寸不变。
- [ ] PA 设计目标达到 `0.1 W into 50 ohms`，计算写入报告。
- [ ] T/R switch 由 GPIO 控制，真值表写入报告。
- [ ] 新增器件均来自 Element14 Australia，BOM 包含 order code。
- [ ] ERC/DRC 已运行，剩余问题已解释。
- [ ] IEEE conference format 报告不超过 4 页。
- [ ] zip 中包含 report PDF、KiCad files、source code、必要图片/测量数据。

## 8. 风险与应对

| 风险 | 影响 | 应对 |
|---|---|---|
| WSPR 编码实现错误 | decoder 无法解码 | 先做 dry-run 和 audio loopback，再做 RF loopback。 |
| Si5351 频率更新时间抖动 | tone 不稳定，SNR 下降 | 预先计算 tone frequencies，symbol 边界只做必要 I2C 更新。 |
| RF loopback 电平过高 | receiver overload 或损坏 | 加电容隔直和可计算衰减网络，先从大衰减开始测试。 |
| PA harmonic 太高 | 输出不干净，decoder 测试失败 | 使用共享 BPF，必要时在 PA 后加 LPF。 |
| T/R switch 隔离不足 | TX 泄漏进 RX 或 RX 灵敏度下降 | 选择隔离度足够的 RF switch，layout 缩短 stub。 |
| PCB 尺寸限制 | 新增 PA/switch 放不下 | 优先使用小封装器件，并移除不必要测试结构。 |
| Element14 缺货 | BOM 不可采购 | 尽早锁定 2 个可替代器件并记录查询日期。 |

## 9. 建议时间线

| 时间 | 软件 | 硬件/PCB | 报告 |
|---|---|---|---|
| Day 1 | 完成 WSPR 编码资料阅读和代码框架 | 找到 `CLK2`、receiver input、可用 GPIO | 建立报告模板 |
| Day 2 | 完成 symbol 生成和 dry-run | 设计 Part A 衰减/loopback 电路 | 画 Part A 框图 |
| Day 3 | 接入 Si5351 `CLK2` 控制 | 焊接 Part A 修改并拍照 | 写 Part A 方法 |
| Day 4 | audio/RF loopback debug | 完成 Part B PA 和 T/R switch 原理图 | 加入测量表 |
| Day 5 | 整理代码和运行说明 | 完成 PCB layout 和 ERC/DRC | 写 Part B 设计 |
| Day 6 | 最终复测 | 生成 Gerber/BOM/截图 | 压缩到 4 页并交叉检查 |

