# libgbinder 学习笔记 — 总索引

> 本项目用于系统学习 libgbinder（GLib-style Android Binder IPC 库），结合源码分析与测试程序，彻底理解其总体框架、使用方法、跨平台特性。

## 项目结构

```
gbinder-note/
├── _doc/                          # 学习笔记（本文档所在目录）
├── _test/                         # 测试程序
└── libgbinder-master/             # libgbinder 源码
    ├── include/                   # 公共 API 头文件
    ├── src/                       # 源码实现
    ├── test/                      # 官方示例程序
    └── unit/                      # 单元测试
```

## 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                 Public API (include/)                      │
│   servicemanager / client / local_object / writer ...      │
├─────────────────────────────────────────────────────────┤
│              ServiceManager 层                              │
│   aidl / aidl2~6 / hidl 多版本后端                          │
├─────────────────────────────────────────────────────────┤
│              IPC 核心层 (gbinder_ipc.c)                     │
│   事务调度 / 异步处理 / looper 管理                           │
├──────────────────────┬──────────────────────────────────┤
│   Driver 层          │    RPC Protocol 层                │
│   gbinder_driver.c   │  gbinder_rpc_protocol.c            │
│   ioctl 封装          │  aidl/hidl 头部格式                 │
├──────────────────────┴──────────────────────────────────┤
│           IO 抽象层 (gbinder_io_32/64.c)                    │
│   32/64 位内核协议适配                                       │
├─────────────────────────────────────────────────────────┤
│           Linux Kernel Binder Driver                        │
│   /dev/binder, /dev/hwbinder                               │
└─────────────────────────────────────────────────────────┘
```

## 学习阶段

| 阶段 | 主题 | 笔记文件 | 关键源码 |
|------|------|----------|----------|
| 1 | Binder 背景知识 | [01-binder-background.md](01-binder-background.md) | `README`, `src/binder.h` |
| 2 | 架构与配置系统 | [02-architecture-and-config.md](02-architecture-and-config.md) | `gbinder_config.c`, `gbinder_rpc_protocol.c` |
| 3 | Driver 与 IO 层 | [03-driver-and-io-layer.md](03-driver-and-io-layer.md) | `gbinder_driver.c`, `gbinder_io_32/64.c` |
| 4 | IPC 核心层 | [04-ipc-core.md](04-ipc-core.md) | `gbinder_ipc.c`, `gbinder_eventloop.c` |
| 5 | 对象模型 | [05-object-model.md](05-object-model.md) | `gbinder_local_object.c`, `gbinder_remote_object.c` |
| 6 | 数据序列化 | [06-data-serialization.md](06-data-serialization.md) | `gbinder_writer.c`, `gbinder_reader.c` |
| 7 | ServiceManager 与 Client | [07-servicemanager-and-client.md](07-servicemanager-and-client.md) | `gbinder_servicemanager*.c`, `gbinder_client.c` |
| 8 | 跨平台特性与实战 | [08-cross-platform-and-practice.md](08-cross-platform-and-practice.md) | `test/`, `_test/` |

## 核心术语速查

| 术语 | 说明 |
|------|------|
| **Binder** | Android 的核心 IPC 机制，基于 Linux 内核驱动 |
| **ServiceManager** | 名称→句柄映射服务，Binder 世界的"DNS" |
| **Transaction** | Binder 事务，一次请求-回复的完整交互 |
| **LocalObject** | 本地对象，接收并处理入站事务（服务端） |
| **RemoteObject** | 远程对象，发起事务的代理（客户端） |
| **AIDL** | Android Interface Definition Language |
| **HIDL** | HAL Interface Definition Language（硬件抽象层） |
| **Looper** | Binder 事件循环线程 |
| **flat_binder_object** | 内核中传递的 binder 对象的扁平表示 |

## 学习建议

1. **先读 README 和公共头文件**（`include/`），建立 API 全景认知
2. **自底向上读源码**：`binder.h` → `driver` → `io` → `ipc` → `objects` → `data` → `servicemanager` → `client`
3. **结合 `test/` 示例**理解使用方法，`binder-service` + `binder-client` 是最佳入门组合
4. **每阶段写笔记 + 画图**，特别是数据流图和调用关系图
5. **参考 `unit/` 测试**理解各模块的边界行为
