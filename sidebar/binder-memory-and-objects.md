# Binder 通讯中的内存分布与对象生命周期

> 本文结合 `aidl-hidl-protocol-explained.md` 中的 binder_transaction_data 结构，说明 Binder 通讯过程中各数据存放在哪些内存区域，以及 LocalObject 与 handle 的对照关系。

## 1. 总览图：数据都在哪里？

```
┌─── Client 进程内存 ───┐   ┌─── 内核内存 ──────────────────────┐   ┌─── Server 进程内存 ───┐
│                       │   │                                    │   │                       │
│  Writer 构建请求:     │   │  binder_transaction_data (信封):    │   │  LocalObject          │
│  ┌─────────────────┐  │   │  ┌──────────────────────────────┐  │   │  ┌─────────────────┐  │
│  │ RPC Header      │  │   │  │ handle   │ code              │  │   │  │ 业务处理函数    │  │
│  │ + 用户数据      │──┼──→│  │ pid/euid │ data_size         │  │   │  │ (cookie 指向)   │  │
│  └─────────────────┘  │   │  │ data.ptr.buffer ──┐           │  │   │  └────────┬────────┘  │
│                       │   │  └───────────────────┼───────────┘  │   │           │            │
│  handle=1 (整数引用)  │   │                      │              │   │           │            │
│                       │   │  mmap 共享内存区 ←────┘              │   │           │            │
│  ioctl(BINDER_WRITE   │   │  ┌──────────────────────────────┐  │   │  mmap 映射 ←┘            │
│   _READ, &bwr)        │   │  │ RPC Header (flags/work/SYST)  │  │   │  (直接读取共享内存)       │
│                       │   │  │ + 用户数据 (payload)          │  │   │                           │
│                       │   │  └──────────────────────────────┘  │   │  Reader 读取数据:         │
│                       │   │                                    │   │  ┌─────────────────┐       │
│                       │   │  binder_node (内核对象):           │   │  │ 从 mmap 区读取  │       │
│                       │   │  ┌──────────────────────────────┐  │   │  │ RPC Header       │       │
│                       │   │  │ proc → Server 进程           │  │   │  │ + 用户数据       │       │
│                       │   │  │ cookie → LocalObject 地址    │──┼───→│  └─────────────────┘       │
│                       │   │  └──────────────────────────────┘  │   │                           │
└───────────────────────┘   └────────────────────────────────────┘   └───────────────────────────┘
```

**三个内存区域，各司其职**：
- **Client 内存**：Writer 构建请求数据，持有 handle 整数引用
- **内核内存**：信封（binder_transaction_data）、共享内存区（数据缓冲区）、binder_node（路由表）
- **Server 内存**：LocalObject（业务对象），通过 mmap 读取共享内存中的数据

## 2. 结合 binder_transaction_data 的详细说明

### 2.1 binder_transaction_data（信封）— 在内核内存中

```
binder_transaction_data 结构体:
┌──────────────┬──────────────────────────────────────────┐
│ 字段          │ 说明                                     │
├──────────────┼──────────────────────────────────────────┤
│ target.handle│ 目标对象句柄（类似收件人地址）             │
│ code         │ 事务码（类似命令码）                       │
│ flags        │ 事务标志（如 TF_ONE_WAY）                 │
│ sender_pid   │ 发送方 PID（内核注入，不可伪造）           │
│ sender_euid  │ 发送方 EUID（内核注入，不可伪造）          │
│ data_size    │ 数据缓冲区大小                            │
│ offsets_size │ 对象偏移表大小                            │
│ data.ptr.buffer ──→ 指向数据缓冲区（RPC Header + 用户数据）
│ data.ptr.offsets → 指向对象偏移表                       │
└──────────────┴──────────────────────────────────────────┘
```

**存放位置**：内核内存。通过 `ioctl(BINDER_WRITE_READ)` 从用户空间拷贝到内核。

**不放共享内存的原因**：
- 内核需要读取 handle 来路由事务
- 内核需要写入 sender_pid/sender_euid（注入调用者身份）
- 数据量很小（几十字节），拷贝开销可忽略

### 2.2 数据缓冲区（RPC Header + 用户数据）— 在内核的 mmap 共享内存中

```
data.ptr.buffer 指向的数据:
┌──────────────────────────────────────────────────┐
│ RPC Header                                        │ ← 协议头（aidl 或 hidl 格式）
│   ├─ flags (int32)                               │
│   ├─ work_source (int32)                         │
│   ├─ SYST (int32, aidl3+)                        │
│   └─ interface_name (string16/string8)           │
├──────────────────────────────────────────────────┤
│ 用户数据 (payload)                                │ ← 业务数据
│   ├─ 参数1                                        │
│   ├─ 参数2                                        │
│   └─ ...                                          │
└──────────────────────────────────────────────────┘
```

**存放位置**：内核的 mmap 共享内存区。

**这是"一次拷贝"的核心**：
- Client 的 Writer 在用户空间构建数据缓冲区
- 内核把数据缓冲区**拷贝一次**到 Server 的 mmap 共享内存区
- Server 通过 mmap 映射直接读取，**不需要第二次拷贝**

### 2.3 对象偏移表（offsets）— 也在 mmap 共享内存中

```
data.ptr.offsets 指向的偏移表:
┌──────────┬──────────┬──────────┐
│ offset_0 │ offset_1 │ ...      │
└──────────┴──────────┴──────────┘
每个 offset 指向数据缓冲区中 flat_binder_object 的位置
```

**存放位置**：和数据缓冲区一起在 mmap 共享内存中。

**用途**：内核需要知道数据中哪些位置包含 binder 对象（flat_binder_object），以便做引用计数和类型翻译。

### 2.4 各数据位置汇总

| 数据 | 存放位置 | 传输方式 | 谁读取 |
|------|---------|---------|--------|
| binder_transaction_data（信封） | 内核内存 | ioctl 拷贝 | 内核（路由+注入身份） |
| 数据缓冲区（Header+payload） | 内核 mmap 共享内存 | 内核拷贝（一次拷贝） | Server 通过 mmap 读取 |
| offsets（对象偏移表） | 内核 mmap 共享内存 | 和数据缓冲区一起 | 内核（引用计数） |
| BC/BR 命令序列 | 内核内存 | ioctl 拷贝 | 内核（命令处理） |
| LocalObject | Server 用户内存 | 不传输 | Server 自己（通过 cookie 定位） |
| binder_node | 内核内存 | 内核创建 | 内核（路由表） |

## 3. 一次拷贝原理

### 3.1 对比传统 IPC（如管道/socket）

```
传统 IPC（两次拷贝）:
  Client 用户空间 → 内核缓冲区（拷贝1）→ Server 用户空间（拷贝2）

Binder（一次拷贝）:
  Client 用户空间 → 内核 mmap 共享内存区（拷贝1）→ Server 通过 mmap 直接读（0次拷贝）
```

### 3.2 mmap 的作用

```
Server 启动时:
  fd = open("/dev/binder")
  mmap(NULL, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE, fd, 0)
  → Server 的虚拟地址空间映射到 binder 设备的内核内存

Client 发送事务时:
  内核把 Client 的数据缓冲区拷贝到 Server 的 mmap 区
  → 这块物理内存同时映射在:
     - 内核的地址空间（内核可写）
     - Server 的地址空间（Server 可读）
  → Server 直接读，不需要再拷贝
```

### 3.3 一次拷贝的完整流程

```
1. Client: Writer 在用户空间构建 [RPC Header + 用户数据]
2. Client: ioctl(BINDER_WRITE_READ)
   → binder_transaction_data（信封）从用户空间拷贝到内核栈
   → 信封中的 data.ptr.buffer 指向用户空间的数据缓冲区
3. 内核: 读取信封中的 handle → 查 binder_node → 找到 Server 进程
4. 内核: 把 data.ptr.buffer 指向的数据缓冲区拷贝到 Server 的 mmap 共享内存区
   ← 这就是"一次拷贝"！
5. 内核: 唤醒 Server 的 Looper 线程
6. Server: Looper 收到 BR_TRANSACTION
   → 事件中包含 data.ptr（指向 mmap 共享内存区）
   → Server 通过 data.ptr 直接读共享内存（0次拷贝）
   → Reader 解析 RPC Header + 用户数据
```

### 3.4 mmap 共享内存是谁分配的？

**mmap 由接收方发起。** 每个打开 `/dev/binder` 的进程都会通过 mmap 分配自己的共享内存区，数据由内核拷贝到接收方的 mmap 区。

```
谁接收数据，就用谁的 mmap 区:

Client 发事务 → Server 接收 → 内核拷贝到 Server 的 mmap 区
Server 回复   → Client 接收 → 内核拷贝到 Client 的 mmap 区
```

#### 物理内存在哪里？

虽然 mmap 是接收方发起的，但物理页属于内核：

```
Server 用户空间          内核                    物理内存
  │                       │                       │
  │ mmap(NULL, size, ...) │                       │
  ├──────────────────────>│  分配内核物理页         │
  │                       ├──────────────────────>│
  │                       │  建立页表映射           │
  │  虚拟地址 0x7f000000  │  内核地址 0xffff8000   │  物理页 0x10000
  │       ↕               │       ↕               │
  │  Server 可读           │  内核可写              │  同一块物理内存
```

- **接收方可读**：通过 mmap 映射，接收方用户空间可以直接读
- **内核可写**：内核把发送方的数据拷贝到这块物理页
- **接收方不可写**：mmap 时用的是 `PROT_READ`（只读），接收方不能修改

#### 释放也是接收方发起的

```
接收方处理完事务后:
  BC_FREE_BUFFER(data_ptr)
  → 告诉内核：这块 mmap 内存我用完了，你可以回收了
  → 内核回收物理页
```

#### TCP/IP 类比

```
TCP/IP:
  Server: socket() + bind() + listen() → Server 分配接收缓冲区
  Client: connect() + send() → 数据到达 Server 的 socket 接收缓冲区
  Server 的接收缓冲区是 Server 创建的，但数据是 Client 发的

Binder:
  Server: open("/dev/binder") + mmap() → Server 分配 binder 共享内存区
  Client: ioctl(BINDER_WRITE_READ) → 数据被内核拷贝到 Server 的 mmap 区
  Server 的 mmap 区是 Server 创建的，但数据是 Client 发的
```

#### 总结

| 问题 | 答案 |
|------|------|
| mmap 谁发起的？ | 接收方（Server 接收请求时是 Server，Client 接收回复时是 Client） |
| 物理内存在哪？ | 内核空间（binder 设备的内核内存） |
| 数据谁写的？ | 内核（从发送方用户空间拷贝过来） |
| 接收方能读吗？ | 能，通过 mmap 映射直接读 |
| 接收方能写吗？ | 不能，mmap 时是 PROT_READ 只读 |
| 谁释放？ | 接收方处理完后用 BC_FREE_BUFFER 通知内核释放 |

> 一句话：**mmap 是接收方发起的，物理页在内核中，数据由内核从发送方拷贝过来，接收方只读，用完后通知内核回收。** 这就是 Binder "一次拷贝"的实现方式。

## 4. LocalObject 与 handle 的对照关系

### 4.1 注册阶段：LocalObject → handle

```
Server 进程                              内核                          ServiceManager 进程

1. 创建 LocalObject
   localObject = gbinder_local_object_new(...)
   地址: 0x7fff1234 (Server 虚拟地址)

2. addService("audio", localObject)
   → Writer 把 localObject 打包成 flat_binder_object:
     { type=BINDER_TYPE_BINDER, cookie=0x7fff1234 }
   → BC_TRANSACTION(handle=0, code=ADD_SERVICE, data)

3. 内核拦截!
   → 发现 data 中有 flat_binder_object
   → 创建 binder_node:
     { proc=Server进程, cookie=0x7fff1234 }
   → 把 flat_binder_object 翻译为:
     { type=BINDER_TYPE_HANDLE, handle=1 }
   → 传给 ServiceManager
                                                    4. ServiceManager 收到
                                                       → flat_binder_object: type=HANDLE, handle=1
                                                       → 记录: "audio" → handle=1
                                                       → SM 不知道 0x7fff1234，只知道 handle=1
```

### 4.2 查询阶段：名字 → handle

```
Client 进程                              内核                          ServiceManager 进程

1. getService("audio")
   → BC_TRANSACTION(handle=0, code=GET_SERVICE, data="audio")

                                                    2. ServiceManager 收到查询
                                                       → 查表: "audio" → handle=1
                                                       → 回复: flat_binder_object
                                                         { type=HANDLE, handle=1 }

3. 内核拦截回复!
   → 发现回复中有 flat_binder_object: handle=1
   → handle=1 → binder_node → Client 进程还没有这个引用
   → 为 Client 创建 binder_ref:
     { proc=Client进程, node=binder_node, desc=1 }
   → 把 flat_binder_object 传给 Client
   → Client 得到 handle=1
```

### 4.3 调用阶段：handle → LocalObject

```
Client 进程                              内核                          Server 进程

1. transact(handle=1, code, data)
   → BC_TRANSACTION(handle=1, code, data)

2. 内核路由!
   → handle=1 → binder_ref → binder_node
   → binder_node: { proc=Server进程, cookie=0x7fff1234 }
   → 把数据缓冲区拷贝到 Server 的 mmap 区
   → 唤醒 Server Looper
   → BR_TRANSACTION 事件中带上 cookie=0x7fff1234

                                                    3. Server Looper 收到
                                                       → 事件中有 cookie=0x7fff1234
                                                       → 📌Server 自己用 cookie 找到 LocalObject
                                                       → 📌Server 自己调用 LocalObject->handler()
                                                       → 处理完毕，写 Reply
```

### 4.4 对照关系总结

```
Server 视角:     LocalObject (0x7fff1234)  ← Server 直接持有
                        ↕ 内核翻译
内核视角:        binder_node { proc=Server, cookie=0x7fff1234 }
                        ↕ 内核分配
SM/Client 视角:  handle=1  ← 只是一个整数
```

| 角色 | 感知 LocalObject 吗？ | 感知什么？ |
|------|---------------------|-----------|
| Server 自己 | ✅ 直接持有 | LocalObject 指针（0x7fff1234） |
| 内核 | ✅ 通过 binder_node | Server 进程 + cookie（LocalObject 指针） |
| ServiceManager | ❌ 不感知 | 只有一个 handle 整数 |
| Client | ❌ 不感知 | 只有一个 handle 整数 |

## 5. LocalObject 不归内核管

### 5.1 内核对 cookie 的操作只有三个

```
1. 存储:  注册时把 cookie 存到 binder_node 中
2. 查找:  handle → binder_node → 取出 cookie
3. 传递:  把 cookie 放到 BR_TRANSACTION 事件中传回给 Server
```

**内核从不解引用 cookie**（不把它当指针用），它只是搬运这个数字。内核不知道 LocalObject 的结构、方法、状态，也不关心。

### 5.2 LocalObject 的生命周期完全由 Server 控制

```
创建:  Server 调用 gbinder_local_object_new() → 在 Server 用户空间分配
注册:  Server 调用 addService() → 内核记录 cookie
使用:  事务到来 → 内核传回 cookie → Server 自己调用 LocalObject
销毁:  Server 调用 gbinder_local_object_unref() → 引用计数归零 → Server 用户空间释放
```

内核在整个过程中：
- ❌ 不参与 LocalObject 的创建
- ❌ 不参与 LocalObject 的方法调用
- ❌ 不参与 LocalObject 的销毁
- ✅ 只记录和搬运 cookie 这个数字

## 6. 内存风险与崩溃

### 6.1 use-after-free（过早释放）

```
Server: localObject = gbinder_local_object_new(...)     ← 创建，地址 0x7fff1234
Server: addService("audio", localObject)                ← 注册，内核记录 cookie=0x7fff1234
Server: gbinder_local_object_unref(localObject)          ← 引用计数归零，内存释放！
Client: transact(handle=1, ...)                          ← 事务到来
内核:   BR_TRANSACTION(cookie=0x7fff1234)                ← 传回已释放的地址
Server: 通过 0x7fff1234 访问 LocalObject                 ← 野指针 → 崩溃 💥
```

**libgbinder 的防护**：通过引用计数防止过早释放。`addService` 内部会 `ref` LocalObject，只要服务还在注册状态，对象不会被释放。

### 6.2 内存踩踏

```
Server: localObject 在 0x7fff1234
Server: 某处 buffer overflow 覆盖了 0x7fff1234 附近的内存
Client: transact(handle=1, ...)                          ← 事务到来
内核:   BR_TRANSACTION(cookie=0x7fff1234)                ← 传回地址
Server: 通过 0x7fff1234 访问 LocalObject                 ← 对象被破坏 → 崩溃或行为异常 💥
```

**无法防护**：引用计数只能防止"过早释放"，不能防止"内存踩踏"。如果代码有 buffer overflow 把 LocalObject 的内存踩坏了，内核和 libgbinder 都无能为力。

### 6.3 为什么不把 LocalObject 放内核？

| 如果 LocalObject 在内核 | 问题 |
|------------------------|------|
| 内核检查对象有效性 | 每次调用都要系统调用（用户态→内核态切换） |
| 业务逻辑在内核执行 | 安全风险（内核 bug = 系统崩溃） |
| 内核知道每种接口逻辑 | 违反分层设计（内核不应包含业务逻辑） |

**Binder 的设计选择**：LocalObject 放用户空间，内核只做路由（"邮局"）。Server 进程的 bug 只影响自己，不会搞崩内核。

### 6.4 崩溃影响范围

```
LocalObject 崩溃:
  → Server 进程崩溃（SIGSEGV）
  → 内核检测到 Server 进程死亡
  → 内核清理 binder_node（引用计数归零）
  → Client 收到死亡通知（DEATH_NOTIFICATION）
  → Client 可以重新查询或报错
  → 内核和其他进程不受影响
```

**隔离性**：Server 崩溃只影响自己。内核和其他进程通过死亡通知感知，不会跟着崩溃。这是 LocalObject 放用户空间的安全优势。
