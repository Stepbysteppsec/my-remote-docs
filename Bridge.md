#### 系统概述：
	存在实体 tms，和zk，其中zk有1-10个。而tms有一个。 zk和tms有各自的协议，所以当zk要和tms沟通时需要一个中间桥梁程序 ：Bridge

#### 主要核心场景：

- 收到zk发起通告需求 
- brdige解析通告需求，提取target_id,等参数 
- 转发通告需求到tms
- 和tms完成会话信令交互过程 ，成功进入业务通信
- 将收到的zk的业务通信，转发到对应的tms侧的终端
- 将收到的tms侧的业务通信转发到对应的zk

- 收到zk 拆链通告需求
- bridge 转发此需求到tms
- tms和bridge拆链信令交互
- 业务通信拆除


##### 详细核心场景流程
- 程序启动
	- 从配置文件中加载cu配置表，cu和target_id映射表
	-  加载tms,zk 的ip port 同时启动监听流程
	- 和tms建立心跳，当存在心跳时，上报设备状态到zk
- 程序运行：
	- 监听到来自zk的报文
	- 这里好像协议自己封装了可靠性机制
	- 判断类型为 需求通告
	- 判断内容为建立点对点通信
	- 提取被叫号码，带宽
	- 通过号码 读取相应配置
	- 创建点对点会话表，并填入zk_ip，被叫号码，带宽，配置，自动分配 bridge->zk 业务端口，bridge->tms业务端口
	- 会话表触发tms侧发起呼叫
	- 通过tcp发送
	- 接收tcp响应，更新会话表状态
	- 会话表检测到之后
	- 解析业务端口，更新到会话表中
    - 

实体2发送通告拆链  


解析拆链消息  

  

提取目的终端,通过目的终端和站控id来查找会话  

  

更新会话状态为正在关闭  

  

通过session数据结构 访问其中的cu号来向tms发送呼叫挂断  

  

·收到呼叫挂断响应  

  

·通过cu号来查找session结构体并更新其中的状态  

  

更新session数据结构中的状态  

  

回收brdige->tms业务udp端口及socket，回收brdige->zkudp端口和socket  

  

更新session中状态为已经拆链  

  

释放session数据结构 请你设计一个架构 我内心已经有一个架构了 我想和你比一下谁设计的更好

#### 结构设计：
- zk_interface: 负责zk协议的数据结构实现，以及这些数据结构的方法，创建方法和解析方法



ubuntu16.04 虚拟机镜像 密码 1234



FROM ubuntu:20.04

# 避免交互式安装

ENV DEBIAN_FRONTEND=noninteractive

# 安装开发工具

RUN apt-get update && apt-get install -y \

build-essential \

cmake \

gdb \

git \

curl \

wget \

vim \

nano \

pkg-config \

valgrind \

&& rm -rf /var/lib/apt/lists/*

# 创建开发用户

RUN useradd -m -s /bin/bash developer && \

usermod -aG sudo developer && \

echo 'developer:developer' | chpasswd && \

echo 'developer ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# 设置工作目录

WORKDIR /workspace

# 切换到开发用户

USER developer

# 设置编译环境变量（兼容性优先）

ENV CXXFLAGS="-std=c++11"

ENV CC=gcc

ENV CXX=g++

这是我的docker file文件 每次我进去这个工程 都提示我要安装 插件 但是安装完以后 当我连接这些容器 时 又提示我 os版本太老了 但是我觉得 不太对，因为我的镜像是20.04的不可能是这个原因 我的理解就是连接时 连接到了错误的容器 上 但是好像工程下的有两个配置文件 一个devcontainer.json 一个dockerfile 我现在需要弄清楚一些基本的概念 ， 你来告诉我这两个配置文件 是干嘛用的 ，谢谢 

而且由于我想在这个docker里使用交叉编译






Okay, here's a structural design document for your C++ "Bridge" application, incorporating our discussions on I/O multiplexing, event-driven architecture, and multithreading. I'll also address your question about using JSON, spdlog, and Redis.

Yes, you absolutely **should consider using libraries like a JSON parser, spdlog, and Redis**, especially given your requirements:

- **JSON Library (e.g., nlohmann/json, RapidJSON):** Excellent for reading configuration files if they are in JSON format. This simplifies configuration management significantly.
- **spdlog:** A very good choice for high-performance, flexible, and structured logging in C++. Logging is crucial for debugging and monitoring any server application.
- **Redis:** For主备容灾 (active-standby disaster recovery), Redis is a strong candidate. It can be used to store and replicate active session state. If a primary Bridge instance fails, a standby instance can potentially take over by loading session information from Redis, allowing for more seamless failover or graceful termination of existing calls.

---

## Bridge 系统结构设计说明文档

本文档描述了 Bridge 系统的结构设计，该系统作为 ZK 实体和 TMS 系统之间的中间桥梁，支持信令和媒体业务的转发。设计考虑了高性能、高并发、可扩展性和可维护性，采用了I/O复用、事件驱动和多线程的架构。

---

### 1. 核心设计原则 🌟

- **模块化:** 系统划分为具有明确职责的独立模块。
- **异步非阻塞:** I/O操作和核心逻辑采用异步非阻塞模式，以提高并发处理能力。
- **事件驱动:** 系统由外部事件（如网络消息、定时器）和内部事件（任务完成）驱动。
- **线程分离:** I/O密集型任务与CPU密集型（或业务逻辑）任务由不同类型的线程处理。
- **会话中心化:** `Session` 对象封装了单个端到端呼叫的核心状态和逻辑。
- **线程安全:** 所有共享资源（会话表、端口池、媒体转发表等）的设计都必须考虑线程安全。

---

### 2. 总体架构图 (概念)

```
+---------------------+      +-----------------------+      +---------------------+
|     ZK Entities     |<---->|      ZK Interface     |<---->|                     |
| (10 ZKs, 1 port ea. |      | (10 Listeners,        |      |                     |
|  Signaling & Media) |      |  Demux, Proto Handler) |      |   Coordinator /   |
+---------------------+      +-----------+-----------+      |   Event Router    |      +-------------------+
                                         ^                          |                     |----->|  Session Module   |
                                         |                          v                     |<-----| (SessionManager,  |
                                         | (Tasks)           +------+------+              |      |  Session Objects) |
                                         |                   | Task Queue  |              |      +-------------------+
                                         v                   +------+------+              |                |
+---------------------+      +-----------+-----------+              | (Tasks)            |                | (Config MediaRelay)
|     TMS System      |<---->|     TMS Interface     |<-------------+                     |                v
| (1 TCP Sig, N UDP   |      | (TCP Conn, N UDP Listen,|             v                     |      +-------------------+
|  Media from TMS)    |      |  Proto Handler)       |      +-----------------------+      |----->| MediaRelay Module |
+---------------------+      +-----------------------+      |   Worker Thread Pool  |------      |(Fast Fwd Table,   |
                                                            +-----------------------+             | Header Manip.)    |
                                                                                                  +-------------------+
                                                                     |  ^
                                                                     |  | (Alloc/Release)
                                                                     v  |
+-----------------------+     +-----------------------+     +-----------------------+
| Communication Layer   |<--->|    Protocol Layer     |<--->|    Port Manager       |
| (Async Sockets)       |     | (ZK/TMS Parsers/Fmts) |     | (UDP Port Pool)       |
+-----------------------+     +-----------------------+     +-----------------------+

+-----------------------+     +-----------------------+
| Configuration Manager |     |     Logging (spdlog)  |
| (JSON for config)     |     +-----------------------+
+-----------------------+

+-----------------------+
| Redis (Optional for HA|
| Session State Sync)   |
+-----------------------+
```

---

### 3. 线程模型 🧵

- **I/O 线程 (少量, e.g., `num_cores / 2` 或 `num_cores`):**
    - 运行 `epoll` (或等效的) 事件循环。
    - 负责所有Socket的非阻塞读写。
    - 进行最基本的数据包解析（如识别来源、判断是信令还是媒体，提取会话标识符）。
    - 将解析后的任务（封装了数据和上下文）放入任务队列，交由工作线程池处理。
    - 直接处理极低延迟的媒体转发（如果包头操作简单）。
- **工作线程池 (固定数量, e.g., `num_cores` 或更多):**
    - 从任务队列中获取任务。
    - 执行所有核心业务逻辑：`Session` 对象的创建、状态更新、决策；与 `ConfigManager`、`PortManager` 交互；准备信令数据；通过 `Coordinator` 指示接口层发送消息；配置 `MediaRelay`。
- **可选的专用线程：**
    - TMS心跳线程。
    - 定时清理线程 (如清理过期会话)。

---

### 4. 代码结构与模块说明

```
bridge_system/
├── main.cpp                     # 程序入口, 初始化各模块, 启动I/O事件循环和工作线程池
│
├── common/                      # 通用基础模块
│   ├── communication/           # 底层异步网络通信封装
│   │   ├── async_udp_socket.{h,cpp}
│   │   └── async_tcp_client.{h,cpp} # (如果TMS信令是Client模式)
│   │   └── async_tcp_server.{h,cpp} # (如果TMS信令是Server模式或管理接口)
│   ├── task_queue/              # 线程安全的任务队列 (I/O线程与工作线程交互)
│   │   └── concurrent_queue.h
│   └── utils/                   # 其他工具类 (e.g., string manipulation, timers)
│       └── ...
│
├── protocols/                   # 协议定义与编解码 (无状态工具库)
│   ├── zk_protocol/
│   │   ├── zk_message_types.h   # ZK协议消息的C++结构体定义
│   │   ├── zk_parser.{h,cpp}    # 解析ZK字节流 -> 结构体
│   │   └── zk_formatter.{h,cpp} # 结构体 -> ZK字节流
│   └── tms_protocol/
│       ├── tms_message_types.h
│       ├── tms_parser.{h,cpp}
│       └── tms_formatter.{h,cpp}
│
├── interfaces/                  # 外部系统接口适配层
│   ├── zk_interface/
│   │   ├── zk_listener.{h,cpp}  # 管理10个UDP监听Socket (接收ZK信令和媒体)
│   │   │                        # 功能: 使用communication层, 注册到epoll, 读取数据,
│   │   │                        #       初步解析(识别ZK ID, 会话ID, 信令/媒体),
│   │   │                        #       创建任务并放入TaskQueue或直接调用MediaRelay
│   │   ├── zk_sender.{h,cpp}    # 封装向ZK发送数据 (使用communication层和zk_formatter)
│   │   └── zk_defs.h            # ZK接口相关的内部常量/定义
│   └── tms_interface/
│       ├── tms_connector.{h,cpp}# 管理与TMS的TCP信令连接 (心跳, 收发)
│       ├── tms_media_listener.{h,cpp} # 管理N个UDP监听Socket (接收TMS媒体)
│       ├── tms_sender.{h,cpp}   # 封装向TMS发送数据
│       └── tms_defs.h
│
├── core_logic/                  # 核心业务逻辑
│   ├── coordinator.{h,cpp}      # 协调器/路由器: 在接口层和会话模块间传递事件和指令
│   │                            # 功能: 由工作线程调用, 路由到SessionManager或具体Session
│   ├── session.h                # Session对象定义
│   │   │                        # 主要数据结构: SessionState (enum), ZK/TMS媒体信息,
│   │   │                        #               会话ID, target_id, 带宽, Cu配置引用等
│   │   │                        # 主要函数: handleZkEvent(), handleTmsEvent(),
│   │   │                        #           processMedia(), setupMediaRelayRules(),
│   │   │                        #           initiateTmsCall(), sendStateToZk(), cleanup()
│   ├── session.cpp              # Session对象实现 (状态机, 核心业务流程)
│   ├── session_manager.h        # SessionManager类定义 (线程安全)
│   │   │                        # 主要数据结构: std::unordered_map<SessionID, std::shared_ptr<Session>> active_sessions;
│   │   │                        #               (可能还有其他索引map, 如 Port -> SessionID)
│   │   │                        # 主要函数: createSession(), findSession(), removeSession()
│   ├── session_manager.cpp      # SessionManager实现
│   ├── port_manager.h           # PortManager类定义 (线程安全)
│   │   │                        # 主要数据结构: std::set<uint16_t> available_ports; std::mutex mtx;
│   │   │                        # 主要函数: allocatePort(), releasePort()
│   ├── port_manager.cpp         # PortManager实现
│   └── config_manager.h         # ConfigManager类定义
│       │                        # 主要数据结构: Structs for ZK/TMS config, CU config table,
│       │                        #               CU-target_id map. (使用JSON库解析)
│       │                        # 主要函数: loadConfigs(), getZkConfig(zkId), getTmsConfig(),
│       │                        #           getCuConfig(cuId), getTargetCuMapping(targetId)
│   └── config_manager.cpp       # ConfigManager实现
│
├── media_processing/            # 媒体处理与转发
│   └── media_relay.{h,cpp}      # 媒体中继模块 (线程安全)
│       │                        # 主要数据结构: ForwardingRule (包含目标IP/Port, 包头处理函数指针/lambda, 发送Socket引用)
│       │                        #               std::unordered_map<uint16_t /*local_rcv_port*/, ForwardingRule> zk_to_tms_flows;
│       │                        #               std::unordered_map<uint16_t /*local_rcv_port*/, ForwardingRule> tms_to_zk_flows;
│       │                        # 主要函数: setupFlow(), teardownFlow(),
│       │                        #           processIncomingZkMedia(packet, local_port_or_session_id),
│       │                        #           processIncomingTmsMedia(packet, local_port)
│
└── logging/                     # 日志模块
    └── logger_wrapper.{h,cpp}   # spdlog的封装和全局访问点
```

---

### 5. 关键数据结构与函数概述 (部分已在上面提及)

- **`Session` 对象:**
    - **数据:** `sessionId`, `zkId`, `targetId`, `bandwidth`, `zkAllocatedBridgeMediaPort` (用于发往ZK), `tmsAllocatedBridgeMediaPort` (用于收发TMS媒体), `tmsRemoteMediaEndpoint`, `cuConfig`, `currentState (enum: IDLE, ZK_INITIATED, TMS_CONNECTING, ACTIVE, TERMINATING, ...)`。
    - **函数:** `handleZkSignaling(parsedMsg)`, `handleTmsSignaling(parsedMsg)`, `allocateMediaPorts()`, `setupMediaRelay()`, `sendToZkViaCoordinator(msg)`, `sendToTmsViaCoordinator(msg)`, `terminate()`.
- **`SessionManager`:**
    - **数据:** `std::unordered_map<std::string /*SessionID*/, std::shared_ptr<Session>> sessions_by_id;` (主表，线程安全), 可能还有其他辅助索引表如 `std::unordered_map<uint16_t /*BridgeLocalMediaPort*/, std::string /*SessionID*/> port_to_session_map;` (用于媒体包初次关联，但后续媒体转发不依赖它)。
    - **函数:** `createSessionFromZk(zkData)`, `getSessionById(id)`, `getSessionByLocalMediaPort(port)`, `removeSession(id)`.
- **`MediaRelay`:**
    - **数据:** 如上所述的 `active_flows_table` (两个方向的转发表)，键为 Bridge 本地接收媒体的端口，值为包含完整转发路径和处理规则的结构体。**这些表必须是线程安全的。**
    - **函数:** `addFlow(sessionId, zkLocalPort, zkRemoteEp, tmsLocalPort, tmsRemoteEp, zkToTmsHeaderProc, tmsToZkHeaderProc)`, `removeFlow(sessionId_or_ports)`, `onUdpPacketReceived(buffer, len, localRecvPort, remoteSenderEp)`.
- **`ZkListener` (in `zk_interface`):**
    - **数据:** `std::vector<std::unique_ptr<AsyncUDPSocket>> zk_listening_sockets;` (10个，每个对应一个ZK的信令/媒体入口)。
    - **函数:** `startListening()`, `onDataReceived(socket_idx, data, remote_ep)` (回调，由此创建任务交工作线程池)。

---

### 6. 库的使用建议

- **JSON 库 (如 nlohmann/json for Modern C++):**
    - **用途:** 在 `ConfigManager` 中用于解析 `.json` 格式的配置文件（cu配置表, cu和target_id映射表, tms/zk ip port等）。
    - **集成:** `ConfigManager` 依赖此库。
- **spdlog:**
    - **用途:** 在整个应用程序中提供高性能的、可配置的日志记录。
    - **集成:** 创建一个全局可访问的 `LoggerWrapper` 或直接在各模块中使用 `spdlog` 实例。日志级别应可配置。
- **Redis (用于主备容灾 - 可选的增强功能):**
    - **用途:**
        1. **会话状态同步/持久化:** 当 `Session` 对象的状态发生关键变化（创建、激活、媒体信息更新、拆除）时，可以将这些状态异步地写入 Redis。
        2. **主备切换:** 如果主 Bridge 实例故障，备用 Bridge 实例启动后可以从 Redis 读取活动的会话信息，尝试恢复会话（例如，重新建立媒体路径，或至少知道哪些会话存在过以便进行清理或通知）。
        3. **分布式锁/领导者选举 (可选):** 如果有多个Active的Bridge实例（更复杂的负载均衡场景），或者严格的主备切换，可能需要。
        4. **(可选) 分布式端口管理:** 如果多个Bridge实例需要从一个全局唯一的端口池中分配端口。
    - **集成:**
        - `SessionManager` 或 `Session` 对象在状态变更时，会产生一个任务（或直接调用）与 Redis 交互的模块。
        - 需要一个 `RedisClient` 或 `RedisInterface` 模块来封装与 Redis 服务器的通信 (可以使用如 `redis-plus-plus` 或 `hiredis` C/C++ 客户端库)。
        - **考虑点:**
            - 写入 Redis 的频率和数据量（避免成为瓶颈）。
            - 数据一致性与序列化/反序列化。
            - Redis 的部署和可用性。
            - 会话恢复的复杂性（重新建立 UDP 媒体流通常比 TCP 连接恢复更复杂，因为UDP无状态）。对于媒体，可能更多的是知道会话存在，然后尝试快速拆除或指示对端重建。

---

这个结构设计考虑了您最新的需求和我们之前的讨论，旨在提供一个清晰、高效且可扩展的 C++ Bridge 系统蓝图。实际实现时，还需要在异步编程、错误处理、线程同步细节上进行细致的工作。




好的，我们来基于之前的讨论，开始整理一份更详细的设计文档。我们将从您最关注的 `protocols/zk_protocol/` 模块开始，然后逐步扩展到其他模块与它交互的部分，以及其他模块自身的一些关键设计点。

这个过程我会尽量细化，但请记住，真正的详细设计是一个非常细致的工作，我们在这里主要是构建一个清晰的框架和关键要素。

---

## Bridge 系统详细设计 (草案 V1)

**文档目标：** 明确各模块的核心职责、接口、关键数据结构和交互方式，为后续编码实现提供指导。本设计基于I/O复用、事件驱动和多线程（I/O线程 + 工作线程池）的架构模型。

---

### **第一部分：协议层 (Protocol Layer)**

#### **模块 1.1: `protocols/zk_protocol/` - ZK 协议处理模块**

1. 概述 (Overview):

定义Bridge与ZK实体间通信协议的消息数据结构，并提供将原始字节流与这些结构体相互转换（编码/解码）的纯功能性方法。本模块不处理网络I/O，也不包含UDP之上的自定义可靠传输逻辑（如重传、ACK状态管理）。

**2. 主要职责 (Key Responsibilities):**

- 定义所有ZK上下行协议消息的C++数据结构/类，包括公共头部和特定消息的载荷。
- 提供解码（解析）功能：将接收到的原始ZK UDP数据包字节流，根据公共头部中的特定字段（如消息类型、信息单元类型）识别消息，并转换为对应的C++消息结构体。支持解析包含多个“信息单元”的复合消息。
- 提供编码（格式化）功能：将内部C++消息结构体序列化为待发送的UDP字节流。
- 处理协议字段的字节序转换（网络字节序 &lt;-> 主机字节序）。

**3. 主要输入 (Key Inputs to its functions):**

- 解码函数输入: `const unsigned char* buffer`, `size_t length` (原始字节流及其长度)。
- 编码函数输入: `const SpecificZkMessage& message_struct` (待编码的C++消息结构体对象)。

**4. 主要输出 (Key Outputs from its functions):**

- 解码函数输出: `std::optional<ParsedZkMessageVariant>` (包含解析后的C++消息结构体，或在失败时为空) 或包含错误码的特定结果类型。
- 编码函数输出: `std::vector<unsigned char>` (包含序列化后的字节流) 或错误码。

**5. 依赖的其他模块 (Dependencies):**

- C++标准库 (如 `<vector>`, `<string>`, `<cstdint>`, `<variant>`, `<optional>`, `<algorithm>`)。
- 字节序转换函数 (如系统提供的 `htons`, `ntohl` 等，通常在 `<arpa/inet.h>` 或由 `common/utils/endian_converter.h` 封装)。

**6. 被哪些模块依赖 (Depended On By):**

- `interfaces/zk_interface/` (主要使用者，调用本模块的编解码函数)。

**7. 核心对外接口 (Key Public APIs - 初步设想):**

- **`zk_message_types.h`**:
    
    C++
    
    ```
    #include <cstdint>
    #include <string>
    #include <vector>
    #include <variant>
    #include <optional>
    
    namespace ZkProtocol {
    
    // --- 枚举定义 ---
    enum class MessageType : uint16_t { // 假设在公共头部中定义
        INITIATE_CALL_REQUEST = 0x0001,
        INITIATE_CALL_RESPONSE = 0x0002,
        CALL_DISCONNECT_REQUEST = 0x0003,
        CALL_DISCONNECT_RESPONSE = 0x0004,
        HEARTBEAT_REQUEST = 0x0005,
        HEARTBEAT_RESPONSE = 0x0006,
        MEDIA_DATA_FORWARD = 0x0100, // 用于媒体数据包，由MediaRelay调用
        // ... 其他信令消息类型
        UNKNOWN = 0xFFFF
    };
    
    enum class ReliabilityFlags : uint8_t {
        NONE = 0x00,
        ACK_REQUESTED = 0x01,
        IS_ACK = 0x02,
        // ... 其他可靠性相关的标志
    };
    
    // --- 公共消息头 ---
    struct CommonHeader {
        uint32_t sequence_number;    // 序列号 (用于可靠性层)
        uint32_t ack_number;         // 确认号 (用于可靠性层)
        ReliabilityFlags flags;       // 标志位 (用于可靠性层)
        MessageType message_type;    // 消息类型 (用于分发到具体消息解析)
        uint16_t total_length;       // 整个消息的总长度 (包括头部和载荷)
        // ... 其他通用头部字段，如版本号，源/目标实体ID（如果协议有规定）
    };
    constexpr size_t COMMON_HEADER_SIZE = sizeof(CommonHeader); // 注意对齐和实际打包大小
    
    // --- 特定信令消息结构体 ---
    struct InitiateCallRequest {
        std::string target_id;       // 被叫号码
        uint32_t bandwidth_kbps;    // 带宽需求
        // ... 其他特定字段
    };
    
    struct InitiateCallResponse {
        bool success;
        std::string zk_session_id;   // ZK侧会话标识
        uint16_t bridge_media_port_for_zk; // Bridge分配给ZK用于发送媒体的端口 (Bridge接收ZK媒体)
                                           // **更正: 此处应为Bridge分配用于从ZK接收媒体的端口，
                                           //  或ZK告知Bridge它将用于发送媒体的端口，具体看协议设计。
                                           //  假设此处是Bridge告知ZK：“你的媒体请发往我的这个端口X”
                                           //  或者，如果是ZK先指定，那这里是ZK的端口。
                                           //  根据用户描述 "zk始终用一个端口来接收200路的业务消息，
                                           //  因为发送给zk消息时会通过一个字段来区分不同的session业务"
                                           //  以及 "bridge对于每个zk有一个信令端口和一路会话就一个业务端口（向zk发送的）"
                                           //  这暗示Bridge分配一个端口用于 *接收* ZK特定会话的媒体，
                                           //  或者ZK发往Bridge的共享信令/媒体端口后，Bridge通过包内字段区分。
                                           //  此处我们假设是Bridge分配用于*接收*ZK媒体的端口。
                                           //  但用户最新描述是ZK所有业务都从一个端口发出到Bridge的对应ZK信令端口。
                                           //  所以此字段可能不需要，或表示其他含义。
                                           //  **暂时以此为Bridge分配用于接收ZK媒体的端口，如果ZK会用不同端口发媒体的话。
                                           //  如果ZK所有媒体都发往Bridge的ZK信令端口，则此字段无直接对应。**
    
                                           // **根据最新理解，ZK所有东西都发给Bridge的ZK信令/媒体统一入口。
                                           // 此处可能为Bridge分配用于 *向ZK发送媒体* 的本地源端口 (ZK会看到来自此端口的媒体)**
        std::string error_message;
    };
    
    // ... 其他信令消息的结构体 ...
    
    
    // --- 用于媒体转发的特定结构 (由MediaRelay调用本模块的编码函数) ---
    // 这个结构可能不是直接通过网络发送的“消息类型”，而是MediaRelay需要封装的数据
    struct MediaPacketToZk {
        CommonHeader common_zk_header; // ZK协议的公共头部 (序列号等由可靠性层填充)
        // 这里是纯媒体负载，实际的“加包头”就是填充上面的CommonHeader
        // 并确保整个包符合ZK协议对媒体数据包的定义
    };
    
    
    // --- 解析结果的Variant ---
    using ParsedZkMessagePayload = std::variant<
        std::monostate, // 表示空或未知，或只有公共头部
        InitiateCallRequest,
        InitiateCallResponse
        // ... 其他特定信令消息的载荷结构体
    >;
    
    struct DecodedZkMessage {
        CommonHeader header;
        ParsedZkMessagePayload payload;
        // 可以加入来源ZK的标识，如果顶层解析函数能获取到
    };
    
    
    } // namespace ZkProtocol
    ```
    
- **`zk_parser.h`**:
    
    C++
    
    ```
    #include "zk_message_types.h"
    #include <optional>
    
    namespace ZkProtocol::Parser {
    
    // 顶层解析函数，负责识别消息类型并分发
    std::optional<DecodedZkMessage> decode_packet(const unsigned char* buffer, size_t len);
    
    // (以下可能是内部辅助函数，或根据需要暴露)
    // ZkMessageType get_message_type_from_header(const CommonHeader& header);
    // std::optional<InitiateCallRequest> decode_initiate_call_request_payload(const unsigned char* payload_buffer, size_t payload_len);
    // ... 其他特定消息载荷的解码函数 ...
    
    } // namespace ZkProtocol::Parser
    ```
    
- **`zk_formatter.h`**:
    
    C++
    
    ```
    #include "zk_message_types.h"
    #include <vector>
    
    namespace ZkProtocol::Formatter {
    
    // 顶层编码函数 (根据传入的结构体类型)
    std::vector<unsigned char> encode_message(const DecodedZkMessage& message); // 或者为每种消息提供单独的encode函数
    
    // (以下可能是内部辅助函数，或根据需要暴露)
    // std::vector<unsigned char> encode_initiate_call_request(const CommonHeader& header, const InitiateCallRequest& payload);
    // ... 其他特定消息的编码函数 ...
    
    // 特别为MediaRelay提供的，用于封装裸媒体数据为ZK媒体包
    // header_template 应包含除序列号等动态字段外的头部信息
    // reliability_handler (在zk_interface中) 会在发送前填写最终的序列号等
    std::vector<unsigned char> encode_media_packet_for_zk(
        const CommonHeader& header_template, // 可靠性字段由调用方（zk_interface的可靠性模块）填充
        const unsigned char* media_payload,
        size_t media_payload_len
    );
    
    } // namespace ZkProtocol::Formatter
    ```
    

**8. 关键内部数据结构 (Key Internal Data Structures - 初步设想):**

- 主要是上述 `zk_message_types.h` 中定义的各种消息结构体。
- 解析函数内部可能使用临时的、基于状态的解析器上下文结构，但这些不公开。

**9. 非功能性需求考量 (NFRs):**

- **性能:** 编解码操作必须非常高效，避免不必要的内存拷贝。
- **健壮性:** 能够优雅处理格式错误、不完整的数据包（例如，通过返回 `std::nullopt` 或特定的错误对象），并提供足够的错误信息供上层模块记录和处理。
- **正确性:** 严格按照ZK协议规范进行编解码，特别是字节序、字段长度和对齐。
- **可测试性:** 容易为各种消息类型和边界条件编写单元测试。

**10. 待讨论/未明确点 (Open Questions):**

- ZK协议是否有官方的、字节级的详细定义文档？（非常重要！）
- `CommonHeader` 中的 `message_type` 字段是如何确定的？它是否足以区分所有顶层消息？
- 如果消息体包含“信息单元个数”和多个可变类型/长度的“信息单元”，`decode_packet` 和相关载荷解码函数需要实现循环解析逻辑。
- 错误处理机制：解析失败时，是返回 `std::nullopt`，还是一个包含错误码和描述的 `Result` 类型对象？
- 字节序：明确协议所有多字节字段的字节序（应为网络字节序），并在编解码时处理。

---

#### **模块 1.2: `protocols/tms_protocol/` - TMS 协议处理模块**

- (设计思路和结构与 `protocols/zk_protocol/` 非常相似，只是针对TMS的协议规范进行定义和实现)
- 需要明确TMS信令是通过TCP还是UDP，以及其消息格式。

---

### **第二部分：通用基础模块 (Common Infrastructure)**

#### **模块 2.1: `common/communication/` - 通信层**

1. 概述 (Overview):

提供底层的、异步的网络Socket操作封装，与具体的ZK或TMS协议无关。与I/O复用机制（如epoll）集成。

**2. 主要职责 (Key Responsibilities):**

- 封装异步UDP Socket的创建、绑定、数据收发 (`async_udp_socket.{h,cpp}`)。
- 封装异步TCP客户端Socket的创建、连接、数据收发、断开 (`async_tcp_client.{h,cpp}`) (用于连接TMS信令)。
- （可选）封装异步TCP服务器Socket的创建、监听、接受连接 (`async_tcp_server.{h,cpp}`) (如果Bridge需要对外提供TCP管理接口)。
- 所有操作都应设计为非阻塞的，并能与外部的事件循环（如epoll）集成（例如，通过回调函数、`std::future` 或其他异步通知机制）。

**3. 核心对外接口 (Key Public APIs - 初步设想):**

- `AsyncUDPSocket::bind(ip, port)`
- `AsyncUDPSocket::async_receive_from(buffer, callback_on_received)`
- `AsyncUDPSocket::async_send_to(buffer, target_ip, target_port, callback_on_sent)`
- `AsyncTCPClient::async_connect(ip, port, callback_on_connected)`
- `AsyncTCPClient::async_send(buffer, callback_on_sent)`
- `AsyncTCPClient::async_receive(buffer, callback_on_received)`
- `AsyncTCPClient::close()`

**4. 依赖的其他模块 (Dependencies):**

- 操作系统Socket API (`<sys/socket.h>`, `<netinet/in.h>`, `<arpa/inet.h>`等，或Boost.Asio等网络库)。
- I/O复用机制的封装 (可能在内部实现，或依赖外部事件循环)。

**5. 被哪些模块依赖 (Depended On By):**

- `interfaces/zk_interface/`
- `interfaces/tms_interface/`

#### **模块 2.2: `common/task_queue/` - 线程安全任务队列**

1. 概述 (Overview):

提供一个线程安全的队列，用于I/O线程向工作线程池分发处理任务。

**2. 主要职责 (Key Responsibilities):**

- 实现线程安全的入队 (`push`) 和出队 (`pop`/`try_pop`) 操作。
- 支持阻塞等待（当队列为空时，工作线程可以等待）和唤醒机制。

**3. 核心对外接口 (Key Public APIs - 初步设想):**

C++

```
template<typename T> // T 是任务对象的类型
class ConcurrentQueue {
public:
    void push(T task);
    bool try_pop(T& out_task);
    T wait_and_pop(); // 阻塞直到有任务
    // ...
};
```

- 任务对象 `T` 可以是一个 `std::function<void()>`，或者一个包含处理所需数据的自定义结构体。

**4. 依赖的其他模块 (Dependencies):**

- C++标准库 (`<queue>`, `<mutex>`, `<condition_variable>`)。

**5. 被哪些模块依赖 (Depended On By):**

- I/O线程 (生产者)
- 工作线程池 (消费者)

#### **模块 2.3: `logging/` - 日志模块**

1. 概述 (Overview):

提供全局的、可配置的、高性能的日志记录功能。推荐使用 spdlog。

**2. 主要职责 (Key Responsibilities):**

- 初始化日志系统（例如，设置日志级别、输出格式、目标文件/控制台）。
- 提供不同日志级别的记录接口（DEBUG, INFO, WARN, ERROR, CRITICAL）。
- （可选）支持异步日志记录以减少对业务线程的影响。

**3. 核心对外接口 (Key Public APIs - 初步设想):**

- (通常是对 `spdlog` 的简单封装或直接使用其API)
- `Logger::init(...)`
- `LOG_DEBUG("message with arg {}", arg1)`
- `LOG_INFO(...)`, `LOG_WARN(...)`, `LOG_ERROR(...)`

**4. 被哪些模块依赖 (Depended On By):**

- 几乎所有其他模块。

---

### **第三部分：核心逻辑模块 (Core Logic)**

#### **模块 3.1: `core_logic/config_manager/` - 配置管理器**

1. 概述 (Overview):

负责在程序启动时加载所有必要的配置信息，并提供给其他模块访问。

**2. 主要职责 (Key Responsibilities):**

- 从配置文件（例如JSON格式）中读取：
    - Bridge自身的监听设置（例如，用于ZK信令的10个端口，TMS信令端口）。
    - TMS服务器的连接信息 (IP, Port)。
    - ZK实体的基础信息列表 (如果每个ZK有特定配置需要预加载)。
    - "cu配置表" 和 "cu和target_id映射表"。
    - 端口管理器可用的端口范围。
    - 日志级别、线程池大小等系统参数。
- 解析并校验配置信息的有效性。
- 提供线程安全的接口供其他模块查询配置。配置通常在启动后是只读的。

**3. 核心对外接口 (Key Public APIs - 初步设想):**

- `ConfigManager::instance().load_config(const std::string& filepath);`
- `const TmsConfig& ConfigManager::instance().get_tms_config();`
- `std::optional<ZkConfig> ConfigManager::instance().get_zk_config(const ZkId& zk_id);`
- `const CuConfigTable& ConfigManager::instance().get_cu_config_table();`
- `// ...`

**4. 依赖的其他模块 (Dependencies):**

- JSON解析库 (例如 `nlohmann/json`)。
- C++标准库 (`<fstream>`, `<string>`, `<vector>`, `<map>`)。

**5. 被哪些模块依赖 (Depended On By):**

- 几乎所有其他模块在初始化或运行时都需要配置信息。

---

**(接下来的模块定义，如 `interfaces/*`, `Coordinator`, `SessionManager`, `Session`, `PortManager`, `MediaRelay` 将遵循类似的模板，细化其职责、接口和关键数据结构，并特别强调它们如何在多线程和事件驱动模型下工作，以及如何保证线程安全。) **

**例如，对于 `Session` 对象，其核心职责之一就是实现一个状态机来管理呼叫的完整生命周期，它的方法（如 `handleZkEvent`）将在工作线程中被调用。**

**对于 `MediaRelay`，其内部的转发表在被工作线程（由Session逻辑驱动）更新时需要加锁，而I/O线程在读取转发表进行媒体转发时，可以使用更轻量级的同步机制（如读写锁或无锁数据结构，如果适用且实现正确的话），或者保证转发表的更新是非常快速的原子操作（例如替换指针）来最小化对I/O线程的影响。**

---

这个详细设计过程会比较耗时，但对于后续的顺利开发至关重要。我们可以先集中精力把 `protocols/zk_protocol/` 模块的 **核心对外接口 (特别是消息结构体和编解码函数签名)** 确定下来，因为这是很多其他模块的基础。

然后我们可以讨论 `interfaces/zk_interface/` 如何使用它，以及它如何与I/O线程和工作线程池的任务队列交互。

您觉得这个方向和细化程度如何？我们可以先针对 `protocols/zk_protocol/` 把第7点“核心对外接口”再具体化一下吗？例如，您ZK协议中最重要的几条信令消息是什么？它们包含哪些关键字段？






好的，这绝对是一个关键的里程碑！现在，我们将把之前所有关于 Bridge 系统架构的讨论，包括模块划分、核心职责、I/O模型、多线程、协议处理细节，以及最新加入的**主备容灾**考量，整合到一份结构设计文档中。

这份文档将作为您后续详细设计和编码的蓝图。

---

## Bridge 系统结构设计说明文档 V1.0 🌉

文档版本： 1.0

日期： 2025-05-27 (根据您当前时间调整)

作者： (您的名字/团队)

### 1. 引言 📜

#### 1.1. 项目概述

Bridge 系统是一个中间件程序，旨在作为 ZK 实体（最多10个，每个ZK使用单一固定端口进行信令和媒体通信）与单一 TMS 系统之间的通信桥梁。它负责在两者之间进行信令交互的协议适配和媒体流的转发。系统设计目标是支持最多200路并发会话，并具备主备容灾能力。

#### 1.2. 设计目标

- **功能性：** 准确实现ZK与TMS间的信令转换和媒体中继。
- **性能：** 高效处理200路并发会话，低延迟。
- **可靠性：** 系统稳定运行，具备主备容灾能力，减少单点故障时间。
- **可维护性：** 模块化设计，代码清晰，易于理解、修改和扩展。
- **可扩展性：** 架构上为未来可能的会话数量增长或功能增强留有余地。

#### 1.3. 关键约束

- C++11 作为开发语言标准。
- 目标运行环境：麒麟服务器操作系统 (Linux 4.4.131 aarch64, GCC 5.4.0)。
- ZK侧通信：每个ZK使用一个固定端口与Bridge的对应ZK监听端口进行所有信令及所有会话的媒体数据交互；Bridge向ZK发送媒体时，使用为该会话分配的独立源端口。
- TMS侧通信：协议待明确（假设TCP信令，UDP媒体）。

---

### 2. 系统总体架构 🏗️

Bridge 系统采用**模块化、事件驱动、I/O复用（基于epoll或等效机制）和多线程（I/O线程 + 工作线程池）**的架构。

#### 2.1. 架构图 (概念)

```
+---------------------+      +-----------------------+      +---------------------+
|     ZK Entities     |<---->|      ZK Interface     |<---->|                     |
| (10 ZKs, fixed port |      | (10 Listeners for ZK  |      |                     |
|  for Sig & Media In)|      |  Sig/Media, Demux,    |      |   Coordinator /   |
+---------------------+      |  Reliability Handler, |      |   Event Router    |      +-------------------+
                             |  Protocol Util)       |      |                     |----->|  Session Module   |
                             +-----------+-----------+      +---------------------+      |<-----| (SessionManager,  |
                                         ^                                               |      |  Session Objects) |
                                         | (Tasks to Worker Pool)                        |      +-------------------+
                                         v                                               |                |
+---------------------+      +-----------+-----------+      +-----------------------+  |                | (Config MediaRelay)
|     TMS System      |<---->|     TMS Interface     |<---->|   Worker Thread Pool  |  |                v
| (TCP Sig, N UDP     |      | (TCP Conn, N UDP Listen|      | (Executes Session Logic,|  |      +-------------------+
|  Media In)          |      |  for TMS Media, Proto |      |  Coordinator, etc.)   |<---------->| MediaRelay Module |
+---------------------+      |  Util)                |      +-----------------------+  |      |(Fast Fwd Table,   |
                             +-----------+-----------+                                   |      | Header Manip.)    |
                                         ^                                               |      +-------------------+
                                         | (Tasks to Worker Pool)                        |
                                         v                                               |  ^
                             +-----------------------+                                   |  | (Alloc/Release)
                             | I/O Threads &         |                                   v  |
                             | Event Loop (epoll)    |      +-----------------------+     +-----------------------+
                             +-----------------------+      | Communication Layer   |<--->|    Protocol Layer     |
                                     ^       ^              | (Async Sockets)       |     | (ZK/TMS Parsers/Fmts) |
                                     |       |              +-----------------------+     +-----------------------+
                                     |       |
                                     |       +-------------------------------------------->+    Port Manager       |
                                     |                                                     | (UDP Port Pool)       |
                                     +----------------------------------------------------->+-----------------------+

+-----------------------+     +-----------------------+     +-----------------------+
| Configuration Manager |     |     Logging (spdlog)  |     | Redis Client Interface|
| (JSON for config)     |     +-----------------------+     | (for HA State Sync)   |
+-----------------------+                                   +-----------------------+
                                                                      ^
                                                                      | (HA Heartbeat & State)
                                                                      v
                                                            +-----------------------+
                                                            | HA Switch Logic Module|
                                                            | (e.g., ms_switch)     |
                                                            +-----------------------+
```

#### 2.2. 线程模型

- **I/O 线程 (少量)：**
    - 运行核心I/O复用循环 (如 `epoll_wait`)。
    - 负责所有Socket的非阻塞读写。
    - 进行最小化解析以识别消息来源和基本类型。
    - 将任务分发到工作线程池。
    - 可能直接处理极低延迟的媒体转发（如果包头操作简单）。
- **工作线程池 (固定数量)：**
    - 执行所有核心业务逻辑：`Session`状态机、`Coordinator`路由、配置读取、端口分配、`MediaRelay`配置。
    - 处理需要CPU计算或可能短时阻塞（非网络I/O）的操作。
- **(可选) 专用线程：** TMS心跳、定时清理、HA状态监控等。

---

### 3. 模块详细设计 🧩

#### 3.1. `common/` - 通用基础模块

- **`common/communication/` - 通信层**
    - **概述：** 提供底层的、异步的、平台无关（或已适配目标平台）的Socket操作封装。
    - **主要职责：** 封装异步UDP Socket的创建、绑定、收发；封装异步TCP客户端/服务器Socket操作。与I/O事件循环集成。
    - **核心接口：** `AsyncUDPSocket::async_receive_from()`, `AsyncUDPSocket::async_send_to()`, `AsyncTCPClient::async_connect()`, `AsyncTCPClient::async_send()` 等。
    - **主要数据结构：** Socket描述符、缓冲区、回调函数对象。
- **`common/task_queue/` - 任务队列**
    - **概述：** 实现线程安全的任务队列，用于I/O线程与工作线程池之间的任务传递。
    - **核心接口：** `ConcurrentQueue::push(Task)`, `ConcurrentQueue::wait_and_pop()`。

#### 3.2. `protocols/` - 协议层

- **`protocols/zk_protocol/` - ZK协议处理**
    - **概述：** 定义ZK协议消息的C++数据结构，提供纯粹的编解码功能（字节流 <-> C++结构体），处理字节序。**不包含自定义可靠传输的状态逻辑。**
    - **主要职责：**
        - 定义 `CommonHeader` (序列号、ACK号、标志位、源/目标实体ID、信息单元数量)。
        - 定义 `InformationUnitType` 枚举。
        - 定义每种 `InformationUnitType` 对应的载荷结构体 (`XxxIU`)。
        - 定义顶层已解析ZK UDP包结构体 `ParsedZkUdpPacket { CommonHeader header; std::vector<std::shared_ptr<BaseIU>> information_units; }` (其中 `BaseIU` 为信息单元的基类，具体单元为其派生类，以适应C++11下 `std::variant` 的缺失)。
    - **核心接口：**
        - `ZkProtocol::Parser::decode_common_header(buffer, len)`
        - `ZkProtocol::Parser::decode_information_unit_payload(buffer, len, type, BaseIU*& out_unit)` (或多个特定类型的解码函数)
        - `ZkProtocol::Parser::decode_packet(buffer, len, ParsedZkUdpPacket& out_packet)` (顶层解析，循环解析信息单元)
        - `ZkProtocol::Formatter::encode_packet(const ParsedZkUdpPacket& packet)` (顶层编码)
        - `ZkProtocol::Formatter::encode_raw_media_for_zk(CommonHeader& header_template, payload, len)` (供MediaRelay使用，可靠性字段由上层填充)
- **`protocols/tms_protocol/` - TMS协议处理**
    - **概述与职责：** 同ZK协议模块，但针对TMS协议。

#### 3.3. `interfaces/` - 接口适配层

- **`interfaces/zk_interface/` - ZK接口适配器**
    - **概述：** 处理与所有ZK实体的网络通信、协议编解码调用、自定义可靠UDP传输机制（如果ZK协议需要），并将业务事件路由到 `Coordinator`。
    - **主要职责：**
        - **`ZkListener`:** 管理10个UDP监听Socket（每个ZK一个，接收其信令和所有会话的媒体）。使用`common/communication`和`epoll`。读取数据后，调用`protocols/zk_protocol`初步解析。
        - **`ZkReliabilityHandler` (如果ZK协议有自定义可靠性):** 基于`CommonHeader`中的序列号/ACK/标志位实现消息的可靠发送（重传、超时）和接收（去重、排序、发送ACK）。它会使用`protocols/zk_protocol`来编解码头部字段，但可靠性状态机在此模块。
        - **内容分发：** 解析ZK包头（或由`ZkReliabilityHandler`处理后的包）中的“信息单元会话标识”和“信息单元类型”。
            - 信令类信息单元：封装成任务，通过`Coordinator`提交给`Session`处理。
            - 媒体类信息单元：直接（或通过高优先级任务）交给`MediaRelay`处理。
        - **`ZkSender`:** 接收来自`Coordinator`（源自`Session`或`MediaRelay`）的发送指令，调用`ZkReliabilityHandler`（如果需要可靠性）或直接调用`protocols/zk_protocol`格式化消息，并通过`common/communication`发送。
    - **核心数据结构：** 每个ZK监听Socket的上下文，可靠传输相关的状态表（如发送窗口、重传队列）。
- **`interfaces/tms_interface/` - TMS接口适配器**
    - **概述与职责：** 类似ZK接口，但针对TMS。管理与TMS的TCP信令连接（心跳），管理N个UDP Socket接收来自TMS的媒体。

#### 3.4. `core_logic/` - 核心业务逻辑模块

- **`core_logic/coordinator/` - 协调器**
    - **概述：** 运行在工作线程中，作为接口层和会话模块之间的逻辑路由器。
    - **主要职责：** 接收来自接口层（`zk_interface`, `tms_interface`）的已解析业务事件/信令，将其路由到`SessionManager`（用于查找或创建`Session`）或直接路由到已存在的`Session`对象的方法。反之，接收来自`Session`对象的指令，将其路由到相应的接口层进行网络发送。
- **`core_logic/session_manager/` - 会话管理器**
    - **概述：** 运行在工作线程中，管理所有活动会话的生命周期。
    - **主要职责：** 维护一个**线程安全**的会话表（如 `std::unordered_map<SessionID, std::shared_ptr<Session>>`）。负责会话的创建、查找、销毁。
    - **与HA/Redis交互：** 在会话创建、关键状态变更、销毁时，**异步**地将必要信息写入Redis（通过`RedisClientInterface`），以供备机恢复。
- **`core_logic/session/` - 会话对象**
    - **概述：** 运行在工作线程中，封装单个端到端呼叫的核心状态机和业务流程。
    - **主要职责：**
        - 管理会话状态（初始化、连接中、激活、拆除中等）。
        - 处理来自ZK或TMS的特定信令事件（由`Coordinator`路由而来）。
        - 决策并触发向ZK或TMS发送信令（通过`Coordinator`）。
        - 调用`PortManager`分配/释放媒体端口。
        - 调用`MediaRelay`配置/清除媒体转发规则。
        - 管理会话相关的定时器（如呼叫超时）。
    - **核心数据结构：** 会话ID，ZK/TMS端点信息，分配的Bridge媒体端口，当前状态，关联的CU配置等。
- **`core_logic/port_manager/` - 端口管理器**
    - **概述：** 运行在工作线程中（或其操作被封装为任务），管理Bridge可用的UDP媒体端口池。
    - **主要职责：** 提供**线程安全**的端口分配和回收接口。
- **`core_logic/config_manager/` - 配置管理器**
    - **概述：** 程序启动时加载配置，之后提供只读访问。
    - **主要职责：** 解析JSON配置文件，提供对ZK/TMS连接信息、CU配置表、映射表等的访问。

#### 3.5. `media_processing/media_relay/` - 媒体中继模块

- **概述：** 负责高效的UDP媒体包转发。
- **主要职责：**
    - 维护一个**线程安全**的、高效的内部媒体转发表（`ActiveMediaFlowTable`），键为Bridge本地接收媒体的UDP端口（对于TMS侧）或ZK会话标识（对于ZK共享入口侧），值为转发规则（目标IP/端口、包头处理函数、发送用的Bridge源端口）。
    - 由`Session`对象在会话建立/拆除时调用其接口来配置/清除转发规则。
    - **转发逻辑（可能在I/O线程中直接执行以降低延迟）：**
        - 收到媒体包后，快速查转发表。
        - 执行指定的“加/去ZK通用报头”操作 (调用`protocols/zk_protocol`的媒体封装/解封装函数)。
        - 通过`common/communication`将处理后的包异步发送出去。
    - （可选）实现简单的媒体包缓冲机制。

#### 3.6. `ha_module/` (或 `ms_switch` 相关逻辑的封装) - 主备容灾模块

- **概述：** 负责实现主备切换逻辑，与Redis交互。
- **主要职责：**
    - **`HaSwitchLogic`:** (类似 `ms_switch.h` 中的功能)
        - 初始化（`switch_init`）：判断初始角色（主/备），连接Redis。
        - 心跳机制：主实例定期向Redis写入心跳；备实例监控心跳。
        - 故障检测：备实例检测到主实例心跳超时。
        - 切换执行（`switch_func`, `system_switch`）：
            - 备实例将自己提升为主，更新Redis中的主实例标识。
            - （如果需要）执行IP漂移等网络配置变更。
            - 通知各核心模块（尤其是 `SessionManager`）系统角色已改变。
        - 信号处理：响应外部信号（如 `SIGUSR1`, `SIGUSR2`）进行角色切换或状态上报。
    - **`RedisClientInterface` (或直接使用 `hiredis` 封装):**
        - 封装与Redis服务器的连接、命令执行（SET EX, GET, DEL, PUBLISH/SUBSCRIBE等）。
        - 供 `HaSwitchLogic` 和 `SessionManager` 等模块使用。
    - **主要数据结构:** 当前系统角色（主/备），对端心跳状态。

#### 3.7. `logging/` - 日志模块

- **概述：** 基于 `spdlog` 实现全局日志记录。
- **主要职责：** 初始化日志配置，提供各级别日志记录接口。

---

### 4. 主要交互流程示例 (简述)

1. **ZK发起呼叫：**
    - ZK -> `ZkInterface` (ZK监听端口) -> I/O线程读取 -> 初步解析 -> 任务入队
    - 工作线程获取任务 -> `ZkInterface` (可靠性处理，如需要) -> `Coordinator` -> `SessionManager` (创建`Session`对象) -> `Session`对象 (`onZkInitiateRequest`):
        - 更新状态。
        - 调用 `ConfigManager` 获取配置。
        - 调用 `PortManager` 分配Bridge媒体端口。
        - **(HA)** 将初始会话信息异步写入Redis。
        - 通过 `Coordinator` 指示 `TmsInterface` 向TMS发起呼叫。
        - 通过 `Coordinator` 指示 `ZkInterface` 回复ZK初步响应。
2. **媒体流建立与转发：**
    - `Session` 对象在收到TMS摘机和媒体端口信息后：
        - 更新状态。
        - **(HA)** 更新Redis中的会话媒体信息。
        - 调用 `MediaRelay::setupFlow()` 配置双向媒体转发规则。
    - `MediaRelay` (可能在I/O线程中):
        - 收到ZK媒体（在ZK信令/媒体共享端口，需根据包内会话ID查找转发表） -> 去ZK协议头 -> 查转发表 -> 加TMS（如有必要）或直接转发 -> 发送给TMS。
        - 收到TMS媒体（在Bridge为该会话分配的TMS媒体接收端口） -> 查转发表（用本地端口做键）-> 加ZK协议头 -> 发送给ZK（使用Bridge为该会话分配的对ZK发送端口）。
3. **主备切换：**
    - `HaSwitchLogic` (备) 检测到主故障 (如Redis心跳超时)。
    - `HaSwitchLogic` (备) 执行切换：更新Redis主标识，(可选)IP漂移。
    - `HaSwitchLogic` (备) 通知 `SessionManager` 加载活动会话。
    - `SessionManager` (新主) 从Redis读取活动会话信息，重建轻量级`Session`对象或状态，尝试恢复媒体路径（这部分较复杂，可能简化为通知对端重邀或清理）。

---

### 5. 第三方库使用

- **Boost.Asio:** 推荐用于 `common/communication/` 实现异步网络I/O。
- **nlohmann/json:** 用于 `ConfigManager` 解析JSON配置文件。
- **spdlog:** 用于 `logging/` 实现日志记录。
- **hiredis (C客户端):** 用于 `RedisClientInterface` (或直接在 `HaSwitchLogic`, `SessionManager` 中封装) 与Redis通信。
- **(可选) Google Test / Catch2:** 用于单元测试。

---

这份文档提供了一个更全面的结构设计框架。您可以基于此，针对每个模块进一步细化其内部类设计、函数签名、错误处理、以及与多线程相关的同步机制。在实现过程中，您会不断地回顾和调整这份设计。祝您项目顺利！





[MODE: PLAN]

现在我将创建Bridge系统的详细技术实现规范。基于我们在RESEARCH和INNOVATE阶段的分析，我将制定完整的开发计划。

## 项目结构设计

```
bridge_system/
├── src/
│   ├── core/
│   │   ├── session_manager.h/cpp     # Session管理核心
│   │   ├── config_manager.h/cpp      # 配置文件管理
│   │   └── port_pool.h/cpp           # 端口池管理
│   ├── network/
│   │   ├── tcp_client.h/cpp          # TMS TCP客户端
│   │   ├── tcp_server.h/cpp          # TMS TCP服务端
│   │   ├── udp_handler.h/cpp         # UDP处理基类
│   │   └── socket_utils.h/cpp        # Socket工具函数
│   ├── protocol/
│   │   ├── tms_protocol.h/cpp        # TMS协议处理
│   │   ├── zk_protocol.h/cpp         # ZK协议处理
│   │   └── message_types.h           # 消息类型定义
│   ├── media/
│   │   ├── media_relay.h/cpp         # 媒体转发核心
│   │   └── media_thread.h/cpp        # 媒体处理线程
│   ├── threads/
│   │   ├── signal_thread.h/cpp       # 信令处理线程
│   │   └── thread_manager.h/cpp      # 线程管理
│   └── utils/
│       ├── logger.h/cpp              # 日志系统
│       └── common_utils.h/cpp        # 通用工具
├── config/
│   └── bridge_config.ini             # 系统配置文件
├── CMakeLists.txt                    # CMake构建文件
├── README.md                         # 项目说明
└── main.cpp                          # 主程序入口
```

## 核心数据结构设计

### Session表结构 (session_manager.h)
```cpp
struct SessionInfo {
    // 唯一标识
    std::string session_id;           // zk_ip + target_id组合
    std::string zk_ip;
    std::string target_id;
    std::string cu_number;            // 电话号码
    
    // 端口信息
    int bridge_tms_recv_port;         // Bridge接收TMS媒体端口
    int bridge_zk_recv_port;          // Bridge接收ZK媒体端口
    
    // 地址信息
    struct sockaddr_in tms_send_addr; // TMS媒体发送地址
    struct sockaddr_in zk_send_addr;  // ZK媒体发送地址
    struct sockaddr_in zk_signal_addr;// ZK信令地址
    
    // 会话状态
    enum SessionState {
        IDLE, CALLING, ANSWERED, TALKING, HANGING, RELEASED
    } state;
    
    // 会话类型
    enum SessionType { P2P, CLUSTER } type;
    
    // 媒体线程分配
    int assigned_media_thread_id;     // 分配的媒体线程ID
    
    // 状态变更回调
    std::function<void(SessionState, SessionState)> state_change_callback;
};
```

### 媒体绑定配置结构 (media_relay.h)
```cpp
struct MediaBindingConfig {
    std::string session_id;
    int listen_port;                  // 监听端口
    struct sockaddr_in target_addr;   // 转发目标地址
    int feedback_socket_fd;           // 反馈socket句柄
    struct sockaddr_in feedback_addr; // 反馈目标地址
    
    // 预编译的转发函数指针
    std::function<void(const char*, size_t)> forward_function;
    std::function<void(const char*, size_t)> feedback_function;
};
```

## 详细实现规范

### 1. 配置管理模块 (config_manager.cpp)
```cpp
class ConfigManager {
private:
    // ZK配置列表
    struct ZKConfig {
        std::string zk_id;
        std::string ip;
        std::string bridge_ip;
        int port;
    };
    std::vector<ZKConfig> zk_configs;
    
    // Target_ID映射表
    std::unordered_map<std::string, std::string> target_cu_map;
    
    // 端口池配置
    int port_pool_start;
    int port_pool_end;
    
    // TMS服务器配置
    std::string tms_server_ip;
    int tms_server_port;
    
    // Bridge服务器配置
    std::string bridge_server_ip;
    int bridge_server_port;

public:
    bool load_config(const std::string& config_file);
    const std::vector<ZKConfig>& get_zk_configs() const;
    std::string get_cu_by_target_id(const std::string& target_id) const;
    std::pair<int, int> get_port_pool_range() const;
};
```

### 2. Session管理器核心 (session_manager.cpp)
```cpp
class SessionManager {
private:
    // Session表 - 多键索引
    std::unordered_map<std::string, SessionInfo> sessions_by_id;           // zk_ip+target_id
    std::unordered_map<std::string, SessionInfo*> sessions_by_cu;          // cu号索引
    std::unordered_map<int, SessionInfo*> sessions_by_tms_port;            // TMS端口索引
    std::unordered_map<int, SessionInfo*> sessions_by_zk_port;             // ZK端口索引
    
    PortPool port_pool;
    ConfigManager* config_manager;
    
    // 媒体线程通信队列
    std::array<std::queue<MediaBindingConfig>, 4> media_thread_queues;
    std::array<std::mutex, 4> queue_mutexes;

public:
    // Session查找方法
    SessionInfo* find_session_by_cu(const std::string& cu);
    SessionInfo* find_session_by_id(const std::string& zk_ip, const std::string& target_id);
    SessionInfo* find_session_by_port(int port);
    
    // Session生命周期管理
    std::string create_session(const std::string& zk_ip, const std::string& target_id, SessionType type);
    void update_session_state(const std::string& session_id, SessionState new_state);
    void destroy_session(const std::string& session_id);
    
    // 媒体配置推送
    void push_media_binding(const MediaBindingConfig& config);
    
    // 状态上报
    void report_session_state_to_zk(const std::string& session_id);
    void report_all_sessions_to_zk(const std::string& zk_ip);
};
```

### 3. 协议处理模块

#### TMS协议处理 (tms_protocol.cpp)
```cpp
class TMSProtocol {
private:
    SessionManager* session_manager;
    
    // JSON-RPC处理函数映射
    std::unordered_map<std::string, std::function<void(const json&)>> method_handlers;
    
    void handle_ringing(const json& params);
    void handle_paging(const json& params);
    void handle_answer(const json& params);
    void handle_hangup(const json& params);

public:
    void process_tms_message(const std::string& json_message);
    std::string create_call_request(const std::string& cu, int recv_port);
    std::string create_hangup_request(const std::string& cu);
};
```

#### ZK协议处理 (zk_protocol.cpp)
```cpp
class ZKProtocol {
private:
    SessionManager* session_manager;
    
    // 信令类型处理函数映射
    std::unordered_map<int, std::function<void(const ZKMessage&)>> signal_handlers;
    
    void handle_query_signal(const ZKMessage& msg);
    void handle_link_control_signal(const ZKMessage& msg);

public:
    void process_zk_message(const char* data, size_t len, const std::string& source_ip);
    std::string create_link_status_response(const SessionInfo& session);
    std::string create_media_feedback(const std::string& session_id, FeedbackType type);
};
```

### 4. 媒体处理线程 (media_thread.cpp)
```cpp
class MediaThread {
private:
    int thread_id;
    int feedback_socket_fd;           // SO_REUSEPORT专用socket
    int epoll_fd;                     // epoll事件循环
    
    // 预绑定配置存储
    std::unordered_map<int, MediaBindingConfig> port_bindings;
    
    // 配置接收队列
    std::queue<MediaBindingConfig> config_queue;
    std::mutex config_mutex;

public:
    void initialize(int thread_id);
    void add_binding_config(const MediaBindingConfig& config);
    void run();                       // 主事件循环
    
private:
    void setup_feedback_socket();
    void handle_media_data(int port, const char* data, size_t len);
    void send_feedback(const std::string& session_id, FeedbackType type);
};
```

### 5. 信令处理线程 (signal_thread.cpp)
```cpp
class SignalThread {
private:
    SessionManager session_manager;
    TMSProtocol tms_protocol;
    ZKProtocol zk_protocol;
    
    // 网络组件
    TCPClient tms_client;
    TCPServer tms_server;
    UDPHandler zk_handler;
    
    // 定时器
    std::timer status_report_timer;

public:
    void initialize();
    void run();                       // 主事件循环
    
private:
    void handle_tms_client_message(const std::string& message);
    void handle_tms_server_message(const std::string& message);
    void handle_zk_message(const char* data, size_t len, const std::string& source_ip);
    void periodic_status_report();
};
```

### 6. 网络组件实现

#### TCP客户端 (tcp_client.cpp)
```cpp
class TCPClient {
private:
    int socket_fd;
    struct sockaddr_in server_addr;
    bool connected;

public:
    bool connect_to_tms(const std::string& ip, int port);
    bool send_message(const std::string& message);
    std::string receive_message();
    void disconnect();
};
```

#### UDP处理器 (udp_handler.cpp)
```cpp
class UDPHandler {
private:
    int socket_fd;
    struct sockaddr_in bind_addr;

public:
    bool bind_port(const std::string& ip, int port);
    ssize_t send_to(const char* data, size_t len, const struct sockaddr_in& addr);
    ssize_t receive_from(char* buffer, size_t buffer_size, struct sockaddr_in& from_addr);
};
```

### 7. 主程序入口 (main.cpp)
```cpp
int main(int argc, char* argv[]) {
    // 1. 加载配置文件
    ConfigManager config_manager;
    if (!config_manager.load_config("config/bridge_config.ini")) {
        return -1;
    }
    
    // 2. 初始化日志系统
    Logger::initialize("bridge_system.log");
    
    // 3. 创建并启动信令处理线程
    SignalThread signal_thread;
    signal_thread.initialize();
    std::thread signal_worker(&SignalThread::run, &signal_thread);
    
    // 4. 创建并启动4个媒体处理线程
    std::array<MediaThread, 4> media_threads;
    std::array<std::thread, 4> media_workers;
    
    for (int i = 0; i < 4; ++i) {
        media_threads[i].initialize(i);
        media_workers[i] = std::thread(&MediaThread::run, &media_threads[i]);
    }
    
    // 5. 等待所有线程完成
    signal_worker.join();
    for (auto& worker : media_workers) {
        worker.join();
    }
    
    return 0;
}
```

## 配置文件格式 (bridge_config.ini)
```ini
[ZK_CONFIGS]
zk1=192.168.1.10,192.168.1.100,5001
zk2=192.168.1.11,192.168.1.100,5001
zk3=192.168.1.12,192.168.1.100,5002

[TARGET_MAPPING]
target001=12345678
target002=87654321

[PORT_POOL]
start_port=20000
end_port=25000

[TMS_SERVER]
ip=192.168.1.50
port=8080

[BRIDGE_SERVER]
ip=192.168.1.100
port=9090
```

## CMake构建配置 (CMakeLists.txt)
```cmake
cmake_minimum_required(VERSION 3.16)
project(BridgeSystem)

set(CMAKE_CXX_STANDARD 17)

# 查找依赖库
find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBJSONRPC REQUIRED libjsonrpc)

# 设置源文件
file(GLOB_RECURSE SOURCES "src/*.cpp")

# 创建可执行文件
add_executable(bridge_system ${SOURCES})

# 链接库
target_link_libraries(bridge_system 
    ${LIBJSONRPC_LIBRARIES}
    pthread
)

# 设置包含目录
target_include_directories(bridge_system PRIVATE 
    src/
    ${LIBJSONRPC_INCLUDE_DIRS}
)
```

## 实施检查清单

IMPLEMENTATION CHECKLIST:
1. 创建项目目录结构和基础文件
2. 实现ConfigManager类和配置文件解析
3. 实现PortPool端口池管理类
4. 实现SessionManager核心类和Session表管理
5. 实现TMSProtocol类和JSON-RPC消息处理
6. 实现ZKProtocol类和ZK消息处理
7. 实现TCPClient类用于TMS客户端连接
8. 实现TCPServer类用于TMS服务端监听
9. 实现UDPHandler类用于ZK通信
10. 实现MediaThread类和媒体处理逻辑
11. 实现SignalThread类和信令处理逻辑
12. 实现Logger日志系统
13. 实现ThreadManager线程管理器
14. 编写main.cpp主程序入口
15. 创建CMakeLists.txt构建配置
16. 创建bridge_config.ini配置文件模板
17. 编写README.md项目文档
18. 集成libjsonrpc库依赖
19. 实现SO_REUSEPORT socket配置
20. 实现预绑定媒体转发机制
21. 实现状态上报和反馈机制
22. 实现会话生命周期管理
23. 实现多键索引的Session查找
24. 实现媒体线程的配置接收机制
25. 完成系统集成测试
