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