# 阶段 3：Driver 与 IO 层

> 对应源码：`src/gbinder_driver.c`、`src/gbinder_driver.h`、`src/gbinder_io.c`、`src/gbinder_io.h`、`src/gbinder_io_32.c`、`src/gbinder_io_64.c`、`src/binder.h`

## 3.1 Driver 层概述

`GBinderDriver` 是 libgbinder 与内核 Binder 驱动的**直接接口层**。它封装了所有与 `/dev/binder` 设备的交互：打开设备、mmap 内存、ioctl 事务、looper 循环等。

### GBinderDriver 结构

```c
// src/gbinder_driver.c
struct gbinder_driver {
    gint refcount;                    // 引用计数
    int fd;                           // 设备文件描述符
    void* vm;                         // mmap 的内存指针
    gsize vmsize;                     // mmap 大小
    char* dev;                        // 设备路径（如 "/dev/binder"）
    const char* name;                 // 短名（用于日志，如 "binder"）
    const GBinderIo* io;              // IO 实现（32 或 64 位）
    const GBinderRpcProtocol* protocol; // RPC 协议
};
```

## 3.2 Driver 初始化流程

```
gbinder_driver_new(dev, protocol)
    │
    ├── 1. open(dev, O_RDWR | O_CLOEXEC)                    // 打开 binder 设备
    │
    ├── 2. ioctl(fd, BINDER_VERSION, &version)              // 查询内核协议版本
    │      ├── version == 7 → 使用 gbinder_io_32（32 位内核）
    │      └── version == 8 → 使用 gbinder_io_64（64 位内核）
    │
    ├── 3. mmap(NULL, BINDER_VM_SIZE, PROT_READ,            // 映射内核缓冲区
    │          MAP_PRIVATE | MAP_NORESERVE, fd)
    │      └── BINDER_VM_SIZE = 1MB - 2*PAGE_SIZE
    │
    ├── 4. ioctl(fd, BINDER_SET_MAX_THREADS, &max_threads)  // 设置最大线程数
    │
    └── 5. 选择 RPC 协议
           ├── 显式指定 → 使用传入的 protocol
           └── 未指定 → gbinder_rpc_protocol_for_device(dev)
```

### 关键常量

```c
// src/gbinder_driver.c
#define BINDER_VM_SIZE ((1024*1024) - sysconf(_SC_PAGE_SIZE)*2)  // ~1MB
#define BINDER_MAX_REPLY_SIZE (256)
#define DEFAULT_MAX_BINDER_THREADS (0)  // 0 = 让内核自己决定

// ioctl 码
#define BINDER_VERSION _IOWR('b', 9, gint32)
#define BINDER_SET_MAX_THREADS _IOW('b', 5, guint32)
```

> **mmap 的作用**：Binder 驱动通过 mmap 将内核缓冲区映射到用户空间，实现"一次拷贝"——数据从发送方用户空间拷贝到内核缓冲区后，接收方可以直接通过 mmap 映射读取，无需第二次拷贝。

## 3.3 核心操作

### 3.3.1 write_read — 底层 ioctl 封装

所有与驱动的交互最终都通过 `BINDER_WRITE_READ` ioctl 完成：

```c
// src/gbinder_io.c — 实际由 gbinder_io_32/64.c 包含
static int gbinder_io_XX_write_read(int fd, GBinderIoBuf* write, GBinderIoBuf* read)
{
    struct binder_write_read bwr;
    memset(&bwr, 0, sizeof(bwr));

    if (write) {
        bwr.write_buffer = write->ptr + write->consumed;
        bwr.write_size = write->size - write->consumed;
    }
    if (read) {
        bwr.read_buffer = read->ptr + read->consumed;
        bwr.read_size = read->size - read->consumed;
    }

    ret = ioctl(fd, BINDER_WRITE_READ, &bwr);
    // 更新 consumed 偏移
    return ret;
}
```

### 3.3.2 发送命令

Driver 提供了几个辅助函数来发送 BC 命令：

```c
// 发送无参数命令（如 BC_ENTER_LOOPER）
gbinder_driver_cmd(self, io->bc.enter_looper);

// 发送带 int32 参数的命令（如 BC_INCREFS）
gbinder_driver_cmd_int32(self, io->bc.increfs, handle);

// 发送带复杂数据的命令（如 BC_TRANSACTION）
gbinder_driver_cmd_data(self, cmd, payload, buf);
```

### 3.3.3 transact — 发起事务

```c
// src/gbinder_driver.c
int gbinder_driver_transact(
    GBinderDriver* self,
    GBinderObjectRegistry* reg,
    GBinderHandler* handler,
    guint32 handle,           // 目标句柄
    guint32 code,             // 事务码
    GBinderLocalRequest* req, // 请求数据
    GBinderRemoteReply* reply) // 回复缓冲区（NULL = oneway）
{
    // 1. 编码 BC_TRANSACTION 或 BC_TRANSACTION_SG
    //    SG = Scatter-Gather（带额外缓冲区）
    if (extra_buffers) {
        *cmd = io->bc.transaction_sg;
        io->encode_transaction_sg(...);
    } else {
        *cmd = io->bc.transaction;
        io->encode_transaction(...);
    }

    // 2. 循环 write_read 直到获得事务状态
    while (txstatus == (-EAGAIN)) {
        gbinder_driver_write_read(self, &write, rbuf);
        txstatus = gbinder_driver_txstatus(self, &context, reply);
    }

    // 3. 处理入站命令（在等待回复期间可能收到其他事务）
    gbinder_driver_handle_commands(self, &context);

    return txstatus;
}
```

### 3.3.4 read — Looper 读取

```c
// src/gbinder_driver.c
int gbinder_driver_read(
    GBinderDriver* self,
    GBinderObjectRegistry* reg,
    GBinderHandler* handler)
{
    // 1. 初始化读缓冲区（栈上分配，128 字节）
    gbinder_driver_read_init(&read);

    // 2. write_read（只读，不写）
    gbinder_driver_write_read(self, NULL, context.rbuf);

    // 3. 处理所有入站命令
    gbinder_driver_handle_commands(self, &context);

    // 4. 如果还有数据且 handler 允许循环，继续读取
    while (read.buf.io.consumed && gbinder_handler_can_loop(handler)) {
        gbinder_driver_write_read(self, NULL, context.rbuf);
        gbinder_driver_handle_commands(self, &context);
    }
}
```

### 3.3.5 Looper 控制

```c
// 进入 looper
gbinder_driver_enter_looper(self)
    → BC_ENTER_LOOPER

// 退出 looper
gbinder_driver_exit_looper(self)
    → BC_EXIT_LOOPER
```

### 3.3.6 引用计数管理

```c
// 增加弱引用
gbinder_driver_increfs(self, handle)  → BC_INCREFS handle

// 减少弱引用
gbinder_driver_decrefs(self, handle)  → BC_DECREFS handle

// 增加强引用
gbinder_driver_acquire(self, handle) → BC_ACQUIRE handle

// 减少强引用
gbinder_driver_release(self, handle)  → BC_RELEASE handle
```

### 3.3.7 死亡通知

```c
// 请求死亡通知
gbinder_driver_request_death_notification(self, obj)
    → BC_REQUEST_DEATH_NOTIFICATION handle, cookie

// 清除死亡通知
gbinder_driver_clear_death_notification(self, obj)
    → BC_CLEAR_DEATH_NOTIFICATION handle, cookie

// 确认死亡通知处理完成
gbinder_driver_dead_binder_done(self, obj)
    → BC_DEAD_BINDER_DONE handle
```

## 3.4 BR 命令处理

`gbinder_driver_handle_command()` 是 BR 命令的分发中心：

```
BR 命令处理流程
├── BR_NOOP              → 忽略
├── BR_OK                → 忽略
├── BR_TRANSACTION_COMPLETE → 事务已提交（在 txstatus 中处理）
├── BR_SPAWN_LOOPER      → 通知 IPC 层创建新 looper 线程
├── BR_FINISHED          → 完成
├── BR_INCREFS           → 本地对象增加弱引用
│   └── 回复 BC_INCREFS_DONE
├── BR_ACQUIRE           → 本地对象增加强引用
│   └── 处理后回复 BC_ACQUIRE_DONE
├── BR_RELEASE           → 本地对象减少强引用（延迟处理）
├── BR_DECREFS           → 本地对象减少弱引用（延迟处理）
├── BR_TRANSACTION       → 收到入站事务
│   └── gbinder_driver_handle_transaction()
├── BR_DEAD_BINDER       → 远程对象死亡
│   └── 通知 GBinderRemoteObject
└── BR_CLEAR_DEATH_NOTIFICATION_DONE → 死亡通知清除完成
```

### BR_TRANSACTION 处理详解

```c
// src/gbinder_driver.c — gbinder_driver_handle_transaction
static void gbinder_driver_handle_transaction(...)
{
    // 1. 解码事务数据
    io->decode_transaction_data(data, &tx);
    // tx 包含: target, code, flags, pid, euid, data, size, objects

    // 2. 创建 RemoteRequest
    req = gbinder_remote_request_new(reg, protocol, tx.pid, tx.euid);

    // 3. 查找目标 LocalObject
    obj = gbinder_object_registry_get_local(reg, tx.target);

    // 4. 将数据包装为 GBinderBuffer
    buf = gbinder_buffer_new(self, tx.data, tx.size, tx.objects);
    gbinder_remote_request_set_data(req, tx.code, buf);

    // 5. 解析接口名
    iface = gbinder_remote_request_interface(req);

    // 6. 分发事务
    switch (gbinder_local_object_can_handle_transaction(obj, iface, tx.code)) {
    case LOOPER_TRANSACTION:
        reply = gbinder_local_object_handle_looper_transaction(...);
        break;
    case SUPPORTED:
        reply = handler
            ? gbinder_handler_transact(handler, obj, req, ...)
            : gbinder_local_object_handle_transaction(obj, req, ...);
        break;
    }

    // 7. 回复（非 oneway 事务）
    if (!(tx.flags & GBINDER_TX_FLAG_ONEWAY)) {
        if (reply)
            gbinder_driver_reply_data(self, reply->data);  // BC_REPLY
        else
            gbinder_driver_reply_status(self, txstatus);    // BC_REPLY + status

        // 等待回复被处理
        do {
            gbinder_driver_write_read(self, NULL, rbuf);
            txstatus = gbinder_driver_txstatus(self, context, NULL);
        } while (txstatus == (-EAGAIN));
    }
}
```

## 3.5 IO 抽象层 — gbinder_io.h / gbinder_io.c

### 设计模式：模板实例化

libgbinder 使用了一个巧妙的**模板模式**来生成 32 位和 64 位两套 IO 实现：

```c
// src/gbinder_io_32.c
#define BINDER_IPC_32BIT              // 定义 32 位
#define GBINDER_IO_PREFIX gbinder_io_32
#include "gbinder_io.c"               // 包含同一份源码！

// src/gbinder_io_64.c
#undef BINDER_IPC_32BIT               // 取消 32 位定义
#define GBINDER_IO_PREFIX gbinder_io_64
#include "gbinder_io.c"               // 包含同一份源码！
```

> **核心思想**：`gbinder_io.c` 是一份源码，通过 `BINDER_IPC_32BIT` 宏控制 `binder_uintptr_t` 是 4 字节还是 8 字节，通过 `GBINDER_IO_PREFIX` 控制函数命名。编译后生成两套完全独立的实现。

### GBinderIo 结构

```c
// src/gbinder_io.h
struct gbinder_io {
    int version;          // 7 (32位) 或 8 (64位)
    guint pointer_size;   // 4 (32位) 或 8 (64位)

    // BC 命令码（用户空间 → 内核）
    struct gbinder_io_command_codes bc;

    // BR 命令码（内核 → 用户空间）
    struct gbinder_io_return_codes br;

    // 对象大小计算
    gsize (*object_size)(const void* obj, const GBinderRpcProtocol* protocol);
    gsize (*object_data_size)(const void* obj);

    // 编码器（用户空间 → 内核格式）
    guint (*encode_pointer)(void* out, const void* pointer);
    guint (*encode_cookie)(void* out, guint64 cookie);
    guint (*encode_local_object)(void* out, GBinderLocalObject* obj, ...);
    guint (*encode_remote_object)(void* out, GBinderRemoteObject* obj);
    guint (*encode_fd_object)(void* out, int fd);
    guint (*encode_fda_object)(void* out, const GBinderFds* fds, ...);
    guint (*encode_buffer_object)(void* out, const void* data, gsize size, ...);
    guint (*encode_handle_cookie)(void* out, GBinderRemoteObject* obj);
    guint (*encode_ptr_cookie)(void* out, GBinderLocalObject* obj);
    guint (*encode_transaction)(void* out, guint32 handle, guint32 code, ...);
    guint (*encode_transaction_sg)(void* out, ...);
    guint (*encode_reply)(void* out, ...);
    guint (*encode_reply_sg)(void* out, ...);
    guint (*encode_status_reply)(void* out, gint32* status);

    // 解码器（内核格式 → 用户空间）
    void (*decode_transaction_data)(const void* data, GBinderIoTxData* tx);
    void* (*decode_ptr_cookie)(const void* data);
    guint (*decode_cookie)(const void* data, guint64* cookie);
    guint (*decode_binder_handle)(const void* obj, guint32* handle, ...);
    guint (*decode_binder_object)(const void* data, gsize size, ...);
    guint (*decode_buffer_object)(GBinderBuffer* buf, gsize offset, ...);
    guint (*decode_fd_object)(const void* data, gsize size, int* fd);

    // ioctl 封装
    int (*write_read)(int fd, GBinderIoBuf* write, GBinderIoBuf* read);
};

// 两个全局实例
extern const GBinderIo gbinder_io_32;
extern const GBinderIo gbinder_io_64;
```

### 32 位 vs 64 位的关键差异

| 特性 | 32 位 (io_32) | 64 位 (io_64) |
|------|---------------|---------------|
| `binder_uintptr_t` | `__u32` (4字节) | `__u64` (8字节) |
| `binder_size_t` | `__u32` (4字节) | `__u64` (8字节) |
| 协议版本 | 7 | 8 |
| 指针大小 | 4 | 8 |
| ioctl 码 | 基于 32 位结构体大小 | 基于 64 位结构体大小 |
| `flat_binder_object` | 16 字节 | 24 字节 |
| `binder_transaction_data` | 较小 | 较大 |

> **ioctl 码差异**：ioctl 码通过 `_IOW`/`_IOR`/`_IOWR` 宏生成，其中包含结构体大小。32 位和 64 位的结构体大小不同，因此 ioctl 码不同。这就是为什么需要两套 IO 实现。

### BC/BR 命令码

```c
// src/gbinder_io.c — 命令码初始化
.bc = {
    .transaction = BC_TRANSACTION,
    .reply = BC_REPLY,
    .acquire_result = BC_ACQUIRE_RESULT,
    .free_buffer = BC_FREE_BUFFER,
    .increfs = BC_INCREFS,
    .acquire = BC_ACQUIRE,
    .release = BC_RELEASE,
    .decrefs = BC_DECREFS,
    .increfs_done = BC_INCREFS_DONE,
    .acquire_done = BC_ACQUIRE_DONE,
    .register_looper = BC_REGISTER_LOOPER,
    .enter_looper = BC_ENTER_LOOPER,
    .exit_looper = BC_EXIT_LOOPER,
    .request_death_notification = BC_REQUEST_DEATH_NOTIFICATION,
    .clear_death_notification = BC_CLEAR_DEATH_NOTIFICATION,
    .dead_binder_done = BC_DEAD_BINDER_DONE,
    .transaction_sg = BC_TRANSACTION_SG,
    .reply_sg = BC_REPLY_SG
},

.br = {
    .error = BR_ERROR,
    .ok = BR_OK,
    .transaction = BR_TRANSACTION,
    .reply = BR_REPLY,
    .dead_reply = BR_DEAD_REPLY,
    .transaction_complete = BR_TRANSACTION_COMPLETE,
    .increfs = BR_INCREFS,
    .acquire = BR_ACQUIRE,
    .release = BR_RELEASE,
    .decrefs = BR_DECREFS,
    .noop = BR_NOOP,
    .spawn_looper = BR_SPAWN_LOOPER,
    .finished = BR_FINISHED,
    .dead_binder = BR_DEAD_BINDER,
    .failed_reply = BR_FAILED_REPLY
    // ...
}
```

> BC/BR 命令码本身在 32 位和 64 位之间是相同的（因为它们通过 `_IO`/`_IOW` 宏生成，参数类型是固定的 `__u32` 等），但涉及 `binder_uintptr_t` 的命令（如 `BC_TRANSACTION`）的 ioctl 码不同，因为 `binder_transaction_data` 结构体大小不同。

## 3.6 事务数据编解码

### GBinderIoTxData — 解码后的事务数据

```c
// src/gbinder_io.h
typedef struct gbinder_io_tx_data {
    int status;        // 事务状态
    guint32 code;      // 事务码
    guint32 flags;     // 事务标志
    pid_t pid;         // 发送方 PID
    uid_t euid;        // 发送方 EUID
    void* target;      // 目标（handle 或 ptr）
    void* data;        // 数据指针
    gsize size;         // 数据大小
    void** objects;     // 对象偏移数组
} GBinderIoTxData;
```

### encode_transaction — 编码事务

```c
// src/gbinder_io.c
static guint gbinder_io_XX_encode_transaction(
    void* out,           // 输出缓冲区
    guint32 handle,      // 目标句柄
    guint32 code,        // 事务码
    const GByteArray* data,  // 事务数据
    guint flags,         // 事务标志
    GUtilIntArray* offsets, // 对象偏移数组
    void** offsets_buf)  // 编码后的偏移缓冲区
{
    struct binder_transaction_data* tx = out + sizeof(guint32);
    // 填充 target.handle, code, flags, data_size, offsets_size
    // 编码 offsets 数组（32位用 __u32，64位用 __u64）
}
```

### offsets 数组的作用

事务数据中可能包含 binder 对象（flat_binder_object、fd_object 等）。`offsets` 数组记录这些对象在数据缓冲区中的位置，内核需要知道这些位置来处理引用计数。

```
事务数据布局:
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ int32    │ string16 │ flat_obj │ int32    │ fd_obj   │
│ (4 bytes)│ (N bytes)│ (16/24B) │ (4 bytes)│ (16B)    │
└──────────┴──────────┴──────────┴──────────┴──────────┘
                        ↑                      ↑
                        │                      │
              offsets[0] = 4+N          offsets[1] = 4+N+16/24+4
```

## 3.7 poll 与事件循环集成

```c
// src/gbinder_driver.c
int gbinder_driver_poll(GBinderDriver* self, struct pollfd* pipefd)
{
    struct pollfd fds[2];
    fds[0].fd = self->fd;
    fds[0].events = POLLIN | POLLERR | POLLHUP | POLLNVAL;

    if (pipefd) {
        fds[1] = *pipefd;  // 同时监听 pipe（用于唤醒）
    }

    err = poll(fds, n, -1);  // 无限等待
    return fds[0].revents;   // 返回 binder fd 的事件
}
```

> Driver 的 poll 被 IPC 层的 looper 线程调用，用于等待内核数据可读。同时可以监听一个 pipe fd，用于在需要退出 looper 时唤醒。

## 3.8 缓冲区管理

### free_buffer

```c
// src/gbinder_driver.c
void gbinder_driver_free_buffer(GBinderDriver* self, void* buffer)
{
    // 发送 BC_FREE_BUFFER 命令
    // 内核分配的事务数据缓冲区必须显式释放
    *cmd = io->bc.free_buffer;
    io->encode_pointer(wbuf + len, buffer);
    gbinder_driver_write(self, &write);
}
```

> **重要**：内核通过 mmap 分配的事务数据缓冲区不会自动释放，必须通过 `BC_FREE_BUFFER` 命令显式归还。libgbinder 通过 `GBinderBuffer` 管理这些缓冲区的生命周期。

### close_fds

```c
// src/gbinder_driver.c
void gbinder_driver_close_fds(GBinderDriver* self, void** objects, const void* end)
{
    // 遍历对象数组，关闭所有 FD 对象
    for (ptr = objects; *ptr; ptr++) {
        if (io->decode_fd_object(obj, size, &fd)) {
            close(fd);
        }
    }
}
```

## 3.9 Driver 生命周期

```
gbinder_driver_new()           gbinder_driver_ref()        gbinder_driver_unref()
      │                              │                           │
      v                              v                           v
┌──────────┐                   ┌──────────┐              ┌──────────┐
│ refcount=1│──────────────────>│refcount++│<─────────────│refcount--│
└──────────┘                   └──────────┘              └────┬─────┘
                                                             │
                                                    refcount == 0?
                                                    ├── 否 → 返回
                                                    └── 是 ↓
                                                gbinder_driver_close()
                                                  ├── munmap(vm)
                                                  └── close(fd)
                                                g_free(dev)
                                                g_slice_free(self)
```

## 3.10 完整事务数据流

```
发起事务（Client 端）:

GBinderClient          GBinderIpc           GBinderDriver          Kernel
     │                     │                     │                     │
     │ transact_sync_reply │                     │                     │
     ├────────────────────>│                     │                     │
     │                     │ gbinder_ipc_transact│                     │
     │                     ├────────────────────>│                     │
     │                     │                     │ encode_transaction  │
     │                     │                     │ (BC_TRANSACTION)     │
     │                     │                     │                     │
     │                     │                     │ write_read(ioctl)   │
     │                     │                     ├────────────────────>│
     │                     │                     │                     │
     │                     │                     │    BR_TRANSACTION_COMPLETE
     │                     │                     │<────────────────────┤
     │                     │                     │                     │
     │                     │                     │    BR_REPLY(data)   │
     │                     │                     │<────────────────────┤
     │                     │                     │                     │
     │                     │  RemoteReply        │                     │
     │                     │<────────────────────┤                     │
     │  RemoteReply         │                     │                     │
     │<────────────────────┤                     │                     │


接收事务（Server 端）:

Kernel              GBinderDriver          GBinderIpc           LocalObject
  │                      │                     │                     │
  │ BR_TRANSACTION(data) │                     │                     │
  ├─────────────────────>│                     │                     │
  │                      │ decode_transaction  │                     │
  │                      │ handle_transaction()│                     │
  │                      ├────────────────────>│                     │
  │                      │                     │ handler_transact()  │
  │                      │                     ├────────────────────>│
  │                      │                     │                     │
  │                      │                     │     LocalReply      │
  │                      │                     │<────────────────────┤
  │                      │ reply_data()        │                     │
  │                      │ (BC_REPLY)          │                     │
  │                      ├────────────────────>│                     │
  │  BC_REPLY             │                     │                     │
  │<─────────────────────┤                     │                     │
```

## 3.11 小结

| 概念 | 要点 |
|------|------|
| GBinderDriver | 内核 Binder 驱动的封装层 |
| 初始化 | open → BINDER_VERSION → mmap → SET_MAX_THREADS |
| 32/64 位检测 | BINDER_VERSION 返回 7(32位) 或 8(64位) |
| 模板模式 | gbinder_io.c 被包含两次，生成 io_32 和 io_64 |
| 核心 ioctl | BINDER_WRITE_READ 同时发送 BC 和接收 BR |
| offsets 数组 | 标记事务数据中 binder 对象的位置 |
| mmap | ~1MB 内核缓冲区映射，实现一次拷贝 |
| free_buffer | 内核缓冲区必须显式释放（BC_FREE_BUFFER） |
| BR_TRANSACTION | 入站事务，解码后分发给 LocalObject |
| BC_REPLY | 回复事务，将 LocalReply 数据发回内核 |
| poll | 用于 looper 线程等待数据可读 |

---

**上一阶段**：[02-architecture-and-config.md](02-architecture-and-config.md)
**下一阶段**：[04-ipc-core.md](04-ipc-core.md) — IPC 核心层
