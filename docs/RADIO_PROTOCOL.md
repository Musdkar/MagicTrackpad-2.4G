# 无线通信协议草案

本文定义发射端与接收端之间的应用层数据格式。第一版以 ESP-NOW v1.0 为传输层；协议应保持传输层可替换，以便未来切换到 nRF52840 ESB。

**状态：草案，尚未实现或实测。** 字段宽度、超时和重试策略可以在 M0/M1 的真实 USB 数据采集后调整。

## 1. 设计目标

- 高频触控 Input Report 尽量单包传输。
- 控制请求与触控数据分离，避免初始化请求阻塞实时输入。
- 每份输入报告带有序号，便于检测丢包、乱序和重复。
- 使用有界队列，不允许旧触控数据无限积压。
- 无线层不解析或转换触点坐标；只负责可靠传送原始 HID 报告。
- 允许未来替换 ESP-NOW 为 ESB，而不修改 USB 代理层。

## 2. ESP-NOW v1.0 载荷预算

ESP-NOW v1.0 的单包数据上限为 250 字节。协议头暂定 12 字节，因此单包应用载荷上限为：

    250 - 12 = 238 bytes

Linux 内核对 USB-C Magic Trackpad 2 报告的解析格式最多为 12 + 15 × 9 = 147 字节。按该上限估算，原始报告加协议头可以放入一个 ESP-NOW v1.0 数据包。

该估算必须通过真实设备确认。固件必须在发送前检查长度，超过上限时显式报错；第一版禁止静默截断。

## 3. 通用包头

所有 ESP-NOW 应用数据包使用相同的 12 字节头部。多字节整数使用 little-endian。

| Offset | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 2 | magic | 固定为 ASCII “MT” 对应的 0x4D 0x54 |
| 2 | 1 | version | 协议主版本，第一版为 0x01 |
| 3 | 1 | type | 消息类型 |
| 4 | 2 | sequence | 发送端递增序号，允许 16-bit 回绕 |
| 6 | 1 | flags | 标志位，未定义位必须置 0 |
| 7 | 1 | payload_len | 后续载荷长度，最大 238 |
| 8 | 4 | timestamp_us | 发射端单调时钟微秒计数低 32 位 |

接收端必须检查 magic、version、payload_len 和实际包长。无效包直接丢弃并增加错误计数。

## 4. 消息类型

| Type | 名称 | 方向 | 用途 |
|---:|---|---|---|
| 0x01 | INPUT_REPORT | TX → RX | 原始 USB HID Input Report |
| 0x02 | CONTROL_REQUEST | RX → TX | 转发 macOS 发来的 HID 控制请求 |
| 0x03 | CONTROL_RESPONSE | TX → RX | 返回真实妙控板的控制请求结果 |
| 0x04 | HEARTBEAT | 双向 | 心跳、链路状态与时钟校准 |
| 0x05 | DEVICE_STATUS | TX → RX | USB 连接状态、设备信息与错误状态 |
| 0x06 | DIAGNOSTIC | 双向 | 调试计数器与诊断数据 |

## 5. INPUT_REPORT

方向：Transmitter → Receiver。

载荷内容为发射端从真实妙控板 USB Interrupt IN 端点取得的原始报告字节。

要求：

- 不把坐标转换为鼠标相对位移。
- 不改变触点顺序、触点 ID、压力、点击状态或报告中的时间字段。
- 如果 USB Host API 对 Report ID 有剥离行为，必须在 USB Host 适配层明确处理，并通过抓包验证最终字节序列与真实 USB 传输一致。
- 记录每份报告的接收时间、无线发送时间和序号。
- 第一版尽量保证一份 USB Input Report 对应一个无线包。

如果 USB 端点一次传输中可能包含多个 HID 报告，必须先确认设备实际行为，再定义拆包规则；不能擅自假设。

## 6. CONTROL_REQUEST

方向：Receiver → Transmitter。

接收端收到电脑发来的 HID Class Request 后，将必要字段转发给发射端。第一版优先支持实际抓包确认的 GET_REPORT 和 SET_REPORT；其他请求按真实设备需求补充。

载荷结构：

| 字段 | 长度 | 说明 |
|---|---:|---|
| transaction_id | 2 | 请求编号 |
| bmRequestType | 1 | USB Setup Packet 字段 |
| bRequest | 1 | USB Setup Packet 字段 |
| wValue | 2 | little-endian |
| wIndex | 2 | little-endian |
| wLength | 2 | little-endian |
| data | 可变 | OUT 请求的数据阶段；IN 请求通常为空 |

发射端将请求转换为真实妙控板上的 USB 控制传输。必须保留请求方向、Report Type、Report ID、接口索引和数据长度。

## 7. CONTROL_RESPONSE

方向：Transmitter → Receiver。

载荷结构：

| 字段 | 长度 | 说明 |
|---|---:|---|
| transaction_id | 2 | 对应请求编号 |
| status | 1 | 0 表示成功；其他值为项目定义错误码 |
| actual_length | 2 | 实际返回数据长度 |
| data | 可变 | IN 请求返回的数据 |

接收端按 transaction_id 匹配等待中的 USB 控制请求。超时、设备拔出或无线断开时，必须结束等待并向 USB Host 返回明确错误，不能无限等待。

控制请求是低频控制路径。输入报告不能等待控制请求完成后才发送。

## 8. HEARTBEAT 与 DEVICE_STATUS

心跳至少应携带：

- 发射端运行状态
- 接收端运行状态
- USB 设备是否已连接
- 无线链路是否可用
- 最近一次有效输入报告序号
- 可选的时钟同步时间戳

DEVICE_STATUS 至少区分：

- USB_DISCONNECTED
- USB_ENUMERATING
- USB_READY
- USB_ERROR
- RADIO_DISCONNECTED
- RADIO_READY

枚举值最终应在公共头文件中统一定义。

## 9. 序号、队列与丢包策略

### 序号

- INPUT_REPORT 使用独立的 16-bit sequence。
- 接收端根据序号差检测丢包、重复和乱序。
- 序号回绕按模 65536 计算。
- 控制请求另用 transaction_id，不能与输入报告序号混用。

### 队列

- 发射端和接收端均使用固定容量队列。
- 队列溢出必须增加计数器并输出诊断信息。
- 不允许无限重传导致旧报告排队。
- 是否合并连续移动报告，需要通过真实触点数据分析后决定。
- 点击、按下、抬起和触点状态变化不能被不加区分地丢弃。
- 对完整状态型报告，可以研究在链路恢复后通过新报告重新同步；必须确认设备报告确实包含足够的当前状态。

### 重试

- 第一版先测量 ESP-NOW 单播发送结果、接收序号缺口和实际丢包。
- 重试次数必须有上限。
- 普通移动报告与控制请求可以采用不同策略。
- 不允许为追求“零丢包”而让实时触控数据产生不可控的队头阻塞。

## 10. 信道与安全

- 初版使用固定 2.4GHz 信道，发射端和接收端配置一致。
- 不连接无线路由器，不创建 AP。
- Bluetooth / BLE 不参与配对、数据传输或恢复。
- 可在后续版本启用 ESP-NOW 加密，并安全保存 PMK / LMK。
- 固件不得在日志中输出密钥。
- 应测试常见 Wi-Fi 信道占用下的延迟和丢包。

## 11. 延迟测量

至少记录：

- T0：发射端 USB Host 收到完整 Input Report 的时间。
- T1：发射端调用无线发送接口的时间。
- T2：接收端收到有效无线包的时间。
- T3：接收端将报告提交到 USB HID IN 队列的时间。

计算：

- 无线发送排队耗时：T1 - T0
- 无线传输及接收处理：T2 - T1
- 接收端处理耗时：T3 - T2
- 桥接新增延迟：T3 - T0

跨芯片时间戳需要时钟偏移校准；在时钟同步尚未实现前，可使用 GPIO 脉冲和逻辑分析仪测量，避免把两个 MCU 的本地时钟直接相减。

## 12. 参考

- [ESP-NOW 官方文档：数据包格式与长度](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- [ESP-IDF ESP-NOW API](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/network/esp_now.html)
- [Linux hid-magicmouse.c](https://github.com/torvalds/linux/blob/master/drivers/hid/hid-magicmouse.c)
