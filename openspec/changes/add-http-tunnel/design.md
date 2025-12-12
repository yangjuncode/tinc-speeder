# Design: HTTP/WebSocket 隧道加速 tinc

## Context

tinc VPN 是一款开源的 mesh VPN 软件。在某些网络环境下，tinc 的流量特征可能被 DPI 设备识别并被限速。本项目通过 HTTP/WebSocket 隧道封装 tinc 流量，使其看起来像普通的 Web 流量。

**约束条件**：

- 使用 Go fasthttp 库实现高性能 HTTP
- 需要 WebSocket 支持双向流数据传输
- 支持多平台（Linux、Windows、macOS）
- 防止长连接被监控识别

## Goals / Non-Goals

**Goals**:

- 实现 Client/Server 双模式的 TCP-over-WebSocket 隧道
- 低延迟、低开销的数据转发
- 简单易用的 CLI 配置
- 连接轮换机制，防止长连接被监控

**Non-Goals**:

- 不实现加密（依赖 tinc 自身加密或 HTTPS）
- 不实现多路复用（初版简单实现）
- 不实现断线重连（初版简单实现）

## Architecture

### 数据流架构（双向 WebSocket）

```
┌─────────────────────────────────────────────────────────────────┐
│                         数据流向                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  tinc ←TCP→ [Client]                    [Server] ←TCP→ tinc    │
│              ↓ HTTP Server ← WebSocket ← ↓ (下行数据)           │
│              ↓ WebSocket → HTTP Server → ↓ (上行数据)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

上行（tinc client → tinc server）：
  tinc → Client(TCP) → Client发起WebSocket → Server(HTTP Server) → tinc server

下行（tinc server → tinc client）：
  tinc server → Server → Server发起WebSocket → Client(HTTP Server) → tinc
```

### 对称设计

Client 和 Server 都需要：

- **HTTP Server**：接收对端发起的 WebSocket 连接
- **WebSocket Client**：主动连接对端发送数据

### 连接轮换

- 每传输 N MB 数据后，关闭当前 WebSocket，发起新连接
- 可配置轮换阈值（默认 10MB）
- 目的：避免长连接特征被 DPI 识别

## Decisions

### 1. 双向 WebSocket 而非单向

- **决定**：上行和下行使用独立的 WebSocket 连接，各自由发送方发起
- **原因**：
  - 避免单一长连接被监控识别
  - 支持连接轮换，每 N MB 发起新连接
  - 更符合正常 Web 流量模式（短连接）
- **备选方案**：单一双向 WebSocket，但容易被识别为长连接

### 2. 使用 fasthttp 库

- **决定**：使用 `github.com/valyala/fasthttp` 作为 HTTP 框架
- **原因**：高性能、低内存分配，适合高吞吐量场景
- **备选方案**：标准库 `net/http`，但性能较低

### 3. 单一可执行文件双模式

- **决定**：通过命令行参数 `-mode client|server` 切换模式
- **原因**：简化部署和分发，一个二进制文件即可运行两种角色

### 4. 连接轮换机制

- **决定**：每传输 N MB 数据发起新 WebSocket 连接
- **原因**：防止长连接被 DPI 监控识别
- **可配置**：通过 `-rotate-size` 参数设置，默认 10MB

## Risks / Trade-offs

| 风险 | 缓解措施 |
|------|----------|
| 双向连接复杂度增加 | 对称设计，Client/Server 共用连接管理逻辑 |
| 连接轮换开销 | 轮换阈值可配置，权衡隐蔽性和性能 |
| 连接关联问题 | 使用 session ID 关联上下行连接 |
| 无断线重连 | 初版简化实现，后续迭代添加 |

## Migration Plan

不适用（新项目，无迁移需求）

## Data Framing Protocol

### 多连接架构

当多个 tinc TCP 连接到 Client 时：

```text
tinc-1 ──TCP──┐                              ┌──TCP── tinc-server
tinc-2 ──TCP──┼─→ [Client] ═══WebSocket═══ [Server] ──TCP── tinc-server
tinc-3 ──TCP──┘                              └──TCP── tinc-server
```

**关键设计**：
- 每个 TCP 连接对应一个独立的 **Session**
- 每个 Session 有独立的 **Session ID**、**序列号**、**发送/接收缓冲区**
- WebSocket 连接是共享的传输通道，通过 Session ID 区分不同 TCP 连接的数据

### 帧格式

每个 WebSocket 二进制帧包含以下结构：

```text
+----------+----------------+----------+------------+-------------+
| Type (1B)| SessionID (16B)| Seq (4B) | Length (4B)| Payload     |
+----------+----------------+----------+------------+-------------+
```

| 字段 | 大小 | 说明 |
|------|------|------|
| Type | 1 byte | 帧类型：0x01=数据帧, 0x02=ACK帧, 0x03=关闭帧 |
| SessionID | 16 bytes | UUID，标识 TCP 连接 |
| Seq | 4 bytes | 序列号（小端序），每 Session 独立 |
| Length | 4 bytes | Payload 长度（小端序） |
| Payload | 可变 | 实际数据 |

### 多连接数据流示例

```text
场景：tinc-1 和 tinc-2 同时发送数据

Client 端：
  tinc-1 → Session-A (Seq=0,1,2...) ─┐
  tinc-2 → Session-B (Seq=0,1,2...) ─┼─→ WebSocket → Server
                                     │
帧序列（WebSocket 中）：
  [DATA, Session-A, Seq=0, "data1"]
  [DATA, Session-B, Seq=0, "data2"]
  [DATA, Session-A, Seq=1, "data3"]
  [ACK,  Session-A, Seq=1]          ← Server 确认
  ...

Server 端：
  收到帧 → 按 SessionID 分发 → Session-A → TCP-1 → tinc-server
                             → Session-B → TCP-2 → tinc-server
```

### 有序传输机制

1. **Session 隔离**：
   - 每个 TCP 连接创建时生成唯一 Session ID（UUID）
   - 每个 Session 独立维护：发送序列号、接收序列号、发送缓冲区、接收缓冲区

2. **序列号管理**：
   - 每个 Session 的序列号从 0 开始，独立递增
   - 不同 Session 的序列号互不影响

3. **确认机制**：
   - ACK 帧携带 SessionID + Seq，确认特定 Session 的数据
   - 发送方按 SessionID 维护发送缓冲区，收到 ACK 后释放对应 Session 的已确认数据

4. **连接轮换时的有序性**：
   - WebSocket 轮换时，所有 Session 的数据都通过新连接发送
   - 每个 Session 的序列号在轮换后继续递增，不重置
   - 接收方通过 SessionID + Seq 保证每个 Session 数据的有序性

### 控制帧

| Type | 用途 |
|------|------|
| 0x01 DATA | 携带 TCP 数据，含 SessionID + Seq |
| 0x02 ACK | 确认特定 Session 已接收的序列号 |
| 0x03 CLOSE | 通知关闭特定 Session（TCP 连接关闭时）|

### Session 生命周期

```text
1. TCP 连接建立
   Client: 生成 SessionID，创建 Session 对象
   Client → Server: 发送第一个 DATA 帧（隐式创建 Session）
   Server: 收到新 SessionID，创建 Session 对象，建立到 tinc-server 的 TCP 连接

2. 数据传输
   双方独立维护每个 Session 的序列号和缓冲区

3. TCP 连接关闭
   发起方: 发送 CLOSE 帧（SessionID）
   对方: 关闭对应 TCP 连接，清理 Session 资源
```

## Open Questions

- 是否需要支持 TLS/HTTPS？（可在后续迭代中添加）
- 是否需要认证机制？（可在后续迭代中添加）
