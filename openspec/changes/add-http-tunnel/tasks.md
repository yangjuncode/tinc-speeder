# Implementation Tasks

## 1. 项目基础设施

- [ ] 1.1 添加 fasthttp 和 websocket 依赖到 go.mod
- [ ] 1.2 创建项目目录结构（pkg/tunnel, pkg/session）

## 2. CLI 和配置

- [ ] 2.1 实现命令行参数解析（mode, tcp-listen, http-listen, server, client, target, rotate-size）
- [ ] 2.2 添加配置验证和默认值
- [ ] 2.3 实现帮助信息和版本输出

## 3. Session 管理（多连接复用）

- [ ] 3.1 实现 Session ID 生成（16字节 UUID）
- [ ] 3.2 实现 Session 注册表（SessionID → Session 对象映射）
- [ ] 3.3 实现 Session 对象（TCP 连接、发送序列号、接收序列号、发送缓冲区、接收缓冲区）
- [ ] 3.4 实现 Session 生命周期管理（创建、数据传输、关闭）
- [ ] 3.5 实现新 SessionID 检测和自动创建对端 TCP 连接

## 4. 数据帧协议

- [ ] 4.1 定义帧结构：Type(1B) + SessionID(16B) + Seq(4B) + Length(4B) + Payload
- [ ] 4.2 实现 DATA 帧编码/解码（Type=0x01，含完整帧头和 Payload）
- [ ] 4.3 实现 ACK 帧编码/解码（Type=0x02，含 SessionID + Seq）
- [ ] 4.4 实现 CLOSE 帧编码/解码（Type=0x03，含 SessionID）

## 5. 可靠传输机制（每 Session 独立）

- [ ] 5.1 实现序列号管理（每 Session 独立，从 0 递增）
- [ ] 5.2 实现发送缓冲区（按 SessionID 分组，保存待确认数据帧）
- [ ] 5.3 实现接收方按 SessionID + Seq 重组数据
- [ ] 5.4 实现 ACK 确认机制（按 SessionID 确认，释放对应缓冲区）

## 6. 连接轮换机制

- [ ] 6.1 实现传输字节计数（所有 Session 共享计数）
- [ ] 6.2 实现达到阈值时的连接轮换
- [ ] 6.3 轮换前等待所有 Session 的 ACK 确认
- [ ] 6.4 新连接继续使用各 Session 当前序列号

## 7. Client 模式实现

- [ ] 7.1 实现 TCP 监听器（接收多个 tinc 连接，每个分配 SessionID）
- [ ] 7.2 实现 HTTP Server（接收 Server 发起的下行 WebSocket）
- [ ] 7.3 实现 WebSocket 客户端（发起到 Server 的上行连接，共享传输所有 Session）
- [ ] 7.4 实现上行数据转发（TCP → 按 SessionID 封装帧 → WebSocket）
- [ ] 7.5 实现下行数据转发（WebSocket → 按 SessionID 分发 → TCP）

## 8. Server 模式实现

- [ ] 8.1 实现 HTTP Server（接收 Client 发起的上行 WebSocket）
- [ ] 8.2 实现按 SessionID 管理到 tinc-server 的 TCP 连接池
- [ ] 8.3 实现 WebSocket 客户端（发起到 Client 的下行连接，共享传输所有 Session）
- [ ] 8.4 实现上行数据转发（WebSocket → 按 SessionID 分发 → TCP）
- [ ] 8.5 实现下行数据转发（TCP → 按 SessionID 封装帧 → WebSocket）

## 9. 连接管理

- [ ] 9.1 实现连接生命周期管理
- [ ] 9.2 实现优雅关闭和资源清理
- [ ] 9.3 添加错误处理和日志记录

## 10. 测试和文档

- [ ] 10.1 编写单元测试
- [ ] 10.2 进行集成测试（client-server 联调）
- [ ] 10.3 更新 README.md 使用说明
