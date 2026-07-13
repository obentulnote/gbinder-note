# 阶段 6：数据序列化 — Writer/Reader/Request/Reply

> 对应源码：`src/gbinder_writer.c`、`include/gbinder_writer.h`、`src/gbinder_reader.c`、`include/gbinder_reader.h`、`src/gbinder_local_request.c`、`include/gbinder_local_request.h`、`src/gbinder_local_reply.c`、`src/gbinder_remote_request.c`、`include/gbinder_remote_request.h`、`src/gbinder_remote_reply.c`、`include/gbinder_remote_reply.h`、`src/gbinder_buffer.c`、`src/gbinder_output_data.h`

⭐GBinderWriter 和 GBinderReader 的报文目标字段都是 payload，也就是写共享内存。

## 6.1 数据模型概述

libgbinder 的数据序列化围绕四对对象展开：

```
发送方（客户端/服务端）:
  LocalRequest  ← Writer 写入 →  发送到内核
  LocalReply    ← Writer 写入 →  发送到内核

接收方（服务端/客户端）:
  RemoteRequest  ← Reader 读取 ← 从内核接收
  RemoteReply    ← Reader 读取 ← 从内核接收
```

### 数据流图

```
客户端:
  LocalRequest ──Writer──> [内核] ──> RemoteRequest ──Reader──> 服务端 handler
                                                          │
  RemoteReply <──Reader── [内核] <──Writer── LocalReply <──┘
```

## 6.2 GBinderWriter — 写入器

### 结构

```c
// include/gbinder_writer.h
struct gbinder_writer {
    gconstpointer d[4];  // 内部状态（不透明）
};

// 父对象引用（用于嵌套 buffer）
struct gbinder_parent {
    guint32 index;   // 父 buffer 对象的索引
    guint32 offset;  // 在父 buffer 中的偏移
};
```

> Writer 通常在栈上分配，由 LocalRequest/LocalReply 初始化。Writer 不引用其所属对象，调用者需确保对象生命周期长于 Writer。

### 基本类型写入

```c
// 基本类型
void gbinder_writer_append_int8(GBinderWriter* writer, guint8 value);
void gbinder_writer_append_int16(GBinderWriter* writer, guint16 value);
void gbinder_writer_append_int32(GBinderWriter* writer, guint32 value);
void gbinder_writer_append_int64(GBinderWriter* writer, guint64 value);
void gbinder_writer_append_float(GBinderWriter* writer, gfloat value);
void gbinder_writer_append_double(GBinderWriter* writer, gdouble value);
void gbinder_writer_append_bool(GBinderWriter* writer, gboolean value);

// 原始字节
void gbinder_writer_append_bytes(GBinderWriter* writer, const void* data, gsize size);
```

### 字符串写入

```c
// AIDL 字符串（UTF-16，带长度前缀）
void gbinder_writer_append_string16(GBinderWriter* writer, const char* utf8);
void gbinder_writer_append_string16_len(GBinderWriter* writer, const char* utf8, gssize num_bytes);
void gbinder_writer_append_string16_utf16(GBinderWriter* writer, const gunichar2* utf16, gssize length);

// HIDL 字符串（UTF-8，带长度前缀）
void gbinder_writer_append_string8(GBinderWriter* writer, const char* str);
void gbinder_writer_append_string8_len(GBinderWriter* writer, const char* str, gsize len);
```

#### string16 格式（AIDL）

```
| int32: length (字符数) | gunichar2[]: UTF-16 数据 | \0 (终止符) |
```

> 输入是 UTF-8，Writer 自动转换为 UTF-16。

#### string8 格式（HIDL）

```
| int32: length (字节数，不含\0) | char[]: UTF-8 数据 | \0 (终止符) |
```

### HIDL 类型写入

```c
// HIDL 向量
void gbinder_writer_append_hidl_vec(GBinderWriter* writer,
    const void* base, guint count, guint elemsize);

// HIDL 字符串
void gbinder_writer_append_hidl_string(GBinderWriter* writer, const char* str);
void gbinder_writer_append_hidl_string_copy(GBinderWriter* writer, const char* str);

// HIDL 字符串向量
void gbinder_writer_append_hidl_string_vec(GBinderWriter* writer,
    const char* data[], gssize count);

// HIDL 结构体（通过类型描述写入）
void gbinder_writer_append_struct(GBinderWriter* writer,
    const void* ptr, const GBinderWriterType* type, const GBinderParent* parent);
```

#### HIDL vec 格式

```
| GBinderHidlVec (16字节) | 数据 buffer_object |
  ├── data.ptr → 指向数据
  ├── count → 元素数量
  └── owns_buffer → 是否拥有缓冲区
```

### 对象写入

```c
// 本地对象（flat_binder_object, type=BINDER_TYPE_BINDER）
void gbinder_writer_append_local_object(GBinderWriter* writer, GBinderLocalObject* obj);

// 远程对象（flat_binder_object, type=BINDER_TYPE_HANDLE）
void gbinder_writer_append_remote_object(GBinderWriter* writer, GBinderRemoteObject* obj);

// 文件描述符
void gbinder_writer_append_fd(GBinderWriter* writer, int fd);

// FD 数组（HIDL）
void gbinder_writer_append_fds(GBinderWriter* writer, const GBinderFds* fds, const GBinderParent* parent);
```

> 写入对象时，Writer 同时记录对象在数据中的偏移到 offsets 数组，内核需要这些偏移来处理引用计数。

### Buffer 对象

```c
// buffer_object（指向额外数据的指针）
guint gbinder_writer_append_buffer_object(GBinderWriter* writer, const void* buf, gsize len);

// 带父对象的 buffer（嵌套引用）
guint gbinder_writer_append_buffer_object_with_parent(GBinderWriter* writer,
    const void* buf, gsize len, const GBinderParent* parent);

// Parcelable（int32 长度 + 数据）
void gbinder_writer_append_parcelable(GBinderWriter* writer, const void* buf, gsize len);
```

### 内存管理

Writer 提供了与事务绑定的内存分配函数：

```c
// 分配内存（事务完成后自动释放）
void* gbinder_writer_malloc(GBinderWriter* writer, gsize size);
void* gbinder_writer_malloc0(GBinderWriter* writer, gsize size);

// 便捷宏
#define gbinder_writer_new(writer, type)  ((type*)gbinder_writer_malloc(writer, sizeof(type)))
#define gbinder_writer_new0(writer, type) ((type*)gbinder_writer_malloc0(writer, sizeof(type)))

// 复制数据（事务完成后自动释放）
void* gbinder_writer_memdup(GBinderWriter* writer, const void* buf, gsize size);
char* gbinder_writer_strdup(GBinderWriter* writer, const char* str);

// 注册清理函数
void gbinder_writer_add_cleanup(GBinderWriter* writer, GDestroyNotify destroy, gpointer data);
```

> **重要**：`gbinder_writer_append_struct()` 和 `gbinder_writer_append_hidl_vec()` 不复制数据，而是写入 buffer_object 指针。调用者必须确保指针在事务完成前有效，通常通过 `gbinder_writer_malloc()` 分配。

### 其他工具函数

```c
// 获取已写入的数据
const void* gbinder_writer_get_data(GBinderWriter* writer, gsize* size);

// 已写入的字节数
gsize gbinder_writer_bytes_written(GBinderWriter* writer);

// 覆盖已写入的 int32（用于回填长度）
void gbinder_writer_overwrite_int32(GBinderWriter* writer, gsize offset, gint32 value);

// parcelable 辅助函数
gssize gbinder_writer_append_parcelable_start(GBinderWriter* writer, gboolean has_data);
void gbinder_writer_append_parcelable_finish(GBinderWriter* writer, gssize start_offset);
```

## 6.3 GBinderReader — 读取器

### 结构

```c
// include/gbinder_reader.h
struct gbinder_reader {
    gconstpointer d[6];  // 内部状态（不透明）
};
```

> Reader 在栈上分配，由 RemoteRequest/RemoteReply 初始化。不复制数据，不引用所属对象。

### 基本类型读取

```c
// 返回 TRUE 表示成功，FALSE 表示数据不足
gboolean gbinder_reader_read_int8(GBinderReader* reader, gint8* value);
gboolean gbinder_reader_read_uint8(GBinderReader* reader, guint8* value);
gboolean gbinder_reader_read_int16(GBinderReader* reader, gint16* value);
gboolean gbinder_reader_read_uint16(GBinderReader* reader, guint16* value);
gboolean gbinder_reader_read_int32(GBinderReader* reader, gint32* value);
gboolean gbinder_reader_read_uint32(GBinderReader* reader, guint32* value);
gboolean gbinder_reader_read_int64(GBinderReader* reader, gint64* value);
gboolean gbinder_reader_read_uint64(GBinderReader* reader, guint64* value);
gboolean gbinder_reader_read_float(GBinderReader* reader, gfloat* value);
gboolean gbinder_reader_read_double(GBinderReader* reader, gdouble* value);
gboolean gbinder_reader_read_bool(GBinderReader* reader, gboolean* value);
gboolean gbinder_reader_read_byte(GBinderReader* reader, guchar* value);

// 检查是否已读完
gboolean gbinder_reader_at_end(const GBinderReader* reader);
```

### 字符串读取

```c
// AIDL string16（UTF-16 存储，返回 UTF-8）
char* gbinder_reader_read_string16(GBinderReader* reader);  // 需 g_free
gboolean gbinder_reader_read_nullable_string16(GBinderReader* reader, char** out);
const gunichar2* gbinder_reader_read_string16_utf16(GBinderReader* reader, gsize* len);
gboolean gbinder_reader_skip_string16(GBinderReader* reader);

// HIDL string8（UTF-8 存储）
const char* gbinder_reader_read_string8(GBinderReader* reader);
gboolean gbinder_reader_read_nullable_string8(GBinderReader* reader, const char** out, gsize* out_len);
```

### HIDL 类型读取

```c
// HIDL vec
const void* gbinder_reader_read_hidl_vec(GBinderReader* reader, gsize* count, gsize* elemsize);
const void* gbinder_reader_read_hidl_vec1(GBinderReader* reader, gsize* count, guint expected_elemsize);

// 便捷宏
#define gbinder_reader_read_hidl_type_vec(reader, type, count) \
    ((const type*)gbinder_reader_read_hidl_vec1(reader, count, sizeof(type)))
#define gbinder_reader_read_hidl_byte_vec(reader, count) \
    gbinder_reader_read_hidl_type_vec(reader, guint8, count)

// HIDL 字符串
char* gbinder_reader_read_hidl_string(GBinderReader* reader);  // 需 g_free
const char* gbinder_reader_read_hidl_string_c(GBinderReader* reader);  // 不需释放
char** gbinder_reader_read_hidl_string_vec(GBinderReader* reader);

// HIDL 结构体
const void* gbinder_reader_read_hidl_struct1(GBinderReader* reader, gsize size);
#define gbinder_reader_read_hidl_struct(reader, type) \
    ((const type*)gbinder_reader_read_hidl_struct1(reader, sizeof(type)))
```

### 对象和缓冲区读取

```c
// 读取 binder 对象（返回 RemoteObject）
GBinderRemoteObject* gbinder_reader_read_object(GBinderReader* reader);
gboolean gbinder_reader_read_nullable_object(GBinderReader* reader, GBinderRemoteObject** obj);
#define gbinder_reader_skip_object(reader) \
    gbinder_reader_read_nullable_object(reader, NULL)

// 读取 buffer
GBinderBuffer* gbinder_reader_read_buffer(GBinderReader* reader);
gboolean gbinder_reader_skip_buffer(GBinderReader* reader);

// 读取 parcelable
const void* gbinder_reader_read_parcelable(GBinderReader* reader, gsize* size);

// 读取 FD
int gbinder_reader_read_fd(GBinderReader* reader);
int gbinder_reader_read_dup_fd(GBinderReader* reader);  // dup 一份

// 读取 byte array
const void* gbinder_reader_read_byte_array(GBinderReader* reader, gsize* len);
```

### 工具函数

```c
// 获取剩余数据
const void* gbinder_reader_get_data(const GBinderReader* reader, gsize* size);
gsize gbinder_reader_bytes_read(const GBinderReader* reader);
gsize gbinder_reader_bytes_remaining(const GBinderReader* reader);

// 复制 reader（用于回溯读取）
void gbinder_reader_copy(GBinderReader* dest, const GBinderReader* src);
```

## 6.4 GBinderLocalRequest — 本地请求

### 公共 API

```c
// include/gbinder_local_request.h
// 引用管理
GBinderLocalRequest* gbinder_local_request_ref(GBinderLocalRequest* request);
void gbinder_local_request_unref(GBinderLocalRequest* request);

// 初始化 Writer
void gbinder_local_request_init_writer(GBinderLocalRequest* request, GBinderWriter* writer);

// 注册清理函数
void gbinder_local_request_cleanup(GBinderLocalRequest* request, GDestroyNotify destroy, gpointer pointer);

// 便捷追加函数（链式调用，返回 request 本身）
GBinderLocalRequest* gbinder_local_request_append_bool(GBinderLocalRequest* request, gboolean value);
GBinderLocalRequest* gbinder_local_request_append_int32(GBinderLocalRequest* request, guint32 value);
GBinderLocalRequest* gbinder_local_request_append_int64(GBinderLocalRequest* request, guint64 value);
GBinderLocalRequest* gbinder_local_request_append_float(GBinderLocalRequest* request, gfloat value);
GBinderLocalRequest* gbinder_local_request_append_double(GBinderLocalRequest* request, gdouble value);
GBinderLocalRequest* gbinder_local_request_append_string8(GBinderLocalRequest* request, const char* str);
GBinderLocalRequest* gbinder_local_request_append_string16(GBinderLocalRequest* request, const char* utf8);
GBinderLocalRequest* gbinder_local_request_append_hidl_string(GBinderLocalRequest* request, const char* str);
GBinderLocalRequest* gbinder_local_request_append_hidl_string_vec(GBinderLocalRequest* request, const char* strv[], gssize count);
GBinderLocalRequest* gbinder_local_request_append_local_object(GBinderLocalRequest* request, GBinderLocalObject* obj);
GBinderLocalRequest* gbinder_local_request_append_remote_object(GBinderLocalRequest* request, GBinderRemoteObject* obj);
```

### 使用方式

```c
// 方式1：使用 Writer（更灵活）
GBinderLocalRequest* req = gbinder_client_new_request(client);
GBinderWriter writer;
gbinder_local_request_init_writer(req, &writer);
gbinder_writer_append_int32(&writer, 42);
gbinder_writer_append_string16(&writer, "hello");
gbinder_writer_append_local_object(&writer, local_obj);

// 方式2：使用便捷函数（链式调用）
GBinderLocalRequest* req = gbinder_client_new_request(client);
gbinder_local_request_append_int32(req, 42);
gbinder_local_request_append_string16(req, "hello");
```

## 6.5 GBinderLocalReply — 本地回复

### API（内部）

```c
// 由 LocalObject 创建
GBinderLocalReply* gbinder_local_object_new_reply(GBinderLocalObject* obj);

// 引用管理
GBinderLocalReply* gbinder_local_reply_ref(GBinderLocalReply* reply);
void gbinder_local_reply_unref(GBinderLocalReply* reply);

// 初始化 Writer
void gbinder_local_reply_init_writer(GBinderLocalReply* reply, GBinderWriter* writer);

// 便捷追加函数（与 LocalRequest 类似）
GBinderLocalReply* gbinder_local_reply_append_int32(GBinderLocalReply* reply, guint32 value);
GBinderLocalReply* gbinder_local_reply_append_string16(GBinderLocalReply* reply, const char* utf8);
// ...

// 获取输出数据（内部使用）
GBinderOutputData* gbinder_local_reply_data(GBinderLocalReply* reply);
```

### 在事务处理中使用

```c
static GBinderLocalReply*
app_reply(GBinderLocalObject* obj, GBinderRemoteRequest* req,
    guint code, guint flags, int* status, void* user_data)
{
    GBinderReader reader;
    gbinder_remote_request_init_reader(req, &reader);

    char* str = gbinder_reader_read_string16(&reader);

    GBinderLocalReply* reply = gbinder_local_object_new_reply(obj);
    gbinder_local_reply_append_string16(reply, str);  // echo back
    g_free(str);

    *status = 0;
    return reply;
}
```

## 6.6 GBinderRemoteRequest — 远程请求

### 公共 API

```c
// include/gbinder_remote_request.h
// 引用管理
GBinderRemoteRequest* gbinder_remote_request_ref(GBinderRemoteRequest* req);
void gbinder_remote_request_unref(GBinderRemoteRequest* req);

// 初始化 Reader
void gbinder_remote_request_init_reader(GBinderRemoteRequest* req, GBinderReader* reader);

// 获取事务信息
const char* gbinder_remote_request_interface(GBinderRemoteRequest* req);  // 接口名
guint32 gbinder_remote_request_code(GBinderRemoteRequest* req);           // 事务码
guint32 gbinder_remote_request_flags(GBinderRemoteRequest* req);          // 事务标志
pid_t gbinder_remote_request_sender_pid(GBinderRemoteRequest* req);       // 发送方 PID
uid_t gbinder_remote_request_sender_euid(GBinderRemoteRequest* req);     // 发送方 EUID

// 异步完成
void gbinder_remote_request_block(GBinderRemoteRequest* req);
void gbinder_remote_request_complete(GBinderRemoteRequest* req,
    GBinderLocalReply* reply, int status);

// 检查是否已被处理
gboolean gbinder_remote_request_is_complete(GBinderRemoteRequest* req);
```

## 6.7 GBinderRemoteReply — 远程回复

### 公共 API

```c
// include/gbinder_remote_reply.h
// 引用管理
GBinderRemoteReply* gbinder_remote_reply_ref(GBinderRemoteReply* reply);
void gbinder_remote_reply_unref(GBinderRemoteReply* reply);

// 初始化 Reader
void gbinder_remote_reply_init_reader(GBinderRemoteReply* reply, GBinderReader* reader);

// 复制为 LocalReply
GBinderLocalReply* gbinder_remote_reply_copy_to_local(GBinderRemoteReply* reply);

// 便捷读取函数
gboolean gbinder_remote_reply_read_int32(GBinderRemoteReply* reply, gint32* value);
gboolean gbinder_remote_reply_read_uint32(GBinderRemoteReply* reply, guint32* value);
gboolean gbinder_remote_reply_read_int64(GBinderRemoteReply* reply, gint64* value);
char* gbinder_remote_reply_read_string16(GBinderRemoteReply* reply);
const char* gbinder_remote_reply_read_string8(GBinderRemoteReply* reply);
GBinderRemoteObject* gbinder_remote_reply_read_object(GBinderRemoteReply* reply);
```

## 6.8 GBinderBuffer — 缓冲区

### 概述

`GBinderBuffer` 包装内核通过 mmap 分配的事务数据缓冲区。这些缓冲区必须通过 `BC_FREE_BUFFER` 显式释放。

```c
// src/gbinder_buffer_p.h
// 创建缓冲区
GBinderBuffer* gbinder_buffer_new(
    GBinderDriver* driver,
    void* data,        // 数据指针（mmap 内存）
    gsize size,        // 数据大小
    void** objects);   // 对象偏移数组

// 引用管理
GBinderBuffer* gbinder_buffer_ref(GBinderBuffer* buf);
void gbinder_buffer_unref(GBinderBuffer* buf);

// 释放到内核
void gbinder_buffer_free(GBinderBuffer* buf);
```

### 生命周期

```
内核分配 (BR_TRANSACTION/BR_REPLY)
    │
    v
GBinderBuffer (包装 data + objects)
    │
    ├── Reader 读取数据
    │
    └── gbinder_buffer_unref()
        └── refcount == 0
            └── gbinder_driver_free_buffer() → BC_FREE_BUFFER
```

## 6.9 GBinderOutputData — 输出数据

```c
// src/gbinder_output_data.h
// 封装 LocalRequest/LocalReply 的输出数据
// 包含:
// - GByteArray* bytes:  序列化后的数据
// - GUtilIntArray* offsets: 对象偏移数组
// - gsize buffers_size: 额外缓冲区大小（用于 SG 事务）

GBinderOutputData* gbinder_output_data_ref(GBinderOutputData* data);
void gbinder_output_data_unref(GBinderOutputData* data);
GUtilIntArray* gbinder_output_data_offsets(GBinderOutputData* data);
gsize gbinder_output_data_buffers_size(GBinderOutputData* data);
```

## 6.10 数据布局示例

### AIDL 事务数据布局

```
RPC Header (aidl2):
┌──────────┬──────────┬──────────────────────┐
│ int32    │ int32    │ string16             │
│ flags    │ work_src │ interface_name       │
│ 0x400000 │ -1       │ "android.os.IService"│
└──────────┴──────────┴──────────────────────┘

用户数据:
┌──────────┬──────────────────────┬──────────┐
│ int32    │ string16             │ int32    │
│ code_arg │ "hello"              │ 42       │
└──────────┴──────────────────────┴──────────┘
                ↑
         offsets[0] 指向这里（如果有对象）
```

### HIDL 事务数据布局

```
RPC Header (hidl):
┌────────────────────────────────┐
│ string8 "android.hidl.base@1.0"│
└────────────────────────────────┘

用户数据:
┌──────────┬──────────────────────────────────┐
│ HidlVec  │ buffer_object → 实际数据          │
│ (16字节) │                                  │
└──────────┴──────────────────────────────────┘
```

## 6.11 完整使用示例

### 客户端发送请求

```c
// 1. 获取 RemoteObject
GBinderRemoteObject* remote = gbinder_servicemanager_get_service_sync(sm, "test", &status);

// 2. 创建 Client
GBinderClient* client = gbinder_client_new(remote, "test@1.0");

// 3. 构建请求
GBinderLocalRequest* req = gbinder_client_new_request(client);
GBinderWriter writer;
gbinder_local_request_init_writer(req, &writer);
gbinder_writer_append_int32(&writer, 42);
gbinder_writer_append_string16(&writer, "hello");

// 4. 发送同步事务
int status;
GBinderRemoteReply* reply = gbinder_client_transact_sync_reply(client,
    GBINDER_FIRST_CALL_TRANSACTION, req, &status);

// 5. 解析回复
if (status == GBINDER_STATUS_OK) {
    GBinderReader reader;
    gbinder_remote_reply_init_reader(reply, &reader);
    char* result = gbinder_reader_read_string16(&reader);
    printf("Reply: %s\n", result);
    g_free(result);
}

// 6. 释放
gbinder_remote_reply_unref(reply);
gbinder_local_request_unref(req);
```

### 服务端处理请求

```c
static GBinderLocalReply*
handle_transaction(GBinderLocalObject* obj, GBinderRemoteRequest* req,
    guint code, guint flags, int* status, void* user_data)
{
    GBinderReader reader;
    gbinder_remote_request_init_reader(req, &reader);

    // 读取请求参数
    gint32 arg = 0;
    gbinder_reader_read_int32(&reader, &arg);
    char* str = gbinder_reader_read_string16(&reader);

    // 构建回复
    GBinderLocalReply* reply = gbinder_local_object_new_reply(obj);
    gbinder_local_reply_append_string16(reply, str);
    g_free(str);

    *status = 0;
    return reply;
}
```

## 6.12 小结

| 概念 | 要点 |
|------|------|
| Writer | 写入 LocalRequest/LocalReply 的数据，栈上分配 |
| Reader | 读取 RemoteRequest/RemoteReply 的数据，栈上分配 |
| LocalRequest | 客户端构建的请求数据 |
| LocalReply | 服务端构建的回复数据 |
| RemoteRequest | 服务端收到的请求数据（含 pid/euid） |
| RemoteReply | 客户端收到的回复数据 |
| string16 | AIDL 字符串，UTF-16 存储，UTF-8 输入 |
| string8 | HIDL 字符串，UTF-8 存储 |
| HIDL vec | 16 字节头 + buffer_object 指向数据 |
| offsets 数组 | 记录数据中 binder 对象的位置 |
| GBinderBuffer | 包装内核 mmap 缓冲区，需显式释放 |
| Writer 内存管理 | malloc/memdup 分配的内存随事务释放 |
| 链式调用 | append_* 返回 request/reply 本身，支持链式 |
| block/complete | 异步完成：block 标记 + complete 回复 |

---

**上一阶段**：[05-object-model.md](05-object-model.md)
**下一阶段**：[07-servicemanager-and-client.md](07-servicemanager-and-client.md) — ServiceManager 与 Client
