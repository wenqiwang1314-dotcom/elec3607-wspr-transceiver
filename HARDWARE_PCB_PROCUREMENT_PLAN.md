# WSPR Transceiver 硬件 / PCB / 采购优先规划

起始日期：2026-04-28  
目标：优先解决采购周期长、PCB 返工慢、硬件风险高的问题。软件可以并行推进，但不能阻塞器件采购和 PCB 方案冻结。

## 1. 核心结论

硬件任务的关键路径是：

```text
确定 TX/RX RF 架构
 -> 选 PA 与 T/R switch
 -> Element14 下单主器件和备选器件
 -> 原理图冻结
 -> footprint/封装核对
 -> PCB layout 冻结
 -> ERC/DRC + Gerber/BOM 输出
 -> 板级 bring-up 测试
```

必须先采购的不是所有小阻容，而是这些长周期/易缺货/封装影响 PCB 的器件：

1. PA final transistor 或 RF gain block。
2. T/R switch：RF switch IC 或 PIN diode switch 所需二极管。
3. RF matching/filter 电感电容，尤其是 RF-grade C0G/NP0 和高 Q 电感。
4. SMA/RF connector、test point、attenuator/matching 电阻。
5. PA 电源去耦、电源开关/enable 相关器件。

## 2. 当前原板可复用信息

从现有工程可见的关键网络：

| 原板网名 | 用途 | 新设计动作 |
|---|---|---|
| `CLK2` | Si5351 剩余输出，用作 TX RF source | 接入 TX driver / PA input。 |
| `BPFIN` | bandpass filter input | 改成 T/R switch 后的共享 BPF 输入。 |
| `BPFOUT` | bandpass filter output | RX 时接 mixer，TX 时视滤波器方向和拓扑决定是否反向使用。 |
| `RFIN` | receiver RF input path | RX path 保留，TX 时隔离/保护。 |
| `RFOUT` | RF amplifier/mixer 相关输出网 | 核对原接收机链路，不要误接为 TX power output。 |
| `U2 Si5351A-B-GT` | 频率源 | `CLK2` 需要新增 TX path footprint 和 test point。 |
| `U6 SMA` | 原 RF 接口 | 尽量复用为收发共用 50 ohm RF port。 |

硬件规划应先在 KiCad 中把这些网名重命名/标注清楚，例如新增：

- `TX_RF_RAW`
- `TX_PA_IN`
- `TX_PA_OUT`
- `TR_CTRL`
- `PA_EN`
- `ANT_50R`
- `BPF_SHARED_IN`
- `BPF_SHARED_OUT`

## 3. 架构冻结：两套方案并行保底

### 方案 A：分立 PA + PIN diode T/R switch（推荐优先推进）

理由：

- 低频 HF/WSPR 更容易用分立器件实现 0.1 W。
- PIN diode 可覆盖作业可能的 HF 频率，器件便宜，封装好焊。
- 即使 switch 性能一般，也容易调试和解释。

信号路径：

```text
TX:
Si5351 CLK2
 -> attenuator / square-wave shaping
 -> driver transistor
 -> class-C / class-E-ish PA final
 -> low-pass / harmonic cleanup if needed
 -> PIN diode T/R switch
 -> shared BPF
 -> SMA 50 ohm

RX:
SMA 50 ohm
 -> PIN diode T/R switch
 -> shared BPF
 -> existing RX amplifier/mixer path
```

必须设计：

- PA 输出目标：`0.1 W = 20 dBm into 50 ohms`。
- `Vrms = sqrt(0.1 * 50) = 2.236 V`
- `Vpp = 2 * sqrt(2) * 2.236 = 6.32 Vpp`
- `Irms = 2.236 / 50 = 44.7 mA`
- PA 要留 3 dB 余量，设计目标建议为 `23 dBm`，实际通过衰减或偏置调到 20 dBm。

采购优先件：

- 2N7000 / BS170 类 MOSFET 先买一批用于原型验证。
- RF BJT 小信号 driver 先买 5-10 个。
- PIN diode 至少买两种：低电阻优先、封装易焊优先。
- RF 电感电容按计算值上下各买一档。

### 方案 B：RF gain block + switch IC（作为备选）

理由：

- 原理图更整洁，50 ohm 接口明确。
- 缺点是价格高、频率下限/输出功率可能不完全满足。

注意：

- 搜到的 HMC741ST89ETR 是 50 MHz 到 3 GHz、输出约 +18.5 dBm 级别，单颗离 20 dBm 目标略低，而且若项目频率在 HF，需要确认 50 MHz 下限是否不可接受。
- 很多现成 RF switch IC 下限从 100 MHz、1 GHz 或更高开始，不适合 HF WSPR，不能只看“SPDT RF switch”就下单。
- 如果选 switch IC，必须确认 minimum frequency、power handling、insertion loss、isolation、control voltage 和 footprint。

## 4. Element14 初筛器件方向

下表是采购前筛选方向，不是最终 BOM。下单前必须重新打开 Element14 页面确认库存、交期、价格和是否停产。

| 类别 | 候选方向 | 初筛观察 | 风险 |
|---|---|---|---|
| PA final | 2N7000 / BS170 类小 MOSFET | Element14 上 2N7000 有大量库存记录，便宜且易原型 | 不是专用 RF PA，HF 输出功率和效率要实测。 |
| RF driver | BFR92/BFR106 类 RF BJT | Element14 有 RF transistor 分类，SOT-23 易布局 | 需要偏置设计，不能直接当 100 mW final。 |
| PIN diode | BAR50 / BAP65 / HSMP 类 | PIN diode 适合 RF switch/attenuator，部分型号覆盖 10 MHz 以上 | 某些型号可能 no longer manufactured，要买替代件。 |
| RF switch IC | BGSA11 / F2912 / ADRF 系列 | 部分器件为 50 ohm switch，控制简单 | 频率下限、价格、封装和焊接难度可能不适合本作业。 |
| RF gain block | HMC741ST89E 类 | 50 ohm gain block、SOT-89、资料完整 | 频率下限 50 MHz，输出功率略低于 20 dBm 目标。 |
| L/C matching | C0G/NP0 RF capacitors，高 Q inductors | 影响 PA matching 和滤波器性能 | 普通 X7R 不适合频率/稳定性关键位置。 |

参考链接：

- Element14 2N7000: https://au.element14.com/on-semiconductor/2n7000/mosfet-n-channel-200ma-60v-to/dp/9845178
- Element14 BAR50-02V: https://au.element14.com/infineon/bar50-02v/diode-rf-pin-sc-79/dp/1095645
- Element14 HMC741ST89ETR: https://au.element14.com/analog-devices/hmc741st89etr/rf-amplifier-50mhz-to-3ghz-sot/dp/4419988
- Element14 RF / PIN diode category: https://au.element14.com/c/semiconductors-discretes/diodes-rectifiers/rf-pin-diodes

## 5. 采购策略：今天就能下的单

### 第一单：风险样品单（今天或最晚明天）

目的：不等最终 PCB，先买会影响架构的器件。

建议数量：

| 器件类别 | 数量 | 原因 |
|---|---:|---|
| PA final MOSFET/BJT 候选 | 每种 5-10 | 焊坏、调偏置、做 dead-bug 原型都需要余量。 |
| RF driver transistor | 5-10 | 驱动级可换型。 |
| PIN diode 候选 | 每种 10-20 | switch 可能要用 2-4 颗，且要做多种拓扑。 |
| RF C0G 电容 | 每值 10-20 | matching/filter 调试要换值。 |
| RF 电感 | 每值 5-10 | PA 输出网络和 LPF 调试。 |
| 50 ohm/attenuator 电阻 | 每值 20 | Pi/T attenuator 与终端匹配。 |
| SMA/test point | 2-5 | PCB 与飞线测试都需要。 |

第一单原则：

- 买主方案和备选方案，不要只买一个“看起来最对”的器件。
- 对 switch IC 这类贵/难焊器件，除非频率下限和封装已经确认，否则先不要压预算。
- 所有新增封装优先选 KiCad 标准库已有 footprint：SOT-23、SOT-223、SOT-89、0805、0603、SMA edge/through-hole。

### 第二单：PCB BOM 冻结单

触发条件：

- PA topology 已在面包板/dead-bug 或小板上能接近 20 dBm。
- T/R switch 在 RX/TX 两态下能连通/隔离。
- KiCad footprint 与 datasheet land pattern 已核对。

第二单内容：

- 最终型号主件。
- 最终 matching/filter 值。
- 备用器件 20%-50%。
- PCB assembly 需要的 reel/cut-tape 封装选项。

## 6. 日期规划（从 2026-04-28 开始）

| 日期 | 硬件/采购目标 | 输出物 | 是否阻塞 PCB |
|---|---|---|---|
| Apr 28 | 冻结 TX/RX 总体架构，列出 PA 与 switch 两套候选 | `hardware_decision_log.md` 初版，采购 basket 初版 | 是 |
| Apr 29 | 完成 Element14 第一单，下单主件+备选+RF L/C | 订单截图/BOM 记录，datasheet 链接 | 是 |
| Apr 30 | 画 Part B 原理图草案：PA、T/R switch、shared BPF 接法 | KiCad schematic draft | 是 |
| May 1 | footprint 核对，确定板上可放位置，检查原板尺寸 | footprint checklist，placement sketch | 是 |
| May 2 | PCB layout 第 1 版：只追关键 RF path 和电源路径 | `wspr_transceiver.kicad_pcb` v1 | 否 |
| May 3 | ERC/DRC 第 1 轮，生成 BOM 草案 | DRC log，BOM v1 | 否 |
| May 4 | 器件到货后做 PA/switch 台架验证；若未到货，用仿真/计算补齐 | PA/switch measurement plan | 否 |
| May 5 | 根据实测或 datasheet 修正匹配、偏置、滤波器 | schematic v2，BOM v2 | 是 |
| May 6 | PCB layout 冻结，导出 Gerber/Drill/position | fabrication package | 是 |
| May 7+ | Bring-up/报告证据收集 | 波形、功率、切换真值表、照片 | 否 |

如果最终提交日期早于这个节奏，需要把 May 4 以后的“实测”改成“prototype test + expected performance”，但采购和 PCB 冻结仍然要在前两天完成。

## 7. 每日硬件检查表

### Day 0 / Apr 28

- [ ] 在 KiCad 中确认 `CLK2` 到新增 TX path 的可布线路径。
- [ ] 确认 SMA 到 BPF 的原始路径，决定 BPF 是否可双向共享。
- [ ] 决定 `TR_CTRL` GPIO 使用哪个 BeagleBone/board header pin。
- [ ] 建立候选 BOM 表，至少每个关键类别 2 个候选。
- [ ] 采购负责人把 Element14 basket 分享给全组复核。

### Day 1 / Apr 29

- [ ] 下第一单。
- [ ] 下载每个新增器件 datasheet，放到 `Resources/datasheets/`。
- [ ] 在 BOM 表记录 order code、数量、价格、库存、预计到货日期、查询日期。
- [ ] KiCad 新增 symbol/footprint 前先确认封装尺寸。

### Day 2 / Apr 30

- [ ] 画 PA 偏置、电源去耦、input attenuator。
- [ ] 画 T/R switch，包含 TX/RX 真值表。
- [ ] 明确 RX 保护：TX 时 receiver input 不可直接承受 PA 输出。
- [ ] 明确 TX harmonic cleanup：BPF 是否足够，不足则加 LPF。

### Day 3 / May 1

- [ ] 把关键器件放入 PCB：PA 靠近 BPF/switch，Si5351 `CLK2` 到 PA input 短。
- [ ] PA 电源走线加粗，就近放 `100 nF + 1 uF/10 uF`。
- [ ] 新增 test points：`TP_CLK2_TX`, `TP_PA_IN`, `TP_PA_OUT`, `TP_TR_CTRL`, `TP_ANT_50R`。
- [ ] 检查地回流：PA、switch、BPF 下方尽量连续地。

### Day 4 / May 2

- [ ] 完成 RF path layout。
- [ ] 检查 board outline 与原 `bbb.kicad_pcb` 一致。
- [ ] 关键 RF 线尽量短、少过孔、少 stub。
- [ ] 输出 Gerber 预览截图给报告使用。

## 8. PCB 设计冻结标准

### 原理图冻结必须满足

- [ ] `CLK2 -> PA -> T/R switch -> shared BPF -> SMA` TX path 可追踪。
- [ ] `SMA -> T/R switch -> shared BPF -> RX` RX path 可追踪。
- [ ] `TR_CTRL` GPIO 控制 switch。
- [ ] `PA_EN` 或同等机制保证 RX 时 PA 关闭。
- [ ] 所有 RF port 有 DC blocking 或明确 DC bias 处理。
- [ ] PA 输出功率计算写在原理图 note 或报告草稿中。

### PCB 冻结必须满足

- [ ] 板框尺寸不变。
- [ ] 新增器件 footprint 与 datasheet land pattern 对照完成。
- [ ] PA 输出到 BPF/switch/SMA 的路径最短。
- [ ] RX input 和 PA output 不平行长距离靠近。
- [ ] 每个新增 IC/PA 管有近端 decoupling。
- [ ] DRC 通过，或剩余 DRC issue 有说明。
- [ ] 生成 Gerber/Drill/BOM/position files。

## 9. Bring-up 测试顺序

不能第一次上电就直接发 0.1 W。顺序如下：

1. 不焊 PA final，检查 3.3 V/5 V 电源和 `TR_CTRL`。
2. 焊 switch，低功率信号源测 RX path 插损和 TX path 连通。
3. 焊 PA driver，不焊 final 或限流供电，测 `CLK2` 到 `PA_IN`。
4. 焊 PA final，先接 20 dB attenuator + 50 ohm dummy load。
5. 从低 duty / 短时间 TX 开始，测电流和输出波形。
6. 达到接近 20 dBm 后再接共享 BPF，检查谐波/输出电平。
7. 切 RX 模式，确认 PA 关闭且 receiver 没有被强信号损坏。

报告证据：

- TX output into 50 ohm 的电压或功率计算。
- `TR_CTRL=0/1` 的状态表。
- PA 输出和 BPF 后输出波形。
- PCB layout 截图。
- 新增 BOM 表和 Element14 order code。

## 10. 风险优先级

| 优先级 | 风险 | 立即动作 |
|---|---|---|
| P0 | 买不到合适 T/R switch | 同时采购 PIN diode 和至少一个 switch IC 备选；方案 A 不依赖高价 IC。 |
| P0 | PA 达不到 20 dBm | PA 设计留 3 dB 余量；买 driver/final 多种候选；预留可调 matching。 |
| P1 | BPF 双向共享后插损/匹配变差 | PCB 上预留 0 ohm / DNP 位置，可改接 BPF 方向或旁路测试。 |
| P1 | TX 泄漏损坏 RX | RX path 加隔离和保护，bring-up 使用 attenuator/dummy load。 |
| P1 | 封装太难焊 | 优先选 SOT-23/SOT-89/0805，LGA/QFN 只作备选。 |
| P2 | PCB 空间不够 | 先放 PA/switch/BPF/SMA，其他测试点可删减。 |

## 11. 给团队的执行规则

- 硬件负责人每天更新一次 BOM 和 decision log。
- 采购下单不等软件完成。
- 每个关键器件至少有一个备选。
- 下单前必须由另一名组员复核 datasheet 的 frequency、power、voltage、footprint。
- 任何“看起来可以”的 RF switch，必须先查 minimum operating frequency。
- 报告里只写最终采用方案，但保留 decision log 供 oral discussion 解释为什么没选其他方案。

