# 阶段 1：Android Binder 机制背景

> 对应源码：`libgbinder-master/README`、`libgbinder-master/src/binder.h`

## 1.1 Binder 是什么

Binder 是 Android 系统的**核心 IPC（进程间通信）机制**。与传统的管道、消息队列、共享内存等 IPC 方式不同，Binder 基于 **Linux 内核驱动**实现，只需一次拷贝即可完成跨进程数据传输。

### 核心特点

| 特点 | 说明 |
|------|------|
| **基于内核驱动** | 通过 `/dev/binder` 字符设备与内核交互 |
| **一次拷贝** | 数据从发送方用户空间 → 内核缓冲区 → 接收方用户空间映射，只需一次 copy |
| **C/S 架构** | 天然的 Client/Server 模型 |
| **引用计数** | 内核维护对象的强/弱引用，自动管理生命周期 |
| **死亡通知** | 可监听对端进程死亡，自动清理资源 |
| **线程池** | 内核自动管理 Binder 线程池，按需 spawn 新线程 |

## 1.2 Binder 设备节点

Android 系统中存在多个 Binder 设备节点，对应不同的通信域：

```
/dev/binder    → Framework 层 IPC（AIDL 接口）
/dev/hwbinder  → HAL 层 IPC（HIDL 接口，旧版硬件抽象）
/dev/vndbinder → Vendor 层 IPC（厂商间通信）
```

在 libgbinder 中，默认配置为：

```c
// src/gbinder_types.h
#define GBINDER_DEFAULT_BINDER     "/dev/binder"
#define GBINDER_DEFAULT_HWBINDER  "/dev/hwbinder"
```

## 1.3 ServiceManager — Binder 世界的"DNS"

ServiceManager 是 Binder 机制的核心组件，充当**名称→句柄的映射服务**。

```
┌─────────────┐         ┌──────────────────┐         ┌─────────────┐
│   Client    │         │  ServiceManager  │         │   Server     │
│             │  3.查询  │                  │  1.注册  │              │
│  get_service├────────>│  "service_name"  │<────────┤ add_service  │
│             │         │    → handle(0x5) │         │              │
│             │<────────┤                  │         │              │
│  4.handle=5 │  返回句柄 │                  │         │              │
│             │         │                  │         │              │
│  5.用 handle│         │                  │         │              │
│     直接通信 │─────────────────────────────────────>│ 处理事务      │
└─────────────┘   transact(handle=5, code, data)    └─────────────┘
```

### ServiceManager 的工作流程

1. **Server 注册服务**：Server 创建 LocalObject，通过 `add_service(name, obj)` 注册到 SM
2. **Client 查询服务**：Client 通过 `get_service(name)` 从 SM 获取 RemoteObject（handle）
3. **直接通信**：拿到 handle 后，Client 与 Server 直接通过 Binder 驱动通信，不再经过 SM

> ServiceManager 本身也是一个 Binder 服务（handle=0），它是所有 Binder 服务的入口。

## 1.4 事务（Transaction）模型

Binder 的通信基本单元是**事务**。一次事务包含：

```
binder_transaction_data {
    target: { handle 或 ptr }     // 目标对象
    cookie:                        // 本地对象的 cookie
    code:                          // 事务码（方法编号）
    flags:                         // 事务标志（如 TF_ONE_WAY）
    sender_pid:                    // 发送方 PID
    sender_euid:                   // 发送方 EUID
    data_size:                     // 数据大小
    offsets_size:                  // offsets 数组大小
    data: { buffer, offsets }      // 数据缓冲区 + 对象偏移数组
}
```

### 事务标志（transaction_flags）

```c
// src/binder.h
enum transaction_flags {
    TF_ONE_WAY     = 0x01,  // 单向事务，不需要回复
    TF_ROOT_OBJECT = 0x04,  // 数据包含根对象
    TF_STATUS_CODE = 0x08,  // 数据是状态码
    TF_ACCEPT_FDS  = 0x10,  // 接受 FD 传递
};
```

### 事务码（code）

事务码标识要调用的方法。在 libgbinder 中：

```c
// include/gbinder_types.h
#define GBINDER_FIRST_CALL_TRANSACTION (0x00000001)
#define GBINDER_TX_FLAG_ONEWAY         (0x01)
```

## 1.5 AIDL vs HIDL

Android 有两种接口定义语言，对应不同的 Binder 域：

| 特性 | AIDL | HIDL |
|------|------|------|
| **全称** | Android Interface Definition Language | HAL Interface Definition Language |
| **设备** | `/dev/binder` | `/dev/hwbinder` |
| **用途** | Framework 层 IPC | 硬件抽象层（HAL） |
| **版本管理** | 接口版本内嵌 | 严格版本化（如 `@1.0`） |
| **头部格式** | 不同的 RPC header | 不同的 RPC header |
| **libgbinder 后端** | `aidl`, `aidl2`~`aidl6` | `hidl` |

> Android 不断在不同版本中修改 AIDL 和 HIDL 的 RPC 头部格式和 ServiceManager 协议。libgbinder 通过可配置后端来适配这些变化。

## 1.6 内核 Binder 协议 — BC/BR 命令

Binder 驱动使用**命令-返回协议**，用户空间发送 BC（Binder Command），内核返回 BR（Binder Return）。

### BC 命令（用户空间 → 内核）

```c
// src/binder.h — binder_driver_command_protocol
BC_TRANSACTION        // 发起事务
BC_REPLY              // 回复事务
BC_ACQUIRE_RESULT     // 确认 acquire
BC_FREE_BUFFER        // 释放缓冲区
BC_INCREFS            // 增加弱引用
BC_ACQUIRE            // 增加强引用
BC_RELEASE            // 减少强引用
BC_DECREFS            // 减少弱引用
BC_INCREFS_DONE       // 确认 increfs 完成
BC_ACQUIRE_DONE       // 确认 acquire 完成
BC_REGISTER_LOOPER    // 注册 looper 线程
BC_ENTER_LOOPER       // 进入 looper
BC_EXIT_LOOPER        // 退出 looper
BC_REQUEST_DEATH_NOTIFICATION  // 请求死亡通知
BC_CLEAR_DEATH_NOTIFICATION    // 清除死亡通知
BC_DEAD_BINDER_DONE   // 确认死亡通知处理完成
BC_TRANSACTION_SG     // 带 scatter-gather 的事务
BC_REPLY_SG           // 带 scatter-gather 的回复
```

### BR 命令（内核 → 用户空间）

```c
// src/binder.h — binder_driver_return_protocol
BR_ERROR              // 发生错误
BR_OK                 // 正常
BR_TRANSACTION        // 收到事务请求
BR_REPLY              // 收到事务回复
BR_ACQUIRE_RESULT     // acquire 结果
BR_DEAD_REPLY         // 对端已死
BR_TRANSACTION_COMPLETE  // 事务已提交
BR_INCREFS            // 请求增加弱引用
BR_ACQUIRE            // 请求增加强引用
BR_RELEASE            // 请求减少强引用
BR_DECREFS            // 请求减少弱引用
BR_NOOP               // 空操作
BR_SPAWN_LOOPER       // 内核请求创建新 looper 线程
BR_FINISHED           // 完成
BR_DEAD_BINDER        // 对端死亡通知
BR_CLEAR_DEATH_NOTIFICATION_DONE  // 死亡通知清除完成
BR_FAILED_REPLY       // 事务失败
```

### 一次同步事务的完整流程

```
Client 线程                          Kernel Driver                        Server 线程
    │                                     │                                    │
    │  BC_TRANSACTION(handle, code, data)  │                                    │
    ├────────────────────────────────────>│                                    │
    │                                     │                                    │
    │  BR_TRANSACTION_COMPLETE            │                                    │
    │<────────────────────────────────────┤  BR_TRANSACTION(data)             │
    │                                     ├───────────────────────────────────>│
    │                                     │                                    │
    │                                     │  (Server 处理请求...)                │
    │                                     │                                    │
    │                                     │  BC_REPLY(reply_data)              │
    │                                     │<───────────────────────────────────┤
    │  BR_REPLY(reply_data)               │                                    │
    │<────────────────────────────────────┤                                    │
    │                                     │                                    │
    │  (处理回复...)                       │                                    │
```

### BC/BR 命令分类与具体例子

Binder 的 BC/BR 命令虽然多，但可以按功能分为 5 类。下面逐类举例说明。

#### 1. 事务类 — 最核心的通信命令

| 命令 | 方向 | 携带数据 | 说明 |
|------|------|---------|------|
| BC_TRANSACTION | 用户→内核 | binder_transaction_data | 发起一次事务请求 |
| BC_REPLY | 用户→内核 | binder_transaction_data | 回复一次事务 |
| BC_TRANSACTION_SG | 用户→内核 | binder_transaction_data_sg | 带 scatter-gather 的事务 |
| BC_REPLY_SG | 用户→内核 | binder_transaction_data_sg | 带 scatter-gather 的回复 |
| BR_TRANSACTION | 内核→用户 | binder_transaction_data | 收到事务请求 |
| BR_REPLY | 内核→用户 | binder_transaction_data | 收到事务回复 |
| BR_TRANSACTION_COMPLETE | 内核→用户 | 无 | 事务已提交（不代表已完成） |
| BR_FAILED_REPLY | 内核→用户 | 无 | 事务失败（如目标已死） |
| BR_DEAD_REPLY | 内核→用户 | 无 | 对端进程已死，回复无法到达 |
| BR_ERROR | 内核→用户 | __s32 | 发生错误，附带错误码 |

**例子：Client 调用 Server 的方法，但 Server 进程已死**

```
Client 线程                          Kernel Driver
    │                                     │
    │  BC_TRANSACTION(handle=5, code=1)    │
    ├────────────────────────────────────>│
    │                                     │  内核: handle=5 → binder_node
    │                                     │        → node->proc == NULL (Server 已死)
    │  BR_DEAD_REPLY                      │
    │<────────────────────────────────────┤
    │                                     │
    │  (Client 知道对端已死，清理资源)       │
```

> **BR_TRANSACTION_COMPLETE vs BR_REPLY**：`BR_TRANSACTION_COMPLETE` 只表示"你的 BC_TRANSACTION 已被内核接收"，对于同步事务，Client 线程会继续等待 `BR_REPLY`。对于 `TF_ONE_WAY` 单向事务，收到 `BR_TRANSACTION_COMPLETE` 就结束了，不会有 `BR_REPLY`。

**例子：单向事务（TF_ONE_WAY）**

```
Client 线程                          Kernel Driver                        Server 线程
    │                                     │                                    │
    │  BC_TRANSACTION(handle=5, code=1,   │                                    │
    │    flags=TF_ONE_WAY)                 │                                    │
    ├────────────────────────────────────>│                                    │
    │                                     │                                    │
    │  BR_TRANSACTION_COMPLETE            │  BR_TRANSACTION(data)             │
    │<────────────────────────────────────┤───────────────────────────────────>│
    │                                     │                                    │
    │  (Client 不等回复，继续干别的)         │  (Server 异步处理)                  │
    │                                     │                                    │
    │  ← 没有 BR_REPLY！                   │                                    │
```

#### 2. 引用计数类 — 内核管理对象生命周期

| 命令 | 方向 | 携带数据 | 说明 |
|------|------|---------|------|
| BC_ACQUIRE | 用户→内核 | __u32 (handle) | 增加强引用 |
| BC_RELEASE | 用户→内核 | __u32 (handle) | 减少强引用 |
| BC_INCREFS | 用户→内核 | __u32 (handle) | 增加弱引用 |
| BC_DECREFS | 用户→内核 | __u32 (handle) | 减少弱引用 |
| BC_ACQUIRE_DONE | 用户→内核 | binder_ptr_cookie | 确认强引用增加完成 |
| BC_INCREFS_DONE | 用户→内核 | binder_ptr_cookie | 确认弱引用增加完成 |
| BR_ACQUIRE | 内核→用户 | binder_ptr_cookie | 请求增加强引用 |
| BR_RELEASE | 内核→用户 | binder_ptr_cookie | 请求减少强引用 |
| BR_INCREFS | 内核→用户 | binder_ptr_cookie | 请求增加弱引用 |
| BR_DECREFS | 内核→用户 | binder_ptr_cookie | 请求减少弱引用 |

**例子：Client 获取 RemoteObject 时的引用计数**

```
Client 进程                           Kernel Driver                        Server 进程
    │                                     │                                    │
    │  (getService 返回 handle=5)          │                                    │
    │                                     │                                    │
    │  BC_ACQUIRE(handle=5)               │  内核: Client 的 binder_ref 强引用+1  │
    ├────────────────────────────────────>│  内核: Server 的 binder_node 强引用+1 │
    │                                     │                                    │
    │                                     │  BR_ACQUIRE(cookie=0x7fff1234)     │
    │                                     ├───────────────────────────────────>│
    │                                     │                                    │
    │                                     │  BC_ACQUIRE_DONE(ptr, cookie)      │
    │                                     │<───────────────────────────────────┤
    │                                     │  内核: 确认 Server 已增加强引用        │
    │                                     │                                    │
    │  (Client 不再需要时)                   │                                    │
    │  BC_RELEASE(handle=5)               │  内核: Client 的 binder_ref 强引用-1  │
    ├────────────────────────────────────>│  内核: Server 的 binder_node 强引用-1 │
    │                                     │                                    │
    │                                     │  BR_RELEASE(cookie=0x7fff1234)     │
    │                                     ├───────────────────────────────────>│
    │                                     │  (Server 减少本地对象强引用)           │
```

> **为什么需要 BC_ACQUIRE_DONE？** 内核请求 Server 增加强引用（BR_ACQUIRE），但内核不能直接操作 Server 用户空间的对象，需要 Server 自己处理。Server 处理完后用 BC_ACQUIRE_DONE 告诉内核"我做完了"。这是一个异步确认机制。

#### 3. Looper 线程管理类 — 内核控制线程池

| 命令 | 方向 | 携带数据 | 说明 |
|------|------|---------|------|
| BC_ENTER_LOOPER | 用户→内核 | 无 | 主线程进入 looper |
| BC_REGISTER_LOOPER | 用户→内核 | 无 | 辅助线程注册为 looper |
| BC_EXIT_LOOPER | 用户→内核 | 无 | 退出 looper |
| BR_SPAWN_LOOPER | 内核→用户 | 无 | 内核请求创建新 looper 线程 |
| BR_NOOP | 内核→用户 | 无 | 空操作（轮询时返回） |
| BR_OK | 内核→用户 | 无 | 正常 |
| BR_FINISHED | 内核→用户 | 无 | 完成 |

**例子：Server 线程池扩容**

```
Server 主线程                        Kernel Driver
    │                                     │
    │  BC_ENTER_LOOPER                     │  (主线程注册为 looper)
    ├────────────────────────────────────>│
    │                                     │
    │  (大量事务涌入，主线程忙不过来)        │
    │                                     │
    │  BR_SPAWN_LOOPER                    │  内核: "你需要更多线程！"
    │<────────────────────────────────────┤
    │                                     │
    │  (Server 创建新线程)                  │
    │                                     │
    │  BC_REGISTER_LOOPER                  │  (新线程注册为 looper)
    ├────────────────────────────────────>│
    │                                     │
    │  (新线程可以接收 BR_TRANSACTION 了)   │
```

> **BR_SPAWN_LOOPER** 是内核主动要求用户空间创建新线程的命令。当内核发现所有 looper 线程都在忙，就会发这个命令。libgbinder 收到后创建新 GLib 线程并调用 `BC_REGISTER_LOOPER` 注册。

#### 4. 缓冲区管理类 — 释放 mmap 共享内存

| 命令 | 方向 | 携带数据 | 说明 |
|------|------|---------|------|
| BC_FREE_BUFFER | 用户→内核 | binder_uintptr_t | 释放 mmap 缓冲区 |

**例子：Server 处理完事务后释放缓冲区**

```
Server 线程                          Kernel Driver
    │                                     │
    │  BR_TRANSACTION(data_ptr=0xABCD)    │  (数据在 mmap 共享内存区)
    │<────────────────────────────────────┤
    │                                     │
    │  (Server 读取 data_ptr 处的数据)      │
    │  (Server 处理请求)                    │
    │  (Server 写 Reply)                   │
    │                                     │
    │  BC_REPLY(reply_data)               │
    ├────────────────────────────────────>│
    │                                     │
    │  BC_FREE_BUFFER(0xABCD)             │  (告诉内核：这块 mmap 内存我用完了)
    ├────────────────────────────────────>│
    │                                     │  内核: 回收 mmap 缓冲区
```

> **为什么需要 BC_FREE_BUFFER？** BR_TRANSACTION 传来的数据在 mmap 共享内存区，内核不会自动回收。Server 读完数据后必须用 `BC_FREE_BUFFER` 主动告诉内核"这块内存我用完了"，否则会内存泄漏。

#### 5. 死亡通知类 — 监听对端进程死亡

| 命令 | 方向 | 携带数据 | 说明 |
|------|------|---------|------|
| BC_REQUEST_DEATH_NOTIFICATION | 用户→内核 | binder_handle_cookie | 请求监听对端死亡 |
| BC_CLEAR_DEATH_NOTIFICATION | 用户→内核 | binder_handle_cookie | 取消死亡监听 |
| BR_DEAD_BINDER | 内核→用户 | binder_uintptr_t | 对端已死通知 |
| BC_DEAD_BINDER_DONE | 用户→内核 | binder_uintptr_t | 确认死亡通知已处理 |
| BR_CLEAR_DEATH_NOTIFICATION_DONE | 内核→用户 | binder_uintptr_t | 取消监听完成 |

**例子：Client 监听 Server 进程死亡**

```
Client 进程                           Kernel Driver                        Server 进程
    │                                     │                                    │
    │  BC_ACQUIRE(handle=5)               │  (先增加强引用)                     │
    ├────────────────────────────────────>│                                    │
    │                                     │                                    │
    │  BC_REQUEST_DEATH_NOTIFICATION      │  内核: 记录 Client 要监听 handle=5  │
    │    (handle=5, cookie=0x1234)        │                                    │
    ├────────────────────────────────────>│                                    │
    │                                     │                                    │
    │  (正常通信中...)                      │                                    │
    │                                     │                                    │
    │                                     │  (Server 进程崩溃！)                 │
    │                                     │  ← 进程死亡                        │
    │                                     │                                    │
    │  BR_DEAD_BINDER(cookie=0x1234)      │  内核: 通知所有监听者               │
    │<────────────────────────────────────┤                                    │
    │                                     │                                    │
    │  (Client 收到死亡通知，清理资源)       │                                    │
    │                                     │                                    │
    │  BC_DEAD_BINDER_DONE(cookie=0x1234) │  (告诉内核：我处理完了)              │
    ├────────────────────────────────────>│                                    │
    │                                     │                                    │
    │  (如果之前取消监听)                    │                                    │
    │  BC_CLEAR_DEATH_NOTIFICATION         │                                    │
    │    (handle=5, cookie=0x1234)        │                                    │
    ├────────────────────────────────────>│                                    │
    │  BR_CLEAR_DEATH_NOTIFICATION_DONE   │                                    │
    │<────────────────────────────────────┤                                    │
```

> **死亡通知的典型场景**：Client 拿到 RemoteObject 后注册死亡监听。如果 Server 进程崩溃，Client 会收到 `BR_DEAD_BINDER`，可以清理引用、重连或报错。libgbinder 的 `GBinderRemoteObject` 内部自动处理这个机制，通过 `GBinderClient` 的 `object-destroyed` 信号通知上层。

#### 6. Scatter-Gather 类 — 大数据传输优化

| 命令 | 方向 | 携带数据 | 说明 |
|------|------|---------|------|
| BC_TRANSACTION_SG | 用户→内核 | binder_transaction_data_sg | 带 scatter-gather 的事务 |
| BC_REPLY_SG | 用户→内核 | binder_transaction_data_sg | 带 scatter-gather 的回复 |

`binder_transaction_data_sg` 在 `binder_transaction_data` 基础上多了 `sg_buffer_size` 和 `sg_buffer` 字段，允许事务数据分散在多个不连续的内存区域中，而不是必须在一个连续缓冲区里。现代 Android（API 29+）默认使用 SG 版本。

> libgbinder 在 `gbinder_driver.c` 中会根据内核能力选择使用 `BC_TRANSACTION` 还是 `BC_TRANSACTION_SG`。

## 1.7 内核数据结构

### flat_binder_object — Binder 对象的扁平表示

```c
// src/binder.h
struct flat_binder_object {
    struct binder_object_header hdr;  // type: BINDER_TYPE_BINDER/HANDLE/...
    __u32 flags;                      // 优先级等标志
    union {
        binder_uintptr_t binder;      // 本地对象指针（发送方）
        __u32 handle;                  // 远程对象句柄（接收方）
    };
    binder_uintptr_t cookie;           // 本地对象的 cookie
};
```

### 对象类型

```c
enum {
    BINDER_TYPE_BINDER       = B_PACK_CHARS('s', 'b', '*', B_TYPE_LARGE),  // 强引用本地对象
    BINDER_TYPE_WEAK_BINDER = B_PACK_CHARS('w', 'b', '*', B_TYPE_LARGE),  // 弱引用本地对象
    BINDER_TYPE_HANDLE      = B_PACK_CHARS('s', 'h', '*', B_TYPE_LARGE),  // 强引用远程句柄
    BINDER_TYPE_WEAK_HANDLE = B_PACK_CHARS('w', 'h', '*', B_TYPE_LARGE),  // 弱引用远程句柄
    BINDER_TYPE_FD          = B_PACK_CHARS('f', 'd', '*', B_TYPE_LARGE),  // 文件描述符
    BINDER_TYPE_FDA         = B_PACK_CHARS('f', 'd', 'a', B_TYPE_LARGE),  // FD 数组
    BINDER_TYPE_PTR         = B_PACK_CHARS('p', 't', '*', B_TYPE_LARGE),  // 缓冲区指针
};
```

### binder_write_read — 核心 ioctl 结构

```c
struct binder_write_read {
    binder_size_t write_size;        // 写缓冲区大小
    binder_size_t write_consumed;    // 已消费的写数据
    binder_uintptr_t write_buffer;   // 写缓冲区（BC 命令）
    binder_size_t read_size;         // 读缓冲区大小
    binder_size_t read_consumed;     // 已消费的读数据
    binder_uintptr_t read_buffer;    // 读缓冲区（BR 返回）
};
```

> `BINDER_WRITE_READ` 是最核心的 ioctl，用户空间通过它同时发送 BC 命令并接收 BR 返回。

### 32 位 vs 64 位

```c
#ifdef BINDER_IPC_32BIT
    typedef __u32 binder_size_t;
    typedef __u32 binder_uintptr_t;
    #define BINDER_CURRENT_PROTOCOL_VERSION 7
#else
    typedef __u64 binder_size_t;
    typedef __u64 binder_uintptr_t;
    #define BINDER_CURRENT_PROTOCOL_VERSION 8
#endif
```

> 这是 libgbinder 需要 `gbinder_io_32.c` 和 `gbinder_io_64.c` 两个实现的原因——32 位和 64 位内核的指针大小、结构体布局、ioctl 码都不同。

## 1.8 为什么需要 libgbinder

libgbinder 由 Jolla（Sailfish OS）开发，核心目标是**在非 Android Linux 系统上使用 Android Binder**。

### 关键设计目标（来自 README）

1. **GLib 事件循环集成** — 与 GLib MainLoop 无缝协作，而非 Android 的 libbinder
2. **运行时 32/64 位检测** — 自动适配 32 位或 64 位内核 Binder 协议
3. **异步事务** — 不阻塞事件线程
4. **稳定的 API** — ServiceManager 和底层事务 API 不变，通过可配置后端适配不同 Android 版本

### 配置驱动

```
# /etc/gbinder.conf
[Protocol]
Default = aidl
/dev/binder = aidl
/dev/hwbinder = hidl

[ServiceManager]
Default = aidl
/dev/binder = aidl
/dev/hwbinder = hidl

# 或直接指定 Android API Level
[General]
ApiLevel = 29
```

> Android 不断修改底层 RPC 和 ServiceManager 协议。libgbinder 通过配置文件选择对应的后端（aidl/aidl2~6/hidl），保持自身 API 不变。

## 1.9 小结

| 概念 | 要点 |
|------|------|
| Binder | Android 核心 IPC，基于内核驱动，一次拷贝 |
| ServiceManager | 名称→句柄映射，Binder 世界的 DNS |
| Transaction | 请求-回复模型，由 code 标识方法 |
| BC/BR 协议 | 用户空间发 BC 命令，内核返回 BR 结果 |
| AIDL/HIDL | 两种接口定义语言，对应不同 Binder 域 |
| 32/64 位 | 内核协议版本不同，需运行时适配 |
| libgbinder | 非 Android 系统的 Binder 封装，GLib 风格 |

---

**下一阶段**：[02-architecture-and-config.md](02-architecture-and-config.md) — 总体架构与配置系统
