# HTTP Tunnel Capability

## ADDED Requirements

### Requirement: Client Mode

系统 SHALL 提供 Client 模式，包含 TCP 监听器和 HTTP Server，实现双向 WebSocket 隧道。

#### Scenario: Client 启动

- **GIVEN** 用户以 client 模式启动程序
- **WHEN** 指定本地 TCP 监听地址、HTTP Server 地址和远程 Server 地址
- **THEN** 程序同时启动 TCP 监听器和 HTTP Server

#### Scenario: Client 上行数据转发

- **GIVEN** Client 正在运行
- **WHEN** tinc 建立 TCP 连接并发送数据到 Client
- **THEN** Client 发起 WebSocket 连接到 Server
- **AND** 将 TCP 数据封装到 WebSocket 二进制帧发送

#### Scenario: Client 下行数据接收

- **GIVEN** Client HTTP Server 正在运行
- **WHEN** Server 发起 WebSocket 连接到 Client
- **THEN** Client 接收 WebSocket 数据
- **AND** 将数据转发到对应的 tinc TCP 连接

### Requirement: Server Mode

系统 SHALL 提供 Server 模式，包含 HTTP Server 和 WebSocket 客户端，实现双向 WebSocket 隧道。

#### Scenario: Server 启动

- **GIVEN** 用户以 server 模式启动程序
- **WHEN** 指定 HTTP Server 地址、远程 Client 地址和目标 tinc 服务器地址
- **THEN** 程序启动 HTTP Server 准备接收连接

#### Scenario: Server 上行数据接收

- **GIVEN** Server HTTP Server 正在运行
- **WHEN** Client 发起 WebSocket 连接发送上行数据
- **THEN** Server 接收数据并转发到目标 tinc 服务器

#### Scenario: Server 下行数据转发

- **GIVEN** tinc 服务器返回数据
- **WHEN** Server 需要发送下行数据
- **THEN** Server 发起 WebSocket 连接到 Client HTTP Server
- **AND** 将数据封装到 WebSocket 二进制帧发送

### Requirement: Connection Rotation

系统 SHALL 实现连接轮换机制，每传输 N MB 数据发起新的 WebSocket 连接。

#### Scenario: 达到轮换阈值

- **GIVEN** WebSocket 连接正在传输数据
- **WHEN** 累计传输数据达到配置的轮换阈值（默认 10MB）
- **THEN** 关闭当前 WebSocket 连接
- **AND** 发起新的 WebSocket 连接继续传输

#### Scenario: 配置轮换阈值

- **GIVEN** 用户启动程序
- **WHEN** 使用 `-rotate-size 20` 参数
- **THEN** 每传输 20MB 数据发起新连接

### Requirement: Session Management

系统 SHALL 使用 Session ID 关联上下行 WebSocket 连接。

#### Scenario: 建立 Session

- **GIVEN** tinc 建立新的 TCP 连接
- **WHEN** 创建隧道会话
- **THEN** 生成唯一 Session ID
- **AND** 在 WebSocket 握手时携带 Session ID

#### Scenario: Session 关联

- **GIVEN** 收到带有 Session ID 的 WebSocket 连接
- **WHEN** 查找对应的 TCP 连接
- **THEN** 将 WebSocket 数据转发到正确的 TCP 连接

### Requirement: CLI Configuration

系统 SHALL 提供命令行参数配置运行模式和网络地址。

#### Scenario: 显示帮助信息

- **GIVEN** 用户运行程序
- **WHEN** 使用 `-h` 或 `--help` 参数
- **THEN** 显示帮助信息，包括所有可用参数

#### Scenario: Client 模式参数

- **GIVEN** 用户以 client 模式运行
- **WHEN** 使用 `-mode client -tcp-listen :1655 -http-listen :8081 -server ws://remote:8080/api`
- **THEN** Client 在 1655 端口监听 TCP，8081 端口监听 HTTP，连接到远程 Server

#### Scenario: Server 模式参数

- **GIVEN** 用户以 server 模式运行
- **WHEN** 使用 `-mode server -http-listen :8080 -client ws://client:8081/api -target 127.0.0.1:655`
- **THEN** Server 在 8080 端口监听 HTTP，转发到本地 tinc 655 端口，下行数据发送到 Client

### Requirement: WebSocket Protocol

系统 SHALL 使用 WebSocket 协议进行数据传输，以二进制帧传输 TCP 数据。

#### Scenario: WebSocket 握手

- **GIVEN** 发起 WebSocket 连接
- **WHEN** 建立连接
- **THEN** 使用配置的路径（默认 `/api`）进行握手
- **AND** 在请求头中携带 Session ID

#### Scenario: 二进制数据传输

- **GIVEN** WebSocket 连接已建立
- **WHEN** 有 TCP 数据需要转发
- **THEN** 使用 WebSocket 二进制帧传输封装后的数据帧

### Requirement: Data Framing Protocol

系统 SHALL 使用自定义帧格式封装数据，支持多 TCP 连接复用和有序传输。

#### Scenario: 数据帧格式

- **GIVEN** 需要发送 TCP 数据
- **WHEN** 封装数据帧
- **THEN** 帧结构为：Type(1B) + SessionID(16B) + Seq(4B) + Length(4B) + Payload
- **AND** Type=0x01 表示数据帧
- **AND** SessionID 为 16 字节 UUID，标识 TCP 连接
- **AND** Seq 为小端序序列号，每 Session 独立
- **AND** Length 为小端序 Payload 长度

#### Scenario: 多连接复用

- **GIVEN** 多个 tinc TCP 连接到 Client
- **WHEN** 各 TCP 连接发送数据
- **THEN** 每个 TCP 连接分配独立的 SessionID
- **AND** 所有 Session 数据通过共享的 WebSocket 连接发送
- **AND** 通过 SessionID 区分不同 TCP 连接的数据

#### Scenario: 发送数据帧

- **GIVEN** 发送方有数据需要发送
- **WHEN** 发送数据帧
- **THEN** 递增该 Session 的序列号
- **AND** 将数据帧保存到该 Session 的发送缓冲区
- **AND** 通过 WebSocket 发送

#### Scenario: 接收数据帧

- **GIVEN** 接收方收到数据帧
- **WHEN** 解析帧内容
- **THEN** 根据 SessionID 找到对应的 Session
- **AND** 按该 Session 的序列号顺序重组数据
- **AND** 将 Payload 转发到对应的 TCP 连接

### Requirement: Acknowledgment Mechanism

系统 SHALL 实现确认机制，确保每个 Session 的数据可靠传输。

#### Scenario: 发送 ACK 帧

- **GIVEN** 接收方收到某 Session 连续的数据帧
- **WHEN** 需要确认已接收数据
- **THEN** 发送 ACK 帧（Type=0x02）
- **AND** 帧格式为：Type(1B) + SessionID(16B) + Seq(4B)
- **AND** Seq 字段为该 Session 已收到的最大连续序列号

#### Scenario: 处理 ACK 帧

- **GIVEN** 发送方收到 ACK 帧
- **WHEN** 解析 SessionID 和 Seq
- **THEN** 从对应 Session 的发送缓冲区移除已确认的数据帧

#### Scenario: 连接轮换时的确认

- **GIVEN** 需要轮换 WebSocket 连接
- **WHEN** 关闭旧连接前
- **THEN** 等待所有 Session 的待确认数据收到 ACK
- **AND** 新连接继续使用各 Session 当前的序列号

### Requirement: Connection Management

系统 SHALL 正确管理连接生命周期，处理连接建立和关闭。

#### Scenario: 连接正常关闭

- **GIVEN** 隧道连接正在使用中
- **WHEN** tinc TCP 连接关闭
- **THEN** 清理对应的 Session
- **AND** 通知对端关闭相关资源

#### Scenario: 连接异常处理

- **GIVEN** 隧道连接正在使用中
- **WHEN** 网络错误导致 WebSocket 连接中断
- **THEN** 记录错误日志
- **AND** 尝试发起新的 WebSocket 连接继续传输
