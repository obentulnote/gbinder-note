# 阶段 2：总体架构与配置系统

> 对应源码：`src/gbinder_config.c`、`src/gbinder_rpc_protocol.c`、`src/gbinder_types_p.h`、`include/gbinder_types.h`

## 2.1 整体架构设计

libgbinder 的核心设计理念是：**API 不变，后端可配置**。

Android 在不同版本中反复修改底层 RPC 协议和 ServiceManager 协议。libgbinder 通过以下机制应对：

1. **配置文件**（`/etc/gbinder.conf`）选择协议变体
2. **API Level 预设**自动选择合适的协议组合
3. **运行时 32/64 位检测**选择对应的 IO 实现
4. **统一的抽象接口**（`GBinderRpcProtocol`、`GBinderIo`）隔离版本差异

### 分层架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Public API (include/)                      │
│                                                              │
│  gbinder_servicemanager_new()  gbinder_client_new()          │
│  gbinder_local_object_new()    gbinder_writer_*()            │
├─────────────────────────────────────────────────────────────┤
│              ServiceManager 层 (可配置后端)                    │
│                                                              │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ aidl    │ │ aidl2~6  │ │  hidl    │ │  (未来)  │         │
│  │ (SM)   │ │ (SM)     │ │ (SM)     │ │          │         │
│  └─────────┘ └──────────┘ └──────────┘ └──────────┘         │
├─────────────────────────────────────────────────────────────┤
│              IPC 核心层 (gbinder_ipc.c)                        │
│                                                              │
│  事务调度 │ 异步处理 │ Looper 线程管理 │ 死亡通知              │
├──────────────────────┬──────────────────────────────────────┤
│   RPC Protocol 层    │    Driver 层                          │
│  (gbinder_rpc_       │  (gbinder_driver.c)                   │
│   protocol.c)         │                                       │
│                      │  open(/dev/binder)                    │
│  aidl/aidl2~4/hidl   │  ioctl(BINDER_WRITE_READ)             │
│  头部格式             │  poll/looper                          │
├──────────────────────┴──────────────────────────────────────┤
│              IO 抽象层 (gbinder_io.h)                          │
│                                                              │
│  ┌──────────────────┐    ┌──────────────────┐               │
│  │ gbinder_io_32   │    │ gbinder_io_64    │               │
│  │ (32位内核)       │    │ (64位内核)        │               │
│  └──────────────────┘    └──────────────────┘               │
├─────────────────────────────────────────────────────────────┤
│              Linux Kernel Binder Driver                       │
│              /dev/binder, /dev/hwbinder                      │
└─────────────────────────────────────────────────────────────┘
```

## 2.2 配置系统 — gbinder_config.c

### 配置文件位置

```c
// src/gbinder_config.c
static const char gbinder_config_default_file[] = "/etc/gbinder.conf";
static const char gbinder_config_default_dir[] = "/etc/gbinder.d";

const char* gbinder_config_file = gbinder_config_default_file;
const char* gbinder_config_dir = gbinder_config_default_dir;
```

### 配置文件格式

```ini
# /etc/gbinder.conf

[General]
ApiLevel = 29

[Protocol]
Default = aidl
/dev/binder = aidl
/dev/hwbinder = hidl

[ServiceManager]
Default = aidl
/dev/binder = aidl
/dev/hwbinder = hidl
```

### 配置加载流程

```
gbinder_config_get()
    │
    ├── 1. 读取 /etc/gbinder.conf（主配置文件）
    │
    ├── 2. 扫描 /etc/gbinder.d/*.conf（覆盖配置目录）
    │      └── 文件列表排序后逐个加载，后加载的覆盖先加载的
    │
    ├── 3. 合并所有 KeyFile（gbinder_config_merge_keyfiles）
    │
    └── 4. 如果设置了 ApiLevel，应用预设（gbinder_config_apply_presets）
           └── 预设只填充未显式设置的键（不覆盖用户配置）
```

### 配置合并逻辑

```c
// src/gbinder_config.c — gbinder_config_merge_keyfiles
// 将 src KeyFile 的所有组/键合并到 dest KeyFile
// src 中的值覆盖 dest 中的同名键
static GKeyFile*
gbinder_config_merge_keyfiles(GKeyFile* dest, GKeyFile* src)
{
    // 遍历 src 的所有组 → 所有键 → 复制到 dest
}
```

### 延迟释放机制

```c
// src/gbinder_config.c
// 配置 KeyFile 不会立即释放，而是延迟到下一个 idle 回调
// 因为配置通常在同一调用栈中被多次查询
static GKeyFile* gbinder_config_keyfile = NULL;
static GBinderEventLoopCallback* gbinder_config_autorelease = NULL;

GKeyFile* gbinder_config_get()
{
    if (!gbinder_config_keyfile) {
        gbinder_config_keyfile = gbinder_config_load_files();
        if (gbinder_config_keyfile) {
            // 安排 idle 回调释放 KeyFile
            gbinder_config_autorelease = gbinder_idle_callback_schedule_new(
                gbinder_config_autorelease_cb, gbinder_config_keyfile, NULL);
        }
    }
    return gbinder_config_keyfile;
}
```

> **设计原因**：配置在初始化时被频繁查询（同一调用栈多次），延迟释放避免重复读取文件。

## 2.3 API Level 预设

libgbinder 内置了不同 Android API Level 的协议预设，用户只需指定 `ApiLevel` 即可自动选择合适的协议组合。

### 预设表

```c
// src/gbinder_config.c — 按降序排列
static const GBinderConfigPreset gbinder_config_presets[] = {
    { 36, gbinder_config_36 },  // Android 16
    { 35, gbinder_config_35 },  // Android 15
    { 33, gbinder_config_33 },  // Android 13
    { 31, gbinder_config_31 },  // Android 12
    { 30, gbinder_config_30 },  // Android 11
    { 29, gbinder_config_29 },  // Android 10
    { 28, gbinder_config_28 },  // Android 9
};
```

### 各 API Level 的协议选择

| API Level | Android 版本 | Protocol | ServiceManager | 说明 |
|-----------|-------------|----------|----------------|------|
| 28 | Android 9 | (默认) | aidl2 | SM 协议变更 |
| 29 | Android 10 | aidl2 | aidl2 | RPC 头部增加 work source |
| 30 | Android 11 | aidl3 | aidl3 | RPC 头部增加 SYST 标识，flat_binder 增加 stability |
| 31 | Android 12 | aidl4 | aidl3 | flat_binder 的 stability 格式变更 |
| 33 | Android 13 | aidl3 | aidl3 | 回退到 aidl3 协议 |
| 35 | Android 15 | aidl3 | aidl5 | SM 协议变更 |
| 36 | Android 16 | aidl3 | aidl6 | SM 协议再次变更 |

### 预设应用逻辑

```c
// src/gbinder_config.c — gbinder_config_apply_presets
// 只填充用户未显式设置的键
static void
gbinder_config_apply_presets(GKeyFile* config, const GBinderConfigPreset* preset)
{
    for (g = preset->groups; g->name; g++) {
        for (e = g->entries; e->key; e++) {
            if (!g_key_file_has_key(config, g->name, e->key, NULL)) {
                g_key_file_set_value(config, g->name, e->key, e->value);
            }
        }
    }
}
```

> **关键**：预设不会覆盖用户显式配置的值。用户可以混合使用 ApiLevel 和手动配置。

## 2.4 RPC 协议变体 — gbinder_rpc_protocol.c

### GBinderRpcProtocol 结构

```c
// src/gbinder_rpc_protocol.h
struct gbinder_rpc_protocol {
    const char* name;                    // "aidl", "aidl2", "aidl3", "aidl4", "hidl"
    guint32 ping_tx;                     // ping 事务码
    void (*write_ping)(GBinderWriter* writer);
    void (*write_rpc_header)(GBinderWriter* writer, const char* iface);
    const char* (*read_rpc_header)(GBinderReader* reader, guint32 txcode, char** iface);

    gsize flat_binder_object_extra;      // flat_binder_object 的额外数据大小
    void (*finish_flatten_binder)(void* out, GBinderLocalObject* obj);
    void (*finish_unflatten_binder)(const void* in, GBinderRemoteObject* obj);
    void (*write_fmq_descriptor)(GBinderWriter* writer, const GBinderFmq* queue);
};
```

### 各协议的 RPC 头部格式对比

#### aidl（原始 AIDL，Android 9 之前）

```
| int32: strict_mode_flags | string16: interface_name |
```

```c
gbinder_writer_append_int32(writer, BINDER_RPC_FLAGS);    // STRICT_MODE_PENALTY_GATHER
gbinder_writer_append_string16(writer, iface);
```

#### aidl2（Android 10 / API 29）

```
| int32: strict_mode_flags | int32: work_source(-1) | string16: interface_name |
```

```c
gbinder_writer_append_int32(writer, BINDER_RPC_FLAGS);
gbinder_writer_append_int32(writer, UNSET_WORK_SOURCE);   // -1
gbinder_writer_append_string16(writer, iface);
```

#### aidl3（Android 11 / API 30）

```
| int32: flags | int32: work_source | int32: SYST_header | string16: interface_name |
```

```c
gbinder_writer_append_int32(writer, BINDER_RPC_FLAGS);
gbinder_writer_append_int32(writer, UNSET_WORK_SOURCE);
gbinder_writer_append_int32(writer, BINDER_SYS_HEADER);   // 'SYST' fourcc
gbinder_writer_append_string16(writer, iface);
```

> aidl3 还增加了 `flat_binder_object_extra = 4`，在 flat_binder_object 后附加 4 字节的 stability 信息。

#### aidl4（Android 12 / API 31）

RPC 头部与 aidl3 相同，但 flat_binder_object 的 stability 格式变更：

```c
// aidl3: 直接写入 stability_level (guint32)
*(guint32*)out = obj->stability;

// aidl4: 写入结构化的 stability_category
struct stability_category {
    guint8 binder_wire_format_version;  // = 1
    guint8 reserved[2];
    guint8 stability_level;
};
```

#### hidl（HIDL 协议）

```
| string8: interface_name |
```

```c
gbinder_writer_append_string8(writer, iface);  // 注意是 string8，不是 string16
```

> HIDL 使用 UTF-8 字符串（string8），AIDL 使用 UTF-16 字符串（string16）。

### 协议选择流程

```
gbinder_rpc_protocol_for_device(dev)
    │
    ├── 1. 首次调用时加载配置（gbinder_rpc_protocol_load_config）
    │      ├── 读取 [Protocol] 配置组
    │      ├── 将配置值（"aidl"/"hidl"等）映射为 GBinderRpcProtocol*
    │      └── 添加默认值：/dev/binder → aidl, /dev/hwbinder → hidl
    │
    ├── 2. 提取 "Default" 特殊值作为默认协议
    │
    └── 3. 查找 dev 对应的协议
           ├── 找到 → 返回对应协议
           └── 未找到 → 返回默认协议
```

### 已注册的协议列表

```c
// src/gbinder_rpc_protocol.c
static const GBinderRpcProtocol* gbinder_rpc_protocol_list[] = {
    &gbinder_rpc_protocol_aidl,    // 原始 AIDL
    &gbinder_rpc_protocol_aidl2,   // Android 10
    &gbinder_rpc_protocol_aidl3,   // Android 11
    &gbinder_rpc_protocol_aidl4,   // Android 12
    &gbinder_rpc_protocol_hidl     // HIDL
};
```

> 注意：aidl5 和 aidl6 是 ServiceManager 协议变体，不是 RPC 协议变体。RPC 协议从 aidl3 之后没有再变。

## 2.5 内部类型定义 — gbinder_types_p.h

### 内部事务码

```c
// src/gbinder_types_p.h
#define GBINDER_TRANSACTION(c2,c3,c4)     GBINDER_FOURCC('_',c2,c3,c4)
#define GBINDER_PING_TRANSACTION          GBINDER_TRANSACTION('P','N','G')  // ping
#define GBINDER_DUMP_TRANSACTION          GBINDER_TRANSACTION('D','M','P')  // dump
#define GBINDER_SHELL_COMMAND_TRANSACTION GBINDER_TRANSACTION('C','M','D')  // shell 命令
#define GBINDER_INTERFACE_TRANSACTION     GBINDER_TRANSACTION('N','T','F')  // 接口查询
#define GBINDER_SYSPROPS_TRANSACTION      GBINDER_TRANSACTION('S','P','R')  // 系统属性

// HIDL 特有事务码
#define HIDL_PING_TRANSACTION         HIDL_FOURCC('P','N','G')
#define HIDL_LINK_TO_DEATH_TRANSACTION    HIDL_FOURCC('L','T','D')
#define HIDL_UNLINK_TO_DEATH_TRANSACTION  HIDL_FOURCC('U','T','D')
// ...

// ServiceManager 的句柄固定为 0
#define GBINDER_SERVICEMANAGER_HANDLE (0)
```

### 内部类型前向声明

```c
// src/gbinder_types_p.h
typedef struct gbinder_buffer_contents GBinderBufferContents;
typedef struct gbinder_buffer_contents_list GBinderBufferContentsList;
typedef struct gbinder_cleanup GBinderCleanup;
typedef struct gbinder_driver GBinderDriver;
typedef struct gbinder_handler GBinderHandler;
typedef struct gbinder_io GBinderIo;
typedef struct gbinder_object_converter GBinderObjectConverter;
typedef struct gbinder_object_registry GBinderObjectRegistry;
typedef struct gbinder_output_data GBinderOutputData;
typedef struct gbinder_proxy_object GBinderProxyObject;
typedef struct gbinder_rpc_protocol GBinderRpcProtocol;
typedef struct gbinder_servicepoll GBinderServicePoll;
typedef struct gbinder_ipc_looper_tx GBinderIpcLooperTx;
typedef struct gbinder_ipc_sync_api GBinderIpcSyncApi;
```

### 关键宏

```c
#define GBINDER_INTERNAL G_GNUC_INTERNAL       // 标记内部符号（不导出）
#define GBINDER_DESTRUCTOR __attribute__((destructor))  // 析构函数属性
```

## 2.6 公共类型定义 — gbinder_types.h

### 核心术语图解

```c
// include/gbinder_types.h — 注释中的术语说明
/*
 * 1. RemoteObject is the one we sent requests to.
 * 2. LocalObjects may receive incoming transactions (and reply to them)
 * 3. We must have a RemoteObject to initiate a transaction.
 *    We send LocalRequest and receive RemoteReply:
 *
 *    LocalObject --- (LocalRequest) --> Client(RemoteObject)
 *    LocalObject <-- (RemoteReply) -- RemoteObject
 *
 * 4. LocalObject knows caller's pid (and therefore credentials)
 *
 *    LocalObject <-- (RemoteRequest) --- (pid)
 *    LocalObject --- (LocalReply)    --> (pid)
 *
 * 5. Writer prepares the data for LocalRequest and LocalReply
 * 6. Reader parses the data coming with RemoteRequest and RemoteReply
 */
```

### 稳定性级别

```c
// include/gbinder_types.h
typedef enum gbinder_stability_level {
    GBINDER_STABILITY_UNDECLARED = 0,   // 未声明
    GBINDER_STABILITY_VENDOR    = 0x03,  // 厂商级
    GBINDER_STABILITY_SYSTEM    = 0x0c,  // 系统级
    GBINDER_STABILITY_VINTF     = 0x3f   // VINTF（Vendor Interface）级
} GBINDER_STABILITY_LEVEL;
```

> 稳定性级别从 aidl3 开始被写入 flat_binder_object 的 extra 数据中，用于声明接口的兼容性承诺。

### HIDL 特有类型

```c
// HIDL vec（向量）
typedef struct gbinder_hidl_vec {
    union { guint64 value; const void* ptr; } data;
    guint32 count;
    guint8 owns_buffer;
    guint8 pad[3];
} GBinderHidlVec;  // 16 字节

// HIDL string
typedef struct gbinder_hidl_string {
    union { guint64 value; const char* str; } data;
    guint32 len;
    guint8 owns_buffer;
    guint8 pad[3];
} GBinderHidlString;  // 16 字节

// HIDL FDs
typedef struct gbinder_fds {
    guint32 version;   // = 12
    guint32 num_fds;
    guint32 num_ints;
} GBinderFds;  // 12 字节，后跟实际 fd 数组

// HIDL handle
typedef struct gbinder_hidl_handle {
    union { guint64 value; const GBinderFds* fds; } data;
    guint8 owns_handle;
    guint8 pad[7];
} GBinderHidlHandle;  // 16 字节

// HIDL memory
typedef struct gbinder_hidl_memory {
    union { guint64 value; const GBinderFds* fds; } data;
    guint8 owns_buffer;
    guint8 pad[7];
    guint64 size;
    GBinderHidlString name;
} GBinderHidlMemory;  // 40 字节
```

## 2.7 配置系统数据流图

```
                    /etc/gbinder.conf
                         │
                         v
              ┌─────────────────────┐
              │  gbinder_config_get() │
              └──────────┬──────────┘
                         │
          ┌──────────────┼──────────────┐
          v              v              v
    /etc/gbinder.d/   合并 KeyFile   ApiLevel 预设
    *.conf (排序)     (后者覆盖前者)  (只填未设置的键)
          │              │              │
          └──────────────┴──────────────┘
                         │
                         v
              ┌─────────────────────┐
              │   GKeyFile (合并后)   │
              └──────────┬──────────┘
                         │
           ┌─────────────┼─────────────┐
           v             v             v
     [Protocol]   [ServiceManager]  [General]
           │             │
           v             v
  gbinder_rpc_protocol   gbinder_servicemanager
  _for_device(dev)       _new2(dev, sm_proto, rpc_proto)
           │             │
           v             v
  GBinderRpcProtocol*   选择对应 SM 后端
  (aidl/aidl2/.../hidl)  (aidl/aidl2~6/hidl)
```

## 2.8 小结

| 概念 | 要点 |
|------|------|
| 配置文件 | `/etc/gbinder.conf` + `/etc/gbinder.d/*.conf`（后者覆盖前者） |
| API Level 预设 | 指定 ApiLevel 自动选择协议组合，不覆盖用户显式配置 |
| RPC 协议变体 | aidl / aidl2 / aidl3 / aidl4 / hidl，头部格式各不同 |
| SM 协议变体 | aidl / aidl2~6 / hidl，独立于 RPC 协议 |
| 协议选择 | 按设备名查找配置，未找到则用 Default |
| 延迟释放 | 配置 KeyFile 延迟到 idle 回调释放，避免重复读取 |
| 稳定性级别 | UNDECLARED / VENDOR / SYSTEM / VINTF，aidl3+ 写入 flat_binder |
| ServiceManager 句柄 | 固定为 0 |

---

**上一阶段**：[01-binder-background.md](01-binder-background.md)
**下一阶段**：[03-driver-and-io-layer.md](03-driver-and-io-layer.md) — Driver 与 IO 层
