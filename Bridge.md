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