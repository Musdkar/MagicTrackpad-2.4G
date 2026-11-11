# MagicTrackpad-2.4G

> 为 Apple USB-C 妙控板制作一个**不使用蓝牙的专用 2.4GHz 无线 USB 桥接器**。

[架构设计](docs/ARCHITECTURE.md) · [无线协议](docs/RADIO_PROTOCOL.md) · [验证计划](docs/VALIDATION.md)

## 项目简介

MagicTrackpad-2.4G 希望把 Apple USB-C Magic Trackpad 的 USB 数据，通过独立的 2.4GHz 无线链路传送到电脑。妙控板通过 USB-C 连接发射端；电脑只连接一个 USB 接收器。接收器向电脑呈现一个尽可能兼容原生 Apple Magic Trackpad 的 USB HID 设备。

**本项目不是蓝牙转 USB，也不是蓝牙转 2.4GHz。** 妙控板与发射端之间使用 USB 有线连接；发射端与接收端之间使用非蓝牙 2.4GHz 无线协议；接收端与电脑之间使用 USB。

项目首先面向 macOS，尤其是 AMD 黑苹果环境。核心目标是绕过主机蓝牙控制器及其驱动，同时尽可能保留原生触控板的多点触控、点击和手势体验。

> **当前状态：设计与协议验证阶段。** 仓库目前以技术方案为主，尚无可用固件、PCB 或实测性能数据。文档中的性能数字均为工程目标，不代表已经实现。

## 设计目标

- [x] 明确非蓝牙 2.4GHz USB 桥接的总体方向
- [x] 建立初版系统架构、无线数据包和测试计划
- [ ] 读取真实 USB-C 妙控板的 USB 描述符与原始报告
- [ ] 在发射端完成 USB Host 读取与多点触控模式初始化
- [ ] 在接收端模拟 USB HID 设备并通过 macOS 原生驱动验证
- [ ] 完成 2.4GHz 无线链路
- [ ] 完成端到端延迟、丢包、抗干扰和长时间稳定性测试
- [ ] 设计低成本 PCB 与外壳

### 必须满足的产品要求

1. **不使用 Bluetooth / BLE。** 无线输入链路使用独立的 2.4GHz 非蓝牙协议。
2. **优先保留原始 HID 数据。** 不把触控板先转换成普通鼠标的 X/Y/滚轮事件。
3. **尽可能让 macOS 识别为原生 Apple Magic Trackpad。** 需要复现必要的 USB Device Descriptor、Configuration Descriptor、HID Report Descriptor、端点和控制请求。
4. **保留多点触控信息。** 包括触点数量、坐标、压力/尺寸、点击状态和设备报告中的其他字段。
5. **低延迟、低抖动。** 不允许无线队列积压大量过期触控数据。
6. **安全供电。** 发射端必须正确处理 USB-C Host 角色、VBUS 供电、过流保护和反向电流问题。

## 系统架构

### 第一版原型：双 ESP32-S3 + ESP-NOW

第一版优先选择两块 ESP32-S3 开发板，减少硬件数量，先验证 USB 协议与端到端链路。

    Apple USB-C Magic Trackpad
              │ USB-C 数据与供电
              ▼
    [Transmitter：ESP32-S3]
       USB Host / HID Raw Report
              │
              │ ESP-NOW（2.4GHz，非蓝牙）
              ▼
    [Receiver：ESP32-S3]
       ESP-NOW / USB HID Device
              │ USB
              ▼
       macOS / AMD Hackintosh

ESP32-S3 的 USB OTG 外设可以工作在 Host 或 Device 模式；发射端使用 Host 模式，接收端使用 Device 模式。ESP-IDF USB Host 文档说明其 Host Library 支持 Full-Speed / Low-Speed 设备，并提供 HID Host 方案。

ESP-NOW v1.0 的单包数据上限为 250 字节，默认数据速率为 1 Mbps。Linux 内核对 Magic Trackpad 2 USB 报告的解析格式为 12 字节前缀加每个触点 9 字节，最多解析 15 个触点，即按该格式计算最大 147 字节。初版无线包头预留 12 字节后，仍可在单包中传输该尺寸的报告。实际设备报告长度必须通过真实硬件抓取确认。

### 性能升级候选：nRF52840 ESB

如果 ESP-NOW 的实测延迟、抖动或抗干扰能力达不到目标，再评估 Nordic nRF52840 的 Enhanced ShockBurst（ESB）专有 2.4GHz 链路：

- 发射端仍需要 ESP32-S3 或其他 USB Host MCU 读取妙控板。
- 发射端增加一颗 nRF52840，负责 ESB 无线发送。
- 接收端使用一颗 nRF52840，负责 ESB 接收及 USB Device 模拟。
- nRF52840 支持专有 2.4GHz 1/2 Mbps 速率，并带 USB Full-Speed Device 控制器。

**暂不把 nRF52840 作为第一版必选硬件。** 它本身不能替代发射端的 USB Host 控制器；采用它会增加一颗芯片、SPI 通信和固件复杂度。

详细取舍见 [架构设计](docs/ARCHITECTURE.md)。

## 关键 USB 兼容性问题

Linux 内核的 hid-magicmouse 驱动将 USB-C Magic Trackpad 2 识别为 Apple USB VID:PID **05AC:0324**，并在 USB 模式下使用 Feature Report 启用多点触控。驱动中对应的报告 ID 为 **0x02**，USB 多点触控报告格式按 12 字节前缀和每个触点 9 字节解析。

这些信息是重要的协议参考，但**不能替代对实际设备的抓包**。在正式冻结协议前，必须记录真实设备的：

- USB VID、PID、bcdUSB、设备速度和完整描述符
- Configuration / Interface / Endpoint Descriptor
- HID Report Descriptor
- Interrupt IN 端点的最大包长与轮询间隔
- Feature Report 的完整控制传输
- 原始 Input Report 的实际长度、频率和字段变化
- 断连、休眠、唤醒及重新连接行为

另一个硬件门槛：ESP32-S3 USB Host 方案只适用于 Full-Speed / Low-Speed 设备。必须先确认目标妙控板实际以 Full-Speed 枚举；如果它要求 High-Speed，第一版发射端就需要更换 USB Host 控制器。

## 延迟与稳定性目标

所有数字均为**待验证的工程目标**。

| 指标 | 第一版目标 | 说明 |
|---|---:|---|
| 桥接新增延迟 P95 | ≤ 3 ms | 从发射端收到 USB Input Report 到接收端报告进入 USB IN 队列 |
| 桥接新增延迟 P99 | ≤ 5 ms | 同上，不包含触控板物理采样和 macOS 图形渲染 |
| 无线报告 | 尽量单包传输 | 避免分片与重组造成额外延迟 |
| 丢包 | 长时间测试中无可感知输入丢失 | 分别统计普通移动报告与按下/抬起状态 |
| 连接恢复 | 自动恢复 | 接收器重启、无线短暂中断、USB 重新插拔 |
| 无线依赖 | 不使用蓝牙 | 不配对、不启动 BLE 服务 |

不能仅凭“2.4GHz”或标称无线速率推导真实延迟。必须测量 P50、P95、P99、最大值、丢包率和重连时间，并在 Wi-Fi 拥挤环境中重复测试。

## 开发阶段

| 阶段 | 工作内容 | 完成标准 |
|---|---|---|
| M0：设备研究 | 抓取 USB 描述符、报告描述符、Feature Report 与原始触控数据 | 形成可复现的设备协议记录 |
| M1：USB Host | ESP32-S3 读取真实妙控板 | 稳定枚举并获得原始多点触控报告 |
| M2：USB Device | 接收端模拟 Apple USB HID 设备 | macOS 能枚举；验证原生驱动与手势 |
| M3：无线链路 | ESP-NOW 双向通信与控制请求转发 | 报告、序号、状态和统计信息可靠传输 |
| M4：端到端 | USB Host → 2.4GHz → USB Device | 完成延迟、丢包、干扰和长时间测试 |
| M5：硬件化 | 原理图、PCB、电源、天线与外壳 | 形成可重复制作的硬件版本 |

**开发顺序原则：先证明 USB 协议可行，再做无线；先证明 macOS 原生识别，再做外壳和 PCB。**

## 仓库结构规划

    .
    ├── README.md
    ├── docs/
    │   ├── ARCHITECTURE.md
    │   ├── RADIO_PROTOCOL.md
    │   └── VALIDATION.md
    ├── firmware/
    │   ├── transmitter/
    │   └── receiver/
    ├── hardware/
    │   ├── transmitter/
    │   └── receiver/
    └── captures/
        ├── usb/
        └── radio/

## 安全与边界

- USB-C 发射端必须正确提供 VBUS，并实现符合 Host 角色的 CC 电阻/控制器设计。
- 需要过流保护、ESD 防护，并避免外部电源与主机 USB 电源反向灌电。
- 不在没有真实设备抓包的情况下硬编码完整 HID Report Descriptor。
- 不以普通鼠标 HID 作为最终实现；它只能作为无线链路调试的临时测试模式。
- 不宣称已经支持所有 Magic Trackpad 型号；第一目标是 USB-C 版本。
- 不将未经测试的延迟、续航或兼容性写成已实现功能。

## 参考资料

- [Espressif ESP32-S3 USB Host 文档](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/usb_host.html)
- [Espressif ESP-NOW 文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- [ESP32-S3 技术参考手册：USB OTG](https://documentation.espressif.com/esp32-s3_technical_reference_manual_en.pdf)
- [Linux Kernel：hid-magicmouse.c](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-magicmouse.c)
- [Linux Kernel：hid-ids.h](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-ids.h)
- [Nordic nRF52840 Product Specification](https://docs.nordicsemi.com/r/bundle/ps_nrf52840/page/keyfeatures_html5.html)
- [Nordic DevZone：nRF52840 ESB throughput discussion](https://devzone.nordicsemi.com/f/nordic-q-a/120889/how-to-achieve-nrf52840-esb-maximum-data-rate)

---

## English summary

MagicTrackpad-2.4G is an experimental non-Bluetooth 2.4 GHz wireless USB bridge for Apple's USB-C Magic Trackpad. The transmitter reads the trackpad over USB Host, sends raw HID reports over a dedicated 2.4 GHz link, and the receiver enumerates as a USB HID device to the computer. The first prototype targets dual ESP32-S3 boards with ESP-NOW. Native macOS Magic Trackpad recognition, full multitouch compatibility, latency, and reliability are all validation goals—not yet verified results.
