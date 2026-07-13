# AIDL vs HIDL 协议：从 TCP/IP 视角理解 Binder 报文

> 本文面向有 TCP/IP Socket 开发经验的工程师，用你熟悉的概念来类比理解 Binder 的 AIDL/HIDL 协议差异。

## 1. 你已经知道的：自定义 TCP 协议

在 TCP Socket 开发中，你一定做过类似的事情：

```
TCP 是字节流，没有边界。你需要自己定义"报文"的格式：

┌──────────┬──────────┬──────────┬──────────────────┐
│ magic    │ length   │ cmd      │ payload          │
│ 2 bytes  │ 4 bytes  │ 2 bytes  │ length bytes     │
└──────────┴──────────┴──────────┴──────────────────┘
```

- **magic**：魔数，用于识别这是不是一个合法报文
- **length**：告诉接收方这个报文有多长，解决 TCP 粘包问题
- **cmd**：命令码，告诉接收方要做什么操作
- **payload**：业务数据

发送方按这个格式打包，接收方按这个格式拆包。**协议头变了，收发双方必须同时升级**。

## 2. Binder 事务就是一次"TCP 报文"

Binder 的一次事务（Transaction）本质上就是一次请求-回复，和你在 TCP 上发一个请求包、收一个回复包是一回事：

```
TCP Socket 通讯:
  Client → [请求报文] → Server
  Client ← [回复报文] ← Server

Binder 通讯:
  Client → BC_TRANSACTION(数据) → Server
  Client ← BR_REPLY(数据)       ← Server
```

区别只在于：
- TCP 你自己管 `send()`/`recv()` 和粘包拆包
- Binder 内核驱动帮你管了"粘包"问题——每次 `BR_TRANSACTION` 就是一个完整的"报文"，你不需要自己找边界

## 3. Binder 报文的"协议头"长什么样

### 3.1 对应关系

```
你的 TCP 自定义协议（全部在一个字节流里）:
┌──────────┬──────────┬──────────┬──────────────────┐
│ magic    │ length   │ cmd      │ payload          │
└──────────┴──────────┴──────────┴──────────────────┘

Binder 事务（信封和数据缓冲区分开）:
┌─ binder_transaction_data（内核结构体 = "信封"）──┐
│ handle   │ code     │ pid/euid │ data_size       │
│ (收件人)  │ (命令码)  │ (寄件人) │ (数据长度)       │
└──────────┴──────────┴──────────┬────────────────┘
                                  │ data.ptr.buffer
                                  v
                  ┌──────────────────────────────┐
                  │ 数据缓冲区（= "信纸"）        │
                  │ ┌──────────────────────────┐ │
                  │ │ RPC Header（协议头）      │ │
                  │ ├──────────────────────────┤ │
                  │ │ 用户数据（payload）       │ │
                  │ └──────────────────────────┘ │
                  └──────────────────────────────┘

其中 RPC Header 的内容取决于你用 AIDL 还是 HIDL。
code 不在数据缓冲区里，它在"信封"（transaction_data）中。
```

> **关于 length、magic 和 code 的说明**：在你的 TCP 协议中，`length`、`magic`、`cmd` 都是你自己写在报文里的，接收方从字节流中解析出来。Binder 的做法不同——这些"信封信息"由内核的 `binder_transaction_data` 结构体单独承载，**不在用户数据缓冲区里**。
>
> 具体来说，`binder_transaction_data` 结构体包含：
> - `target.handle`：目标对象句柄（类似收件人地址）
> - **`code`**：事务码（类似你的 cmd 命令码）
> - `sender_pid` / `sender_euid`：发送方身份（类似寄件人地址，由内核填写，不可伪造）
> - `data_size`：数据长度（类似你的 length 字段）
> - `data.ptr.buffer`：指向数据缓冲区的指针
>
> 而 `data.ptr.buffer` 指向的数据缓冲区，才是 Writer 写入 / Reader 读取的内容，其结构为 `RPC Header + 用户数据`。
>
> **所以"剥壳"后的层次关系是**：
>
> ```
> 内核给你的:
>   binder_transaction_data {
>       handle, code, flags, sender_pid, sender_euid, data_size,
>       data.ptr.buffer ──→ [RPC Header + 用户数据]
>   }
>
> 你实际面对的:
>   code（从 transaction_data 取出）+ 数据缓冲区（RPC Header + 用户数据）
>
> libgbinder 帮你剥掉 RPC Header 后:
>   code + 用户数据（通过 Reader 读取）
> ```
>
> 对比你的 TCP 协议：
>
> | TCP 自定义协议 | Binder |
> |---------------|--------|
> | magic（在报文里） | 无对应（内核帮你校验合法性） |
> | length（在报文里） | data_size（在 transaction_data 里，不在数据缓冲区里） |
> | cmd（在报文里） | code（在 transaction_data 里，不在数据缓冲区里） |
> | payload（在报文里） | 数据缓冲区 = RPC Header + 用户数据 |
>
> 简单说：你的 TCP 协议把所有东西都塞在一个字节流里；Binder 把"信封信息"（handle、code、pid 等）和"信纸内容"（RPC Header + 用户数据）分开放，信封信息由内核结构体承载，信纸内容在数据缓冲区里。

### 3.1.1 RPC Header 要么是 AIDL 格式，要么是 HIDL 格式

RPC Header 不是一种固定格式——**AIDL 和 HIDL 是 RPC Header 的两种不同格式，二选一，不会混用**。

由配置文件按设备决定用哪种格式：

```ini
# /etc/gbinder.conf
[Protocol]
/dev/binder = aidl3      # → 这个设备上的 RPC Header 用 AIDL 格式
/dev/hwbinder = hidl     # → 这个设备上的 RPC Header 用 HIDL 格式
```

用你的 TCP 经验来类比：就像你有两套协议头格式，配置文件决定用哪一套：

```
你的配置文件:
server_protocol = V3     # → 用 V3 格式的协议头

gbinder.conf:
/dev/binder = aidl3      # → 用 AIDL 格式的 RPC Header
```

这个选择是**按设备绑定**的：
- 打开 `/dev/binder` → 自动用 AIDL 格式的 RPC Header
- 打开 `/dev/hwbinder` → 自动用 HIDL 格式的 RPC Header

libgbinder 在创建 Client 时，根据设备名查配置，拿到对应的 `GBinderRpcProtocol`，之后每次构建请求都调用 `protocol->write_rpc_header()` 写入对应格式的头。收发双方必须用同一种格式，否则解析会出错。

```
RPC Header 是一个概念（协议头）
├── AIDL 格式的 RPC Header（flags + work_source + SYST + interface_name）
│   ├── aidl   （最早期格式）
│   ├── aidl2  （+work_source）
│   ├── aidl3  （+SYST）
│   └── aidl4  （stability 格式变更）
└── HIDL 格式的 RPC Header（仅 interface_name）
    └── hidl   （唯一版本，从未变过）
```

### 3.2 AIDL 的协议头

AIDL 的协议头就像你的 TCP 协议头一样，**随着版本演进不断加字段**：

#### aidl 版本（最早期，类似你的协议 V1）

```
你的 TCP V1:
┌──────────┬──────────┐
│ magic    │ cmd      │
└──────────┴──────────┘

AIDL V1:
┌──────────┬──────────────────────┐
│ flags   │ interface_name       │
│ (int32) │ (string16)           │
└──────────┴──────────────────────┘
```

- `flags`：类似你的 magic，标识这是严格模式事务
- `interface_name`：类似你的 cmd，告诉接收方"我要调用哪个接口"

#### aidl2 版本（Android 9，类似你的协议 V2 加字段）

```
你的 TCP V2:
┌──────────┬──────────┬──────────┐
│ magic    │ version  │ cmd      │  ← 加了 version 字段
└──────────┴──────────┴──────────┘

AIDL V2:
┌──────────┬──────────┬──────────────────────┐
│ flags   │ work_src │ interface_name       │  ← 加了 work_source 字段
│ (int32) │ (int32)  │ (string16)           │
└──────────┴──────────┴──────────────────────┘
```

就像你在 V2 协议里加了 `version` 字段一样，Google 在 aidl2 里加了 `work_source`（用于追踪是谁发起的操作，类似"请求来源标记"）。

#### aidl3 版本（Android 11，又加字段）

```
你的 TCP V3:
┌──────────┬──────────┬──────────┬──────────┐
│ magic    │ version  │ seq      │ cmd      │  ← 又加了 seq
└──────────┴──────────┴──────────┴──────────┘

AIDL V3:
┌──────────┬──────────┬──────────┬──────────────────────┐
│ flags   │ work_src │ SYST     │ interface_name       │  ← 加了 SYST 标识
│ (int32) │ (int32)  │ (int32)  │ (string16)           │
└──────────┴──────────┴──────────┴──────────────────────┘
```

`SYST` 是一个 fourcc（'S','Y','S','T'），类似你的 magic 魔数，标记这是"系统域"事务。

### 3.3 HIDL 的协议头

HIDL 的协议头**极其简单，而且从未变过**：

```
你的 TCP 协议（如果设计得好，可能一直没变）:
┌──────────┬──────────┐
│ magic    │ cmd      │  ← 从 V1 到现在都是这个格式
└──────────┴──────────┘

HIDL:
┌──────────────────────┐
│ interface_name       │  ← 就这一个字段，从来没变过
│ (string8)            │
└──────────────────────┘
```

为什么 HIDL 不需要加字段？因为 HIDL 把"版本"放在了**接口名里**：

```
HIDL 接口名: android.hidl.foo@1.0::IFoo
                              ^^^
                         版本号在这里，不在协议头里
```

这就像你的 TCP 协议如果这样设计：

```
你的 TCP 协议（版本在 cmd 里）:
┌──────────┬───────────────────┐
│ magic    │ cmd = "FOO_V1"    │  ← 版本在命令名里
└──────────┴───────────────────┘

升级时:
┌──────────┬───────────────────┐
│ magic    │ cmd = "FOO_V2"    │  ← 新命令名，但协议头格式没变
└──────────┴───────────────────┘
```

**协议头不变，版本在 payload 里**——这就是 HIDL 的策略。

## 4. AIDL vs HIDL：两种协议演进策略

用你的 TCP 开发经验来理解：

### AIDL 策略：改协议头

```
就像你的 TCP 协议每次升级都加字段:

V1: | magic | cmd |
V2: | magic | version | cmd |        ← 加字段
V3: | magic | version | seq | cmd |  ← 再加字段
V4: | magic | version | seq | flag | cmd |  ← 还加
```

**优点**：协议头里信息丰富，接收方解析头就能知道很多元信息
**缺点**：收发双方必须版本匹配，否则解析出错

### HIDL 策略：协议头不变，版本在数据里

```
就像你的 TCP 协议头永远不变:

V1: | magic | cmd="FOO_V1" | payload |
V2: | magic | cmd="FOO_V2" | payload |  ← 头没变，cmd 变了
V3: | magic | cmd="FOO_V3" | payload |  ← 头还是没变
```

**优点**：协议头永远不变，解析器代码不用改
**缺点**：协议头信息少，所有元信息都要从 payload 里解析

### 对比表

| 特性 | AIDL（像你的 V1→V2→V3） | HIDL（像你的固定头协议） |
|------|--------------------------|------------------------|
| 协议头 | 随版本变 | 永远不变 |
| 版本信息 | 隐含在协议头格式中 | 显式在接口名中（@1.0） |
| 收发方版本 | 必须匹配 | 可以不同（老客户端调新接口的旧版本） |
| 演进方式 | 加字段 | 加接口版本号 |
| 复杂度 | 高（5个 RPC 版本 + 6个 SM 版本） | 低（1个版本） |

## 5. string16 vs string8：编码差异

这就像你在 TCP 协议里选择用 UTF-8 还是 GBK 编码字符串：

### AIDL 用 string16（UTF-16）

```
你的 TCP 协议可能这样存字符串:
| int32: 字节数 | char[]: UTF-8 数据 | \0 |

AIDL 的 string16:
| int32: 字符数 | gunichar2[]: UTF-16 数据 | \0\0 |
                 ^^^^^^^^^^
                 每个字符 2 字节
```

为什么 AIDL 用 UTF-16？因为 Android 的 Java 层原生使用 UTF-16（`String` 内部就是 UTF-16），直接拷贝不需要转换。

### HIDL 用 string8（UTF-8）

```
HIDL 的 string8:
| int32: 字节数 | char[]: UTF-8 数据 | \0 |
```

HIDL 用 UTF-8 是因为 HAL 层是 C/C++ 代码，天然使用 UTF-8。

> **类比**：就像你的 TCP 服务器是 Java 写的用 UTF-16，C 写的用 UTF-8——选择取决于实现语言，不是协议优劣。

## 6. 完整的 Binder "报文"结构

把 Binder 事务数据和你熟悉的 TCP 报文做个完整对比：

```
你的 TCP 自定义协议报文（全部在一个字节流里）:
┌──────────┬──────────┬──────────┬──────────────────┐
│ magic    │ length   │ cmd      │ payload          │
│ 2B       │ 4B       │ 2B       │ N bytes          │
└──────────┴──────────┴──────────┴──────────────────┘
              ↑          ↑          ↑
              全部在同一个字节流里，接收方自己解析

Binder AIDL 事务（信封和数据缓冲区分开）:

  binder_transaction_data（内核结构体 = "信封"）
  ┌──────────┬──────────┬──────────┬──────────┐
  │ handle   │ code     │ pid/euid │ data_size│
  │ (收件人)  │ (命令码)  │ (寄件人)  │ (长度)   │
  └──────────┴──────────┴──────────┬──────────┘
                                    │ data.ptr.buffer
                                    v
  数据缓冲区（= "信纸"，Writer 写入 / Reader 读取的内容）
  ┌──────────┬──────────┬──────────┬──────────────────┐
  │ flags    │ work_src │ SYST     │ interface_name   │ ← RPC Header
  │ 4B       │ 4B       │ 4B       │ 变长 string16     │
  ├──────────┴──────────┴──────────┴──────────────────┤
  │ 用户数据（payload）                                │ ← 业务数据
  │ N bytes                                            │
  └───────────────────────────────────────────────────┘

  注意：code 不在数据缓冲区里！它在"信封"中。
        数据缓冲区 = RPC Header + 用户数据

Binder HIDL 事务（同样信封和数据分开）:

  binder_transaction_data（内核结构体 = "信封"）
  ┌──────────┬──────────┬──────────┬──────────┐
  │ handle   │ code     │ pid/euid │ data_size│
  └──────────┴──────────┴──────────┬──────────┘
                                    │ data.ptr.buffer
                                    v
  数据缓冲区（= "信纸"）
  ┌──────────────────────────────────────────────────┐
  │ interface_name                                    │ ← RPC Header
  │ 变长 string8                                      │
  ├──────────────────────────────────────────────────┤
  │ 用户数据（payload）                                │ ← 业务数据
  │ N bytes                                            │
  └───────────────────────────────────────────────────┘
```

> **关键区别**：你的 TCP 协议把 magic、length、cmd、payload 全部塞在一个字节流里；Binder 把 handle、code、pid 等放在内核结构体（信封）里，数据缓冲区（信纸）里只有 RPC Header + 用户数据。code 不在数据缓冲区里。

## 7. 为什么 libgbinder 需要配置文件

回到你的 TCP 开发经验：如果你的协议有 V1/V2/V3 三个版本，你的客户端代码需要知道连的是哪个版本的服务器，才能用正确的格式打包。

```
你的 TCP 客户端:
// 连接前先协商版本
send(sock, "VERSION?", 8, 0);
recv(sock, buf, 8, 0);
if (strcmp(buf, "V3") == 0) {
    // 用 V3 格式打包
} else if (strcmp(buf, "V2") == 0) {
    // 用 V2 格式打包
}
```

libgbinder 面临同样的问题：不同的 Android 设备用不同版本的 AIDL 协议。但 Binder 没有"版本协商"机制（不像 TCP 的握手），所以需要**配置文件**预先告诉 libgbinder 用哪个版本：

```ini
# /etc/gbinder.conf — 相当于你的 "版本配置"
[Protocol]
/dev/binder = aidl3      # "这个设备用 V3 协议"
/dev/hwbinder = hidl     # "这个设备用 HIDL 协议"
```

> **类比**：就像你在部署 TCP 客户端时，在配置文件里写 `server_version=V3`，而不是运行时动态协商。

## 8. 一句话总结

| 你的 TCP 经验 | Binder 对应 |
|--------------|-------------|
| 自定义协议头 | RPC Header（aidl/hidl 格式不同） |
| magic 魔数 | flags / SYST fourcc |
| cmd 命令码 | code（事务码） |
| payload 业务数据 | 用户数据（Writer/Reader 读写） |
| 协议 V1→V2 加字段 | aidl→aidl2→aidl3 加字段 |
| 版本在 cmd 名里 | HIDL 版本在接口名里（@1.0） |
| UTF-8 vs GBK | string16(UTF-16) vs string8(UTF-8) |
| 配置文件指定服务器版本 | gbinder.conf 指定协议版本 |
| TCP 粘包要自己处理 | Binder 内核帮你处理了"粘包" |
| send()/recv() | ioctl(BINDER_WRITE_READ) |

> **核心**：Binder 事务 = 一次有协议头的请求-回复，和你在 TCP 上发自定义协议报文本质相同。AIDL 的版本演进就像你的协议加字段，HIDL 的版本在接口名里就像你的版本在 cmd 名里。配置文件告诉 libgbinder 用哪个"协议版本"打包，就像你的配置文件告诉客户端用 V2 还是 V3 格式。

## 9. 如何 dump 和观察报文数据

### 9.1 开启 verbose 日志

libgbinder 内置了报文 hexdump 功能，通过环境变量开启：

```bash
# 方式1：设置 gbinder 模块日志级别为 verbose（推荐）
export GBINDER_DEFAULT_LOG_LEVEL=4

# 方式2：通过 GLib 通用日志机制
export G_MESSAGES_DEBUG=all

# 方式3：通过 gutil 通用日志级别
export GUTIL_LOG_LEVEL=4
```

日志级别对照：

| 级别 | 值 | 说明 |
|------|---|------|
| inherit | 0 | 默认（继承上层设置） |
| error | 1 | 只输出错误 |
| warning | 2 | 输出警告 |
| info/debug | 3 | 输出调试信息 |
| **verbose** | **4** | **输出 hexdump（报文内容）** |

> 源码位置：`src/gbinder_log.c` 中的 `gbinder_log_init()` 读取 `GBINDER_DEFAULT_LOG_LEVEL` 环境变量。

### 9.2 dump 的源码位置

dump 功能在 `src/gbinder_driver.c` 中实现，受 `#if GUTIL_LOG_VERBOSE` 条件编译控制：

```c
// src/gbinder_driver.c

#if GUTIL_LOG_VERBOSE
// hexdump 任意内存区域（用 gutil_hexdump 格式化）
static void gbinder_driver_verbose_dump(const char mark, uintptr_t ptr, gsize len)
{
    char line[GUTIL_HEXDUMP_BUFSIZE];
    while (len > 0) {
        const guint dumped = gutil_hexdump(line, (void*)ptr, len);
        GVERBOSE("%s%s", prefix, line);  // 输出: "< 000000: 00 00 40 00 ..."
        len -= dumped;
        ptr += dumped;
    }
}

// dump 事务元信息（信封信息，不含数据内容）
static void gbinder_driver_verbose_transaction_data(const char* name, const GBinderIoTxData* tx)
{
    // 输出: "> BR_TRANSACTION 0x00000001 (48 bytes, 2 objects)"
    //       即: 事务名 + code + 数据大小 + 对象数量
}
#else
// 未开启 verbose 时，替换为空操作
#  define gbinder_driver_verbose_dump(x,y,z) GLOG_NOTHING
#  define gbinder_driver_verbose_transaction_data(x,y) GLOG_NOTHING
#endif
```

### 9.3 dump 的触发位置

这些 dump 函数在以下关键路径被调用：

**发送时（BC_TRANSACTION / BC_REPLY）**：
```
< BC_TRANSACTION 0x00000001 0x00000001    ← 命令 + handle + code
< [hexdump of offsets]                      ← 对象偏移表
< [hexdump of data bytes]                   ← 完整数据缓冲区（RPC Header + 用户数据）
```

**接收时（BR_TRANSACTION / BR_REPLY）**：
```
> BR_TRANSACTION 0x00000001 (48 bytes, 2 objects)  ← 事务元信息
  [hexdump of tx.data]                               ← 完整数据缓冲区
```

**ioctl 读写时**：
```
< [hexdump of write buffer]   ← 发给内核的原始数据
> [hexdump of read buffer]    ← 从内核读回的原始数据
```

### 9.4 dump 输出示例与解读

假设我们发起一个 AIDL（aidl3）事务，调用 `android.os.IServiceManager` 的 `getService` 方法，verbose 日志会输出类似这样的内容：

```
< BC_TRANSACTION 0x00000001 0x00000001
< 00000000: 00 00 40 00 ff ff ff ff 53 59 53 54 12 00 00 00  .....@..SYST....
< 00000010: 61 00 6e 00 64 00 72 00 6f 00 69 00 64 00 2e 00  a.n.d.r.o.i.d...
< 00000020: 6f 00 73 00 2e 00 49 00 53 00 65 00 72 00 76 00  o.s...I.S.e.r.v.
< 00000030: 69 00 63 00 65 00 4d 00 61 00 6e 00 61 00 67 00  i.c.e...M.a.n.a.g.
< 00000040: 65 00 72 00 00 00 00 00 0f 00 00 00 74 00 65 00  e.r.......t.e...
< 00000050: 73 00 74 00 00 00                                s.t...
```

逐字节解读：

```
RPC Header (aidl3 格式):
  00 00 40 00          → flags = 0x00400000 (STRICT_MODE_PENALTY_GATHER)
  ff ff ff ff          → work_source = -1 (UNSET_WORK_SOURCE)
  53 59 53 54          → SYST fourcc ('S','Y','S','T')
  12 00 00 00          → string16 长度 = 0x12 = 18 个字符
  61 00 6e 00 ...      → "android.os.IServiceManager" (UTF-16, 每字符2字节)
  72 00 00 00          → string16 结尾的 \0 + padding

用户数据 (payload):
  0f 00 00 00          → int32 = 15 (可能是参数1)
  74 00 65 00 73 00 ...→ string16 "test" (业务参数)
  00 00                → string16 结尾
```

### 9.5 如何阅读 hexdump

用你的 TCP 经验来理解：就像你用 Wireshark 抓包后看 hex view 一样，libgbinder 的 verbose dump 就是"Binder 版 Wireshark"，只不过输出的是文本格式的 hex。

阅读步骤：

1. **先看事务元信息行**：`> BR_TRANSACTION 0x00000001 (48 bytes, 2 objects)` 告诉你 code 和数据大小
2. **根据协议确定 Header 格式**：如果是 `/dev/binder`，用 AIDL 格式；如果是 `/dev/hwbinder`，用 HIDL 格式
3. **按协议头格式逐字段解析 hex**：
   - AIDL: 先读 4 字节 flags → 4 字节 work_source → (aidl3+) 4 字节 SYST → string16 interface_name
   - HIDL: 直接读 string8 interface_name
4. **Header 之后的就是用户数据**：根据你的业务协议解析

### 9.6 限制说明

| 限制 | 说明 |
|------|------|
| 只有原始 hex | 不会自动标注"这里是 flags，这里是 interface_name"，需手动对照协议格式解析 |
| 条件编译 | 受 `#if GUTIL_LOG_VERBOSE` 控制，编译时未开启则代码为空。默认编译通常包含 |
| 无公共 API | dump 函数都是 `static`，只能通过日志级别开启，不能编程式调用 |
| 性能影响 | verbose 级别会输出大量日志，生产环境不建议开启 |

> **类比**：就像你在 TCP 调试时用 `tcpdump -X` 看 hex 一样——libgbinder 的 verbose dump 给你的是原始字节，需要你自己对照协议头格式来理解每个字段的含义。如果你想要更结构化的报文 inspect（自动标注字段），需要自己写工具或修改源码添加。
