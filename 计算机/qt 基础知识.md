
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