# Change: 添加 HTTP/WebSocket 隧道加速 tinc

## Why

tinc VPN 流量可能被网络设备通过 DPI（深度包检测）识别并限速。通过将 tinc TCP 连接封装在 HTTP/WebSocket 隧道中，可以伪装成普通 Web 流量，绕过流量识别和限速限制。

## What Changes

- 新增 **Client 模式**：
  - 监听本地 TCP 端口，接收 tinc TCP 连接
  - 通过 WebSocket 主动连接到 Server 发送上行数据
  - 运行 HTTP Server 接收 Server 发起的 WebSocket 连接（下行数据）
- 新增 **Server 模式**：
  - 运行 HTTP Server 接收 Client 发起的 WebSocket 连接（上行数据）
  - 通过 WebSocket 主动连接到 Client 发送下行数据
  - 将数据转发到目标 tinc 服务器
- 新增 **连接轮换机制**：每传输 N MB 数据发起新的 WebSocket 连接，防止长连接被监控
- 新增 CLI 命令行参数解析（模式选择、地址配置、轮换阈值等）
- 新增配置选项（监听地址、目标地址、WebSocket 路径、轮换大小等）

## Impact

- **Affected specs**: `http-tunnel`（新增）
- **Affected code**:
  - `cmd/tinc-speeder/main.go`：CLI 入口和参数解析
  - 新增核心隧道逻辑代码
