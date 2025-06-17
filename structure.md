![[

Bridge系统的详细技术实现规范。

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

## 实施检查清单:
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









## SessionManager模块设计探索

### 核心数据结构探索
```cpp
// Session基础信息结构
struct SessionInfo {
    std::string zk_ip;
    std::string target_id;
    std::string cu_number;
    
    // 端口信息
    int bridge_recv_from_tms_port;    // Bridge接收TMS媒体的端口
    int bridge_recv_from_zk_port;     // Bridge接收ZK媒体的端口
    
    // 发送地址信息
    sockaddr_in tms_send_addr;        // 发送给TMS的地址
    sockaddr_in zk_send_addr;         // 发送给ZK的地址
    sockaddr_in zk_signal_addr;       // ZK信令地址
    
    // 会话状态和类型
    enum SessionState state;
    enum SessionType type;            // P2P 或 CLUSTER
};
```

### 主要函数接口探索
```cpp
class SessionManager {
public:
    // 会话查找函数
    SessionInfo* find_session_by_cu(const std::string& cu);
    SessionInfo* find_session_by_zkip_targetid(const std::string& zk_ip, const std::string& target_id);
    
    // 会话创建和管理
    std::string create_session(const std::string& zk_ip, const std::string& target_id, SessionType type);
    bool update_session_field(const std::string& session_id, const std::string& field_name, const std::string& value);
    void destroy_session(const std::string& session_id);
    
    // 字段变更触发机制
    void set_field_change_callback(const std::string& field_name, std::function<void(SessionInfo&)> callback);
};
```

## Protocol模块设计探索

### TMS协议模块
```cpp
class TMSProtocol {
public:
    // 信令解析和处理
    void process_tms_message(const std::string& json_rpc_message);
    
    // 信令生成函数
    std::string generate_call_request(const std::string& cu, int recv_port);
    std::string generate_hangup_request(const std::string& cu);
    
private:
    // method处理函数
    void handle_ringing(const std::string& cu, const json& params);
    void handle_answer(const std::string& cu, const json& params);
    void handle_hangup_notification(const std::string& cu, const json& params);
};
```

### ZK协议模块
```cpp
class ZKProtocol {
public:
    // ZK信令处理
    void process_zk_signal(const char* data, size_t len, const std::string& source_zk_ip);
    
    // 状态信令生成
    std::string generate_link_status(const SessionInfo& session);
    std::string generate_media_feedback(const std::string& session_id, const std::string& feedback_type);
    
private:
    // 信令类型处理函数
    void handle_query_signal(const std::string& zk_ip);
    void handle_link_control(const std::string& zk_ip, const std::string& target_id, const std::string& control_type);
};
```

## MediaRelay模块设计探索

### 媒体绑定配置结构
```cpp
struct MediaBindingConfig {
    std::string session_id;
    int listen_port;                  // 要监听的端口
    sockaddr_in forward_addr;         // 转发目标地址
    sockaddr_in feedback_addr;        // 状态反馈地址
    int thread_feedback_socket_fd;    // 该线程的反馈socket
};
```

### MediaRelay接口设计
```cpp
class MediaRelay {
public:
    // 配置接收和设置
    void receive_binding_config(const MediaBindingConfig& config);
    
    // 媒体数据处理
    void handle_media_packet(int recv_port, const char* data, size_t len);
    
    // 状态反馈
    void send_receive_feedback(const std::string& session_id);
    void send_forward_feedback(const std::string& session_id);
    
private:
    // 数据转换处理
    void process_zk_to_tms(const char* zk_data, size_t len, char* tms_data, size_t* tms_len);
    void process_tms_to_zk(const char* tms_data, size_t len, char* zk_data, size_t* zk_len);
};
```

## 线程模块设计探索

### 信令线程设计
```cpp
class SignalThread {
public:
    void initialize(ConfigManager* config);
    void run();    // 主循环
    
private:
    // 信令处理函数
    void handle_tms_tcp_message(const std::string& message);
    void handle_zk_udp_message(const char* data, size_t len, const std::string& source_ip);
    
    // 定时任务
    void periodic_status_report();
};
```

### 媒体线程设计
```cpp
class MediaThread {
public:
    void initialize(int thread_id);
    void run();    // 主循环
    
    // 配置接收
    void add_media_binding(const MediaBindingConfig& config);
    
private:
    int thread_id;
    int feedback_socket_fd;           // SO_REUSEPORT socket
    
    // 配置存储
    std::map<int, MediaBindingConfig> port_to_config;
};
```

## 配置和工具模块探索

### 配置管理器
```cpp
class ConfigManager {
public:
    bool load_config_file(const std::string& file_path);
    
    // 配置查询函数
    std::string get_cu_by_target_id(const std::string& target_id);
    std::pair<std::string, int> get_tms_server_config();
    std::pair<std::string, int> get_bridge_server_config();
    std::vector<ZKConfig> get_zk_configs();
};
```

### 端口池管理
```cpp
class PortPool {
public:
    void initialize(int start_port, int end_port);
    int allocate_port();
    void release_port(int port);
    
private:
    std::queue<int> available_ports;
    std::set<int> allocated_ports;
};
```
]]