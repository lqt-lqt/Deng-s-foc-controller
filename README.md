# DengFOC 双路无刷电机 FOC 控制器 — 硬件学习与复现

> **Forked from [ToanTech/Deng-s-foc-controller](https://github.com/ToanTech/Deng-s-foc-controller)** | 原项目作者：灯哥 | 许可证：GPL-3.0
>
> 本仓库是我深入学习 DengFOC V3.0 硬件设计的完整记录，包含原理图分析、关键器件选型解读、外设配置思路，以及基于 STM32F405 平台的固件复现方案。

---

## 我为什么选择 DengFOC 来深入学习

在对比研究了 10 个开源 FOC 驱动项目（SimpleFOC Shield、miniFOC、moco、Glitch752、VESC、MESC、ODrive 等）之后，DengFOC V3 在以下方面最适合作为进阶学习范本：

| 维度 | DengFOC V3 的优势 |
|------|------------------|
| **双电机架构** | 单板同时驱动两个 BLDC，功率级复制是理解 FOC 硬件最直接的方式 |
| **中文文档** | 详尽的使用手册 + 25 个测试例程，降低入门门槛 |
| **社区验证** | 数千人复刻成功，设计可靠性经过充分验证 |
| **立创EDA** | 国内工具设计，可直接导入修改和打样 |
| **编码器兼容** | 同时支持 I2C/SPI/ABI/PWM/霍尔，覆盖主流磁编码器 |

---

## 硬件架构分析

### 系统框图

```
┌─────────────────────────────────────────────────────────┐
│                      电源系统                            │
│   12-24V DC → Buck(LDO) → 5V/3.3V                      │
│   母线电容 + 去耦网络                                    │
├──────────┬──────────────────────┬───────────────────────┤
│  MOTOR A │   DRV8301 (3相驱动)  │  MOTOR B             │
│  ├─3半桥 │   + 电流检测运放     │  ├─3半桥             │
│  ├─电流采样│  + 保护电路        │  ├─电流采样           │
│  └─编码器 │                      │  └─编码器            │
├──────────┴──────────────────────┴───────────────────────┤
│              STM32F405 / ESP32 主控                      │
│   ├─TIM1: 6ch 互补PWM (MOTOR A)                         │
│   ├─TIM8: 6ch 互补PWM (MOTOR B)                         │
│   ├─ADC1/2: 双路注入组同步电流采样                       │
│   ├─I2C1/SPI2: 编码器接口                               │
│   ├─CAN: 多轴通信 (SN65HVD232)                          │
│   └─UART: 调试/上位机                                    │
└─────────────────────────────────────────────────────────┘
```

### 关键器件选型分析

#### 1. 栅极驱动 — DRV8301

| 特性 | 参数 | 学习要点 |
|------|------|---------|
| 驱动方式 | 三相独立半桥驱动 | 内置电荷泵，无需自举电容 |
| 栅极电流 | 1.7A 拉/2.3A 灌 | 驱动能力直接影响 MOSFET 开关速度 |
| 保护功能 | 过流/过温/欠压 | 硬件保护 vs 软件保护的分工 |
| 电流运放 | 内置 2 路可调增益运放 | 省去外部运放，简化 BOM |

**设计启示**：V3.0 版本从分立栅极驱动升级到 DRV8301，是在"集成度 vs 灵活性"之间的合理折中。对比 ODrive 也用了 DRV8301，验证了这颗料在 FOC 驱动中的标杆地位。

#### 2. 电流采样方案

```
分流电阻 (低侧) → DRV8301 内置运放 → ADC 注入组 → FOC 算法
      ↓                                  ↓
   0.5mΩ ~ 10mΩ                     双路同步采样
```

**关键设计点**：
- **低侧采样 vs 高侧采样**：低侧成本低但无法检测对地短路，V3.0 选择低侧是合理的成本优化
- **注入组同步采样**：利用 STM32 的 ADC 注入组在 PWM 中点触发，避免开关噪声
- **运放增益选择**：需在"电流分辨率"和"量程"之间平衡，V3.0 设计中增益可调

#### 3. 编码器接口兼容性

| 编码器 | 接口 | 分辨率 | 适用场景 |
|--------|------|--------|---------|
| AS5600 | I2C | 12bit | 低成本位置控制 |
| AS5047P | SPI | 14bit | 高精度速度控制 |
| ABI 增量式 | GPIO 定时器编码器模式 | — | 通用兼容 |
| 霍尔传感器 | GPIO 中断 | 6步 | 低成本速度检测 |

**设计启示**：V3.0 同时引出 I2C、SPI、ABI 接口，牺牲了少量 PCB 面积换取了编码器兼容性，是面向实验和学习场景的务实设计。

---

## 从 ESP32 Arduino 到 STM32F405 HAL 的移植思路

原项目基于 ESP32 + Arduino + SimpleFOC 库。在深入理解 FOC 算法后，我选择用 STM32F405 + CubeMX + HAL 重新实现固件，原因如下：

| 对比维度 | ESP32 Arduino 原方案 | STM32F405 HAL 方案 |
|---------|---------------------|-------------------|
| 实时性 | FreeRTOS，非确定性延迟 | 裸机/HAL，定时器精确触发 |
| PWM 精度 | LEDC 外设 | HRTIM / 高级定时器 168MHz |
| ADC 同步 | 单路轮询 | 双路注入组同步采样 |
| 死区插入 | 软件配置 | 硬件自动，精度更高 |
| 调试工具 | 串口打印 | J-Link SWD + 逻辑分析仪 |
| 行业认可度 | 创客/教育 | 工业/产品级 |

### 外设配置要点（CubeMX）

```
TIM1_CH1/CH1N/CH2/CH2N/CH3/CH3N → Motor A 三路互补 PWM
TIM8_CH1/CH1N/CH2/CH2N/CH3/CH3N → Motor B 三路互补 PWM
  - 死区时间: 100-200ns（根据 MOSFET 开关特性调整）
  - PWM 频率: 20kHz（超声频，避免人耳可闻啸叫）
  - 中心对齐模式: 适合 SVPWM 调制

ADC1_INJ + ADC2_INJ → 双路电流注入组同步采样
  - 触发源: TIM1/TIM8 的 TRGO（PWM 中点触发）
  - 采样时间: 3-7 个 ADC 周期（匹配运放输出阻抗）

I2C1 → AS5600 磁编码器
SPI2 → AS5047P 磁编码器
CAN1 → SN65HVD232 → 多轴通信
```

---

## 学习进度记录

- [x] 下载并分析 DengFOC V3.0 原理图（立创EDA）
- [x] 研读使用文档（含25个测试例程说明）
- [x] 对比 V1.0 → V2.0 → V3.0 三版硬件迭代
- [x] 对比研究 10 个开源 FOC 项目的架构差异
- [x] 分析 DRV8301 驱动芯片的选型逻辑
- [x] 设计 STM32F405 移植方案（CubeMX 外设配置）
- [ ] 固件实现：电流环 PI + SVPWM
- [ ] 固件实现：位置/速度/电流三环级联 PID
- [ ] 自制 FOC 驱动板设计与打样

---

## 相关资源

- **原项目仓库**: [ToanTech/Deng-s-foc-controller](https://github.com/ToanTech/Deng-s-foc-controller)
- **DengFOC 配套库**: [ToanTech/DengFOC_Lib](https://github.com/ToanTech/DengFOC_Lib)
- **SimpleFOC 文档**: [docs.simplefoc.com](https://docs.simplefoc.com/)
- **FOC 算法原理课**: [B站 灯哥开源](https://space.bilibili.com/3946555)

---

## 其他 FOC 开源项目学习笔记

| 项目 | 核心架构 | 学习重点 |
|------|---------|---------|
| [SimpleFOC Shield v3](https://github.com/simplefoc/Arduino-SimpleFOCShield) | DRV8313 集成驱动 | FOC 入门首选，理解三半桥集成方案 |
| [miniFOC](https://github.com/ZhuYanzhen1/miniFOC) | GD32F130 + 分立 MOSFET | 低成本小体积设计，Robomaster 实战验证 |
| [moco](https://github.com/ziteh/moco) | STSPIN32G4 / DRV8301 | 对比"单芯片"和"分立"两种架构 |
| [Glitch752 FOC](https://github.com/Glitch752/focMotorController) | RP2040 + IR2104 + INA240 | 经典分立栅极驱动 + 精密电流采样 |
| [VESC](https://github.com/vedderb/bldc-hardware) | STM32F405 + DRV8302 | 开源 FOC 硬件的事实标准 |
| [MESC](https://github.com/davidmolony/MESC_FOC_ESC) | STM32F303 + 分立驱动 | 极致成本优化（BOM $23） |
| [ODrive v3](https://github.com/odriverobotics/ODriveHardware) | STM32F405 + DRV8301 + INA240 | 科研级精密设计标杆 |

---

*本仓库持续更新中，欢迎 Star 和 PR。*
