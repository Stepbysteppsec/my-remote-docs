
### 回调函数和信号与槽的区别
	回调函数是函数指针的一种用法，如果多个类都关注某个类的状态变化，此时需要维护一个列表，以存放多个回调函数的地址。对于每一个被关注的类，都需要做类似的工作，因此这种做法效率低，不灵活。

`readyRead` 是 Qt 中 `QUdpSocket`（以及 `QTcpSocket`、`QIODevice` 等）类提供的**信号（signal）**，其作用是**通知应用程序“Socket 接收缓冲区中有新数据可读取”**。以下是详细说明：

---

### **1. `readyRead` 信号的作用**
- **触发条件**：当 Socket 的接收缓冲区中收到新数据（如 UDP 报文、TCP 数据流）时，Qt 会自动发射此信号。
- **典型用途**：  
  用于异步处理接收到的数据，避免主动轮询（polling）造成的性能浪费。

---

### **2. 代码解析**
```cpp
connect(entity1_socket, &QUdpSocket::readyRead, this, &P2pUdpWorker::onEntity1DataReady);
```
- **含义**：  
  当 `entity1_socket`（UDP Socket）收到数据时，自动调用 `P2pUdpWorker::onEntity1DataReady` 槽函数处理数据。
- **工作流程**：  
  1. 对方发送 UDP 报文 → 2. 系统将数据存入接收缓冲区 → 3. Qt 触发 `readyRead` → 4. 你的槽函数读取数据。

---

### **3. 如何处理接收的数据？**
在槽函数 `onEntity1DataReady` 中，通常需要：
1. **读取数据**：使用 `readDatagram()`（UDP）或 `read()`（TCP）。  
2. **解析数据**：根据协议解码报文内容。  
3. **业务逻辑**：如转发、存储或响应。

#### 示例代码：
```cpp
void P2pUdpWorker::onEntity1DataReady() {
    while (entity1_socket->hasPendingDatagrams()) {  // 检查是否有待处理数据
        QByteArray datagram;
        datagram.resize(entity1_socket->pendingDatagramSize());
        QHostAddress senderAddr;
        quint16 senderPort;

        // 读取数据（UDP需获取来源地址和端口）
        qint64 bytesRead = entity1_socket->readDatagram(
            datagram.data(), datagram.size(), &senderAddr, &senderPort
        );

        if (bytesRead > 0) {
            qDebug() << "Received data from:" << senderAddr << ":" << senderPort;
            processData(datagram);  // 自定义数据处理函数
        }
    }
}
```

---

### **4. 关键注意事项**
- **数据可能分片**：  
  UDP 报文是独立的数据包，但 TCP 是流式协议，可能需要拼接或拆包。
- **缓冲区管理**：  
  如果数据量大，确保及时读取，避免缓冲区溢出。
- **多线程安全**：  
  若 Socket 在子线程中创建，需通过信号槽跨线程传递数据（Qt 自动处理）。

---

### **5. 对比其他相关信号**
| 信号               | 适用场景                          |
|--------------------|----------------------------------|
| `readyRead`        | 有数据到达（最常用）              |
| `bytesWritten(qint64)` | 数据发送完成时触发（如确认写入成功） |
| `disconnected`     | 连接断开时触发（TCP 专用）         |

---

### **6. 为什么用 `readyRead` 而非轮询？**
- **高效**：事件驱动模型，无 CPU 空转。  
- **低延迟**：数据到达后立即响应。  
- **Qt 事件循环集成**：与其他事件（如 UI 事件）协同工作。

---

总结：`readyRead` 是 Qt 网络编程中**异步接收数据的核心信号**，实现了数据到达时的自动处理。如果需要进一步优化（如超时处理或错误恢复），可以结合其他信号（如 `errorOccurred`）完善逻辑。


### 错误信息特征

错误信息 `undefined reference to 'P2pUdpWorker::onStatusReportTimer()'` 是典型的链接错误提示。在编译流程中，编译器和链接器的职责不同，编译器负责将源代码转换为目标文件，而链接器负责将多个目标文件以及库文件组合成一个可执行文件。当出现 `undefined reference` 这类错误时，意味着链接器在尝试将各个部分组合在一起时，找不到某个符号（这里是函数 `P2pUdpWorker::onStatusReportTimer()` 的定义）的实现，所以这是链接阶段特有的错误类型。




### 快捷键

ctrl+shift+z 撤销刚刚的撤销


### qsettings

qsettings 的错误说明 在使用别人的库之前 最保险的办法 就是找示例跟着写。 先了解示例。



#### qt库安装
我理解您需要解决Qt项目的依赖问题，并且是在离线环境中。让我为您提供解决方案。

## 在银河麒麟ARM64上处理Qt项目的离线安装

如果您需要解决Qt依赖问题以编译或运行Qt项目，您应该处理以下方面：

### 1. 安装基础Qt库

从您提供的列表中，对于基础Qt5库，您需要：

- `qtbase-opensource-src` (Qt基础源码包)

### 2. 编译安装步骤

1. 首先解压源码包：
    
    ```bash
    tar -xf qtbase-opensource-src-5.11.3.tar.xz
    cd qtbase-opensource-src-5.11.3
    ```
    
2. 配置并编译：
    
    ```bash
    # 创建构建目录
    mkdir build && cd build
    
    # 配置Qt (根据需要调整选项)
    ../configure -prefix /usr/local/qt5 -opensource -confirm-license
    
    # 编译
    make -j4  # 使用4个CPU核心编译，根据您的机器调整
    
    # 安装
    sudo make install
    ```
    
3. 设置环境变量：
    
    ```bash
    # 编辑.bashrc文件
    echo 'export PATH=/usr/local/qt5/bin:$PATH' >> ~/.bashrc
    echo 'export LD_LIBRARY_PATH=/usr/local/qt5/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
    source ~/.bashrc
    ```
    

### 3. 处理特定模块的依赖

如果您的项目需要特定的Qt模块（如QtSerialPort），您可能还需要编译其他模块：

1. 解决QtSerialPort依赖 (示例)：
    
    ```bash
    # 在编译完qtbase后cd <源码目录>/qtserialportqmakemakesudo make install
    ```
    

### 4. 使用预编译的二进制包 (如果有)

如果您能够获取兼容您银河麒麟系统的预编译Qt二进制包，这将是更简单的方法：

1. 确认所有必要的.deb包
2. 使用dpkg安装它们：
    
    ```bash
    sudo dpkg -i *.deb
    ```
    
3. 修复可能的依赖问题：
    
    ```bash
    sudo apt --fix-broken install  # 如果有本地依赖库
    ```
    

### 5. 离线项目编译配置

编译您自己的Qt项目时，确保使用正确的qmake路径：

```bash
/usr/local/qt5/bin/qmake  # 如果是从源码安装的
# 或
qmake  # 如果环境变量已经设置好
```

如果您可以提供更多关于您要解决的具体Qt项目的信息，我可以给您更有针对性的建议。


#### 安装包
## 虚拟依赖包工作原理详解

这个脚本创建了一个"虚拟依赖包"，用来解决软件包依赖问题。让我详细解释它是如何工作的：

### 1. 原理概述

在Debian/Ubuntu软件包管理系统中，有一个重要的概念叫"Provides"（提供）。当软件包A声明它"提供"软件包B时，任何依赖于B的软件包也可以通过安装A来满足依赖关系。

### 2. 脚本步骤解析

```bash
# 创建目录结构
mkdir -p ./qtbase-abi-5-11-3-fix/DEBIAN
```

- 这一行创建了一个名为"qtbase-abi-5-11-3-fix"的目录，以及其中的"DEBIAN"子目录，这是Debian软件包的标准结构。

```bash
# 创建控制文件
echo 'Package: qtbase-abi-5-11-3-fix
Version: 1.0
Section: custom
Priority: optional
Provides: qtbase-abi-5-11-3    # 关键行！声明提供qtbase-abi-5-11-3
Depends: libqt5core5a          # 自己依赖于libqt5core5a
Architecture: arm64
Essential: no
Installed-Size: 0
Maintainer: custom
Description: Fix lack of qtbase-abi-5-11-3 in Kylin OS
' > ./qtbase-abi-5-11-3-fix/DEBIAN/control
```

- 这部分创建了"control"文件，定义了软件包的元数据
- 最关键的是`Provides: qtbase-abi-5-11-3`行，告诉包管理器该软件包提供"qtbase-abi-5-11-3"功能
- `Depends: libqt5core5a`表示该包依赖于libqt5core5a

```bash
# 构建Debian包
dpkg-deb --build ./qtbase-abi-5-11-3-fix
```

- 这一行使用dpkg-deb工具将目录打包成一个.deb文件

### 3. 工作流程

1. 假设您要安装软件包X，它依赖于qtbase-abi-5-11-3
2. 系统中没有qtbase-abi-5-11-3，所以安装会失败
3. 您安装这个虚拟依赖包(qtbase-abi-5-11-3-fix)
4. 包管理器认为qtbase-abi-5-11-3已经被"提供"了
5. 现在尝试安装X，包管理器不再抱怨缺少qtbase-abi-5-11-3

### 4. 为什么有效

- 在许多情况下，qtbase-abi-5-11-3仅仅是一个"版本标识符"，表示存在特定版本的Qt库的ABI(应用程序二进制接口)
- 实际功能已经包含在libqt5core5a等实际的库文件中
- 这个虚拟包没有实际内容(0字节)，只是声明提供了特定功能
- 它自己依赖于libqt5core5a确保实际库被安装

这种方法特别适合解决那些形式上的依赖关系，尤其是当您确定系统中已经有了所需的功能，只是缺少特定的标识符软件包。这是一种"欺骗"包管理系统的巧妙方法，在离线环境中非常有用。