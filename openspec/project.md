# Project Context

## Purpose

tinc-speeder 是一个用于加速 tinc VPN 连接的命令行工具。由于 tinc 流量可能被网络设备识别并限速，本项目通过将 tinc TCP 连接封装在 HTTP/WebSocket 隧道中来绕过流量识别和限速。

核心功能：

- **Client 模式**：将本地 tinc TCP 连接转发到远程 server
- **Server 模式**：接受来自 client 的连接并转发到 tinc 服务器

## Tech Stack

- **语言**：Go 1.25+
- **HTTP 框架**：fasthttp（高性能 HTTP 库）
- **协议**：WebSocket（用于隧道传输）
- **构建工具**：Go modules
- **版本管理**：通过 `version.txt` 嵌入版本号

## Project Conventions

### Code Style

- 遵循标准 Go 代码规范（`gofmt`、`go vet`）
- 包名使用小写下划线命名（如 `tinc_speeder`）
- 无特殊风格指南要求，保持代码简洁清晰

### Architecture Patterns

- **CLI 应用架构**：单一可执行文件，通过命令行参数切换 client/server 模式
- **项目结构**：
  - `cmd/tinc-speeder/`：程序入口
  - 根目录：核心库代码
  - `openspec/`：规范文档

### Testing Strategy

- 无特殊覆盖率要求
- 按需编写单元测试和集成测试

### Git Workflow

- 主分支：`main`
- 功能开发使用 feature 分支
- 遵循常规提交规范

## Domain Context

- **tinc**：开源的 mesh VPN 软件，支持点对点和中心化网络拓扑
- **流量识别**：网络设备可能通过 DPI（深度包检测）识别 tinc 协议特征
- **隧道绕过**：通过将流量伪装成普通 HTTP/WebSocket 流量来规避限速

## Important Constraints

- 需要保持低延迟，避免增加过多开销
- 支持多平台（Go 原生跨平台支持：Linux、Windows、macOS）

## External Dependencies

- **fasthttp**：高性能 HTTP 服务器/客户端库
- **tinc**：目标加速的 VPN 软件（外部依赖，非代码依赖）
