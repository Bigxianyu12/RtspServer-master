# RtspServer

一个使用 C++ 从零实现的轻量级 RTSP 服务器，支持 H264 和 AAC 的音视频流媒体传输。

---

## 目录

- [项目介绍](#项目介绍)
- [功能特性](#功能特性)
- [工程结构](#工程结构)
- [架构设计](#架构设计)
- [核心模块详解](#核心模块详解)
  - [基础组件层](#基础组件层-base)
  - [网络核心层](#网络核心层-net)
  - [扩展模块](#扩展模块-extend)
  - [示例与测试](#示例与测试)
- [开发环境](#开发环境)
- [编译与使用](#编译与使用)
- [关键设计要点](#关键设计要点)
- [联系方式](#联系方式)

---

## 项目介绍

本项目是一个基于 C++ 实现的 RTSP 流媒体服务器，采用 **Reactor 事件驱动模型** 和 **非阻塞 I/O**，完全从零构建（不依赖任何第三方流媒体库）。支持从文件或硬件设备采集音视频数据，通过 RTP 协议进行流式传输。

> **核心设计理念**：轻量、可学习、模块化。代码结构清晰，适合学习 RTSP 协议、Reactor 网络模型、RTP 打包、音视频采集等知识。

---

## 功能特性

### 媒体格式支持

- **视频**：H.264 编码，支持 RTP FU-A 分片打包（NAL 单元 > 1400 字节时自动分片）
- **音频**：AAC 编码，支持 ADTS 格式解析与 RTP 打包

### 传输方式

- **单播**：RTP_OVER_UDP、RTP_OVER_RTSP(TCP)
- **多播**：Multicast（通过取消注释即可启用）

### 媒体来源

- **文件**：H.264 裸流文件（.h264）、AAC 音频文件（.aac，含 ADTS 头）
- **设备采集**：
  - V4L2 摄像头 → x264 软编码 → H.264（需编译启用）
  - ALSA 音频设备 → FAAC 编码 → AAC（需编译启用）

### 服务器特性

- 支持 RTSP 标准命令：OPTIONS、DESCRIBE、SETUP、PLAY、TEARDOWN、GET_PARAMETER
- 自动生成 SDP 描述
- 支持多客户端同时播放
- 支持单 Session 多 Track（视频+音频同时传输）

---

## 工程结构

```
RtspServer/
├── src/                           # 源码目录
│   ├── base/                      # 【基础组件层】- 独立于网络的基础能力
│   │   ├── Allocator.h/.cpp       #   内存分配器（Slab 分配器）
│   │   ├── AsyncLogging.h/.cpp    #   异步日志（双缓冲 + 独立写盘线程）
│   │   ├── Condition.h/.cpp       #   条件变量封装（pthread_cond）
│   │   ├── Construct.h            #   placement new 构造器模板
│   │   ├── Logging.h/.cpp         #   日志前端（宏定义日志接口）
│   │   ├── Mutex.h/.cpp           #   互斥锁封装
│   │   ├── New.h                  #   统一内存分配/释放模板（New/Delete）
│   │   ├── Sem.h/.cpp             #   信号量封装
│   │   ├── Thread.h/.cpp          #   线程封装（pthread）
│   │   └── ThreadPool.h/.cpp      #   线程池
│   ├── net/                       # 【网络核心层】- Reactor + RTSP协议
│   │   ├── poller/                #   IO多路复用模块
│   │   │   ├── Poller.h/.cpp      #     抽象基类
│   │   │   ├── SelectPoller.h/.cpp#     select 实现
│   │   │   ├── PollPoller.h/.cpp  #     poll 实现
│   │   │   └── EPollPoller.h/.cpp #     epoll 实现
│   │   ├── Event.h/.cpp           #   事件系统（IOEvent/TimerEvent/TriggerEvent）
│   │   ├── EventScheduler.h/.cpp  #   事件调度器（Reactor 核心循环）
│   │   ├── Timer.h/.cpp           #   定时器管理（timerfd + multimap）
│   │   ├── Buffer.h/.cpp          #   网络缓冲区（readv 分散读优化）
│   │   ├── InetAddress.h/.cpp     #   IPv4 地址封装
│   │   ├── SocketsOps.h/.cpp      #   Socket 系统调用封装
│   │   ├── TcpSocket.h/.cpp       #   TCP Socket 封装
│   │   ├── Acceptor.h/.cpp        #   连接接收器
│   │   ├── TcpConnection.h/.cpp   #   TCP 连接抽象
│   │   ├── TcpServer.h/.cpp       #   TCP 服务器基类
│   │   ├── UsageEnvironment.h/.cpp#   环境上下文（连接 Scheduler 和 ThreadPool）
│   │   ├── RtspServer.h/.cpp      #   RTSP 服务器
│   │   ├── RtspConnection.h/.cpp  #   RTSP 连接（协议解析与命令处理）
│   │   ├── MediaSession.h/.cpp    #   媒体会话（SDP生成/Track管理/多播）
│   │   ├── MediaSource.h/.cpp     #   媒体源抽象（生产者-消费者模型）
│   │   ├── Rtp.h                  #   RTP 协议头与 RtpPacket
│   │   ├── RtpInstance.h          #   RTP/RTCP 实例（UDP/TCP 发送）
│   │   ├── RtpSink.h/.cpp         #   RTP 打包基类（定时器驱动）
│   │   ├── H264RtpSink.h/.cpp     #   H.264 RTP 打包（FU-A 分片）
│   │   ├── AACRtpSink.h/.cpp      #   AAC RTP 打包（AU-headers）
│   │   ├── H264FileMediaSource.h/.cpp # H.264 文件媒体源
│   │   └── AACFileMediaSource.h/.cpp  # AAC 文件媒体源
│   └── extend/                    # 【扩展模块】- 依赖第三方库
│       ├── v4l2/
│       │   ├── V4l2.h/.cpp        #   V4L2 设备操作封装
│       │   └── V4l2MediaSource.h/.cpp  # V4L2 摄像头媒体源（依赖 libx264）
│       └── alsa/
│           └── AlsaMediaSource.h/.cpp  # ALSA 音频采集（依赖 alsa-lib + libfaac）
├── example/                       # 示例程序
│   ├── 01_h264_rtsp_server.cpp    #   传输 H.264 视频文件
│   ├── 02_aac_rtsp_server.cpp     #   传输 AAC 音频文件
│   ├── 03_h264_aac_rtsp_server.cpp#   同时传输音视频文件
│   ├── 04_v4l2_rtsp_server.cpp    #   采集 V4L2 摄像头并编码传输（需编译启用）
│   └── 05_alsa_rtsp_server.cpp    #   采集 ALSA 音频并编码传输（需编译启用）
├── test/                          # 单元测试程序
│   ├── 01_test_log.cpp            #   日志系统测试
│   ├── 02_test_add_event.cpp      #   事件系统测试
│   ├── 03_test_thread_pool.cpp    #   线程池测试
│   └── 04_test_acceptor.cpp       #   TCP 接收器测试
├── pic/                           # 项目图片资源
├── Makefile                       # 编译配置文件
└── README.md                      # 本文件
```

---

## 架构设计

### 整体架构图

```
┌──────────────────────────────────────────────────────────┐
│                    示例层 (example/)                      │
│   01_h264_rtsp_server  02_aac_rtsp_server  ...           │
└──────────────┬───────────────────────────────────────────┘
               │
┌──────────────▼───────────────────────────────────────────┐
│                  RTSP 协议层 (net/)                       │
│  ┌──────────┐  ┌────────────────┐  ┌─────────────────┐  │
│  │RtspServer│  │RtspConnection  │  │MediaSession     │  │
│  │(继承TCPS)│  │(OPTIONS/DE-    │  │(SDP/Track管理/  │  │
│  │ 连接管理 │  │ SCRIBE/SETUP/  │  │ 多播)           │  │
│  │ 会话管理 │  │ PLAY/TEARDOWN) │  └────────┬────────┘  │
│  └──────────┘  └────────────────┘           │            │
├──────────────────────────────────────────────┴──────────┤
│                  RTP 传输层                               │
│  ┌────────────┐  ┌──────────┐  ┌─────────────────────┐  │
│  │RtpSink     │  │RtpInst-  │  │H264RtpSink/         │  │
│  │(定时器驱动 │  │ ance     │  │AACRtpSink           │  │
│  │  +回调分发)│  │(UDP/TCP) │  │(具体编解码器打包)    │  │
│  └──────┬─────┘  └──────────┘  └─────────────────────┘  │
│         │                                                │
│  ┌──────▼─────┐  ┌────────────────────────────────────┐  │
│  │MediaSource │  │  H264FileMediaSource/              │  │
│  │(生产者-    │  │  AACFileMediaSource/               │  │
│  │ 消费者)    │  │  V4l2MediaSource/AlsaMediaSource   │  │
│  └────────────┘  └────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                Reactor 网络核心                            │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │EventScheduler│  │Poller        │  │TimerManager   │  │
│  │(事件循环)    │  │(select/poll/ │  │(timerfd +     │  │
│  │              │  │ epoll)       │  │ multimap)     │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │Acceptor      │  │TcpConnection │  │Buffer         │  │
│  │(接收连接)    │  │(IO事件处理)   │  │(读写缓冲区)    │  │
│  └──────────────┘  └──────────────┘  └───────────────┘  │
├─────────────────────────────────────────────────────────┤
│              基础组件层 (base/)                           │
│  ┌───────────┐ ┌──────────┐ ┌────────┐ ┌────────────┐  │
│  │ThreadPool │ │Allocator │ │Logger  │ │AsyncLogging│  │
│  │           │ │(内存池)  │ │(前端)  │ │(后端+双缓冲) │  │
│  └───────────┘ └──────────┘ └────────┘ └────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 核心设计模式

#### 1. Reactor 事件驱动模型
```
EventScheduler::loop()
    ├── 处理 TriggerEvent（一次性触发事件队列）
    ├── Poller::handleEvent()（IO多路复用等待事件）
    │       └── IOEvent::handleEvent()（分发到对应回调）
    └── handleOtherEvent()（处理跨线程回调）
```

- 非阻塞 IO + IO 多路复用（select/poll/epoll）
- 统一的事件抽象：IOEvent（读写错误）/ TimerEvent（定时器）/ TriggerEvent（触发事件）
- 通过 eventfd 实现事件唤醒机制

#### 2. 生产者-消费者模型（媒体数据流）
```
线程池任务（生产者）             定时器（消费者-RtpSink）
    │                                  │
    │  readFrame()                     │  timeoutCallback()
    │  ┌──────────┐                    │
    │  │ 读取/编码 │                    │
    │  │ 音视频帧  │                    │
    │  └────┬─────┘                    │
    │       ▼                          │
    │  ┌──────────┐    getFrame()      │
    │  │ 输出队列  │◄──────────────────│
    │  │(AVFrame) │──── putFrame() ──►│
    │  └──────────┘                    │
    │       │                          │
    │  ┌────▼─────┐                   │
    │  │ 输入队列  │                   │
    │  └──────────┘                    │
```

#### 3. 分层抽象
- **Poller 抽象**：统一接口，三种实现（select/poll/epoll），可运行时切换
- **MediaSource 抽象**：统一数据源接口，文件/设备多种实现
- **RtpSink 抽象**：统一 RTP 打包接口，H264/AAC 不同封装

---

## 核心模块详解

### 基础组件层 (base/)

#### Allocator — 内存分配器

采用经典的 **Slab 分配器** 设计（类似 SGI STL 的 `__default_alloc_template`）：

- **16 个自由链表**，分别管理 8/16/24/.../128 字节的内存块
- **<= 128 字节**：从对应自由链表获取，释放时回收到链表
- **> 128 字节**：直接 malloc/free
- **批量分配**：当链表为空时，一次性从堆分配 20 块（或不少于 1 块），避免频繁系统调用
- **内存碎片控制**：缓存块中剩余的小碎片会插入到对应大小的自由链表中
- **线程安全**：分配/释放操作由 Mutex 保护

#### New/Delete — 统一分配接口

```cpp
// 统一使用 New<T>::allocate() 代替 new，Delete::release() 代替 delete
RtspServer* server = New<RtspServer>::allocate(env, addr);
// 等价于 Allocator::allocate + placement new

Delete::release(server);
// 等价于 destructor call + Allocator::deallocate
```

#### 日志系统 — 前后端分离

**前端 (Logger)**：通过宏 `LOG_DEBUG` / `LOG_WARNING` / `LOG_ERROR` 格式化日志，包含时间戳、日志级别、文件名、函数名、行号。

**后端 (AsyncLogging)**：

- **独立线程**：日志写入由后台线程负责，不阻塞主流程
- **双缓冲技术**：4 个 LogBuffer（每个 1MB）循环使用
  - `mCurBuffer`：当前写入缓冲区
  - `mFreeBuffer`：空闲缓冲区队列
  - `mFlushBuffer`：待写入磁盘队列
- **写入策略**：每 3 秒或缓冲区满时触发写入
- **RAII 设计**：Logger 对象析构时自动输出日志内容

#### 线程池 (ThreadPool)

- 固定数量线程（创建时指定）
- 任务队列 + 条件变量驱动
- 线程安全：Mutex 保护任务队列

---

### 网络核心层 (net/)

#### IO 多路复用 — Poller 系列

```
Poller (抽象基类)
  ├── SelectPoller: 基于 select()，最大 fd 数受限（FD_SETSIZE）
  ├── PollPoller:   基于 poll()，无最大 fd 限制，O(n) 扫描
  └── EPollPoller:  基于 epoll，O(1) 事件复杂度，适合高并发
```

三者通过工厂模式在 `EventScheduler::createNew()` 时选择，默认示例中使用 `POLLER_SELECT`。

#### 事件系统 — Event

**IOEvent**：封装文件描述符的读写错误事件
- 位掩码设计：`EVENT_READ | EVENT_WRITE | EVENT_ERROR`
- 支持动态启用/禁用各类事件
- 事件触发时回调对应的 Read/Write/Error 处理函数

**TimerEvent**：定时事件，配合 TimerManager 使用
**TriggerEvent**：一次性触发事件，在事件循环的每次迭代开始时处理

#### 事件调度器 — EventScheduler

事件循环核心：

```cpp
void EventScheduler::loop()
{
    while(mQuit != true)
    {
        handleTriggerEvents();  // 处理触发事件
        mPoller->handleEvent(); // 等待并处理 IO 事件
        handleOtherEvent();     // 处理跨线程回调
    }
}
```

- **唤醒机制**：通过 `eventfd` 实现 `wakeup()`，可跨线程唤醒事件循环
- **跨线程回调**：`runInLocalThread()` 将回调投递到事件循环所在的线程执行

#### 定时器管理 — Timer

- 基于 **Linux timerfd_create**，将定时器文件描述符纳入 IO 多路复用管理
- 使用 **multimap** 按超时时间排序存储定时器
- 支持一次性定时器和周期性定时器
- 支持动态添加/删除定时器

#### 网络缓冲区 — Buffer

- 设计类似 Muduo 网络库的 Buffer
- **readv 分散读优化**：结合栈上 extrabuf（64KB），减少系统调用
- 自动扩容：`ensureWritableBytes()` + `makeSpace()`
- 查找 `\r\n`：`findCRLF()` / `findLastCrlf()` 用于 RTSP 协议解析

#### RTSP 协议处理

**支持的命令**：
| 命令 | 处理函数 | 功能 |
|------|---------|------|
| OPTIONS | `handleCmdOption()` | 返回服务器支持的 RTSP 方法列表 |
| DESCRIBE | `handleCmdDescribe()` | 返回 SDP 会话描述 |
| SETUP | `handleCmdSetup()` | 建立 RTP/RTCP 传输通道 |
| PLAY | `handleCmdPlay()` | 开始播放（使 RTP 实例进入活跃状态） |
| TEARDOWN | `handleCmdTeardown()` | 断开连接 |
| GET_PARAMETER | `handleCmdGetParamter()` | 获取参数（当前为空实现） |

**SETUP 流程**：
```
客户端请求 SETUP
    │
    ├─ 解析 Transport 头
    │   ├─ RTP/AVP/TCP → RTP_OVER_TCP 模式
    │   └─ RTP/AVP     → RTP_OVER_UDP 模式（自动绑定端口）
    │
    ├─ 解析 MediaTrack（track0/track1）
    │
    ├─ 根据传输模式创建 RtpInstance
    │   ├─ createRtpOverTcp()    — TCP 隧道模式
    │   └─ createRtpRtcpOverUdp()— UDP 模式（自动分配端口）
    │
    └─ 将 RtpInstance 添加到 MediaSession 的对应 Track
```

#### 媒体会话 — MediaSession

- 管理最多 **2 个 Track**（TrackId0=视频, TrackId1=音频）
- 每个 Track 包含：`RtpSink` + `RtpInstance` 列表
- **SDP 生成**：根据 Track 信息自动生成 SDP 描述
- **多播支持**：启用时自动分配多播地址和端口

#### 媒体源 — MediaSource（生产者-消费者）

```cpp
class MediaSource {
    AVFrame mAVFrames[DEFAULT_FRAME_NUM];  // 帧缓冲池
    queue<AVFrame*> mAVFrameInputQueue;    // 待填充队列（生产者）
    queue<AVFrame*> mAVFrameOutputQueue;   // 已就绪队列（消费者）
};
```

- **生产者**：线程池任务调用 `readFrame()` 读取/编码音视频帧
- **消费者**：RtpSink 定时器定期调用 `getFrame()` 获取帧数据进行 RTP 打包
- **循环缓冲**：DEFAULT_FRAME_NUM=4 个 AVFrame 循环使用

#### RTP 打包

**H.264 RTP 打包 (H264RtpSink)**：
- NAL 单元 ≤ 1400 字节：单包发送
- NAL 单元 > 1400 字节：**FU-A 分片**（RFC 3984）
  - 第一个分片：Start bit = 1
  - 中间分片：仅有 FU Indicator + FU Header
  - 最后分片：End bit = 1
- SPS/PPS 包（NAL Type 7/8）不加时间戳递增
- 时间戳：`timestamp += 90000 / fps`（H.264 默认时钟频率 90kHz）

**AAC RTP 打包 (AACRtpSink)**：
- 去掉 ADTS 头部（7 字节），只传输裸 AAC 数据
- AU-headers 格式：2 字节头描述 AAC 帧长度
- 时间戳：`timestamp += sampleRate * (1000/fps) / 1000`

---

### 扩展模块 (extend/)

#### V4L2 摄像头采集（依赖 libx264）

**数据流**：
```
V4L2 设备 → YUYV 原始数据 → x264 编码器 → H.264 NAL 单元 → RTP 打包
```

**初始化流程**：打开设备 → 查询能力 → 设置输入 → 设置格式(YUYV, 640x480) → 申请 mmap 缓存 → 入队列 → 开始采集

**编码流程**：
1. `v4l2_poll()` 等待帧就绪
2. `v4l2_dqbuf()` 取出帧数据
3. `x264_encoder_encode()` 编码为 H.264
4. NAL 单元入队列 → `readFrame()` 时取出
5. `v4l2_qbuf()` 将缓存重新入队列

#### ALSA 音频采集（依赖 alsa-lib + libfaac）

**数据流**：
```
ALSA 设备 → PCM 原始数据 → FAAC 编码器 → AAC 帧 → RTP 打包
```

- 默认配置：44100Hz 采样率、双通道、16 位有符号、1024 帧/周期
- 帧率自适应：`fps = sampleRate / frames`（约 43fps）

---

### 示例与测试

#### 示例程序

| 示例 | 文件 | 说明 | 依赖 |
|------|------|------|------|
| 01 | `h264_rtsp_server` | 传输 H.264 视频文件 | 无 |
| 02 | `aac_rtsp_server` | 传输 AAC 音频文件 | 无 |
| 03 | `h264_aac_rtsp_server` | 同时传输音视频文件 | 无 |
| 04 | `v4l2_rtsp_server` | 采集 V4L2 摄像头并编码传输 | libx264 |
| 05 | `alsa_rtsp_server` | 采集 ALSA 音频并编码传输 | alsa-lib, libfaac |

#### 测试程序

| 测试 | 文件 | 说明 |
|------|------|------|
| 01 | `test_log` | 日志系统压力测试 |
| 02 | `test_add_event` | 事件系统（IO/定时/触发事件）功能测试 |
| 03 | `test_thread_pool` | 线程池任务调度测试 |
| 04 | `test_acceptor` | TCP 连接接收器功能测试 |

---

## 开发环境

- **系统**：Linux (Ubuntu 14.04+)
- **编译器**：gcc 4.8.4+（支持 C++11）
- **链接库**：`-lpthread -lrt`
- **可选依赖**：
  - `libx264-dev` — V4L2 示例需要
  - `libasound2-dev` — ALSA 示例需要
  - `libfaac-dev` — ALSA 示例需要

---

## 编译与使用

### 1、基本使用（传输音视频文件）


# 编译
cd RtspServer/
make

# 编译后 example/ 目录下生成：
#   h264_rtsp_server      — 传输 H.264 视频文件
#   aac_rtsp_server       — 传输 AAC 音频文件
#   h264_aac_rtsp_server  — 同时传输音视频文件

# 运行 H.264 视频服务器
cd example/
./h264_rtsp_server test.h264
# 输出：Play the media using the URL "rtsp://192.168.31.115:8554/live"

# 打开 VLC，输入 URL 即可播放
```

```bash
# 运行 AAC 音频服务器
./aac_rtsp_server test.aac

# 同时传输音视频
./h264_aac_rtsp_server test.h264 test.aac
```

### 2、采集 V4L2 摄像头

需要依赖 libx264 库，编译步骤详见原版 README。

```bash
# 启用 V4L2 支持（修改 Makefile 第一行为 V4L2_SUPPORT=y）
make

# 运行
./v4l2_rtsp_server /dev/video0
```

### 3、采集 ALSA 音频设备

需要依赖 alsa-lib 和 libfaac 库，编译步骤详见原版 README。

```bash
# 启用 ALSA 支持（修改 Makefile 第二行为 ALSA_SUPPORT=y）
make

# 运行
./alsa_rtsp_server hw:0,0
```

### 4、切换传输模式

**RTP_OVER_RTSP (TCP)**：

在 VLC 中设置：`工具` >> `首选项` >> `输入/编解码器` >> `live555 流传输` >> `RTP over RTSP(TCP)`

**多播模式**：

取消示例中的注释 `//session->startMulticast();`，重新编译即可。

**切换 IO 多路复用模型**：

```cpp
// 在示例中修改创建参数即可切换
EventScheduler* scheduler = EventScheduler::createNew(EventScheduler::POLLER_SELECT);
EventScheduler* scheduler = EventScheduler::createNew(EventScheduler::POLLER_POLL);
EventScheduler* scheduler = EventScheduler::createNew(EventScheduler::POLLER_EPOLL);
```

---

## 关键设计要点

### 1、内存管理

自定义 Allocator 管理小内存（≤128 字节），减少系统调用和内存碎片。所有对象的分配和释放统一通过 `New<T>::allocate()` 和 `Delete::release()` 接口，便于切换分配策略。

### 2、事件驱动

Reactor 模型将 IO 事件、定时器、触发事件统一管理。定时器通过 `timerfd` 实现，将定时器描述符加入 IO 多路复用，避免专门的定时器线程。

### 3、媒体数据流

生产者-消费者模式解耦了数据采集/编码（线程池）和数据传输（定时器驱动）。循环队列缓冲 4 帧数据，平衡生产和消费的速度。

### 4、日志双缓冲

前端格式化日志到栈上缓冲区，后端独立线程将数据写入磁盘。双缓冲（4 个 1MB 缓冲区）减少磁盘 IO 次数，避免频繁锁竞争。

### 5、可扩展性

- **新的媒体源**：继承 `MediaSource`，实现 `readFrame()` 即可
- **新的编码格式**：继承 `RtpSink`，实现 `handleFrame()`、`getMediaDescription()`、`getAttribute()` 即可
- **新的 IO 多路复用**：继承 `Poller`，实现四个纯虚函数即可

---

