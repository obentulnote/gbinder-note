# 阶段 7：ServiceManager 与 Client — 完整使用流程

> 对应源码：`src/gbinder_servicemanager.c`、`src/gbinder_servicemanager_aidl.c`~`gbinder_servicemanager_aidl6.c`、`src/gbinder_servicemanager_hidl.c`、`src/gbinder_client.c`、`src/gbinder_servicename.c`、`src/gbinder_servicepoll.c`、`src/gbinder_bridge.c`、`src/gbinder_system.c`、`include/gbinder_servicemanager.h`、`include/gbinder_client.h`

## 7.1 ServiceManager 概述

ServiceManager（SM）是 Binder 世界的"DNS"——它将服务名映射到内核句柄。libgbinder 为不同 Android 版本的 SM 协议提供了多个后端实现。

### SM 后端变体

| 后端 | Android 版本 | 说明 |
|------|-------------|------|
| `aidl` | 早期 | 原始 AIDL SM |
| `aidl2` | API 28 (Android 9) | SM 协议变更 |
| `aidl3` | API 30 (Android 11) | SM 协议变更 |
| `aidl4` | API 31 (Android 12) | SM 协议变更 |
| `aidl5` | API 35 (Android 15) | SM 协议变更 |
| `aidl6` | API 36 (Android 16) | SM 协议变更 |
| `hidl` | - | HIDL SM（/dev/hwbinder） |

### SM 的句柄

```c
// src/gbinder_types_p.h
#define GBINDER_SERVICEMANAGER_HANDLE (0)  // SM 固定使用 handle=0
```

> ServiceManager 本身是一个 Binder 服务，其句柄固定为 0。所有对 SM 的操作都是对 handle=0 的事务。

## 7.2 GBinderServiceManager — 公共 API

### 创建 ServiceManager

```c
// include/gbinder_servicemanager.h
// 自动选择协议（从配置文件）
GBinderServiceManager* gbinder_servicemanager_new(const char* dev);

// 显式指定协议
GBinderServiceManager* gbinder_servicemanager_new2(
    const char* dev,
    const char* sm_protocol,   // "aidl"/"aidl2"/.../"hidl"
    const char* rpc_protocol); // "aidl"/"aidl2"/.../"hidl"
```

### 创建 LocalObject（服务端）

```c
// 单接口
GBinderLocalObject* gbinder_servicemanager_new_local_object(
    GBinderServiceManager* sm,
    const char* iface,              // 接口名（如 "test@1.0"）
    GBinderLocalTransactFunc handler, // 事务处理回调
    void* user_data);

// 多接口
GBinderLocalObject* gbinder_servicemanager_new_local_object2(
    GBinderServiceManager* sm,
    const char* const* ifaces,      // 接口名数组
    GBinderLocalTransactFunc handler,
    void* user_data);
```

### 服务管理

```c
// 异步列表
gulong gbinder_servicemanager_list(
    GBinderServiceManager* sm,
    GBinderServiceManagerListFunc func, void* user_data);

// 同步列表
char** gbinder_servicemanager_list_sync(GBinderServiceManager* sm);

// 异步获取服务
gulong gbinder_servicemanager_get_service(
    GBinderServiceManager* sm,
    const char* name,
    GBinderServiceManagerGetServiceFunc func, void* user_data);

// 同步获取服务
GBinderRemoteObject* gbinder_servicemanager_get_service_sync(
    GBinderServiceManager* sm,
    const char* name,
    int* status);

// 异步注册服务
gulong gbinder_servicemanager_add_service(
    GBinderServiceManager* sm,
    const char* name,
    GBinderLocalObject* obj,
    GBinderServiceManagerAddServiceFunc func, void* user_data);

// 同步注册服务
int gbinder_servicemanager_add_service_sync(
    GBinderServiceManager* sm,
    const char* name,
    GBinderLocalObject* obj);
```

### 状态与回调

```c
// 检查 SM 是否存在
gboolean gbinder_servicemanager_is_present(GBinderServiceManager* sm);

// 等待 SM 出现
gboolean gbinder_servicemanager_wait(GBinderServiceManager* sm, long max_wait_ms);

// presence 回调（SM 出现/消失）
gulong gbinder_servicemanager_add_presence_handler(
    GBinderServiceManager* sm,
    GBinderServiceManagerFunc func, void* user_data);

// 注册回调（服务名出现时通知）
gulong gbinder_servicemanager_add_registration_handler(
    GBinderServiceManager* sm,
    const char* name,
    GBinderServiceManagerRegistrationFunc func, void* user_data);

// 取消操作
void gbinder_servicemanager_cancel(GBinderServiceManager* sm, gulong id);
void gbinder_servicemanager_remove_handler(GBinderServiceManager* sm, gulong id);
void gbinder_servicemanager_remove_handlers(GBinderServiceManager* sm, gulong* ids, guint count);
#define gbinder_servicemanager_remove_all_handlers(sm, ids) \
    gbinder_servicemanager_remove_handlers(sm, ids, G_N_ELEMENTS(ids))
```

## 7.3 ServiceManager 完整流程

### 服务端注册服务

```c
// 参考 test/binder-service/binder-service.c

int main(int argc, char* argv[])
{
    // 1. 创建 ServiceManager
    GBinderServiceManager* sm = gbinder_servicemanager_new("/dev/binder");

    // 2. 等待 SM 就绪
    if (!gbinder_servicemanager_wait(sm, -1)) {
        return 1;
    }

    // 3. 创建 LocalObject（服务实现）
    GBinderLocalObject* obj = gbinder_servicemanager_new_local_object(
        sm, "test@1.0", app_reply, &app);

    // 4. 注册 presence handler（SM 重启时重新注册）
    gulong presence_id = gbinder_servicemanager_add_presence_handler(
        sm, app_sm_presence_handler, &app);

    // 5. 注册服务
    gbinder_servicemanager_add_service(sm, "test", obj,
        app_add_service_done, &app);

    // 6. 运行事件循环
    GMainLoop* loop = g_main_loop_new(NULL, TRUE);
    g_main_loop_run(loop);

    // 7. 清理
    gbinder_servicemanager_remove_handler(sm, presence_id);
    gbinder_local_object_unref(obj);
    gbinder_servicemanager_unref(sm);
}
```

### SM presence handler（处理 SM 重启）

```c
static void
app_sm_presence_handler(GBinderServiceManager* sm, void* user_data)
{
    App* app = user_data;
    if (gbinder_servicemanager_is_present(app->sm)) {
        // SM 重新出现，重新注册服务
        gbinder_servicemanager_add_service(app->sm, app->opt->name,
            app->obj, app_add_service_done, app);
    }
}
```

### 客户端获取服务

```c
// 参考 test/binder-client/binder-client.c

int main(int argc, char* argv[])
{
    // 1. 创建 ServiceManager
    GBinderServiceManager* sm = gbinder_servicemanager_new("/dev/binder");

    // 2. 创建 LocalObject（用于接收回调）
    GBinderLocalObject* local = gbinder_servicemanager_new_local_object(
        sm, NULL, NULL, NULL);

    // 3. 尝试获取服务
    GBinderRemoteObject* remote = gbinder_servicemanager_get_service_sync(
        sm, "test@1.0/test", NULL);

    if (!remote) {
        // 服务未就绪，注册等待
        gbinder_servicemanager_add_registration_handler(sm,
            "test@1.0/test", app_registration_handler, &app);
    }

    // 4. 运行事件循环
    GMainLoop* loop = g_main_loop_new(NULL, TRUE);
    g_main_loop_run(loop);
}

// 服务出现时的回调
static void
app_registration_handler(GBinderServiceManager* sm, const char* name, void* user_data)
{
    App* app = user_data;
    if (!strcmp(name, app->fqname)) {
        // 获取 RemoteObject
        app->remote = gbinder_servicemanager_get_service_sync(sm, app->fqname, NULL);
        // 创建 Client
        app->client = gbinder_client_new(app->remote, "test@1.0");
        // 注册死亡通知
        app->death_id = gbinder_remote_object_add_death_handler(
            app->remote, app_remote_died, app);
    }
}
```

## 7.4 GBinderClient — 客户端

### 创建 Client

```c
// include/gbinder_client.h
// 单接口
GBinderClient* gbinder_client_new(
    GBinderRemoteObject* object,  // 从 SM 获取的远程对象
    const char* iface);            // 接口名

// 多接口
GBinderClient* gbinder_client_new2(
    GBinderRemoteObject* object,
    const GBinderClientIfaceInfo* ifaces,  // 接口信息数组
    gsize count);
```

### 事务操作

```c
// 创建请求
GBinderLocalRequest* gbinder_client_new_request(GBinderClient* client);
GBinderLocalRequest* gbinder_client_new_request2(GBinderClient* client, guint32 code);

// 同步事务（等待回复）
GBinderRemoteReply* gbinder_client_transact_sync_reply(
    GBinderClient* client,
    guint32 code,                  // 事务码
    GBinderLocalRequest* req,
    int* status);

// 同步单向事务（不等待回复）
int gbinder_client_transact_sync_oneway(
    GBinderClient* client,
    guint32 code,
    GBinderLocalRequest* req);

// 异步事务
gulong gbinder_client_transact(
    GBinderClient* client,
    guint32 code,
    guint32 flags,                 // GBINDER_TX_FLAG_ONEWAY 等
    GBinderLocalRequest* req,
    GBinderClientReplyFunc reply,  // 回复回调
    GDestroyNotify destroy,
    void* user_data);

// 取消异步事务
void gbinder_client_cancel(GBinderClient* client, gulong id);
```

### 回复回调

```c
typedef void (*GBinderClientReplyFunc)(
    GBinderClient* client,
    GBinderRemoteReply* reply,
    int status,
    void* user_data);
```

### 客户端完整调用示例

```c
static void
app_call(App* app, char* str)
{
    // 1. 构建请求
    GBinderLocalRequest* req = gbinder_client_new_request(app->client);

    // 2. 写入参数
    gbinder_local_request_append_string16(req, str);

    // 3. 同步事务
    int status;
    GBinderRemoteReply* reply = gbinder_client_transact_sync_reply(
        app->client, GBINDER_FIRST_CALL_TRANSACTION, req, &status);

    // 4. 释放请求
    gbinder_local_request_unref(req);

    // 5. 解析回复
    if (status == GBINDER_STATUS_OK) {
        GBinderReader reader;
        gbinder_remote_reply_init_reader(reply, &reader);
        char* ret = gbinder_reader_read_string16(&reader);
        printf("Reply: %s\n", ret);
        g_free(ret);
    }

    // 6. 释放回复
    gbinder_remote_reply_unref(reply);
}
```

## 7.5 ServiceManager 内部架构

### SM 后端接口

```c
// src/gbinder_servicemanager_p.h (概念)
// 每个 SM 后端实现一组函数:
typedef struct gbinder_servicemanager_backend {
    const char* name;               // "aidl", "aidl2", ..., "hidl"
    // 获取 SM 的 RemoteObject
    GBinderRemoteObject* (*get_service_manager)(GBinderIpc* ipc);
    // 列出所有服务
    char** (*list)(GBinderServiceManager* sm);
    // 获取服务
    GBinderRemoteObject* (*get_service)(GBinderServiceManager* sm, const char* name, int* status);
    // 注册服务
    int (*add_service)(GBinderServiceManager* sm, const char* name, GBinderLocalObject* obj);
    // ...
} GBinderServiceManagerBackend;
```

### SM 事务码

各版本的 SM 使用不同的事务码来表示 list/get/add 操作：

```
aidl:   GET_SERVICE_TRANSACTION, CHECK_SERVICE_TRANSACTION, ADD_SERVICE_TRANSACTION, LIST_SERVICES_TRANSACTION
aidl2:  (不同的码)
aidl3:  (不同的码)
...
hidl:   (HIDL 特有的码)
```

> 这就是为什么需要多个 SM 后端——不同 Android 版本的 SM 接口事务码和参数格式不同。

## 7.6 GBinderServiceName — 服务名管理

```c
// include/gbinder_servicename.h
// 管理服务名的生命周期和通知
GBinderServiceName* gbinder_servicename_new(
    GBinderServiceManager* sm,
    const char* name,
    GBinderServiceNameFunc notify, void* user_data);
```

> ServiceName 封装了服务名出现/消失的监控逻辑，简化客户端等待服务的代码。

## 7.7 GBinderServicePoll — 服务轮询

```c
// src/gbinder_servicepoll.c
// 定期轮询 SM 检查服务是否可用
// 用于不支持 registration handler 的旧版 SM
```

## 7.8 GBinderBridge — 桥接

```c
// include/gbinder_bridge.h
// 在不同 Binder 域间桥接服务
GBinderBridge* gbinder_bridge_new(
    GBinderLocalObject* local,    // 域 A 的本地对象
    GBinderRemoteObject* remote); // 域 B 的远程对象
```

### 桥接工作原理

```
域 A (/dev/binder)              域 B (/dev/hwbinder)

Client ──> ProxyObject ──> RemoteObject ──> Server
          (LocalObject     (在域 B 上)
           在域 A 上)

ProxyObject 收到域 A 的事务后:
  1. 从 RemoteRequest 读取数据
  2. 构建新的 LocalRequest
  3. 通过域 B 的 Client 发送到 Server
  4. 收到 RemoteReply
  5. 构建 LocalReply 返回给域 A 的 Client
```

## 7.9 GBinderSystem — 系统封装

```c
// src/gbinder_system.c / src/gbinder_system.h
// 封装系统调用，便于测试 mock:
int gbinder_system_open(const char* path, int flags);
int gbinder_system_ioctl(int fd, unsigned long request, void* data);
void* gbinder_system_mmap(gsize size, int prot, int flags, int fd);
int gbinder_system_munmap(void* addr, gsize size);
int gbinder_system_close(int fd);
```

> 通过抽象系统调用，单元测试可以 mock 这些函数而不需要真实的 binder 设备。

## 7.10 完整的 Client-Server 交互图

```
┌─────────────────────────────────────────────────────────────────┐
│                        ServiceManager (handle=0)                  │
└──────┬───────────────────────────────────────────┬──────────────┘
       │ 3. add_service("test", obj)               │ 5. get_service("test")
       v                                           v
┌──────────────┐                              ┌──────────────┐
│   Server     │                              │   Client     │
│              │                              │              │
│ 1. sm_new()  │                              │ 1. sm_new()  │
│ 2. new_local │                              │ 4. get_service
│    _object() │                              │    → remote  │
│ 3. add_service                              │ 6. client_new│
│              │                              │    (remote)  │
│ 7. handler <─┼──────────────────────────────┼── transact   │
│    (req)     │     BC_TRANSACTION           │    (code,req)│
│              │                              │              │
│ 8. reply ────┼──────────────────────────────┼─> reply      │
│              │     BC_REPLY                  │              │
│              │                              │ 9. reader    │
│              │                              │    read reply│
└──────────────┘                              └──────────────┘
```

## 7.11 小结

| 概念 | 要点 |
|------|------|
| ServiceManager | 名称→句柄映射，handle=0 |
| SM 后端 | aidl/aidl2~6/hidl，各版本协议不同 |
| sm_new | 创建 SM，自动选择协议 |
| new_local_object | 创建服务端对象，注册事务处理回调 |
| add_service | 注册服务到 SM |
| get_service | 从 SM 获取 RemoteObject |
| presence handler | SM 重启时重新注册服务 |
| registration handler | 服务出现时通知客户端 |
| GBinderClient | 封装 RemoteObject，提供事务 API |
| transact_sync_reply | 同步事务，阻塞等待回复 |
| transact_sync_oneway | 单向事务，不等待回复 |
| transact (async) | 异步事务，回调通知 |
| GBinderBridge | 跨 Binder 域桥接 |
| GBinderSystem | 系统调用封装，便于测试 |

---

**上一阶段**：[06-data-serialization.md](06-data-serialization.md)
**下一阶段**：[08-cross-platform-and-practice.md](08-cross-platform-and-practice.md) — 跨平台特性与实战
