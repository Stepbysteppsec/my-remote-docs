# 网络设备监控日志系统 - 架构设计教学文档

## 1. 系统概述

### 1.1 功能需求

本系统是一个**实时网络设备监控与日志记录系统**，主要功能包括：

- 监听网络中多个设备发送的UDP数据包
- 实时处理设备数据并识别设备类型
- 将设备数据写入本地日志文件
- 监控设备连接状态
- 提供用户界面显示设备状态

### 1.2 技术栈选择

- **开发框架**: Qt/C++ - 提供跨平台GUI和网络支持
- **网络协议**: UDP - 适合实时数据传输，低延迟
- **并发模型**: 多线程 - 避免阻塞，提高响应性
- **数据存储**: 文本文件 - 简单可靠的日志存储

## 2. 系统架构设计

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        主界面线程                             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  Interface                              ││
│  │  • 系统协调器                                             ││
│  │  • 设备状态管理 (devmap)                                  ││
│  │  • 定时器监控                                            ││
│  │  • 配置文件读写                                           ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              │
                 ┌────────────┼────────────┐
                 │            │            │
         ┌───────▼───────┐    │   ┌────────▼────────┐
         │  UDP网络线程   │    │   │   数据处理线程    │
         │ ┌─────────────┐│   │   │ ┌──────────────┐│
         │ │FUdpControler││   │   │ │DataProcessor ││
         │ │ • UDP监听   ││    │   │ │ • 数据解析   ││
         │ │ • 数据接收  ││    │   │ │ • 设备识别   ││
         │ │ • 发送控制  ││    │   │ │ • 状态更新   ││
         │ └─────────────┘│   │   │ └──────────────┘│
         │ ┌─────────────┐│   │   └─────────────────┘
         │ │ UdpWorker   ││   │
         │ │ • Socket操作 ││    │
         │ │ • 实际网络IO  ││    │
         │ └─────────────┘│   │
         └─────────────────┘   │
                              │
                    ┌─────────▼─────────┐
                    │   文件写入线程     │
                    │ ┌───────────────┐ │
                    │ │  FileWriter   │ │
                    │ │ • 日志文件管理 │ │
                    │ │ • 数据写入     │ │
                    │ │ • 文件轮转     │ │
                    │ └───────────────┘ │
                    └───────────────────┘

        网络设备1 ──┐
        网络设备2 ──┼──► UDP:4001 端口
        网络设备3 ──┘
```

### 2.2 核心设计模式

#### 2.2.1 生产者-消费者模式

```
UDP接收 → 数据队列 → 数据处理 → 文件写入
 (生产)              (消费)     (消费)
```

#### 2.2.2 观察者模式 (Qt信号槽)

```cpp
// 数据流向的信号槽连接
UDP数据接收 ──signal──► 数据处理
数据处理完成 ──signal──► 界面更新
数据处理完成 ──signal──► 文件写入
```

## 3. 关键组件详解

### 3.1 Interface类 - 系统协调中心

**职责**:

- 系统初始化和资源管理
- 线程创建和生命周期管理
- 设备状态监控
- 配置文件管理

**关键代码模式**:

```cpp
// 线程安全的设备管理
QMap<int, DevItems*> devmap;  // 设备映射表
QMutex devMapMutex;           // 线程同步

// 定时监控模式
QTimer *timer = new QTimer();
connect(timer, &QTimer::timeout, this, &Interface::timer_1s_slot);
```

### 3.2 DataProcessor类 - 数据处理引擎

**设计亮点**:

```cpp
void DataProcessor::processData(QByteArray data, QString peerip, quint16 peerport)
{
    // 1. 数据包解析 - 协议设计
    int device_id = data[1] & 0xFF;    // 设备标识
    int device_type = data[2] & 0xFF;  // 设备类型
    
    // 2. 设备注册机制 - 动态发现
    if (!devmap.contains(device_id)) {
        // 自动注册新设备
        DevItems *newDevice = new DevItems();
        // ... 设备信息填充
        devmap.insert(device_id, newDevice);
    }
    
    // 3. 心跳机制 - 连接状态维护
    device->timer_count = 0;  // 重置心跳计数
}
```

### 3.3 FileWriter类 - 文件管理系统

**核心设计思想**:

```cpp
// 设备独立的文件上下文
struct FileContext {
    QFile *file;        // 文件句柄
    QTextStream *stream; // 格式化输出
    qint64 size;        // 大小监控
};
QMap<int, FileContext> fileMap; // 每个设备独立文件

// 文件轮转机制
if (context.size >= 100MB) {
    // 关闭当前文件，创建新文件
    rotateLogFile(deviceId);
}
```

## 4. 多线程设计策略

### 4.1 线程职责分离

|线程|职责|优势|
|---|---|---|
|主线程|UI更新、系统协调|保持界面响应|
|UDP线程|网络IO操作|避免网络阻塞|
|处理线程|数据解析、逻辑处理|CPU密集任务隔离|
|文件线程|磁盘IO操作|避免文件写入阻塞|

### 4.2 线程间通信

```cpp
// Qt信号槽 - 线程安全的异步通信
connect(udpThread, &UdpWorker::dataReceived,
        processor, &DataProcessor::processData, 
        Qt::QueuedConnection);  // 异步队列连接

// 互斥锁 - 共享数据保护
QMutexLocker locker(&devMapMutex);
// 临界区操作...
```

## 5. 网络协议设计

### 5.1 数据包格式

```
字节位置: [0] [1] [2] [3] [4] [5] ...
内容:    [?] [设备ID] [设备类型] [数据负载...]
         
设备类型常量:
- DEV_I   = 1
- DEV_II  = 2  
- DEV_III = 3
```

### 5.2 设备发现机制

```cpp
// 被动发现模式
// 设备主动发送数据包 → 服务器解析 → 自动注册
if (!deviceExists(device_id)) {
    registerNewDevice(device_id, device_type, sender_ip, sender_port);
}
```

## 6. 文件系统设计

### 6.1 目录结构

```
工作目录/
├── logTMS_config.ini          # 配置文件
└── tmslog/                    # 日志根目录
    ├── dev_1/                 # 设备类型1
    │   ├── 101/              # 设备ID 101
    │   │   ├── 101_2024-01-01_10-30-00.log
    │   │   └── 101_2024-01-01_11-45-00.log
    │   └── 102/              # 设备ID 102
    ├── dev_2/                # 设备类型2
    └── dev_3/                # 设备类型3
```

### 6.2 日志格式

```
[2024-01-01 10:30:15.123]  设备数据内容...
[2024-01-01 10:30:16.456]  设备数据内容...
```

## 7. 系统可扩展性设计

### 7.1 新设备类型支持

要添加新的设备类型，只需：

1. 在`devitems.h`中定义新的`DEV_IV`常量
2. 系统会自动识别并创建对应目录结构

### 7.2 多端口监听扩展

当前架构可轻松扩展为多端口监听：

```cpp
// 在Interface::initData()中添加
udptxt[LAN_LOG_LOCAL_2].ip = "172.17.11.95";
udptxt[LAN_LOG_LOCAL_2].port = 4002;

// 创建额外的UDP控制器
FUdpControler *pfudp_chn2 = new FUdpControler(
    udptxt[LAN_LOG_LOCAL_2].ip, 
    udptxt[LAN_LOG_LOCAL_2].port
);
```

## 8. 设计经验总结

### 8.1 优秀的设计决策

1. **单一职责原则** - 每个类职责明确
2. **异步解耦** - 使用信号槽实现松耦合
3. **资源管理** - 明确的线程生命周期管理
4. **错误容错** - 网络异常时的优雅处理

### 8.2 可优化点

1. **内存管理** - 可考虑智能指针
2. **配置灵活性** - 支持运行时配置修改
3. **日志压缩** - 大文件自动压缩
4. **监控仪表板** - 实时状态可视化

## 9. 学习要点

作为学生，通过这个项目你应该掌握：

1. **多线程编程模式** - 如何合理分离任务到不同线程
2. **网络编程基础** - UDP协议的使用场景和实现
3. **Qt框架应用** - 信号槽机制、线程管理
4. **系统架构设计** - 如何设计可扩展的软件架构
5. **文件系统管理** - 大量文件的组织和管理策略

这个系统展现了**企业级应用**的典型架构模式，是学习系统设计的优秀案例。



## 第一步：理解需求和整体架构

**需求分析：**

- 从固定地址接收UDP数据包
- 数据来自三种不同类型的设备（DEV_I、DEV_II、DEV_III）
- 需要按设备类型和ID分类存储日志文件

**基础知识1：UDP协议** UDP（用户数据报协议）是一种无连接的传输层协议：

- 特点：快速、无连接、不保证可靠性
- 适用场景：日志传输、实时数据等对速度要求高的场景
- 与TCP区别：UDP更轻量，但不保证数据完整性

## 第二步：设计软件架构

**多线程架构设计思路：**

```
主线程(Interface) 
    ├── UDP接收线程(UdpWorker)
    ├── 数据处理线程(DataProcessor) 
    └── 文件写入线程(FileWriter)
```

**基础知识2：为什么要用多线程？**

- **UI响应性**：避免阻塞主界面
- **性能优化**：并行处理不同任务
- **模块解耦**：每个线程专注单一职责

## 第三步：UDP接收模块设计

**设计UDP接收器：**

```cpp
// 核心组件：UdpWorker + FUdpControler
class UdpWorker : public QObject {
    // 负责实际的UDP操作
    void doInit(QString localip, quint16 localport);
    void doRead();  // 接收数据
    void doWrite(QByteArray data, QString peerip, quint16 peerport);
};

class FUdpControler : public QObject {
    // 控制器，管理UdpWorker的生命周期
};
```

**基础知识3：Qt的信号槽机制**

- **信号(Signal)**：事件通知，如数据到达
- **槽函数(Slot)**：响应信号的函数
- **优势**：松耦合、线程安全的通信方式

```cpp
connect(udpsock, SIGNAL(readyRead()), this, SLOT(doRead()));
```

## 第四步：数据处理模块设计

**数据处理器的职责：**

```cpp
class DataProcessor : public QObject {
public slots:
    void processData(QByteArray data, QString peerip, quint16 peerport);
    
private:
    // 解析数据包格式
    int device_id = data[1] & 0xFF;    // 第2字节：设备ID
    int device_type = data[2] & 0xFF;  // 第3字节：设备类型
};
```

**设备管理策略：**

- 使用QMap存储设备信息：`QMap<int, DevItems*> devmap`
- 每个设备包含：ID、类型、IP、端口、连接状态
- 线程安全：使用QMutex保护共享数据

**基础知识4：线程同步**

```cpp
QMutexLocker locker(&devMapMutex);  // RAII方式自动加锁解锁
```

## 第五步：文件写入模块设计

**文件组织结构：**

```
/tmslog/
  ├── dev_1/     # DEV_I类型设备
  │   └── 设备ID/
  │       └── ID_yyyy-MM-dd_hh-mm-ss.log
  ├── dev_2/     # DEV_II类型设备
  └── dev_3/     # DEV_III类型设备
```

**基础知识5：文件操作**

```cpp
class FileWriter : public QObject {
private:
    struct FileContext {
        QFile *file;           // 文件对象
        QTextStream *stream;   // 文本流，方便写入
        qint64 size;          // 文件大小追踪
    };
    QMap<int, FileContext> fileMap;  // 每个设备一个文件上下文
};
```

**文件管理策略：**

- 按设备ID维护独立的文件句柄
- 文件大小超过100MB时自动创建新文件
- 添加时间戳格式：`[yyyy-MM-dd hh:mm:ss.zzz]`

## 第六步：主控制器设计

**Interface类的作用：**

```cpp
class Interface : public QObject {
public:
    // 1. 初始化各个组件
    void initData();
    
    // 2. 建立信号槽连接
    void iniConnect();
    
    // 3. 管理设备连接状态
    void freshNetConnectState();
    
private:
    DataProcessor *dataProcessor;
    FileWriter *fileWriter;
    QThread dataProcessorThread;
    QThread fileWriterThread;
};
```

**基础知识6：Qt线程管理**

```cpp
// 将对象移动到指定线程
dataProcessor->moveToThread(&dataProcessorThread);

// 线程结束时自动清理对象
connect(&dataProcessorThread, &QThread::finished, 
        dataProcessor, &QObject::deleteLater);
```

## 第七步：配置管理

**配置文件设计：**

```ini
[LOCAL_NETWORK]
local_ip=172.17.11.94
local_port=4001
```

使用QSettings类读写配置：

```cpp
QSettings settings(QDir::currentPath() + "/logTMS_config.ini", 
                   QSettings::IniFormat);
udptxt[LAN_LOG_LOCAL].ip = settings.value("LOCAL_NETWORK/local_ip").toString();
```

## 第八步：错误处理和健壮性

**连接状态监控：**

- 定时器每2秒检查设备连接状态
- 超过5个周期无数据则标记为断线
- timer_count机制追踪设备活跃度

**文件操作安全性：**

- 创建目录前检查是否存在
- 文件打开失败时的清理机制
- 线程安全的文件映射管理

## 总结：设计模式和最佳实践

这个软件体现了几个重要的设计模式：

1. **生产者-消费者模式**：UDP接收→数据处理→文件写入
2. **观察者模式**：信号槽机制实现组件间通信
3. **单例模式**：Interface作为全局控制器
4. **策略模式**：不同设备类型的处理策略

**Qt开发的关键概念：**

- **事件驱动**：基于信号槽的异步编程
- **对象树**：父子关系自动管理内存
- **线程亲和性**：对象只能在创建线程中操作
- **RAII**：资源获取即初始化，自动管理生命周期

这样的设计确保了系统的可扩展性、稳定性和维护性。每个模块职责单一，通过信号槽松耦合连接，符合现代软件开发的最佳实践。





### 开发者拿到需求后的思考过程：

**需求：** "需要一个软件从一个地址中接收三种类型的日志，然后需要把这三种类型的日志分别显示和分类存储。当接收到日志时候，需要先判断属于哪个设备的日志，判断出以后就去显示，以及保存。为了保证流畅，把判断日志属于哪个类型单独做一个线程，然后保存单独一个线程，ui显示单独一个线程。"

------------------------------------------------------------------
第一步：开发者的需求分析思维
# 开发者思维过程

## 需求理解和关键点提取

### 开发者内心独白：

> "让我理解一下这个需求..."

**需求拆解：**

1. **"从一个地址接收"** → UDP/TCP网络编程，固定监听地址
2. **"三种类型的日志"** → 需要区分设备类型，可能是设备型号不同
3. **"判断属于哪个设备"** → 数据包里有设备标识，需要解析
4. **"分别显示和分类存储"** → UI展示 + 文件系统按类型存储
5. **"保证流畅"** → 多线程，避免阻塞
6. **"单独线程"** → 明确指出了线程架构

### 开发者的疑问和假设：

- 数据包格式是什么？（需要和客户确认）
- 三种类型具体是什么？（假设：设备型号A、B、C）
- 显示什么内容？（假设：实时日志内容）
- 存储格式？（假设：按日期和设备类型分文件）

### 技术选型思考：

- **网络：** UDP比TCP更适合日志传输（性能好，丢包可容忍）
- **UI框架：** Qt（跨平台，网络支持好，多线程友好）
- **存储：** 文本文件（简单，可读性好）
- **线程架构：** 生产者-消费者模式


---

## 第二步：模块划分思维

开发者脑中的模块化思考：
# 模块划分思维过程

## 开发者的模块化思考

### 第一轮思考：按功能划分

> "需求里提到了几个关键动作：接收、判断、显示、存储"

1. **网络接收模块** - 负责从网络接收数据
2. **数据解析模块** - 判断设备类型和解析内容
3. **UI显示模块** - 界面展示
4. **文件存储模块** - 保存到文件

### 第二轮思考：按线程需求重新划分

> "需求明确说了要多线程，让我重新考虑..."

**线程架构设计：**

- **主线程：** UI显示 + 总控制
- **网络线程：** 接收数据包
- **解析线程：** 判断设备类型，解析数据
- **存储线程：** 写入文件

### 第三轮思考：考虑数据流向

> "数据怎么在线程间流转？"

```
网络线程 --[原始数据包]--> 解析线程 --[解析后数据]--> UI线程
                                           |
                                           +--> 存储线程
```

### 最终模块设计：

1. **UdpReceiver** - UDP接收器（独立线程）
2. **DataProcessor** - 数据解析器（独立线程）
3. **FileWriter** - 文件写入器（独立线程）
4. **MainController** - 主控制器（主线程）
5. **DeviceManager** - 设备状态管理（解析线程）

### 模块间通信方式：

- **Qt信号槽** - 线程安全，异步通信
- **共享数据结构** - 需要加锁保护

---
### 第三步：核心数据结构设计思维
```
// 开发者的数据结构设计思维过程

// ==================== 开发者思考过程 ====================
/*
开发者内心独白：
"现在我需要设计数据结构，让我想想数据是怎么流动的..."

1. 网络接收到的是原始字节数组
2. 解析后需要知道：设备ID、设备类型、日志内容
3. UI需要显示：设备信息、日志内容、时间戳
4. 存储需要：按设备分类的文件路径
5. 设备管理需要：设备状态、统计信息

"我需要几个核心数据结构..."
*/

// ==================== 第一个数据结构：原始数据包 ====================
// 开发者思考："网络接收到什么？"
struct RawDataPacket {
    QByteArray rawData;        // 原始字节数据
    QString senderIp;          // 发送方IP
    quint16 senderPort;        // 发送方端口
    QDateTime receiveTime;     // 接收时间
    
    // 开发者思考："需要基本的验证"
    bool isValid() const {
        return rawData.size() >= 3; // 至少要有设备ID和类型
    }
};

// ==================== 第二个数据结构：设备信息 ====================  
// 开发者思考："需要区分三种设备类型"
enum DeviceType {
    DEV_UNKNOWN = 0,
    DEV_I = 1,       // 第一种设备
    DEV_II = 2,      // 第二种设备  
    DEV_III = 3      // 第三种设备
};

// 开发者思考："每个设备的基本信息"
struct DeviceInfo {
    int deviceId;              // 设备ID（数据包中解析出来）
    DeviceType deviceType;     // 设备类型（数据包中解析出来）
    QString ipAddress;         // 设备IP地址
    quint16 port;             // 设备端口
    bool isOnline;            // 是否在线
    QDateTime lastActiveTime; // 最后活跃时间
    
    // 开发者思考："需要一些统计信息用于监控"
    qint64 totalPackets;      // 总接收包数
    qint64 totalBytes;        // 总接收字节数
    int timeoutCounter;       // 超时计数器
};

// ==================== 第三个数据结构：解析后的日志条目 ====================
// 开发者思考："解析后的数据应该包含什么？"
struct LogEntry {
    int deviceId;             // 设备ID
    DeviceType deviceType;    // 设备类型
    QString senderIp;         // 发送方IP
    quint16 senderPort;       // 发送方端口
    QDateTime timestamp;      // 时间戳
    QByteArray logContent;    // 日志内容（从数据包第3字节开始）
    bool isValid;             // 是否有效
    
    // 开发者思考："UI显示需要格式化的字符串"
    QString toDisplayString() const {
        return QString("[%1] 设备%2(类型%3): %4")
               .arg(timestamp.toString("hh:mm:ss"))
               .arg(deviceId)
               .arg(static_cast<int>(deviceType))
               .arg(QString::fromUtf8(logContent));
    }
};

// ==================== 第四个数据结构：文件上下文 ====================
// 开发者思考："文件写入需要管理文件句柄和状态"
struct FileContext {
    QFile *file;              // 文件句柄
    QTextStream *stream;      // 文本流
    QString filePath;         // 文件路径
    qint64 currentSize;       // 当前文件大小
    QDateTime createTime;     // 创建时间
    
    // 开发者思考："需要文件轮转，避免单个文件太大"
    bool needRotation(qint64 maxSize = 100 * 1024 * 1024) const {
        return currentSize >= maxSize; // 100MB轮转
    }
};

// ==================== 第五个数据结构：设备映射表 ====================
// 开发者思考："需要快速查找设备信息"
using DeviceMap = QMap<int, DeviceInfo*>;  // 设备ID -> 设备信息
using FileMap = QMap<int, FileContext>;    // 设备ID -> 文件上下文

// ==================== 配置结构 ====================
// 开发者思考："系统需要一些配置参数"
struct SystemConfig {
    QString listenIp;         // 监听IP
    quint16 listenPort;       // 监听端口
    QString logBasePath;      // 日志根目录
    int deviceTimeout;        // 设备超时时间（秒）
    qint64 maxFileSize;       // 最大文件大小
    
    // 默认配置
    SystemConfig() 
        : listenIp("127.0.0.1")
        , listenPort(4001)
        , logBasePath("./tmslog")
        , deviceTimeout(10)
        , maxFileSize(100 * 1024 * 1024) {}
};

// ==================== 开发者的设计原则总结 ====================
/*
设计原则：
1. 数据结构要反映业务流程：原始数据 -> 解析数据 -> 显示/存储
2. 每个结构体职责单一：RawDataPacket负责网络数据，LogEntry负责业务数据
3. 考虑性能：使用QMap快速查找，避免每次遍历
4. 考虑扩展性：枚举类型便于增加新设备类型
5. 线程安全：结构体本身无状态，状态由管理类负责同步
*/
```

### 第四步 ： 模块话头文件设计思维
```
// ==================== 开发者的头文件设计思维 ====================

/*
开发者思考过程：
"现在我有了数据结构，需要设计各个模块的接口。
每个模块应该：
1. 只暴露必要的接口
2. 通过信号槽通信（线程安全）
3. 职责单一，便于测试和维护
"
*/

// ==================== 1. UDP接收模块 ====================
// udp_receiver.h
#ifndef UDP_RECEIVER_H
#define UDP_RECEIVER_H

#include <QObject>
#include <QUdpSocket>
#include "core_types.h" // 包含我们定义的数据结构

/*
开发者思考：
"UDP接收器的职责就是接收网络数据，转换成RawDataPacket发送给下一层"
*/
class UdpReceiver : public QObject
{
    Q_OBJECT
public:
    explicit UdpReceiver(QObject *parent = nullptr);
    
    // 开发者思考："需要启动和停止接口"
    bool startListening(const QString &ip, quint16 port);
    void stopListening();
    bool isListening() const;

signals:
    // 开发者思考："收到数据就发信号，让其他模块处理"
    void dataReceived(const RawDataPacket &packet);
    void errorOccurred(const QString &error);

private slots:
    // 开发者思考："Qt的槽函数处理UDP socket事件"
    void handleReadyRead();

private:
    QUdpSocket *m_udpSocket;
    bool m_isListening;
};

#endif

// ==================== 2. 数据解析模块 ====================
// data_processor.h  
#ifndef DATA_PROCESSOR_H
#define DATA_PROCESSOR_H

#include <QObject>
#include <QMutex>
#include "core_types.h"

/*
开发者思考：
"数据处理器负责：
1. 解析原始数据包
2. 管理设备状态
3. 生成日志条目
这是系统的核心业务逻辑"
*/
class DataProcessor : public QObject
{
    Q_OBJECT
public:
    explicit DataProcessor(QObject *parent = nullptr);
    
    // 开发者思考："提供查询接口给UI"
    DeviceInfo* getDevice(int deviceId);
    QList<DeviceInfo*> getAllDevices();
    int getOnlineDeviceCount();

signals:
    // 开发者思考："解析完成后发送给UI和存储模块"
    void logEntryReady(const LogEntry &entry);
    void deviceStatusChanged(int deviceId, bool isOnline);
    void newDeviceDetected(const DeviceInfo &device);

public slots:
    // 开发者思考："接收UDP模块发来的原始数据"
    void processRawData(const RawDataPacket &packet);
    
    // 开发者思考："定时检查设备状态"
    void checkDeviceTimeout();

private:
    // 开发者思考："核心业务逻辑方法"
    LogEntry parsePacket(const RawDataPacket &packet);
    void updateDeviceStatus(const LogEntry &entry);
    DeviceInfo* createNewDevice(const LogEntry &entry);
    
    // 开发者思考："设备管理需要线程安全"
    DeviceMap m_devices;
    QMutex m_deviceMutex;
};

#endif

// ==================== 3. 文件写入模块 ====================
// file_writer.h
#ifndef FILE_WRITER_H
#define FILE_WRITER_H

#include <QObject>
#include <QMutex>
#include "core_types.h"

/*
开发者思考：
"文件写入器负责：
1. 按设备分类创建文件
2. 写入日志条目
3. 管理文件轮转
4. 维护文件句柄"
*/
class FileWriter : public QObject
{
    Q_OBJECT
public:
    explicit FileWriter(QObject *parent = nullptr);
    ~FileWriter();
    
    // 开发者思考："配置接口"
    void setBasePath(const QString &path);
    void setMaxFileSize(qint64 size);

signals:
    void fileWritten(int deviceId, const QString &filePath);
    void writeError(int deviceId, const QString &error);
    void fileRotated(int deviceId, const QString &newPath);

public slots:
    // 开发者思考："接收解析模块发来的日志条目"
    void writeLogEntry(const LogEntry &entry);

private:
    // 开发者思考："文件管理的核心方法"
    QString generateFilePath(int deviceId, DeviceType type);
    void createNewFile(int deviceId, DeviceType type);
    void rotateFile(int deviceId, DeviceType type);
    void closeFile(int deviceId);
    
    // 开发者思考："文件操作需要线程安全"
    FileMap m_fileContexts;
    QMutex m_fileMutex;
    QString m_basePath;
    qint64 m_maxFileSize;
};

#endif

// ==================== 4. 主控制器模块 ====================
// main_controller.h
#ifndef MAIN_CONTROLLER_H
#define MAIN_CONTROLLER_H

#include <QObject>
#include <QThread>
#include "core_types.h"

// 前向声明
class UdpReceiver;
class DataProcessor; 
class FileWriter;

/*
开发者思考：
"主控制器负责：
1. 创建和管理各个模块
2. 管理线程
3. 连接信号槽
4. 提供统一的对外接口"
*/
class MainController : public QObject
{
    Q_OBJECT
public:
    explicit MainController(QObject *parent = nullptr);
    ~MainController();
    
    // 开发者思考："系统控制接口"
    bool startSystem(const SystemConfig &config);
    void stopSystem();
    bool isRunning() const;
    
    // 开发者思考："给UI提供的查询接口"
    QList<DeviceInfo*> getDeviceList();
    int getOnlineDeviceCount();

signals:
    // 开发者思考："转发各模块的重要信号给UI"
    void newLogEntry(const LogEntry &entry);
    void deviceStatusChanged(int deviceId, bool isOnline);
    void systemError(const QString &error);

private slots:
    // 开发者思考："处理各模块的错误"
    void handleModuleError(const QString &error);

private:
    // 开发者思考："模块管理"
    void initializeModules();
    void connectModules();
    void moveModulesToThreads();
    void cleanupModules();
    
    // 各功能模块
    UdpReceiver *m_udpReceiver;
    DataProcessor *m_dataProcessor;
    FileWriter *m_fileWriter;
    
    // 线程管理
    QThread *m_networkThread;
    QThread *m_processingThread;
    QThread *m_storageThread;
    
    SystemConfig m_config;
    bool m_isRunning;
};

#endif

// ==================== 5. UI主窗口（可选） ====================
// main_window.h
#ifndef MAIN_WINDOW_H  
#define MAIN_WINDOW_H

#include <QMainWindow>
#include <QTextEdit>
#include <QLabel>
#include <QListWidget>
#include "core_types.h"

class MainController;

/*
开发者思考：
"UI负责：
1. 显示实时日志
2. 显示设备状态
3. 提供系统控制
4. 配置参数设置"
*/
class MainWindow : public QMainWindow
{
    Q_OBJECT
public:
    explicit MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

private slots:
    // 开发者思考："响应控制器的信号"
    void onNewLogEntry(const LogEntry &entry);
    void onDeviceStatusChanged(int deviceId, bool isOnline);
    void onSystemError(const QString &error);
    
    // 开发者思考："用户操作"
    void onStartButtonClicked();
    void onStopButtonClicked();

private:
    void setupUI();
    void updateDeviceList();
    
    MainController *m_controller;
    QTextEdit *m_logDisplay;        // 日志显示区
    QListWidget *m_deviceList;      // 设备列表
    QLabel *m_statusLabel;          // 状态栏
};

#endif

// ==================== 开发者设计总结 ====================
/*
设计原则总结：
1. 单一职责：每个类只做一件事
2. 依赖注入：通过信号槽解耦，避免直接依赖
3. 线程安全：共享数据用互斥锁保护
4. 错误处理：每个模块都有错误信号
5. 可测试性：接口清晰，便于单元测试
6. 可扩展性：通过枚举和虚函数支持扩展

数据流设计：
UdpReceiver → DataProcessor → FileWriter
                  ↓
             MainController → MainWindow
*/
```

