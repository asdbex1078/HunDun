---
name: ELF（可执行与可链接格式）
field: 计算机系统
type: 概念
aliases:
  - Executable and Linkable Format
  - ELF 文件
tags:
  - 二进制格式
  - 链接
  - 加载
  - Linux
  - CSAPP
desc: Unix/Linux 最核心的二进制容器格式：目标文件、可执行文件、共享库、core dump、内核模块、eBPF 对象都用它；核心设计是链接视图（节）与执行视图（段）分离
learned: 2026-09-24
sources:
  - ELF
---
# ELF（可执行与可链接格式）

## 描述

ELF（Executable and Linkable Format，可执行与可链接格式）是 Unix/Linux 世界最核心的二进制文件格式。它不只是「可执行文件格式」，还是目标文件、共享库、核心转储、内核模块、动态插件、eBPF 对象共用的容器。理解 ELF，就是理解**程序怎么从磁盘上的文件变成进程地址空间里的代码和数据**。

一句话定位：**ELF 是链接器和加载器之间的契约。**

## 核心内容

### 来源与支持范围

ELF 源自 System V ABI，后来由 TIS（Tool Interface Standard）标准化。它支持：

- 32 位 / 64 位
- 小端 / 大端
- 多种 CPU 架构：x86、x86-64、ARM、AArch64、RISC-V、MIPS、PowerPC 等
- 多种文件类型：目标文件、可执行文件、共享库、core dump

### 核心设计：两种视图

- **链接视图**：以「节（section）」为单位，给链接器、调试器、静态分析工具看 → [[ELF节]]
- **执行视图**：以「段（segment）」为单位，给操作系统加载器和动态链接器看 → [[ELF段]]

节头表可以被 `strip` 删掉，程序照样能跑；程序头表对可执行文件和共享库的加载必不可少。

### 顶层结构

```text
+----------------------+
| ELF Header           |  e_ident, e_type, e_machine, e_entry, e_phoff, e_shoff ...
+----------------------+
| Program Header Table |  描述「段」，加载器使用
+----------------------+
| Sections / Segments  |  实际代码、数据、符号、重定位、调试信息
+----------------------+
| Section Header Table |  描述「节」，链接器/调试器使用
+----------------------+
```

ELF 头位于文件开头，关键字段：

- `e_ident`：魔数 `0x7F 'E' 'L' 'F'`，以及位数、字节序、OSABI 等
- `e_type`：文件类型
- `e_machine`：目标架构
- `e_entry`：入口虚拟地址
- `e_phoff` / `e_shoff`：程序头表 / 节头表在文件中的偏移
- `e_phentsize` / `e_phnum`、`e_shentsize` / `e_shnum`：两张表的表项大小与数量
- `e_shstrndx`：节名字符串表索引

### 文件类型（`e_type`）

| 类型 | 值 | 含义 | 典型文件 |
|---|---:|---|---|
| `ET_NONE` | 0 | 未知 | 很少见 |
| `ET_REL` | 1 | 可重定位目标文件 | `.o` |
| `ET_EXEC` | 2 | 可执行文件，固定地址 | 传统非 PIE 可执行文件 |
| `ET_DYN` | 3 | 共享目标文件 / PIE 可执行文件 | `.so`、PIE 程序 |
| `ET_CORE` | 4 | 核心转储 | `core` |

注意：现代 Linux 的可执行文件通常是 `ET_DYN`，也就是 PIE，为了支持 ASLR（见 [[ELF安全机制]]）。

### 用在哪里

1. 可执行文件：`/bin/ls`、`/usr/bin/python`
2. 共享库：`libc.so.6`、`libssl.so`、`libstdc++.so`
3. 目标文件：`.o`，编译器（[[汇编器]]）输出、链接器输入
4. 静态库：`.a` 是 `ar` 归档，里面装多个 `.o`
5. 核心转储：`core` 文件，内存快照 + note，用于事后调试
6. 内核模块：Linux `.ko`，模块加载器解析符号和重定位
7. 内核镜像：`vmlinux` 是 ELF，`bzImage` 是压缩引导镜像
8. Android Native 库：`.so`
9. [[eBPF]] 对象：BPF 程序编译为 ELF，由 libbpf 加载
10. [[JIT]] 目标文件：LLVM ORC、MCJIT 可生成 ELF 对象并加载——ELF 在这里是「运行时生成的代码容器」
11. 数据库插件：PostgreSQL、MySQL、SQLite 扩展动态库（见 [[数据库C扩展加载]]）
12. 调试符号文件：`.debug`、build-id 对应文件
13. 容器镜像里的二进制：几乎全是 ELF
14. 嵌入式与 RTOS：很多嵌入式 Linux、QNX、VxWorks

### 对操作系统的启发

1. **统一加载接口**：内核只实现一个 ELF 加载器，就能支持可执行文件、共享库、PIE、core
2. **链接与加载分离**：编译时生成可重定位信息，加载时由动态链接器完成地址绑定
3. **虚拟内存映射**：`PT_LOAD` 直接映射到虚拟地址，支持按需分页、写时复制、mmap
4. **动态链接器用户态化**：内核只负责加载解释器，复杂的符号解析放在用户态
5. **进程启动模型**：`fork` + `execve`，`execve` 替换地址空间
6. **模块化**：内核模块 `.ko` 用 ELF，支持动态加载、符号导出、重定位
7. **安全机制**：PIE、ASLR、NX、RELRO、CET 都依赖 ELF 元数据
8. **核心转储**：core 是 ELF
9. **可扩展性**：节/段类型、NOTE、GNU 扩展让格式持续演进
10. **ABI 稳定性**：符号版本、SONAME、动态标签支撑长期二进制兼容

### 对 CSAPP 学习的启发

CSAPP 第 7 章「链接」和第 9 章「虚拟内存」几乎就是 ELF 的教科书：

- **第 7 章 链接**：`.c` → `.s` → `.o` → 可执行文件；`ET_REL` 目标文件、符号表、重定位；静态/动态链接、共享库；GOT/PLT、位置无关代码；库打桩、LD_PRELOAD
- **第 9 章 虚拟内存**：ELF 段映射到虚拟地址；`mmap`、按需分页、写时复制；加载器怎么把可执行文件变成进程
- **第 8 章 异常控制流**：`fork`、`execve`、进程上下文
- **第 3 章 机器级表示**：`.text` 里的机器码、汇编、反汇编
- **第 5 章 优化**：PIC 开销、内联、符号可见性
- **安全实验**：Attack Lab 的缓冲区溢出、NX、ASLR、RELRO 都和 ELF 权限与布局有关

作者原话：读懂 ELF，CSAPP 的链接、加载、虚拟内存、动态库章节会从「概念」变成「可验证的工程现实」。

### 实践工具

```sh
file a.out
readelf -h a.out          # ELF 头
readelf -l a.out          # 程序头 / 段
readelf -S a.out          # 节头 / 节
readelf -s a.out          # 符号表
readelf -d a.out          # 动态段
readelf -r a.out          # 重定位
objdump -d a.out          # 反汇编
objdump -t a.out          # 符号表
objdump -s -j .rodata a.out
nm a.out
ldd a.out
patchelf --print-interpreter a.out
strip a.out
gdb a.out
cat /proc/$(pidof a.out)/maps
LD_DEBUG=libs ./a.out
LD_PRELOAD=./hook.so ./a.out
```

这些工具能把 ELF 的节、段、符号、重定位一一对应起来。

## 参考资料

- System V ABI, ELF 规范：https://refspecs.linuxfoundation.org/elf/gabi4+/ch4.intro.html
- TIS ELF Specification 1.2：https://refspecs.linuxfoundation.org/elf/elf.pdf
- `elf(5)`：https://man7.org/linux/man-pages/man5/elf.5.html
- CSAPP 第 7 章链接、第 9 章虚拟内存
- Ian Lance Taylor, *Linkers and Loaders*
- Wikipedia, *Executable and Linkable Format*
- GNU Binutils BFD 文档：https://sourceware.org/binutils/docs/bfd/

## 待办

- [ ] 拿一个 hello world 分别编成非 PIE / PIE，用 `readelf -h` 对比 `e_type`
- [ ] 对照 CSAPP 第 7 章把 `.c → .s → .o → a.out` 每一步的产物跑一遍

## 关系
- 包含:: [[ELF节]] — 链接视图
- 包含:: [[ELF段]] — 执行视图
- 产出自:: [[汇编器]] — 汇编器把 .s 翻成 ET_REL 目标文件 .o
- 相关:: [[eBPF]] — eBPF 程序编译成 ELF 对象（程序、map、BTF），由 libbpf 加载进内核
- 相关:: [[JIT]] — LLVM ORC / MCJIT 生成 ELF 对象在运行时加载，数据库表达式 JIT 常用
