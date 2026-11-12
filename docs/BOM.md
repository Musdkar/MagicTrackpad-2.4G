# 第一版原型物料清单（BOM）

> 目标：以 Apple Magic Trackpad USB-C（A3120）为目标设备，搭建双 ESP32-S3 + ESP-NOW 的非蓝牙 2.4GHz USB 桥接原型。
>
> **本清单是开发验证 BOM，不是最终量产 BOM。** 先验证 USB Host、USB Device 与原始 HID 兼容性，再决定是否设计专用 PCB。

## 1. 推荐采购方案

优先使用乐鑫官方 **ESP32-S3-USB-OTG** 开发板，而不是随便购买只有 USB-C 下载口的 ESP32-S3 小板。

该板已经集成：
- ESP32-S3-MINI-1-N8（8 MB Flash）
- USB-A Host 母座
- USB-A Device 公头
- USB Host / Device 数据通路切换电路
- USB 转 UART 调试接口（Micro-USB）
- Host 口 5V 供电与 500mA 限流电路

两块板分别承担发射端 USB Host 和接收端 USB Device。板上的 USB 数据通路通过开关切换，单块板不能同时承担两个角色；本项目使用两块板解决这个问题。

## 2. 核心 BOM

| 编号 | 物料 | 推荐型号 / 规格 | 数量 | 参考单价（人民币） | 用途 |
|---|---|---|---:|---:|---|
| U1 | USB-OTG 开发板 | Espressif ESP32-S3-USB-OTG | 2 | ¥169 | 一块作 USB Host 发射端，一块作 USB Device 接收端 |
| C1 | USB 数据线 | USB-A 公头 → USB-C 公头，USB 2.0 数据线 | 1 | ¥5–15 | 妙控板接发射端 USB-A Host 口 |
| C2 | Micro-USB 数据线 | USB-A → Micro-B，支持数据 | 1 | ¥5–10 | 烧录固件、UART 调试 |
| P1 | USB 电源适配器 | 5V / 1A 或更高，USB-A 输出 | 1 | ¥10–25 | 给发射端 USB_DEV 口供电，从而为 USB_HOST 口提供 VBUS |
| J1 | USB-A 公头转接 / 延长线 | USB 2.0，按桌面布置选购 | 0–1 | ¥5–15 | 可选，方便接收端连接电脑 |

**官方价格参考：** 乐鑫开发板产品页曾列出 ESP32-S3-USB-OTG 淘宝样品参考价 ¥169/块；实际零售价格、库存和套装内容以购买页面为准。两块按该参考价约 ¥338。

### 预算估算

| 采购方式 | 估算金额 | 说明 |
|---|---:|---|
| 先买一块做 M0 USB 研究 | 约 ¥190–220 | 一块开发板、线材和供电 |
| 完整双板原型 | 约 ¥365–405 | 两块开发板、必要线材和供电 |
| 完整原型 + 可选转接线 | 约 ¥370–420 | 视线材与电源是否已有而定 |

以上是询价预算，不是实时成交报价；如果你已有 5V USB-A 充电器和 Micro-USB 数据线，可以进一步降低成本。A3120 妙控板本身不计入上述金额。

## 3. 两块开发板分别怎么连接

### 发射端（Transmitter）

    5V USB-A 充电器
          │
          ▼
    ESP32-S3-USB-OTG 的 USB_DEV（A 公头，供电输入）
          │
          ├── USB_HOST（A 母座）
          │       │
          │       └── USB-A → USB-C 数据线
          │                    │
          │                    ▼
          │             Magic Trackpad A3120
          │
          └── ESP32-S3：USB Host + ESP-NOW TX

Micro-USB 调试口用于烧录和串口调试。乐鑫文档指出，仅从 Micro-USB 口供电时，USB_HOST 口不会获得 VBUS；需要通过 USB_DEV 供电路径或板载电池路径给 Host 口供电。

### 接收端（Receiver）

    ESP32-S3-USB-OTG 的 USB_DEV（A 公头）
          │
          ▼
    电脑 USB-A Host 端口

接收端配置为 USB Device，通过 ESP-NOW 收到原始报告后，再向电脑提供 HID 输入。Micro-USB 调试口用于烧录和串口日志。

## 4. 采购时必须核对的细节

### 开发板

搜索完整型号：**ESP32-S3-USB-OTG（Espressif）**。

不要误买成：
- ESP32-S3-DevKitC-1
- ESP32-S3-USB-Bridge
- 只有一个 USB-C 下载口的普通 ESP32-S3 开发板
- ESP32-C3 / ESP32-C6（不能直接替代本方案的 USB OTG Host + Device 原型板）

普通 ESP32-S3 板也可能用于自制方案，但往往还需要 USB Host 连接器、VBUS 电源开关、Type-C 角色配置和 USB 数据通路设计。第一版不建议增加这些工作。

### USB-A → USB-C 数据线

必须支持 USB 2.0 D+ / D− 数据传输，不能买仅充电线。线的一端是 USB-A 公头，插开发板 USB_HOST 母座；另一端 USB-C 公头插妙控板。

### 发射端供电

乐鑫开发板文档说明 USB_HOST 口提供 5V，最大电流 500mA，并有电流限制电路。开发时要启用相应的 Host 口供电控制 GPIO（官方板定义中的 IDEV_LIMIT_EN），并确认 VBUS 实测正常。

如果 A3120 枚举失败、掉线或供电不足，先测量 VBUS 和电流，再判断是否需要独立、限流的 5V 供电方案；不要直接短接两个电源口。

### 接收端连接

开发板的 USB_DEV 是 USB-A 公头，可直接插入电脑的 USB-A 母座。如果电脑端只有 USB-C，可再选择合规的 USB-A 母头到 USB-C 公头数据转接器；转接器必须支持数据，不只是供电。

## 5. 暂时不要买的物料

| 物料 | 暂不购买原因 |
|---|---|
| nRF52840 开发板 / 模块 | 第一版先用 ESP-NOW，等实测发现性能不足再评估 ESB |
| nRF24L01+ 模块 | 单包载荷、分包与固件复杂度不适合第一版 |
| USB-C PD 控制器 / Type-C Host 转接板 | 官方 ESP32-S3-USB-OTG 已有 USB-A Host 口；用 A-to-C 数据线连接 A3120，暂时不需要自行实现 Type-C DFP |
| 定制 PCB | USB 协议和 macOS 识别尚未验证 |
| 电池、充电管理板 | 桌面原型先用外部 5V 电源 |
| 高速 USB 分析仪 | 先用 Linux usbmon / Wireshark 或 Windows USBPcap 获取描述符和控制传输；确认需要后再采购专用分析仪 |

## 6. 采购顺序建议

为控制试错成本，建议分两次采购：

**第一批：先验证 A3120 USB Host**

- ESP32-S3-USB-OTG × 1
- USB-A → USB-C 数据线 × 1
- 5V USB-A 电源 × 1（如果手头没有）
- USB-A → Micro-USB 数据线 × 1

第一块板先完成 M0：确认 A3120 能否被枚举、USB 速度、描述符、HID 接口、Feature Report 和原始触控报告。

**第二批：M0 通过后**

- ESP32-S3-USB-OTG × 1

第二块板作为 USB Device 接收端，先用录制的 HID 报告验证 macOS 原生识别，再接入无线链路。

## 7. 关键技术门槛

ESP32-S3 USB Host 只支持 Full-Speed / Low-Speed 设备。必须先确认 A3120 的实际 USB 总线速度。如果设备需要 High-Speed，当前 Host 方案必须更换，不能仅靠软件解决。

此外，USB-C A3120 的 HID 描述符、接口数量和 Feature Report 必须从真实设备抓取。不能只凭 Linux 驱动中的通用解析格式，就认定 ESP32-S3 与 macOS 透明代理已经成功。

## 8. 参考资料

- [乐鑫 ESP32-S3-USB-OTG 官方产品页（含淘宝样品参考价）](https://www.espressif.com/zh-hans/products/devkits)
- [ESP32-S3-USB-OTG 官方硬件指南（USB Host/Device 接口、电源、500mA 限流）](https://docs.espressif.com/projects/esp-dev-kits/zh_CN/latest/esp32s3/esp32-s3-usb-otg/user_guide.html)
- [ESP32-S3 USB Host 官方文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/peripherals/usb_host.html)
- [ESP32-S3 USB Device 官方文档](https://docs.espressif.com/projects/esp-usb/en/latest/esp32s3/usb_device.html)
- [ESP-NOW 官方文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- [Linux hid-magicmouse.c](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-magicmouse.c)
