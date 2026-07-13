# 阶段 5：对象模型

> 对应源码：`src/gbinder_local_object.c`、`src/gbinder_local_object_p.h`、`src/gbinder_remote_object.c`、`src/gbinder_remote_object_p.h`、`src/gbinder_object_registry.h`、`src/gbinder_proxy_object.c`、`src/gbinder_object_converter.h`、`include/gbinder_local_object.h`、`include/gbinder_remote_object.h`

## 5.1 对象模型概述

libgbinder 的对象模型围绕两个核心概念展开：

- **LocalObject** — 本地对象，存在于服务端，接收并处理入站事务
- **RemoteObject** — 远程对象，存在于客户端，作为发起事务的代理

### 术语图解（来自 gbinder_types.h）

```
客户端（发起事务）:
LocalObject --- (LocalRequest) --> Client(RemoteObject)
LocalObject <-- (RemoteReply) -- RemoteObject

服务端（接收事务）:
LocalObject <-- (RemoteRequest) --- (pid, euid)
LocalObject --- (LocalReply)    --> (pid, euid)

数据层:
Writer → 准备 LocalRequest 和 LocalReply 的数据
Reader → 解析 RemoteRequest 和 RemoteReply 的数据
```

## 5.2 GBinderLocalObject — 本地对象

### 结构

```c
// src/gbinder_local_object_p.h
struct gbinder_local_object {
    GObject object;                    // 继承 GObject
    GBinderLocalObjectPriv* priv;      // 私有数据
    GBinderIpc* ipc;                   // 所属 IPC
    const char* const* ifaces;         // 支持的接口名列表
    gint weak_refs;                    // 弱引用计数
    gint strong_refs;                  // 强引用计数
    GBINDER_STABILITY_LEVEL stability; // 稳定性级别
};
```

### 事务处理支持级别

```c
// src/gbinder_local_object_p.h
typedef enum gbinder_local_transaction_support {
    GBINDER_LOCAL_TRANSACTION_NOT_SUPPORTED,  // 不支持
    GBINDER_LOCAL_TRANSACTION_SUPPORTED,       // 支持（在主线程处理）
    GBINDER_LOCAL_TRANSACTION_LOOPER           // 支持（在 looper 线程处理）
} GBINDER_LOCAL_TRANSACTION_SUPPORT;
```

### 虚函数表（Class）

```c
// src/gbinder_local_object_p.h
typedef struct gbinder_local_object_class {
    GObjectClass parent;

    // 判断是否能处理该事务
    GBINDER_LOCAL_TRANSACTION_SUPPORT (*can_handle_transaction)
        (GBinderLocalObject* self, const char* iface, guint code);

    // 在主线程处理事务
    GBinderLocalReply* (*handle_transaction)
        (GBinderLocalObject* self, GBinderRemoteRequest* req,
         guint code, guint flags, int* status);

    // 在 looper 线程处理事务
    GBinderLocalReply* (*handle_looper_transaction)
        (GBinderLocalObject* self, GBinderRemoteRequest* req,
         guint code, guint flags, int* status);

    void (*acquire)(GBinderLocalObject* self);
    void (*release)(GBinderLocalObject* self);
    void (*drop)(GBinderLocalObject* self);
} GBinderLocalObjectClass;
```

### 公共 API

```c
// include/gbinder_local_object.h
// 创建本地对象
GBinderLocalObject* gbinder_local_object_new(
    GBinderIpc* ipc,
    const char* const* ifaces,        // 支持的接口名列表
    GBinderLocalTransactFunc handler,  // 事务处理回调
    void* user_data);

// 引用管理
GBinderLocalObject* gbinder_local_object_ref(GBinderLocalObject* obj);
void gbinder_local_object_unref(GBinderLocalObject* obj);
void gbinder_local_object_drop(GBinderLocalObject* obj);

// 创建回复
GBinderLocalReply* gbinder_local_object_new_reply(GBinderLocalObject* obj);

// 设置稳定性级别
void gbinder_local_object_set_stability(
    GBinderLocalObject* self, GBINDER_STABILITY_LEVEL stability);
```

### 事务处理回调

```c
// include/gbinder_types.h
typedef GBinderLocalReply*
(*GBinderLocalTransactFunc)(
    GBinderLocalObject* obj,
    GBinderRemoteRequest* req,
    guint code,
    guint flags,
    int* status,
    void* user_data);
```

> 这是用户注册的事务处理函数。当收到入站事务时，libgbinder 调用此函数。返回值是 LocalReply（NULL 表示无回复或异步完成）。

### 引用计数管理

LocalObject 的引用计数由内核驱动通过 BR_INCREFS/BR_ACQUIRE/BR_RELEASE/BR_DECREFS 命令管理：

```
内核请求                    libgbinder 处理
─────────────────────────────────────────────
BR_INCREFS  (ptr, cookie) → gbinder_local_object_handle_increfs()
                            weak_refs++
                            回复 BC_INCREFS_DONE

BR_ACQUIRE  (ptr, cookie) → gbinder_local_object_handle_acquire()
                            strong_refs++
                            回复 BC_ACQUIRE_DONE

BR_RELEASE  (ptr, cookie) → gbinder_local_object_handle_release()
                            strong_refs--
                            (延迟处理，清空命令队列后)

BR_DECREFS  (ptr, cookie) → gbinder_local_object_handle_decrefs()
                            weak_refs--
                            (延迟处理)
```

> **延迟处理**：RELEASE 和 DECREFS 被延迟到命令队列清空后处理，避免在处理引用减少时触发对象销毁导致问题。

## 5.3 GBinderRemoteObject — 远程对象

### 结构

```c
// src/gbinder_remote_object_p.h
struct gbinder_remote_object {
    GObject object;              // 继承 GObject
    GBinderRemoteObjectPriv* priv;
    GBinderIpc* ipc;             // 所属 IPC
    guint32 handle;              // 内核句柄
    gboolean dead;               // 是否已死亡
};
```

### 公共 API

```c
// include/gbinder_remote_object.h
// 引用管理
GBinderRemoteObject* gbinder_remote_object_ref(GBinderRemoteObject* obj);
void gbinder_remote_object_unref(GBinderRemoteObject* obj);

// 获取所属 IPC
GBinderIpc* gbinder_remote_object_ipc(GBinderRemoteObject* obj);

// 检查是否已死亡
gboolean gbinder_remote_object_is_dead(GBinderRemoteObject* obj);

// 死亡通知
gulong gbinder_remote_object_add_death_handler(
    GBinderRemoteObject* obj,
    GBinderRemoteObjectNotifyFunc func,  // 死亡回调
    void* user_data);
void gbinder_remote_object_remove_handler(GBinderRemoteObject* obj, gulong id);
```

### 创建选项

```c
// src/gbinder_remote_object_p.h
typedef enum gbinder_remote_object_create {
    REMOTE_OBJECT_CREATE_DEAD,      // 创建为已死亡状态
    REMOTE_OBJECT_CREATE_ALIVE,     // 创建为活跃状态
    REMOTE_OBJECT_CREATE_ACQUIRED   // 创建并获取引用
} REMOTE_OBJECT_CREATE;
```

### 死亡通知机制

```
内核                          GBinderRemoteObject          用户回调
  │                                │                          │
  │ BR_DEAD_BINDER(handle)         │                          │
  ├───────────────────────────────>│                          │
  │                                │ handle_death_notification()│
  │                                │ dead = TRUE               │
  │                                │ 遍历 death_handlers        │
  │                                ├─────────────────────────>│
  │                                │                          │ func(obj, user_data)
  │                                │                          │
  │  BC_DEAD_BINDER_DONE(handle)   │                          │
  │<───────────────────────────────┤                          │
```

### 死亡通知的请求与清除

```c
// 请求死亡通知（内部）
gbinder_driver_request_death_notification(driver, obj)
    → BC_REQUEST_DEATH_NOTIFICATION(handle, cookie)

// 清除死亡通知（内部）
gbinder_driver_clear_death_notification(driver, obj)
    → BC_CLEAR_DEATH_NOTIFICATION(handle, cookie)

// 确认死亡通知处理完成（内部）
gbinder_driver_dead_binder_done(driver, obj)
    → BC_DEAD_BINDER_DONE(handle)
```

### 远程对象的生命周期

```
创建:
  gbinder_remote_object_new(ipc, handle, CREATE_ALIVE)
    ├── BC_INCREFS(handle)     // 增加弱引用
    ├── BC_ACQUIRE(handle)     // 增加强引用
    ├── BC_REQUEST_DEATH_NOTIFICATION(handle, cookie)
    └── 注册到 remote_objects 表

销毁:
  gbinder_remote_object_unref() → refcount == 0
    ├── BC_RELEASE(handle)     // 减少强引用
    ├── BC_DECREFS(handle)     // 减少弱引用
    ├── BC_CLEAR_DEATH_NOTIFICATION(handle, cookie)
    └── 从 remote_objects 表移除

死亡:
  BR_DEAD_BINDER(handle)
    ├── dead = TRUE
    ├── 通知所有 death_handler
    └── BC_DEAD_BINDER_DONE(handle)
```

## 5.4 GBinderObjectRegistry — 对象注册表

### 结构

```c
// src/gbinder_object_registry.h
struct gbinder_object_registry {
    const GBinderObjectRegistryFunctions* f;  // 虚函数表
    const GBinderIo* io;                       // IO 实现
};

typedef struct gbinder_object_registry_functions {
    void (*ref)(GBinderObjectRegistry* reg);
    void (*unref)(GBinderObjectRegistry* reg);
    GBinderLocalObject* (*get_local)(GBinderObjectRegistry* reg, void* pointer);
    GBinderRemoteObject* (*get_remote)(GBinderObjectRegistry* reg,
        guint32 handle, REMOTE_REGISTRY_CREATE create);
} GBinderObjectRegistryFunctions;
```

### 远程对象创建选项

```c
typedef enum gbinder_remote_registry_create {
    REMOTE_REGISTRY_DONT_CREATE,           // 不创建，仅查找
    REMOTE_REGISTRY_CAN_CREATE,            // 可以创建
    REMOTE_REGISTRY_CAN_CREATE_AND_ACQUIRE // 创建并获取引用
} REMOTE_REGISTRY_CREATE;
```

### 注册表的作用

ObjectRegistry 是 LocalObject 和 RemoteObject 的**中央查找表**：

```
入站事务 (BR_TRANSACTION):
  target (ptr/cookie) → ObjectRegistry.get_local(ptr) → LocalObject

出站事务 (BC_TRANSACTION):
  handle → ObjectRegistry.get_remote(handle) → RemoteObject

死亡通知 (BR_DEAD_BINDER):
  handle → ObjectRegistry.get_remote(handle, DONT_CREATE) → RemoteObject
```

### 注册表与 IPC 的关系

```c
// src/gbinder_ipc.h
GBinderObjectRegistry* gbinder_ipc_object_registry(GBinderIpc* ipc);

// IPC 的 priv 中维护两个哈希表:
// - local_objects:  ptr → GBinderLocalObject
// - remote_objects: handle → GBinderRemoteObject
```

## 5.5 GBinderProxyObject — 代理对象

### 概述

`GBinderProxyObject` 是 `GBinderLocalObject` 的子类，用于**桥接**不同的 Binder 域。它将收到的入站事务转发到另一个 IPC 实例上的远程对象。

```
Binder 域 A                        Binder 域 B
/dev/binder                        /dev/hwbinder

Client ──> ProxyObject ──> RemoteObject ──> Server
           (LocalObject     (在域 B上)
            在域 A上)
```

### 使用场景

- 将 `/dev/binder` 上的请求转发到 `/dev/hwbinder`
- 在不同 Binder 设备间桥接服务

### 公共 API

```c
// include/gbinder_bridge.h
// ProxyObject 通过 GBinderBridge 创建
GBinderBridge* gbinder_bridge_new(
    GBinderLocalObject* local,    // 域 A 上的本地对象
    GBinderRemoteObject* remote);  // 域 B 上的远程对象
```

## 5.6 GBinderObjectConverter — 对象转换

```c
// src/gbinder_object_converter.h
// 处理 flat_binder_object 在不同上下文间的转换
// 当事务数据中包含 binder 对象时，需要将其在 local↔remote 间转换
```

### 转换场景

当事务数据从一端传到另一端时，binder 对象需要转换：

```
发送方 (LocalObject):
  flat_binder_object {
      type: BINDER_TYPE_BINDER
      binder: 本地指针
      cookie: 本地 cookie
  }

内核转换:
  type: BINDER_TYPE_BINDER → BINDER_TYPE_HANDLE
  binder → handle (分配的句柄)

接收方 (RemoteObject):
  flat_binder_object {
      type: BINDER_TYPE_HANDLE
      handle: 分配的句柄
  }
```

> 内核自动完成这个转换。libgbinder 的 ObjectConverter 负责在用户空间处理这些对象的编解码。

## 5.7 稳定性级别

```c
// include/gbinder_types.h
typedef enum gbinder_stability_level {
    GBINDER_STABILITY_UNDECLARED = 0,   // 未声明（默认）
    GBINDER_STABILITY_VENDOR    = 0x03,  // 厂商级
    GBINDER_STABILITY_SYSTEM    = 0x0c,  // 系统级
    GBINDER_STABILITY_VINTF     = 0x3f   // VINTF（Vendor Interface）级
} GBINDER_STABILITY_LEVEL;
```

### 稳定性级别的含义

| 级别 | 值 | 含义 |
|------|-----|------|
| UNDECLARED | 0 | 默认，无稳定性承诺 |
| VENDOR | 0x03 | 厂商接口，跨版本兼容 |
| SYSTEM | 0x0c | 系统接口，系统内兼容 |
| VINTF | 0x3f | 最稳定，Vendor Interface 级别 |

### 与协议的关系

- **aidl / aidl2**：不写入 stability（`flat_binder_object_extra = 0`）
- **aidl3**：直接写入 `guint32 stability`
- **aidl4**：写入 `struct stability_category { version, reserved, level }`

```c
// 设置稳定性级别
gbinder_local_object_set_stability(obj, GBINDER_STABILITY_VINTF);
```

## 5.8 对象关系图

```
┌─────────────────────────────────────────────────────────────┐
│                    GBinderIpc                                 │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              ObjectRegistry                               │ │
│  │                                                           │ │
│  │  ┌─────────────────┐     ┌─────────────────────────┐    │ │
│  │  │  local_objects   │     │  remote_objects          │    │ │
│  │  │  (ptr → Local)   │     │  (handle → Remote)       │    │ │
│  │  └────────┬────────┘     └───────────┬─────────────┘    │ │
│  └───────────┼──────────────────────────┼──────────────────┘ │
│              │                          │                     │
│              v                          v                     │
│  ┌──────────────────┐     ┌──────────────────────┐           │
│  │  GBinderLocalObject│     │  GBinderRemoteObject  │           │
│  │  (服务端)          │     │  (客户端代理)          │           │
│  │                   │     │                      │           │
│  │  • ifaces[]       │     │  • handle            │           │
│  │  • weak_refs      │     │  • dead              │           │
│  │  • strong_refs    │     │  • death_handlers[]   │           │
│  │  • stability      │     │                      │           │
│  │  • transact_func  │     │  • add_death_handler │           │
│  │                   │     │  • is_dead           │           │
│  │  继承自 GObject   │     │  继承自 GObject       │           │
│  └────────┬─────────┘     └──────────┬──────────┘           │
│           │                          │                       │
│           │ 子类                      │ 用于创建               │
│           v                          v                       │
│  ┌──────────────────┐     ┌──────────────────────┐           │
│  │  GBinderProxyObject│     │  GBinderClient        │           │
│  │  (桥接代理)        │     │  (事务客户端)          │           │
│  └──────────────────┘     └──────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

## 5.9 事务处理完整流程

### 服务端接收事务

```
1. Looper 收到 BR_TRANSACTION
   ↓
2. Driver 解码事务数据
   io->decode_transaction_data(data, &tx)
   ↓
3. ObjectRegistry 查找 LocalObject
   obj = gbinder_object_registry_get_local(reg, tx.target)
   ↓
4. LocalObject 判断是否支持
   support = gbinder_local_object_can_handle_transaction(obj, iface, code)
   ↓
5a. SUPPORTED → 主线程处理
    reply = gbinder_local_object_handle_transaction(obj, req, code, flags, &status)
    → 调用用户的 GBinderLocalTransactFunc 回调
    
5b. LOOPER → looper 线程处理
    reply = gbinder_local_object_handle_looper_transaction(obj, req, ...)
    
5c. NOT_SUPPORTED → 返回错误
   ↓
6. 回复（非 oneway）
   if (reply) → BC_REPLY(reply_data)
   else       → BC_REPLY(status)
```

### 客户端发起事务

```
1. 获取 RemoteObject
   obj = gbinder_servicemanager_get_service_sync(sm, name, &status)
   ↓
2. 创建 Client
   client = gbinder_client_new(obj, iface)
   ↓
3. 构建请求
   req = gbinder_client_new_request(client)
   gbinder_local_request_append_string16(req, "hello")
   ↓
4. 发起事务
   reply = gbinder_client_transact_sync_reply(client, code, req, &status)
   ↓
5. IPC 层调度
   gbinder_ipc_transact(ipc, obj->handle, code, 0, req, ...)
   ↓
6. Driver 执行
   gbinder_driver_transact(driver, reg, handler, handle, code, req, reply)
   → BC_TRANSACTION(handle, code, data)
   → 等待 BR_REPLY
   ↓
7. 解析回复
   gbinder_reader_read_string16(&reader)
```

## 5.10 小结

| 概念 | 要点 |
|------|------|
| LocalObject | 服务端对象，接收入站事务，GObject 子类 |
| RemoteObject | 客户端代理，发起事务，包含 handle |
| ObjectRegistry | 中央查找表，ptr→Local, handle→Remote |
| 事务处理回调 | GBinderLocalTransactFunc，用户注册的处理函数 |
| 引用计数 | 内核通过 BR_INCREFS/ACQUIRE/RELEASE/DECREFS 管理 |
| 死亡通知 | RemoteObject 可注册死亡回调，内核通过 BR_DEAD_BINDER 通知 |
| ProxyObject | LocalObject 子类，桥接不同 Binder 域 |
| 稳定性级别 | UNDECLARED/VENDOR/SYSTEM/VINTF，aidl3+ 写入 flat_binder |
| 事务支持级别 | NOT_SUPPORTED / SUPPORTED(主线程) / LOOPER(looper线程) |
| 对象转换 | 内核自动将 BINDER_TYPE_BINDER ↔ BINDER_TYPE_HANDLE 转换 |

---

**上一阶段**：[04-ipc-core.md](04-ipc-core.md)
**下一阶段**：[06-data-serialization.md](06-data-serialization.md) — 数据序列化
