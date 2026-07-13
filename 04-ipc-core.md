# 阶段 4：IPC 核心层

> 对应源码：`src/gbinder_ipc.c`、`src/gbinder_ipc.h`、`src/gbinder_eventloop.c`、`src/gbinder_eventloop_p.h`、`src/gbinder_handler.h`

## 4.1 GBinderIpc 概述

`GBinderIpc` 是 libgbinder 的**核心调度层**，位于 Driver 和高层 API 之间。它负责：

1. **事务调度** — 同步/异步事务的排队与执行
2. **Looper 线程管理** — 创建和管理 Binder looper 线程
3. **对象注册表** — 维护 Local/Remote 对象映射
4. **GLib 事件循环集成** — 将入站事务分发到主线程处理
5. **异步事务** — 通过线程池实现非阻塞事务

### GBinderIpc 结构

```c
// src/gbinder_ipc.h
struct gbinder_ipc {
    GObject object;              // 继承 GObject
    GBinderIpcPriv* priv;       // 私有数据
    GBinderDriver* driver;      // 底层驱动
    const char* dev;            // 设备路径
};

// src/gbinder_ipc.c
struct gbinder_ipc_priv {
    GBinderIpc* self;
    GThreadPool* tx_pool;                    // 事务线程池
    GHashTable* tx_table;                    // 事务表
    char* dev;
    char* key;                               // "protocol:dev" 格式
    const char* name;
    GBinderObjectRegistry object_registry;   // 对象注册表

    GMutex remote_objects_mutex;
    GHashTable* remote_objects;              // handle → RemoteObject

    GMutex local_objects_mutex;
    GHashTable* local_objects;               // ptr → LocalObject

    GMutex looper_mutex;
    GBinderIpcLooper* primary_loopers;       // 主 looper 链表
    GBinderIpcLooper* blocked_loopers;       // 被阻塞的 looper 链表
};
```

### IPC 全局表

```c
// src/gbinder_ipc.c
// 每个 "protocol:dev" 组合对应一个 GBinderIpc 实例
static GHashTable* gbinder_ipc_table = NULL;
static pthread_mutex_t gbinder_ipc_mutex = PTHREAD_MUTEX_INITIALIZER;

// 常量
#define GBINDER_IPC_MAX_TX_THREADS       (15)  // 最大事务线程数
#define GBINDER_IPC_MAX_PRIMARY_LOOPERS  (5)   // 最大主 looper 数
#define GBINDER_IPC_LOOPER_START_TIMEOUT_SEC  (2)
#define GBINDER_IPC_LOOPER_JOIN_TIMEOUT_MS    (500)
```

## 4.2 Looper 线程模型

### 为什么需要 Looper

Binder 驱动是**阻塞式**的——调用 `BINDER_WRITE_READ` ioctl 会阻塞直到有数据可读。因此不能在主线程（事件循环线程）上直接执行 Binder I/O。

libgbinder 的解决方案：
- **主线程**：运行 GLib MainLoop，处理事务回调
- **Looper 线程**：阻塞在 Binder 驱动上，等待入站事务
- **Worker 线程**：线程池，执行同步 Binder 事务

### GBinderIpcLooper 结构

```c
// src/gbinder_ipc.c
struct gbinder_ipc_looper {
    gint refcount;
    GBinderIpcLooper* next;       // 链表指针
    char* name;
    GBinderHandler handler;       // 事务处理器
    GBinderDriver* driver;        // 独立的 driver 实例
    GBinderIpc* ipc;              // 所属 IPC（非引用）
    pthread_t thread;             // 线程句柄
    GMutex mutex;
    GCond start_cond;             // 启动条件变量
    gint exit;                     // 退出标志
    gint started;                  // 已启动标志
    gint joined;                   // 已 join 标志
    int pipefd[2];                 // 唤醒 pipe
    int txfd[2];                   // 事务 pipe
};
```

### Looper 的工作流程

```
Looper 线程启动
    │
    ├── 1. gbinder_driver_enter_looper()     → BC_ENTER_LOOPER
    │
    ├── 2. 主循环:
    │   ┌──→ gbinder_driver_poll(driver, pipefd)  // 等待数据
    │   │       │
    │   │       ├── binder fd 可读 → BR_TRANSACTION
    │   │       │   ├── 创建 GBinderIpcLooperTx
    │   │       │   ├── 投递到主线程 (g_idle_add)
    │   │       │   └── 等待 pipe 唤醒 (主线程处理完成)
    │   │       │       ├── TX_DONE → 发送 BC_REPLY，继续循环
    │   │       │       └── TX_BLOCKED → spawn 新 looper，继续等待
    │   │       │
    │   │       └── pipe fd 可读 → 退出信号
    │   │           └── break
    │   │
    │   └── gbinder_driver_read()  // 处理入站命令
    │
    └── 3. gbinder_driver_exit_looper()     → BC_EXIT_LOOPER
```

### Looper 与主线程的协作

```
Looper 线程                         主线程 (GLib MainLoop)
    │                                     │
    │ BR_TRANSACTION(data)                │
    ├─ 创建 GBinderIpcLooperTx            │
    │  (包含 obj, req, code, flags)       │
    │                                     │
    │  g_idle_add(looper_tx_handle, tx)   │
    ├────────────────────────────────────>│
    │                                     │
    │  poll(pipefd)  // 阻塞等待          │  looper_tx_handle(tx):
    │                                     │  ├── tx->state = PROCESSING
    │                                     │  ├── reply = local_object_handle_transaction()
    │                                     │  ├── tx->state = COMPLETE
    │                                     │  └── write(pipe, TX_DONE)
    │<────────────────────────────────────┤
    │  read(pipe) = TX_DONE               │
    │                                     │
    │  gbinder_driver_reply_data(reply)   │
    │  → BC_REPLY                         │
    │                                     │
    └── 继续循环                           │
```

## 4.3 事务状态机

入站事务在主线程中处理时，经历以下状态：

```
SCHEDULED
    │
    │ (主线程开始处理)
    v
PROCESSING
    │
    ├── handler 直接返回 reply ──────────────> COMPLETE
    │                                         (正常完成)
    │
    ├── gbinder_remote_request_block() ──> BLOCKING
    │                                         │
    │                                    handler 返回
    │                                         │
    │                                         v
    │                                      BLOCKED
    │                                         │
    │                                    gbinder_remote_request_complete()
    │                                         │
    │                                         v
    │                                      COMPLETE
    │
    └── gbinder_remote_request_complete() ──> PROCESSED
        (在 handler 中直接调用)                   │
                                                 v
                                             COMPLETE
```

### 状态说明

| 状态 | 说明 |
|------|------|
| `SCHEDULED` | 事务已投递到主线程，等待处理 |
| `PROCESSING` | 主线程正在调用 handler |
| `BLOCKING` | handler 调用了 `gbinder_remote_request_block()`，表示需要异步完成 |
| `PROCESSED` | handler 在阻塞模式下调用了 `complete()` |
| `BLOCKED` | handler 返回但事务未完成，等待后续 `complete()` |
| `COMPLETE` | 事务完成，可以发送 BC_REPLY |

### 异步事务处理（block 模式）

```c
// 服务端 handler 中：
static GBinderLocalReply*
app_reply(GBinderLocalObject* obj, GBinderRemoteRequest* req,
    guint code, guint flags, int* status, void* user_data)
{
    if (opt->async) {
        // 标记为异步：不立即回复
        gbinder_remote_request_block(req);

        // 稍后在其他地方调用 complete
        g_idle_add_full(G_PRIORITY_DEFAULT_IDLE, app_async_resp,
            resp, app_async_free);

        return NULL;  // 不返回 reply
    } else {
        // 同步：直接返回 reply
        return reply;
    }
}

// 异步完成
static gboolean app_async_resp(gpointer user_data)
{
    Response* resp = user_data;
    gbinder_remote_request_complete(resp->req, resp->reply, 0);
    return G_SOURCE_REMOVE;
}
```

## 4.4 同步事务 API

libgbinder 提供两套同步事务 API：

### sync_main — 在主线程执行

```c
// src/gbinder_ipc.c
// 在调用线程（通常是主线程）直接执行 ioctl
// 会阻塞调用线程直到收到回复
const GBinderIpcSyncApi gbinder_ipc_sync_main = {
    .sync_reply = gbinder_ipc_transact_sync_reply_main,
    .sync_oneway = gbinder_ipc_transact_sync_oneway_main
};
```

### sync_worker — 在 worker 线程执行

```c
// src/gbinder_ipc.c
// 将事务提交到线程池，不阻塞主线程
// 通过 GLib idle 回调将结果投递回主线程
const GBinderIpcSyncApi gbinder_ipc_sync_worker = {
    .sync_reply = gbinder_ipc_transact_sync_reply_worker,
    .sync_oneway = gbinder_ipc_transact_sync_oneway_worker
};
```

### 同步事务流程对比

```
sync_main (阻塞主线程):
Main Thread
    │
    ├── gbinder_driver_transact(handle, code, req, &reply)
    │   └── 阻塞在 ioctl(BINDER_WRITE_READ) 直到 BR_REPLY
    ├── 处理 reply
    └── 返回

sync_worker (不阻塞主线程):
Main Thread              Worker Thread
    │                         │
    ├── 提交到线程池 ──────────>│
    │                         ├── gbinder_driver_transact()
    │                         │   └── 阻塞等待 BR_REPLY
    │   (处理其他事件)          │
    │                         ├── 完成
    │<────────────────────────┤  g_idle_add(回调)
    ├── 回调: 处理 reply       │
    └── 返回                    │
```

## 4.5 异步事务

```c
// src/gbinder_ipc.h
gulong gbinder_ipc_transact(
    GBinderIpc* ipc,
    guint32 handle,
    guint32 code,
    guint32 flags,              // GBINDER_TX_FLAG_ONEWAY 等
    GBinderLocalRequest* req,
    GBinderIpcReplyFunc func,   // 回复回调
    GDestroyNotify destroy,
    void* user_data);
```

### 异步事务流程

```
调用方                 GBinderIpc              Worker Thread
  │                        │                        │
  │ gbinder_ipc_transact() │                        │
  ├───────────────────────>│                        │
  │                        ├── 创建 GBinderIpcTxInternal │
  │                        ├── 提交到线程池 ──────────>│
  │                        │                        │
  │  返回 tx_id            │                        │
  │<───────────────────────┤                        │
  │                        │                        │
  │  (继续执行)             │                        ├── gbinder_driver_transact()
  │                        │                        │   └── 阻塞等待回复
  │                        │                        │
  │                        │  回复到达               │
  │                        │<────────────────────────┤
  │                        │                        │
  │                        ├── g_idle_add(回调)      │
  │  func(reply, status)   │                        │
  │<───────────────────────┤                        │
```

## 4.6 GBinderHandler — 事务处理接口

```c
// src/gbinder_handler.h
typedef struct gbinder_handler_functions {
    gboolean (*can_loop)(GBinderHandler* handler);
    GBinderLocalReply* (*transact)(GBinderHandler* handler,
        GBinderLocalObject* obj, GBinderRemoteRequest* req,
        guint code, guint flags, int* status);
} GBinderHandlerFunctions;

struct gbinder_handler {
    const GBinderHandlerFunctions* f;
};
```

### Handler 的作用

- `can_loop()` — 询问 handler 是否允许 looper 继续循环
- `transact()` — 处理入站事务，返回 reply

> 当 handler 为 NULL 时，表示同步调用在主线程上执行，直接调用 `gbinder_local_object_handle_transaction()`。

## 4.7 事件循环集成 — gbinder_eventloop.c

### GBinderEventLoopIntegration 接口

```c
// src/gbinder_eventloop_p.h
typedef struct gbinder_eventloop_integration {
    GBinderEventLoopTimeout* (*timeout_add)(guint interval,
        GSourceFunc func, gpointer data);
    void (*timeout_remove)(GBinderEventLoopTimeout* timeout);
    GBinderEventLoopCallback* (*callback_new)(GBinderEventLoopCallbackFunc func,
        gpointer data, GDestroyNotify finalize);
    void (*callback_ref)(GBinderEventLoopCallback* cb);
    void (*callback_unref)(GBinderEventLoopCallback* cb);
    void (*callback_schedule)(GBinderEventLoopCallback* cb);
    void (*callback_cancel)(GBinderEventLoopCallback* cb);
    void (*cleanup)(void);
} GBinderEventLoopIntegration;
```

### GLib 实现

```c
// src/gbinder_eventloop.c
static const GBinderEventLoopIntegration gbinder_eventloop_glib = {
    .timeout_add    = gbinder_eventloop_glib_timeout_add,     // g_timeout_add_full
    .timeout_remove = gbinder_eventloop_glib_timeout_remove,  // g_source_remove
    .callback_new   = gbinder_eventloop_glib_callback_new,    // g_source_new
    .callback_ref   = gbinder_eventloop_glib_callback_ref,    // g_source_ref
    .callback_unref = gbinder_eventloop_glib_callback_unref,  // g_source_unref
    .callback_schedule = gbinder_eventloop_glib_callback_schedule, // g_source_attach
    .callback_cancel = gbinder_eventloop_glib_callback_cancel,    // g_source_destroy
    .cleanup         = gbinder_eventloop_glib_cleanup
};
```

> **可替换性**：通过 `gbinder_eventloop_set()` 可以替换事件循环后端。默认使用 GLib，但理论上可以适配其他事件循环框架。

### Idle 回调机制

```c
// src/gbinder_eventloop.c
// 创建并调度一个 idle 回调
GBinderEventLoopCallback*
gbinder_idle_callback_schedule_new(
    GBinderEventLoopCallbackFunc func,
    gpointer data,
    GDestroyNotify finalize)
{
    GBinderEventLoopCallback* cb =
        gbinder_eventloop->callback_new(func, data, finalize);
    gbinder_idle_callback_schedule(cb);  // 附加到默认 MainContext
    return cb;
}
```

> 这个机制被配置系统（延迟释放 KeyFile）和异步事务（结果投递到主线程）使用。

## 4.8 IPC 实例管理

### 创建与缓存

```c
// src/gbinder_ipc.c
// 每个 "protocol:dev" 组合创建一个 GBinderIpc 实例
// 实例被缓存在全局哈希表中，重复请求返回同一实例

GBinderIpc* gbinder_ipc_new(const char* dev, const char* protocol)
{
    char* key = gbinder_ipc_make_key(protocol, dev);  // "protocol:dev"
    // 查找或创建
    // 如果创建：
    //   ├── gbinder_driver_new(dev, protocol)
    //   ├── 初始化对象注册表
    //   ├── 创建线程池
    //   └── 启动 looper 线程
}
```

### 对象注册表

```c
// src/gbinder_ipc.h
GBinderObjectRegistry* gbinder_ipc_object_registry(GBinderIpc* ipc);

// 对象注册表维护：
// - local_objects: ptr → GBinderLocalObject（本地对象）
// - remote_objects: handle → GBinderRemoteObject（远程对象代理）
```

## 4.9 Looper 生命周期管理

### Looper 创建时机

1. **IPC 初始化时**：创建第一个 primary looper
2. **BR_SPAWN_LOOPER**：内核请求创建新 looper 线程
3. **事务阻塞时**：当 looper 被异步事务阻塞（TX_BLOCKED），spawn 新 looper

### Looper 限制

```c
#define GBINDER_IPC_MAX_PRIMARY_LOOPERS (5)  // 最多 5 个 primary looper
```

### Looper 退出

```
IPC 销毁
    │
    ├── 设置 exit 标志
    ├── 写入 pipe 唤醒所有 looper
    ├── 等待 looper 线程退出 (join, 超时 500ms)
    └── 释放 looper 资源
```

## 4.10 完整架构图

```
┌──────────────────────────────────────────────────────────────┐
│                      Public API                                │
│  gbinder_servicemanager_*()  gbinder_client_*()              │
├──────────────────────────────────────────────────────────────┤
│                      GBinderIpc                                │
│                                                                │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐   │
│  │ 事务调度     │  │ 对象注册表    │  │ Looper 管理       │   │
│  │ sync_main   │  │ local_objects │  │ primary_loopers   │   │
│  │ sync_worker │  │ remote_objects│  │ blocked_loopers   │   │
│  │ async       │  │              │  │                   │   │
│  └──────┬──────┘  └──────────────┘  └────────┬──────────┘   │
│         │                                      │              │
│  ┌──────┴──────────────────────────────────────┴──────────┐  │
│  │              GThreadPool (事务线程池)                      │  │
│  │              (最多 15 个 worker 线程)                     │  │
│  └─────────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│                      GBinderDriver                             │
│  (每个 looper 有独立的 driver 实例)                              │
├──────────────────────────────────────────────────────────────┤
│                      GLib MainLoop                             │
│  (主线程处理入站事务回调)                                        │
└──────────────────────────────────────────────────────────────┘
```

## 4.11 小结

| 概念 | 要点 |
|------|------|
| GBinderIpc | 核心调度层，GObject 子类 |
| Looper 线程 | 阻塞在 Binder 驱动上等待入站事务 |
| 主线程协作 | Looper 收到事务后投递到主线程处理，通过 pipe 同步 |
| 事务状态机 | SCHEDULED → PROCESSING → COMPLETE（或 BLOCKING → BLOCKED → COMPLETE） |
| sync_main | 在调用线程直接执行，阻塞 |
| sync_worker | 提交到线程池，不阻塞主线程 |
| 异步事务 | 通过线程池 + idle 回调实现 |
| block 模式 | handler 可标记异步完成，looper spawn 新线程继续等待 |
| 事件循环 | 可替换的 GBinderEventLoopIntegration，默认 GLib |
| IPC 缓存 | 每个 "protocol:dev" 组合一个实例 |
| Looper 限制 | 最多 5 个 primary looper，15 个 worker 线程 |

---

**上一阶段**：[03-driver-and-io-layer.md](03-driver-and-io-layer.md)
**下一阶段**：[05-object-model.md](05-object-model.md) — 对象模型
