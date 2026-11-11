# 系统架构设计

本文定义 MagicTrackpad-2.4G 的硬件分工、USB 代理方式、无线链路候选与关键技术风险。当前为设计草案；真实 USB 描述符和时序以实测抓包为准。

## 1. 设计原则

1. 触控板到发射端保持 USB 有线连接。
2. 发射端与接收端之间只使用非蓝牙 2.4GHz 无线链路。
3. 接收端通过 USB 连接电脑，并尽可能呈现原生 Apple Magic Trackpad。
4. 传输原始 HID Input Report，不在发射端转换为普通鼠标事件。
5. USB 控制请求与输入报告分开设计：输入报告走低延迟数据通道，Feature Report 等控制请求走请求/响应通道。
6. 所有性能结论必须由测量支持。

## 2. 逻辑拓扑

    ┌──────────────────────────────┐
    │ Apple USB-C Magic Trackpad   │
    └──────────────┬───────────────┘
                   │ USB-C：数据 + VBUS
                   ▼
    ┌──────────────────────────────┐
    │ Transmitter                  │
    │ USB Host                     │
    │ - 枚举真实妙控板             │
    │ - 读取 HID Report Descriptor │
    │ - 读取原始 Input Report      │
    │ - 处理 Feature Report        │
    └──────────────┬───────────────┘
                   │
                   │ 非蓝牙 2.4GHz
                   ▼
    ┌──────────────────────────────┐
    │ Receiver                    │
    │ USB HID Device               │
    │ - 模拟设备描述符             │
    │ - 暴露 HID Report Descriptor │
    │ - 提供 Interrupt IN 报告     │
    │ - 处理控制请求               │
    └──────────────┬───────────────┘
                   │ USB
                   ▼
    ┌──────────────────────────────┐
    │ macOS                        │
    │ 尽可能使用系统原生驱动       │
    └──────────────────────────────┘

## 3. 第一版硬件方案：双 ESP32-S3

### 发射端

建议职责：

- ESP32-S3 USB OTG 配置为 Host。
- 使用 ESP-IDF USB Host Stack / HID Host 组件枚举妙控板。
- 获取设备描述符、配置描述符、接口描述符和 HID Report Descriptor。
- 接收原始 Interrupt IN Input Report。
- 发送必要的 HID Feature Report，例如 USB 多点触控模式初始化请求。
- 通过 ESP-NOW 把原始报告发送给接收端。
- 统计 USB 报告间隔、无线发送时间、丢包和重连次数。

### 接收端

建议职责：

- ESP32-S3 USB OTG 配置为 Device。
- 使用 TinyUSB 或 ESP-IDF USB Device 方案实现自定义 HID 设备。
- 提供经真实设备抓取确认的 USB Device / Configuration / Interface / Endpoint / HID Report Descriptor。
- 接收 ESP-NOW 数据包，将原始 Input Report 放入 USB IN 报告队列。
- 对 macOS 发来的 HID Feature Report 等控制请求，通过无线控制通道转发给发射端，并将结果返回。
- 记录无线序号、接收时间、USB 上报时间和异常计数。

### 为什么先用 ESP-NOW

- 两端都可以使用同一种 ESP32-S3 开发板，硬件与工具链简单。
- 不需要 Wi-Fi 路由器或蓝牙配对。
- ESP-NOW v1.0 单包最大 250 字节；在初步报告尺寸估算下，单个触控报告可以连同小型协议头一起传输。
- 可以快速完成链路验证，先把研发重点放在 USB 代理与 macOS 兼容性上。

限制：

- ESP-NOW 默认速率为 1 Mbps，实际吞吐与延迟还受帧开销、调度、信道拥挤和重传影响。
- ESP-NOW 与 Wi-Fi 共用 2.4GHz 频谱，不能把“非蓝牙”理解为“不会受 Wi-Fi 干扰”。
- v1.0 最大数据长度为 250 字节；协议必须在编译期检查数据包长度，禁止静默截断。
- ESP32-S3 的 USB OTG 外设在单颗芯片上只能按 Host 或 Device 角色使用。本项目用两块芯片分别承担两种角色。

## 4. USB 速度是架构门槛

ESP32-S3 USB Host 支持 Full-Speed / Low-Speed。第一版采用 ESP32-S3 前，必须通过 Linux lsusb -t、USB 分析仪或其他可靠方式确认目标妙控板的实际 USB 速度。

- 若目标设备以 Full-Speed 或 Low-Speed 枚举，可继续评估 ESP32-S3。
- 若目标设备要求 High-Speed，ESP32-S3 Host 方案不成立，需要更换支持 High-Speed Host 的 MCU / USB Host Controller。
- 不允许仅凭接口外形、USB 版本字符串或充电速度判断总线速度。

USB Host Library 的设备数量、传输类型和异步模型等限制也要在固件设计时考虑，尤其是 USB 重连、控制传输超时和任务调度。

## 5. USB 代理：透明转发而不是鼠标翻译

### 5.1 接收端的 USB 身份

目标是让 macOS 把接收器识别为 USB-C Magic Trackpad，并使用系统原生触控板驱动。Linux 内核目前将 USB-C Magic Trackpad 2 的 USB ID 定义为 Apple VID 0x05AC、PID 0x0324。

这是协议研究参考，不代表 macOS 一定只凭 VID/PID 就会加载完整功能。必须验证：

- 接收器提供的设备描述符与配置是否足够接近真实设备。
- HID Report Descriptor 是否完全匹配。
- 接口类、子类、协议、端点地址、最大包长和轮询间隔是否正确。
- macOS 是否绑定预期的 Apple 驱动。
- 系统设置中的触控板选项和多指手势是否正常。

如果仅被识别为普通鼠标，或系统只能收到相对位移数据，则不满足项目的核心目标。

### 5.2 描述符与控制请求

接收端本地提供静态 USB 描述符；运行时控制请求按需转发。优先考虑：

- HID GET_REPORT
- HID SET_REPORT
- HID GET_IDLE / SET_IDLE（如果设备实际使用）
- 必要的标准 USB 请求

Linux 内核的 hid-magicmouse 驱动对 USB-C Magic Trackpad 2 使用 Feature Report 启用多点触控。驱动源码中 USB 路径使用 Report ID 0x02，数据为 0x02、0x01 两个字节的原始缓冲区表示。具体 Setup Packet、wValue、数据阶段和返回状态必须通过 USB 抓包确认，不要仅按源码数组猜测总线字节序。

控制请求通常只发生在初始化、状态查询或配置阶段；输入报告是高频数据路径。两者应独立排队，避免控制请求阻塞连续触控报告。

### 5.3 原始输入报告

USB-C Magic Trackpad 的 Linux 驱动解析路径使用 Input Report ID 0x02。源码描述其 USB 报告由 12 字节前缀和每个触点 9 字节组成，最多解析 15 个触点。

因此当前估算的最大报告长度为：

    12 + 15 × 9 = 147 bytes

该值来自 Linux 驱动的解析上限，不是已经从本项目目标硬件测得的报告长度。实际 Report Descriptor、端点包长、报告分段方式和最大报告长度必须以抓包为准。

## 6. USB-C 供电与连接器

发射端是 USB Host，不能把妙控板直接接到一个仅支持 USB Device 的 USB-C 口。

发射端 USB-C 接口需要：

- 正确的 Host / DFP CC 配置或 Type-C Port Controller。
- 为妙控板提供受控的 5V VBUS。
- 具备合理的限流、短路保护和 ESD 防护。
- 防止外部电源与调试 USB 或其他供电路径反向灌电。
- 记录连接和拔出事件，处理设备重新枚举。

第一版优先使用成熟的 USB Host 开发板或经过验证的 Type-C Host 转接方案，不建议一开始就自行设计 Type-C 电源路径。

## 7. 性能升级方案：ESP32-S3 + nRF52840 ESB

如果双 ESP32-S3 + ESP-NOW 的测试结果不能满足延迟或抗干扰目标，再升级为：

    Trackpad
       │ USB
       ▼
    ESP32-S3 USB Host
       │ SPI
       ▼
    nRF52840 ESB Transmitter
       )))) 2.4GHz proprietary ESB ((((
    nRF52840 ESB Receiver + USB Device
       │ USB
       ▼
    macOS

nRF52840 的优势：

- Nordic 规格列出专有 2.4GHz 1 Mbps / 2 Mbps 速率。
- 集成 USB Full-Speed Device 控制器。
- 可以使用 ESB 风格的短包通信，适合低延迟数据通道。

限制：

- nRF52840 不提供 USB Host，发射端仍需要 ESP32-S3 或其他 Host 控制器。
- ESB 最大载荷与配置取决于所用 SDK / ESB 实现；开发配置中出现的 252 字节上限需要在选定的 SDK 版本中实测和确认。
- 需要额外 SPI 驱动、双端无线固件和更复杂的调试。

## 8. 固件模块建议

### Transmitter

- usb_host_manager：设备发现、枚举、重连和电源状态。
- apple_trackpad_usb：描述符解析、Feature Report、原始 Input Report。
- radio_transport：ESP-NOW 或后续 ESB 的统一接口。
- control_proxy：处理接收端转发的 HID 控制请求。
- metrics：报告频率、USB 错误、发送耗时、丢包计数。

### Receiver

- usb_device_identity：设备、配置、接口、端点和 HID 描述符。
- hid_control_handler：处理或转发 HID 控制请求。
- radio_transport：无线接收、校验、序号和链路状态。
- input_report_queue：低延迟有界队列。
- usb_hid_in：按 USB 主机轮询节奏提交 Input Report。
- metrics：无线接收时间、USB 提交时间、序号缺口和队列溢出。

### 实时数据队列原则

- 使用固定大小、有界队列，避免动态内存分配造成不可预测抖动。
- 为每份报告增加序号和本地时间戳。
- 不允许旧报告无限积压；过期报告策略必须可观测并计数。
- 触点按下/抬起、点击状态变化需要特别保护，不能简单地把所有队列内容都合并成最后一份报告。
- 如果出现序号缺口，记录丢包并依靠后续完整状态报告恢复；是否需要关键状态重传由实测决定。

## 9. 关键风险与验证门槛

| 风险 | 影响 | 验证方法 |
|---|---|---|
| 妙控板 USB 速度不是 FS/LS | ESP32-S3 Host 不可用 | 枚举速度实测 |
| HID Report Descriptor 不完整 | macOS 无法绑定原生驱动 | 抓取并逐字节比对 |
| Feature Report 处理不正确 | 无法开启多点触控 | USB 控制传输抓包 |
| macOS 不接受模拟设备身份 | 不能获得原生手势 | macOS USB 树与系统设置验证 |
| ESP-NOW 延迟或抖动过大 | 触控体验不佳 | P50/P95/P99 测试 |
| 2.4GHz 拥挤或丢包 | 手势断续 | 多信道和干扰测试 |
| Type-C 供电设计错误 | 断连、过流或设备损坏 | 电压、电流和保护测试 |

## 10. 参考资料

- [ESP32-S3 USB Host](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-reference/peripherals/usb_host.html)
- [ESP-NOW](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- [ESP32-S3 Technical Reference Manual](https://documentation.espressif.com/esp32-s3_technical_reference_manual_en.pdf)
- [Linux hid-magicmouse.c](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-magicmouse.c)
- [Linux hid-ids.h](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-ids.h)
- [nRF52840 Product Specification](https://docs.nordicsemi.com/r/bundle/ps_nrf52840/page/keyfeatures_html5.html)
- [Nordic ESB throughput discussion](https://devzone.nordicsemi.com/f/nordic-q-a/120889/how-to-achieve-nrf52840-esb-maximum-data-rate)
