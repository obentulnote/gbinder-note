# 阶段 8：跨平台特性与实战测试

> 对应源码：`src/gbinder_rpc_protocol.c`、`src/gbinder_rpc_protocol.h`、`src/gbinder_config.c`、`src/gbinder_system.c`、`test/binder-client/`、`test/binder-service/`、`_test/`

## 8.1 跨平台架构概述

libgbinder 的设计目标是**在非 Android 平台上**（如 Sailfish OS / Mer）使用 Android 的 Binder IPC。它通过以下机制实现跨平台：

1. **配置系统** — 通过配置文件适配不同设备和 Android 版本
2. **协议抽象** — 支持多种 RPC 协议（aidl/aidl2~4/hidl）
3. **IO 模板** — 同时支持 32 位和 64 位内核 Binder
4. **系统调用封装** — 便于 mock 和移植
5. **事件循环抽象** — 可替换的事件循环后端

## 8.2 RPC 协议系统

### GBinderRpcProtocol 结构

```c
// src/gbinder_rpc_protocol.h
typedef struct gbinder_rpc_protocol {
    const char* name;              // "aidl", "aidl2", "aidl3", "aidl4", "hidl"
    gsize tx_data_offset;          // 事务数据在 binder_transaction_data 中的偏移
    gsize tx_objects_offset;       // 对象数组偏移
    gsize tx_code_offset;          // 事务码偏移
    gsize tx_flags_offset;         // 事务标志偏移
    gsize tx_pid_offset;           // PID 偏移
    gsize tx_euid_offset;           // EUID 偏移
    gsize tx_target_offset;        // 目标偏移
    gsize tx_data_size_offset;     // 数据大小偏移
    gsize tx_objects_size_offset;  // 对象数量偏移
    // ...
} GBinderRpcProtocol;
```

### 协议变体

| 协议 | 说明 | RPC Header |
|------|------|------------|
| `aidl` | 原始 AIDL | int32 flags + int32 work_source + string16 iface |
| `aidl2` | API 28+ | 同 aidl，但 SM 事务码不同 |
| `aidl3` | API 30+ | 同 aidl2，但支持 stability |
| `aidl4` | API 31+ | 同 aidl3，但 stability 格式不同 |
| `hidl` | HIDL | string8 iface + string8 package |

### 协议选择

```c
// src/gbinder_rpc_protocol.c
const GBinderRpcProtocol* gbinder_rpc_protocol_for_device(const char* dev);
// 根据 /etc/gbinder.conf 配置选择协议
// 如果未配置，默认:
//   /dev/binder   → aidl2
//   /dev/hwbinder → hidl
//   /dev/vndbinder → aidl2
```

## 8.3 配置系统回顾

### 配置文件示例

```ini
# /etc/gbinder.conf
[binder]
dev = /dev/binder
rpc-protocol = aidl2
servicemanager = aidl3

[hwbinder]
dev = /dev/hwbinder
rpc-protocol = hidl
servicemanager = hidl
```

### 配置覆盖

```ini
# /etc/gbinder.d/*.conf
# 可以覆盖主配置文件的设置
# 按字母顺序加载，后加载的覆盖先加载的
```

## 8.4 32/64 位兼容性

### 运行时检测

```c
// src/gbinder_driver.c
// Driver 初始化时通过 BINDER_VERSION ioctl 检测:
ioctl(fd, BINDER_VERSION, &version);
if (version == 7) → gbinder_io_32  // 32 位内核
if (version == 8) → gbinder_io_64  // 64 位内核
```

### 模板实例化机制

```
gbinder_io_32.c                    gbinder_io_64.c
┌─────────────────┐               ┌─────────────────┐
│ #define 32BIT   │               │ #undef 32BIT    │
│ #define PREFIX  │               │ #define PREFIX  │
│   gbinder_io_32 │               │   gbinder_io_64 │
│ #include io.c   │               │ #include io.c   │
└─────────────────┘               └─────────────────┘
         │                                │
         v                                v
    编译生成:                        编译生成:
    gbinder_io_32_*()               gbinder_io_64_*()
    (使用 __u32 指针)               (使用 __u64 指针)
```

> **关键**：同一份 `gbinder_io.c` 源码被编译两次，通过宏控制生成 32 位和 64 位两套实现。运行时根据内核版本选择使用哪一套。

## 8.5 测试程序分析

### _test/ 目录

项目的 `_test/` 目录包含学习用的测试程序。参考 `test/` 目录中的官方测试：

#### binder-service（服务端）

```
test/binder-service/
├── binder-service.c    # 服务端主程序
└── Makefile
```

服务端核心流程：
1. 创建 ServiceManager
2. 等待 SM 就绪
3. 创建 LocalObject（注册事务处理回调）
4. 注册 presence handler
5. add_service 注册服务
6. 运行 GLib MainLoop

#### binder-client（客户端）

```
test/binder-client/
├── binder-client.c    # 客户端主程序
└── Makefile
```

客户端核心流程：
1. 创建 ServiceManager
2. 尝试 get_service
3. 如果服务不存在，注册 registration handler
4. 服务出现后创建 Client
5. 注册死亡通知
6. 发起事务调用

### 编译与运行

```bash
# 编译 libgbinder
cd libgbinder-master
make

# 编译测试程序
cd test/binder-service
make
cd ../binder-client
make

# 运行（需要 binder 内核模块）
# 终端1: 启动服务
./binder-service -n test@1.0

# 终端2: 启动客户端
./binder-client -n test@1.0/test
```

## 8.6 单元测试架构

### test/ 目录结构

```
test/
├── binder-service/          # 服务端测试程序
├── binder-client/           # 客户端测试程序
├── common/
│   ├── test_binder.c        # 测试辅助函数
│   └── test_common.h
├── unit/                    # 单元测试
│   ├── unit_servicemanager.c
│   ├── unit_client.c
│   ├── unit_writer.c
│   ├── unit_reader.c
│   └── ...
└── Makefile
```

### GBinderSystem 的 mock 机制

```c
// src/gbinder_system.h
// 通过函数指针封装系统调用:
typedef struct gbinder_system_functions {
    int (*open)(const char* path, int flags);
    int (*ioctl)(int fd, unsigned long request, void* data);
    void* (*mmap)(gsize size, int prot, int flags, int fd);
    int (*munmap)(void* addr, gsize size);
    int (*close)(int fd);
} GBinderSystemFunctions;

// 测试时替换为 mock 实现:
void gbinder_system_set_functions(const GBinderSystemFunctions* fn);
```

> 这使得单元测试可以在没有真实 binder 设备的环境中运行，通过 mock 系统调用来模拟 Binder 驱动行为。

## 8.7 事件循环可替换性

```c
// src/gbinder_eventloop.c
// 默认使用 GLib MainLoop:
static const GBinderEventLoopIntegration gbinder_eventloop_glib = { ... };

// 可以替换为其他事件循环:
void gbinder_eventloop_set(const GBinderEventLoopIntegration* loop);
```

### 自定义事件循环示例

```c
// 假设要适配 libuv:
static GBinderEventLoopIntegration uv_eventloop = {
    .timeout_add    = uv_timeout_add,
    .timeout_remove = uv_timeout_remove,
    .callback_new   = uv_callback_new,
    .callback_ref   = uv_callback_ref,
    .callback_unref = uv_callback_unref,
    .callback_schedule = uv_callback_schedule,
    .callback_cancel = uv_callback_cancel,
    .cleanup         = uv_cleanup
};

gbinder_eventloop_set(&uv_eventloop);
```

## 8.8 完整学习路径总结

```
阶段1: 背景知识
  └── 理解 Android Binder 的基本概念
      └── 一次拷贝、ServiceManager、BC/BR 命令

阶段2: 总体架构
  └── 理解 libgbinder 的分层和配置
      └── 5层架构、配置文件、协议选择

阶段3: Driver 与 IO 层
  └── 理解与内核的交互
      └── open/mmap/ioctl、32/64位模板、BC/BR 处理

阶段4: IPC 核心层
  └── 理解线程模型和调度
      └── Looper 线程、事务状态机、线程池

阶段5: 对象模型
  └── 理解 Local/Remote 对象
      └── 引用计数、死亡通知、ObjectRegistry

阶段6: 数据序列化
  └── 理解数据编解码
      └── Writer/Reader、string16/string8、HIDL 类型

阶段7: ServiceManager 与 Client
  └── 理解完整使用流程
      └── 注册服务、获取服务、发起事务

阶段8: 跨平台与实战
  └── 理解跨平台设计和测试
      └── 协议适配、配置系统、测试程序
```

## 8.9 关键设计模式总结

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| 模板方法 | gbinder_io_32/64.c | 同一源码编译两次，生成 32/64 位实现 |
| 策略模式 | GBinderRpcProtocol | 运行时选择 RPC 协议 |
| 策略模式 | GBinderEventLoopIntegration | 可替换事件循环后端 |
| 策略模式 | GBinderServiceManagerBackend | 多个 SM 后端实现 |
| 工厂模式 | gbinder_ipc_new | 缓存并复用 IPC 实例 |
| 观察者模式 | death_handler/presence_handler | 事件通知回调 |
| 引用计数 | 所有 GObject | g_ref/g_unref 管理生命周期 |
| 代理模式 | GBinderProxyObject | 桥接不同 Binder 域 |
| 桥接模式 | GBinderBridge | 连接两个 Binder 域 |
| 外观模式 | GBinderClient/ServiceManager | 简化复杂 IPC 操作 |

## 8.10 实战建议

### 编写服务端

```c
// 最小服务端模板
GBinderServiceManager* sm = gbinder_servicemanager_new("/dev/binder");
gbinder_servicemanager_wait(sm, -1);

GBinderLocalObject* obj = gbinder_servicemanager_new_local_object(
    sm, "my.service@1.0", my_handler, NULL);

gbinder_servicemanager_add_service(sm, "my-service", obj, NULL, NULL);

GMainLoop* loop = g_main_loop_new(NULL, TRUE);
g_main_loop_run(loop);
```

### 编写客户端

```c
// 最小客户端模板
GBinderServiceManager* sm = gbinder_servicemanager_new("/dev/binder");
int status;
GBinderRemoteObject* remote = gbinder_servicemanager_get_service_sync(
    sm, "my-service", &status);

GBinderClient* client = gbinder_client_new(remote, "my.service@1.0");

GBinderLocalRequest* req = gbinder_client_new_request(client);
gbinder_local_request_append_string16(req, "hello");

GBinderRemoteReply* reply = gbinder_client_transact_sync_reply(
    client, 1, req, &status);

// 解析 reply...
```

### 调试技巧

1. **设置环境变量**：`G_MESSAGES_DEBUG=all` 启用 libgbinder 的调试日志
2. **检查配置**：`cat /etc/gbinder.conf` 确认协议配置
3. **列出服务**：使用 `gbinder-servicelist-pull` 或类似工具
4. **strace**：`strace -e ioctl` 跟踪 Binder ioctl 调用

## 8.11 小结

| 概念 | 要点 |
|------|------|
| 跨平台 | 通过配置文件和协议抽象适配不同设备 |
| 32/64 位 | 运行时通过 BINDER_VERSION 检测，模板编译两套实现 |
| 协议系统 | aidl/aidl2~4/hidl，通过配置选择 |
| 事件循环 | 可替换后端，默认 GLib |
| 系统调用封装 | GBinderSystem 便于 mock 测试 |
| 测试程序 | binder-service + binder-client 完整示例 |
| 单元测试 | 通过 mock 系统调用实现无设备测试 |
| 设计模式 | 模板/策略/工厂/观察者/代理/桥接/外观 |

---

**上一阶段**：[07-servicemanager-and-client.md](07-servicemanager-and-client.md)
**返回索引**：[00-index.md](00-index.md)
