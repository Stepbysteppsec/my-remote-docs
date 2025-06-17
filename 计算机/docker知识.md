
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