
### ** Docker相关命令**

|语法|类别|说明|
|---|---|---|
|`docker ps -a`|容器列表|查看所有容器|
|`docker start/stop/rm`|容器管理|启停/删除容器|
|`docker exec`|容器内执行|在运行中的容器执行命令|
|`docker cp`|文件拷贝|宿主机与容器间复制文件|
|`docker run -v`|卷挂载|实现数据持久化|
### **Docker运行选项详解**

|选项|类别|说明|示例|
|---|---|---|---|
|`-i` (`--interactive`)|输入控制|保持STDIN打开，允许容器接收输入|`echo "hello" \| docker run -i alpine cat`|
|`-t` (`--tty`)|终端模拟|分配伪终端，支持交互式Shell环境|`docker run -t ubuntu bash`（需配合-i使用）|
|`-it`|交互组合|组合使用，进入容器交互式Shell|`docker run -it ubuntu bash`|
|`-d` (`--detach`)|后台运行|容器在后台运行，不占用当前终端|`docker run -d nginx`|
|`-d` vs `-it`|运行模式|后台服务用-d，交互操作用-it|服务：`-d`，调试：`-it`|

### **标准流说明**

|流类型|方向|作用|Docker中的体现|
|---|---|---|---|
|`STDIN`|外部→程序|程序接收输入数据|`-i`选项保持此流开放|
|`STDOUT`|程序→外部|程序正常输出结果|如`echo "hello"`的输出|
|`STDERR`|程序→外部|程序错误信息输出|如命令执行失败的错误信息|

### **使用场景对比**

|场景|推荐选项|命令示例|说明|
|---|---|---|---|
|交互式操作|`-it`|`docker run -it ubuntu bash`|需要在容器内执行命令|
|后台服务|`-d`|`docker run -d nginx`|Web服务、数据库等长期运行|
|管道输入|`-i`|`echo "data" \| docker run -i alpine cat`|处理输入数据但不需要终端|
|一次性任务|无选项|`docker run alpine echo "hello"`|执行后立即退出|
|进入运行容器|`-it`|`docker exec -it container_id bash`|调试或管理已运行的容器|
<details>
<summary>交互选项详细解释</summary>
`-it`: `-i` (interactive) 保持 STDIN 打开即使没有附加, `-t` (tty) 分配一个伪终端。这对于你想进入bash通常是需要的。如果你只是想让它在后台运行，可以省略 `-it` 并主要用 `-d`。


### **`-i` (`--interactive`)**

- **作用**：保持容器的标准输入（`STDIN`）打开，即使没有直接附加（比如从终端输入）。
    
- **`STDIN` 是什么？**
    
    - `STDIN`（Standard Input，标准输入）是程序接收输入数据的流（比如键盘输入、管道传递的数据等）。
        
    - 在 Docker 中，默认情况下，如果未指定 `-i`，容器会立即进入后台运行，无法接收输入。
        
- **示例**：
    
    bash
    
    # 不启用 -i：容器启动后无法交互（直接退出）
    docker run alpine echo "hello"
    
    # 启用 -i：可以接收输入（如通过管道或终端）
    echo "hello" | docker run -i alpine cat
    

---

### **2. `-t` (`--tty`)**

- **作用**：为容器分配一个伪终端（pseudo-TTY），模拟真实的终端环境（如支持命令行编辑、信号处理等）。
    
- **为什么需要 `-t`？**
    
    - 当你想以交互方式进入容器的 Shell（如 `bash`）时，必须启用 `-t`，否则会报错：
        
        bash
        
        # 错误：没有 -t 时无法进入交互式 Shell
        docker run -i ubuntu bash
        # 输出：bash: cannot set terminal process group (-1): Inappropriate ioctl for device
        

---

### **3. `-it` 的组合使用**

- **典型场景**：交互式运行容器（比如进入容器的 Shell）：
    
    bash
    
    docker run -it ubuntu bash
    
    - `-i`：保证你可以输入命令（如 `ls`）。
        
    - `-t`：让容器的 Shell 行为像本地终端（支持快捷键如 `Ctrl+C`、`Tab` 补全等）。
        
- **省略 `-it` 的情况**：  
    如果容器只是后台运行（如服务），只需 `-d`（detach 模式）：
    
    bash
    
    docker run -d nginx

在 Docker 命令 `docker run -it` 中，`-i` 和 `-t` 是两个常用的选项，它们的作用和 `STDIN` 的关系如下：

---

### **1. `-i` (`--interactive`)**
- **作用**：保持容器的标准输入（`STDIN`）打开，即使没有直接附加（比如从终端输入）。
- **`STDIN` 是什么？**  
  - `STDIN`（Standard Input，标准输入）是程序接收输入数据的流（比如键盘输入、管道传递的数据等）。  
  - 在 Docker 中，默认情况下，如果未指定 `-i`，容器会立即进入后台运行，无法接收输入。
- **示例**：
  ```bash
  # 不启用 -i：容器启动后无法交互（直接退出）
  docker run alpine echo "hello"

  # 启用 -i：可以接收输入（如通过管道或终端）
  echo "hello" | docker run -i alpine cat
  ```

---

### **2. `-t` (`--tty`)**
- **作用**：为容器分配一个伪终端（pseudo-TTY），模拟真实的终端环境（如支持命令行编辑、信号处理等）。  
- **为什么需要 `-t`？**  
  - 当你想以交互方式进入容器的 Shell（如 `bash`）时，必须启用 `-t`，否则会报错：  
    ```bash
    # 错误：没有 -t 时无法进入交互式 Shell
    docker run -i ubuntu bash
    # 输出：bash: cannot set terminal process group (-1): Inappropriate ioctl for device
    ```

---

### **3. `-it` 的组合使用**
- **典型场景**：交互式运行容器（比如进入容器的 Shell）：
  ```bash
  docker run -it ubuntu bash
  ```
  - `-i`：保证你可以输入命令（如 `ls`）。  
  - `-t`：让容器的 Shell 行为像本地终端（支持快捷键如 `Ctrl+C`、`Tab` 补全等）。

- **省略 `-it` 的情况**：  
  如果容器只是后台运行（如服务），只需 `-d`（detach 模式）：
  ```bash
  docker run -d nginx
  ```

---

### **4. `STDIN` 的扩展说明**
- **数据流方向**：  
  `STDIN` 是程序输入的数据流（从外部到程序），与之相对的还有：
  - `STDOUT`（标准输出）：程序打印的正常结果（如 `echo "hello"`）。  
  - `STDERR`（标准错误）：程序的错误消息（如 `ls /nonexistent`）。  

- **Docker 中的输入来源**：  
  - 终端直接输入（需要 `-i`）。  
  - 通过管道传递（如 `echo "input" | docker run -i alpine cat`）。  
  - 文件重定向（如 `docker run -i alpine sh < script.sh`）。

---

### **常见问题**
#### **Q：为什么有时候只用 `-i` 不用 `-t`？**
- 当程序不需要终端功能（如 `cat`、`grep`），但需要接收输入时：
  ```bash
  # 仅用 -i 接收管道输入
  echo "test" | docker run -i alpine cat
  ```

#### **Q：为什么 `docker exec` 也需要 `-it`？**
- 同理，进入已运行容器的 Shell 时：
  ```bash
  docker exec -it <container-id> bash
  ```

---

总结：  
- **`-i`** → 保持输入流开放（`STDIN`）。  
- **`-t`** → 模拟终端行为（需要交互时必加）。  
- **`-it`** → 交互式容器的黄金搭档。

</details>


# Docker 容器管理完整流程文档

## 1. 基本概念

- **镜像 (Image)**: 容器的模板，只读
- **容器 (Container)**: 镜像的运行实例
- **挂载 (Mount/Volume)**: 将宿主机目录同步到容器内

## 2. 容器生命周期管理

### 2.1 创建并运行容器

```bash
docker run [选项] 镜像名 [命令]

# 常用选项：
# -d: 后台运行
# --name: 指定容器名称
# -p: 端口映射 宿主机端口:容器端口
# -v: 目录挂载 宿主机路径:容器路径
# -it: 交互式运行

# 示例：
docker run -d --name my_container -p 2222:22 -v /host/path:/container/path my-image
```

### 2.2 容器状态管理

```bash
# 查看所有容器（包括停止的）
docker ps -a

# 查看运行中的容器
docker ps

# 启动已停止的容器
docker start 容器名

# 停止运行中的容器
docker stop 容器名

# 重启容器
docker restart 容器名

# 删除容器（需先停止）
docker rm 容器名

# 强制删除运行中的容器
docker rm -f 容器名
```

### 2.3 容器交互

```bash
# 进入正在运行的容器
docker exec -it 容器名 bash
docker exec -it 容器名 /bin/sh

# 查看容器日志
docker logs 容器名
docker logs -f 容器名  # 实时查看

# 在容器内执行命令
docker exec 容器名 命令
```

## 3. 文件操作

### 3.1 目录挂载（推荐用于开发）

```bash
# 创建容器时挂载
docker run -v /宿主机路径:/容器路径 镜像名

# 实时同步，双向修改
```

### 3.2 文件复制

```bash
# 从宿主机复制到容器
docker cp /宿主机路径 容器名:/容器路径

# 从容器复制到宿主机
docker cp 容器名:/容器路径 /宿主机路径
```

## 4. SSH 连接容器配置

### 4.1 容器要求

- 安装 SSH 服务
- SSH 服务正在运行
- 开放 22 端口
- 配置用户密码

### 4.2 检查 SSH 服务

```bash
# 进入容器
docker exec -it 容器名 bash

# 检查 SSH 服务状态
systemctl status ssh
service ssh status

# 启动 SSH 服务
systemctl start ssh
service ssh start

# 检查端口监听
netstat -tlnp | grep :22
```

### 4.3 VSCode SSH 配置

在 `~/.ssh/config` 文件中添加：

```
Host 别名
  HostName localhost
  Port 宿主机端口
  User 用户名
  PreferredAuthentications password
  PubkeyAuthentication no
  PasswordAuthentication yes
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
```

## 5. 常见问题排查

### 5.1 容器无法启动

```bash
# 查看错误日志
docker logs 容器名

# 常见原因：
# - 端口冲突
# - 挂载路径不存在
# - 启动命令错误
# - 资源不足
```

### 5.2 SSH 连接失败

```bash
# 检查步骤：
1. docker ps 确认容器运行
2. docker exec -it 容器名 bash 能否进入
3. 容器内 service ssh status 检查服务
4. netstat -tlnp | grep :22 检查端口
5. 宿主机 lsof -i :2222 检查端口映射
```

### 5.3 容器名冲突

```bash
# 错误：name already in use
# 解决：删除旧容器
docker stop 旧容器名
docker rm 旧容器名

# 或重命名
docker rename 旧容器名 新容器名
```

## 6. 实际操作流程示例

### 6.1 完整的开发环境搭建流程

```bash
# 1. 停止并清理旧容器（如果存在）
docker stop my_dev_container 2>/dev/null || true
docker rm my_dev_container 2>/dev/null || true

# 2. 创建新容器（带目录挂载）
docker run -d \
  --name my_dev_container \
  -p 2222:22 \
  -v /Users/Apple/bridge_0:/workspace/bridge_0 \
  my-arm64-dev:v1.0 \
  /usr/bin/entry.sh

# 3. 检查容器状态
docker ps

# 4. 确保 SSH 服务运行
docker exec -it my_dev_container service ssh start

# 5. 测试 SSH 连接
ssh -p 2222 root@localhost

# 6. VSCode 连接测试
```

### 6.2 日常维护命令

```bash
# 查看容器资源使用
docker stats my_dev_container

# 查看容器详细信息
docker inspect my_dev_container

# 查看挂载信息
docker inspect my_dev_container | grep -A 10 Mounts

# 备份容器为镜像
docker commit my_dev_container my-backup:latest
```

## 7. 最佳实践

### 7.1 开发环境

- 使用目录挂载 (-v) 实现代码同步
- 固定容器名称便于管理
- 开放必要端口 (SSH, 应用端口)
- 定期备份重要修改

### 7.2 安全建议

- 不要在生产环境禁用 SSH 密钥认证
- 修改默认 SSH 端口
- 使用非 root 用户
- 定期更新镜像

### 7.3 故障排查顺序

1. `docker ps -a` - 检查容器状态
2. `docker logs 容器名` - 查看启动日志
3. `docker exec -it 容器名 bash` - 进入容器调试
4. 检查服务状态和端口监听
5. 检查网络连接和防火墙

---

## 快速参考命令

```bash
# 容器管理
docker ps -a                    # 查看所有容器
docker start/stop/restart 容器名  # 启动/停止/重启
docker rm 容器名                 # 删除容器
docker logs 容器名               # 查看日志

# 进入容器
docker exec -it 容器名 bash     # 交互式进入

# 文件操作
docker cp 源路径 目标路径        # 复制文件

# SSH 调试
service ssh status             # 检查SSH服务
netstat -tlnp | grep :22      # 检查端口监听
```


---
### docker 启动记录
	我判断出这个错误的主要依据来自日志中的**关键错误信息**和你的**Docker操作历史**，结合常见的SSH和容器问题模式。以下是详细分析过程：

---

### **核心判断依据**
#### 1. **SSH连接失败的明确错误（日志关键行）**
   ```log
   kex_exchange_identification: read: Connection reset by peer
   Connection reset by ::1 port 2222
   ```
   - **`Connection reset by peer`**：表明SSH客户端尝试握手时被服务器端强制断开，通常是因为：
     - SSH服务未运行。
     - 端口监听失败（服务崩溃或未启动）。
     - 防火墙/网络策略阻止连接。
   - **`port 2222`**：指向你映射的容器SSH端口，说明问题出在容器内的SSH服务。

#### 2. **Docker操作历史中的关键线索**
   - 你通过 `-p 2222:22` 将容器的22端口映射到主机的2222端口，但：
     ```bash
     docker run -d --name my_dev_container -p 2222:22 ... /usr/bin/entry.sh
     ```
     - **未显式启动SSH服务**：如果`entry.sh`脚本没有包含`service ssh start`或类似命令，容器启动时SSH服务可能不会自动运行。
   - 你后续尝试的修复命令：
     ```bash
     bash -c "/usr/bin/entry.sh && tail -f /dev/null"
     ```
     说明你意识到容器会退出（因为`entry.sh`可能未保持进程前台运行），但**仍未解决SSH服务启动问题**。

#### 3. **容器状态与日志的间接证据**
   - 你的`docker ps -a`输出显示：
     ```bash
     Exited (137) 4 minutes ago   # 容器曾异常退出（137通常表示被强制终止）
     ```
     结合日志中SSH连接失败，说明容器可能未长期运行SSH服务。

---

### **辅助判断依据**
1. **常见Docker+SSH问题模式**：
   - 容器默认不启动SSH服务，需显式配置。
   - `entry.sh`脚本若未设计为长期运行（如只执行一次性任务），容器会退出导致SSH服务终止。

2. **VS Code Remote-SSH的行为**：
   - 它依赖容器内持久的SSH服务，如果服务未运行，会直接报`Connection reset`。

3. **你的调试尝试**：
   - 你曾运行`docker exec -it my_dev_container bash`，说明容器本身可交互，但SSH服务独立于Shell存在。

---

### **排除其他可能的原因**
1. **非网络问题**：
   - 错误是`Connection reset`而非`Connection refused`，说明连接已到达容器，但被拒绝（SSH服务异常）。
   - 如果是防火墙问题，通常会直接超时或无响应。

2. **非认证问题**：
   - 错误发生在握手初期（`kex_exchange_identification`阶段），尚未到密钥认证步骤。

3. **非端口冲突**：
   - 如果是主机2222端口被占用，错误会是`Address already in use`，而非`Connection reset`。

---

### **结论**
综合日志中的SSH握手失败、Docker操作历史中缺失SSH服务启动命令、以及容器状态，**根本原因是容器内的SSH服务未运行**。解决方案是确保容器启动时自动启动SSH服务（如修改`entry.sh`或直接通过`docker run`命令启动）。


---
选择：
 独立头文件的问题

  1. 不属于任何线程：MediaStatus枚举本身不"属于"某个线程，它只是一个类型定义
  2. 编译时包含：头文件在编译时被包含，运行时不存在"访问"问题
  3. 没有性能损失：枚举值传递是最高效的，比字符串快得多

  但是独立头文件增加了复杂性

  - 需要维护额外的文件
  - 概念上不如放在自然的位置清晰

  重新评估各方案

  方案1：字符串传递

  status_callback("session123", "CONGESTED");
  - 性能：需要字符串拷贝和比较，较慢
  - 类型安全：运行时才能发现拼写错误

  方案2：枚举传递（独立头文件）

  // src/common/media_status.h
  enum class MediaStatus { IDLE, ACTIVE, CONGESTED, ERROR, TERMINATED };

  status_callback("session123", MediaStatus::CONGESTED);
  - 性能：最快，枚举值拷贝
  - 类型安全：编译时检查

  方案3：整数传递

  status_callback("session123", 2); // 2表示CONGESTED
  - 性能：很快
  - 可读性：差

  现在我觉得方案1（字符串）可能是最实用的，因为：
  - 简单直接
  - 性能损失在可接受范围内（媒体状态变化不频繁）
  - 易于调试和日志记录

  ---
src/media/media_worker_thread.cpp                                                                                                                                      │ │
│ │                                                                                                                                                                        │ │
│ │   162            // 运行io_service事件循环                                                                                                                             │ │
│ │   163            while (running.load()) {                                                                                                                              │ │
│ │   164                try {                                                                                                                                             │ │
│ │   165 -                  // 运行一段时间的事件循环                                                                                                                     │ │
│ │   166 -                  io_service.run_for(std::chrono::milliseconds(100));                                                                                           │ │
│ │   165 +                  // 使用poll_one代替run_for，实现非阻塞处理                                                                                                    │ │
│ │   166 +                  io_service.poll_one();                                                                                                                        │ │
│ │   167                                                                                                                                                                  │ │
│ │   168 +                  // 短暂休眠，避免过度占用CPU                                                                                                                  │ │
│ │   169 +                  std::this_thread::sleep_for(std::chrono::milliseconds(1));                                                                                    │ │
│ │   170 +                                                                                                                                                                │ │
│ │   171                    // 重新启动io_service以便下次循环                                                                                                             │ │
│ │   172                    if (io_service.stopped()) {                                                                                                                   │ │
│ │   173 -                      io_service.restart();                                                                                                                     │ │
│ │   173 +                      io_service.reset();                                                                                                                       │ │
│ │   174                    }                                                                                                                                             │ │
│ │   175                                                                                                                                                                  │ │
│ │   176                } catch (const std::exception& e) {        
---
---
好的！**Boost.Asio** 是 C++ 中用于网络和异步 I/O 编程的强大库，广泛应用于高性能服务器、游戏引擎等场景。以下是系统化的学习路径和核心知识，从基础到实践逐步深入：

---

### **一、Boost.Asio 的核心概念**
#### 1. **异步 vs 同步 I/O**
- **同步 I/O**：操作阻塞当前线程直到完成（如 `socket.read()` 会一直等待数据到达）。
- **异步 I/O**：操作提交后立即返回，完成后通过回调通知（不阻塞线程，适合高并发）。

#### 2. **核心组件**
| 组件                | 作用                                                                 |
|---------------------|----------------------------------------------------------------------|
| `io_context`        | 异步调度引擎，管理所有 I/O 操作和回调                                 |
| `socket`            | 网络通信端点（TCP/UDP）                                              |
| `resolver`          | 域名解析（如将 `www.example.com` 转为 IP）                           |
| `deadline_timer`    | 定时器，用于延迟或周期性任务                                          |
| `strand`            | 保证回调顺序执行的调度器（避免多线程竞争）                             |

---

### **二、基础示例：从同步到异步**
#### 1. **同步 TCP 客户端**
```cpp
#include <boost/asio.hpp>
using namespace boost::asio;

int main() {
    io_context io_ctx;
    ip::tcp::socket socket(io_ctx);

    // 同步连接和读写
    socket.connect(ip::tcp::endpoint(
        ip::address::from_string("127.0.0.1"), 8080));
    write(socket, buffer("Hello, Server!"));
    
    char buf[1024];
    size_t len = socket.read_some(buffer(buf));
    std::cout << "Received: " << std::string(buf, len) << std::endl;
}
```

#### 2. **异步 TCP 服务端**
```cpp
#include <boost/asio.hpp>
using namespace boost::asio;

class Server {
public:
    Server(io_context& io_ctx) : acceptor_(io_ctx, ip::tcp::endpoint(ip::tcp::v4(), 8080)) {
        accept_connection();
    }

private:
    void accept_connection() {
        auto socket = std::make_shared<ip::tcp::socket>(acceptor_.get_executor());
        acceptor_.async_accept(*socket, [this, socket](boost::system::error_code ec) {
            if (!ec) {
                async_read(*socket, buffer(buf_), 
                    [socket](auto ec, auto len) { /* 处理数据 */ });
            }
            accept_connection(); // 继续接受新连接
        });
    }

    ip::tcp::acceptor acceptor_;
    std::array<char, 1024> buf_;
};
```

---

### **三、关键机制详解**
#### 1. **`io_context` 事件循环**
- **`run()`**：阻塞等待所有异步操作完成。
- **`poll()`**：非阻塞执行就绪的回调。
- **`post()`**：提交一个非 I/O 任务到事件循环。

#### 2. **异步操作链**
```cpp
// 异步写 → 异步读 → 处理
async_write(socket, buffer("Ping"), [&](auto ec, auto) {
    async_read(socket, buffer(buf_), [](auto ec, auto len) {
        std::cout << "Received: " << std::string(buf_.data(), len);
    }
});
```

#### 3. **多线程安全**
- 多线程调用 `io_context::run()` 可提升性能。
- 用 `strand` 保证回调顺序：
  ```cpp
  auto strand = make_strand(io_ctx);
  post(strand, [] { /* 线程安全的任务 */ });
  ```

---

### **四、常见应用场景**
#### 1. **高并发服务器**
- 使用 `async_accept` + `async_read`/`async_write` 处理数千连接。
- 配合线程池：
  ```cpp
  std::vector<std::thread> threads;
  for (int i = 0; i < 4; ++i) {
      threads.emplace_back([&io_ctx] { io_ctx.run(); });
  }
  ```

#### 2. **UDP 广播**
```cpp
ip::udp::socket socket(io_ctx, ip::udp::v4());
socket.set_option(ip::udp::socket::broadcast(true));
socket.send_to(buffer("Broadcast"), ip::udp::endpoint(ip::address_v4::broadcast(), 8888));
```

#### 3. **定时任务**
```cpp
deadline_timer timer(io_ctx, boost::posix_time::seconds(1));
timer.async_wait([](auto ec) {
    std::cout << "1秒后执行" << std::endl;
});
```

---

### **五、学习资源推荐**
1. **官方文档**：  
   [Boost.Asio 官方文档](https://www.boost.org/doc/libs/master/doc/html/boost_asio.html)
2. **书籍**：  
   - *Boost.Asio C++ Network Programming* (by John Torjo)
3. **实践项目**：  
   - 实现一个简易 HTTP 服务器。
   - 写一个多线程聊天程序。

---

### **六、调试技巧**
- **打印错误码**：
  ```cpp
  socket.async_connect(endpoint, [](const boost::system::error_code& ec) {
      if (ec) std::cerr << "Error: " << ec.message() << std::endl;
  });
  ```
- **日志跟踪**：  
  编译时定义宏 `BOOST_ASIO_ENABLE_HANDLER_TRACKING`，可生成回调流程图。

---

### **七、总结**
- **核心**：理解 `io_context` 的调度机制和异步操作链。
- **关键点**：  
  - 异步操作通过回调处理结果。
  - 多线程需用 `strand` 或锁保证安全。
- **进阶**：学习协程（C++20 的 `co_await` 与 Asio 结合）。

通过实际项目练习（如实现一个 Echo Server），你会快速掌握 Boost.Asio 的精髓！