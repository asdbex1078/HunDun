---
name: 数据库 C 扩展加载（dlopen/dlsym）
field: 计算机系统
type: 模式
aliases:
  - 数据库插件
  - UDF
  - dlopen
tags:
  - ELF
  - 数据库
  - 插件
  - dlopen
  - ABI
  - PostgreSQL
desc: PostgreSQL/MySQL/SQLite 的 C 扩展都是 ELF 共享库：服务器 dlopen 加载 .so、dlsym 按名查函数，靠 PG_MODULE_MAGIC 之类的魔数做 ABI 校验；插件就是任意代码，必须信任
learned: 2026-09-24
sources:
  - ELF
---
# 数据库 C 扩展加载（dlopen/dlsym）

## 描述

数据库的「用 C 写扩展函数 / 插件」机制，底层全是 ELF 共享库 + [[动态链接]]：扩展编成 `.so`，数据库进程运行时 `dlopen` 把它映射进自己的地址空间，再用 `dlsym` 在动态符号表里按名字找到入口函数。

## 核心内容

### PostgreSQL

```sql
CREATE FUNCTION add_one(integer) RETURNS integer
AS '$libdir/add_one', 'add_one'
LANGUAGE C STRICT;
```

```c
#include "postgres.h"
#include "fmgr.h"

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(add_one);

Datum
add_one(PG_FUNCTION_ARGS)
{
    int32 arg = PG_GETARG_INT32(0);
    PG_RETURN_INT32(arg + 1);
}
```

```sh
gcc -fPIC -shared -I$(pg_config --includedir-server) -o add_one.so add_one.c
```

PostgreSQL 服务器通过 `dlopen` 加载 `.so`，通过 `dlsym` 找到 `add_one`。`PG_MODULE_MAGIC` 用于 ABI 校验。

### MySQL / SQLite

```sql
INSTALL PLUGIN myplugin SONAME 'myplugin.so';
```

```sql
SELECT load_extension('./myext.so');
```

这些机制都依赖 ELF 动态库、动态符号表、重定位和 ABI。

### 要注意的点

- **插件 ABI**：扩展必须匹配服务器 ABI、glibc 版本、C++ ABI
- **符号冲突**：多个插件可能导出同名符号，要 `-fvisibility=hidden` 隐藏内部符号
- **安全**：插件是任意代码，必须信任；RELRO、ASLR、NX 只能降低风险

### 旁支：数据库 JIT 与 eBPF 观测

- 数据库表达式 JIT 常用 LLVM 生成 ELF 目标文件、运行时加载（PostgreSQL JIT、ClickHouse、部分商业数据库）
- eBPF 程序编译为 ELF 对象，libbpf 加载进内核，可用来观测数据库的 I/O、锁、调度、网络

## 参考资料

- PostgreSQL: C-Language Functions：https://www.postgresql.org/docs/current/xfunc-c.html
- SQLite: Run-Time Loadable Extensions：https://www.sqlite.org/loadext.html
- MySQL: Plugin API：https://dev.mysql.com/doc/refman/8.0/en/plugin-api.html

## 待办

- [ ] 把 add_one 真编一次，`readelf -s --dyn-syms add_one.so` 找到 `add_one` 和 `Pg_magic_func`

## 关系
- 依赖:: [[动态链接]] — dlopen/dlsym 走的就是动态链接器
- 使用:: [[GCC]] — gcc -fPIC -shared 编出扩展
