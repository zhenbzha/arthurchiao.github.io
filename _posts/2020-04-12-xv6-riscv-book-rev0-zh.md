---
layout    : post
title     : "[笔记] xv6 risc-v book（MIT 2019, rev0）"
date      : 2020-04-12
lastupdate: 2020-04-12
categories: os assembly
---

### 译者序

本文内容自 2019 年 MIT 的一门本科生课程 [“Operating System Engineering”](https://pdos.csail.mit.edu/6.828/2019/xv6.html)，

* 教材 [xv6 risc-v rev0](https://pdos.csail.mit.edu/6.828/2019/xv6/book-riscv-rev0.pdf)
* [配套代码](https://github.com/mit-pdos/xv6-riscv)

课程目的（6.S081）：

* 理解操作系统的设计与实现
* 切身实践如何对一个小型操作系统进行扩展（extending a small O/S）
* 编写系统软件（systems software），积累经验

----

# 1. OS

本书使用一个具体的操作系统 xv6 作为例子，解释操作系统的若干概念。

xv6 提供了与Ken Thompson 和 Dennis Ritchie 设计的 Unix 类似的基本接口（basic
interfaces），并模仿了 Unix 的内部设计。

理解了 xv6，再去看其他类 Unix 设计时就会轻松很多。

内核使用 **CPU 的硬件保护机制**来确保每个运行在用户空间的进程只能访问它们各自的
内存。

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/1-1.png" width="60%" height="60%"></p>
<p align="center">Fig 1-1.</p>

当用户程序发起系统调用时，硬件会**提升特权级别**（raises the privilege level），
然后执行内核中某个预设的函数。

本章接下来介绍 xv6 的下列服务：

* 进程
* 内存
* 文件描述符
* 管道
* 文件系统

**并用具体的代码片段来解释它们**。还会讨论 shell —— 传统类 Unix 操作系统的主要用
户接口—— 如何使用这些资源。

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/1-2.png" width="60%" height="60%"></p>
<p align="center">Fig 1-2. xv6 提供的系统调用</p>

shell 只是一个**普通程序**（ordinary program）：获取用户命令然后执行。

shell 是一个**用户空间程序** —— 而非内核空间程序 —— 的事实**体现了系统调用接口的
强大**：shell 并没有什么特殊之处。

> The fact that the shell is a user program and not part of the kernel,
> illustrates the power of the system call interface: there is nothing special
> about the shell. 

这也意味着**可以轻易地用其他 shell 替换系统自带的 shell**；事实上，现代 Unix 系
统上有多种 shell 可以选择，每种都有自己的用户接口和脚本特性（user interface and
scripting features）。

xv6 的 shell 基本上是 Unix Bourne shell 的一个简单实现。 

## 1.1 进程（process）与内存（memory）

### 1.1.1 fork 系统调用

fork 创建一个子进程。

子进程有**独立的内存和寄存器**，但**初始的内存内容完全拷贝自父进程**，其中包括父
进程打开的**文件描述符**。

> Although the child has the same memory contents as the parent initially, the
> parent and child are executing with different memory and different registers:
> changing a variable in one does not affect the other.

### 1.1.2 exec 系统调用

exec 系统调用**从文件系统加载内存镜像**（memroy image），**替换当前进程的内存内容**。

> The exec system call replaces the calling process’s memory with a new memory
> image loaded from a file stored in the file system. 

这个文件必须符合特定的格式，例如，哪个部分是指令、哪个部分是数据、指令从什么
位置开始等等。xv6 使用的 ELF 格式。

**exec 成功后不会再返回到原进程**，而是从 ELF 头中规定的位置（entry point）开始执行。

```c
char *argv[3];
argv[0] = "echo"; // 参数列表中第一个仍然是可执行文件名，但大部分程序会忽略这个参数
argv[1] = "hello";
argv[2] = 0;

exec("/bin/echo", argv);

// exec 成功后不会再返回到原程序，因此能执行到这里说明 exec 一定出错了
printf("exec error\en");
```

exec 接受两个参数：

* 可执行文件路径（例如 `/bin/echo`）
* 参数列表（例如 `["echo", "hello"]`）：注意列表的第一个参数仍然是文件名，但大部
  分程序都会忽略第一个参数。

### 1.1.3 shell 实现原理

**循环**（代码 `sh.c`）：

1. 读取输入
1. `fork` 子进程，`exec` 子进程，`exit` 子进程
1. 父进程 `wait` 子进程退出

fork 和 exec 实现为两个单独的系统调用，对于 I/O redirect 非常有用，后面会看到例子。

### 1.1.4 内存分配

xv6 显式分配内存：

* fork：为**子进程**分配内存
* exec：为**保存可执行文件分**配内存（hold the executable file）
* sbrk：**进程需要增加/减少自己的内存空间**时，可以调用 sbrk

xv6 没有用户的概念，用 Unix 术语来说就是：所有命令都以 root 权限运行。

## 1.2 I/O 和文件描述符

> A file descriptor is a small integer representing a kernel-managed object that
> a process may read from or write to.

文件描述符是一个**整数值**，表示进程能读或写的某个**由内核管理的对象**。

> the file descriptor interface abstracts away the differences between files,
> pipes, and devices, making them all look like streams of bytes.

每个进程都有完整的文件描述符空间，默认：

* 0：stdin
* 1：stdout
* 2：stderr
* 3+：其他

### read/write 系统调用

例子，`cat` 的实现：

```c
char buf[512];
int n;
for(;;){
    n = read(0, buf, sizeof buf);
    if(n == 0)
        break;

    if(n < 0){
        fprintf(2, "read error\en");
        exit();
    }

    if(write(1, buf, n) != n){
        fprintf(2, "write error\en"); exit();
    }
}
```

### close 系统调用

close 用于关闭一个文件描述符。

> A newly allocated file descriptor is always the lowest- numbered unused
> descriptor of the current process.

**每次新打开一个文件时，都会从可用的文件描述符中选最小的一个**。非常重要！见下面
I/O 重定向的例子。

### I/O 重定向：fork/close/open/exec

利用这个特点，可以很方便地实现 I/O 重定向（将某个文件重定向到 stdin 或 stdout）：

1. fork 创建子进程后，子进程完全复制了父进程的内存，因此有与父进程一样的文件描述符。
1. 接下来的 exec 会替换子进程的内存，但不会修改其文件描述符表。
1. 因此，在 fork 之后 exec 之前关闭再打开某些文件，就可以实现重定向。

例如，shell 执行命令 `cat < input.txt` 时的逻辑如下：

```c
char *argv[2];
argv[0] = "cat";
argv[1] = 0;

if(fork() == 0) {
	close(0);                     // 关闭 0（stdin）
	open("input.txt", O_RDONLY);  // open 返回最小可用的文件描述符，此时是 0，因此该文件被重定向到 stdin
	exec("cat", argv);
}
```

这也是前面提到，为什么 fork 和 exec 分为两个系统调用比较灵活的原因。

### fork/dup 后的父子进程共享 underlying file offset

子进程复制了父进程的 file descriptor table，但每个 file 的 offset 是父子进程共享的：

```c
if(fork() == 0) {
  write(1, "hello ", 6);
  exit(0);
} else {
  wait(0);
  write(1, "world\en", 6);
}
```

这个例子最后会打印 `hello, world`（`wait` 使父进程在子进程结束之后才开始执行）。
这种行为使得多条 shell 命令会产生顺序输出，例如 `(echo hello; echo world) >output.txt`。

`dup` 系统调用复制一个文件描述符，返回指向**同一个相同底层 I/O 对象**的新描述符。

用 `dup` 重写这个例子：

```c
fd = dup(1);
write(1, "hello ", 6);
write(fd, "world\en", 6);
```

> Two file descriptors share an offset if they were derived from the same
> original file descriptor by a sequence of fork and dup calls. Otherwise file
> descriptors do not share offsets, even if they resulted from open calls for the
> same file. 

**只有 fork 或 dup 之后得到的文件描述符才共享原文件的 offset**。否则不会共享，即便是
用 `open` 打开同一文件得到的描述符。

`dup` 使得 shell 可以实现如下命令：

```shell
$ ls existing-file non-existing-file > tmp1 2>&1
```

其中 `2>&1` 表示的意思是：**将文件描述符 `1` dup 一份，用作文件描述符 `2`**（give the command a
file descriptor 2 that is a duplicate of descriptor 1），即将 stderr 重定向到
stdout。这条命令的最终效果就是将 stderr 和 stdout 都重定向到 `tmp1` 文件。

> File descriptors are a powerful abstraction, because they hide the details of
> what they are con- nected to: a process writing to file descriptor 1 may be
> writing to a file, to a device like the console, or to a pipe.

## 1.3 Pipe

> A pipe is a small kernel buffer exposed to processes as a pair of file
> descriptors, one for reading and one for writing. 

Pipe 是**自带两个文件描述符的一段内核缓冲区**，一个描述符用于读，一个用于写。

下面这个例子执行 `wc` 命令，并将 pipe 的读端重定向到 `wc` 进程的 stdin：

```c
int p[2];

char *argv[2];
argv[0] = "wc";
argv[1] = 0;

pipe(p);
if(fork() == 0) {
  close(0);    // 关闭 stdin
  dup(p[0]);   // dup 时会选择最小可用的文件描述符，由于上一行刚将 0 释放，因此这里会将 p[0] dup 到 0
  close(p[0]);
  close(p[1]);
  exec("/bin/wc", argv);
} else {
  close(p[0]);
  write(p[1], "hello world\en", 12);
  close(p[1]); // 这一行很重要：所有写端 close 后，读端会读到 EOF，而 wc 读到 EOF 之后才会执行
}
```

pipe 没数据时读端的行为：

1. 写端未关闭情况下：阻塞。
1. 写端（由于 fork 或 dup，写端可能有多个）全部关闭情况下：读到 EOF，表示 pipe 文件被另一端关闭了。

形如 `grep fork sh.c | wc -l` 就是**前者创建一个 pipe，然后 fork
一个子进程执行后者，父子进程用这个 pipe 传数据**。

多个 pipe 可以级联（` a | b | c | d`），
对于每个 pipe，后者都是前者 fork 出来的，最终形成一棵进程树。

Pipe 完成的事情，用临时文件也能完成：

```shell
$ echo hello world | wc
```

```shell
$ echo hello world >/tmp/xyz; wc </tmp/xyz
```

**pipe 相比于临时文件至少有 4 个优势**：

1. 自动清理，不用再去删临时文件
2. 可以传递不限长度的数据流（写、读会阻塞）
3. pipe 是并行的，而临时文件方式是串行的
4. 如果是进程间通信场景，pipe 的阻塞式语义会比临时文件的非阻塞式语义更有用

## 1.4 文件系统


### `mknod` 系统调用：创建一个 device 文件

```c
mknod("/console", 1, 1);
```

设备文件**没有内容**（content），其 **metadata 中记录了设备号**（major, minor）
，唯一地标识一个内核设备。当用 `open` 打开设备文件时，内核会将其 read/write 系统
调用重定向到该设备提供的输入输出方法。

### `fstat` 系统调用：获取文件描述符对应的文件对象信息

```c
#define T_DIR 1    // Directory
#define T_FILE 2   // File
#define T_DEVICE 3 // Device

struct stat {
    int dev;     // File system’s disk device
    uint ino;    // Inode number
    short type;  // Type of file
    short nlink; // Number of links to file
    uint64 size; // Size of file in bytes
};
```

### `link`, `unlink` 系统调用

> A file’s name is distinct from the file itself; the same underlying file,
> called an inode, can have multiple names, called links. 

文件的名字（file name）是与文件本身并不是一一对应：

* **底层用 inode 唯一地表示一个文件**
* 一个 inode 可以被多个文件名 link

下面是一种非常地道的**创建临时 inode** 的方式：

```c
fd = open("/tmp/xyz", O_CREATE|O_RDWR);
unlink("/tmp/xyz");
```

当 fd 被关闭时，这个 inode 就会自动释放。

**绝大部分文件系统命令都是在用户空间实现的**，例如`ls`、`mkdir`，但有一个例外：
`cd`，它是 shell builtin 的。因为 `cd` 会切换进程工作目录。

> One exception is cd, which is built into the shell (user/sh.c:160). cd must
> change the current working directory of the shell itself. If cd were run as a
> regular command, then the shell would fork a child process, the child process
> would run cd, and cd would change the child ’s working directory. The parent’s
> (i.e., the shell’s) working directory would not change.

## 1.5 Real World

> Xv6 is not POSIX compliant. It misses system calls (including basic
> ones such as lseek), it implements system calls only partially, as well as other
> differences. Our
> main goals for xv6 are simplicity and clarity while providing a simple UNIX-like
> system-call interface.

文件系统（file system）和文件描述符（file descriptors）是功能强大的抽象，但并不
是只有这一种**操作系统接口模型**（models for operating system interfaces）。

> Multics, a predecessor of Unix, abstracted file storage in a way that made it
> look like memory, producing a very different flavor of interface.

## 练习题

1. Write a program that uses UNIX system calls to “ping-pong” a byte between two
   processes over a pair of pipes, one for each direction. Measure the program’s
   performance, in exchanges per second.

# 2 Operating system organization

操作系统必须满足的三个需求：

* multiplexing
* isolation
* interaction

本章介绍要满足以上三个需求，操作系统应该如何组织。

* 有多种组织方式，本书使用的是主流方式：**monolithic kernel**
* 很多 Unix 系统都使用了 monolithic kernel 设计

本书假设 xv6 运行在 RISC-V 多处理器上（前几年的 xv6 假设运行在 x86 之上，如果没
有 RISC-V 机器，可自行搜索 xv6 x86 书和代码，建议跟着做练习）。

Xv6 能够用 QEMU 模拟器启动（指定 `-machine virt`），这需要 xv6 提供：

* RAM
* a ROM containing boot code
* a serial connection to the user’s key-board/screen
* a disk for storage

## 2.1 物理资源抽象

考虑这样一个问题：**为什么要设计系统调用，直接将它们实现为库函数（lib），应用（
applications）直接调用库函数来访问硬件资源不就行了**？

某些**嵌入式实时系统**确实是这样做的。这种方式的**缺点**：如果同时有多个应用，必
须确保每个应用的运行行为都是良好的，即，**运行一段时间后主动让出 CPU 给其他应用**。
这种称为协作分时方案（cooperative time-sharing scheme），需要应用彼此信任，
并且应用中没有 bug。

对于更通用的场景来说，应用之间彼此不信任，而且有 bug 是必然的，因此更需求的是
**强隔离**（stronger isolation）而不是**协作**（cooperative scheme）。

**实现强隔离比较好的方式：禁止应用直接访问关键硬件资源**。将这些资源抽象为服务（
services），应用只能通过定义良好的接口访问这些服务。例如对于文件系统，只能通过
`ls`、`mkdir`、等访问，好处：

1. 应用访问更加方便，例如应用只需知道文件名
1. 操作系统（服务接口提供方）管理磁盘更方便

> As another example, Unix processes use exec to build up their memory image,
> instead of directly interacting with physical memory. This allows the operating
> system to decide where to place a process in memory; if memory is tight, the
> operating system might even store some of a process’s data on disk. Exec also
> provides users with the convenience of a file system to store executable program
> images.

其他例子：

* 文件描述符抽象
* 进程切换抽象：内核负责上下文的保存和恢复，对应用透明

系统调用的设计：

* 方便程序员
* 提供了强隔离

## 2.2 User mode, supervisor mode, and system calls

RISC-V CPU 的三种模式：

* machine mode
* supervisor mode
* user mode

supervisor mode 允许 CPU 执行特权指令，例如：打开或关闭中断、读或写保存了 page
table 地址的寄存器等。

应用只能执行 user-mode 指令。

## 2.3 Kernel organization

操作系统设计时一个核心问题：**操作系统的哪些部分应该运行在 supervisor mode**？

* **monolithic kernel**：整个内核都运行在 supervisor mode

    The entire operating system runs with full hardware privilege. 

* **microkernel**：最小化需要运行在 supervisor mode 的代码

    Reduce the risk of mistakes in the kernel.

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/2-1.png" width="60%" height="60%"></p>
<p align="center">Fig 2-1. A microkernel with a file-system server</p>

## 2.4 Code: xv6 organization

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/2-2.png" width="60%" height="60%"></p>
<p align="center">Fig 2-2. Xv6 kernel source files</p>

## 2.5 Process overview

xv6（以及其他 Unix 系统）中隔离（isolation）的基本单位是进程（process）。

实现进程用到了下列机制：

* user/supervisor mode flag
* 地址空间（address spaces）
* 线程的分时切片（time-slicing of threads）

为实现隔离，进程抽象（process abstraction）给程序提供了一个幻象：每个程序都完全
拥有这台计算机（it has its own private machine）。
进程给程序提供了：

1. 其他进程无法访问私有地址空间
1. 看起来是自己独占的 CPU

xv6 **使用 page tables（硬件实现）**来给每个进程提供私有地址空间。

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/2-3.png" width="30%" height="30%"></p>
<p align="center">Fig 2-3. Layout of a virtual address space of a user process</p>

xv6 为每个进程维护一个 page table，进程的地址空间如图 2.3。

> At the top of the address space xv6 reserves a page for a tram- poline and a
> page mapping the process’s trapframe to switch to the kernel, as we will explain
> in Chapter 4.

每个进程最重要的组成部分：

* page table
* kernel stack
* run state

每个进程有两个栈：a user stack and a kernel stack (p->kstack)。

> When the process is executing user instructions, only its user stack is in use, and
> its kernel stack is empty. When the process enters the kernel (for a system call
> or interrupt), the kernel code executes on the process’s kernel stack; while a
> process is in the kernel, its user stack still contains saved data, but isn’t
> ac- tively used. 

## 2.6 Code: starting xv6 and the first process

本节从较高层次介绍 xv6 内核的启动过程，以及如何运行第一个进程。后面的章节会对其
中一些过程进程更详细介绍。

1. 计算机上电（power on），初始化
2. 计算机从 ROM 运行 BootLoader
3. BootLoader 将 xv6 内核加载到内存
4. CPU 以 machine mode 从 `_entry`（`kernel/entry.S`）开始运行 xv6

xv6 开始运行时 paging hardware 是关闭的：**此时虚拟内存直接映射到物理内存**。

BootLoader 将 xv6 加载到地址 `0x80000000`，没有加载到 `0x00000000` 是因为：
**`0x00000000 ~ 0x80000000`（共 256MB）是 I/O 设备的地址**。

`_entry` 处的指令设置一个栈（setup a stack），为后面执行 C 代码做准备：

1. xv6 在`start.c` 中声明了初始栈 `stack0`。
2. `_entry` 将这个栈的起始地址 `stack0+4096` 加载到栈指针寄存器 `sp`。
3. 有了这个栈之后，就可以执行 C 调用了：从 `_entry` 调用到`start.c` 中的 `start` 函数。

## 2.8 Exercises

1. You can use gdb to observe the very first kernel-to-user transition. Run make
   qemu-gdb. Inanotherwindow,inthesamedirectory,rungdb.Typethegdbcommandbreak
   *0x3ffffff07e, which sets a breakpoint at the sret instruction in the kernel
   that jumps into user space. Type
   the continue gdb command. gdb should stop at the breakpoint, about to execute
   sret.
   Type stepi. gdb should now indicate that it is executing at address 0x4,
   which is in user
   space at the start of initcode.S.

# 3 Page Tables

Page tables also provide a level of indirection that allows xv6 to perform a few
tricks: mapping the same memory (a trampoline page) in several address spaces,
and guarding kernel and user stacks with an unmapped page.

## 3.1 Paging hardware

page table gives the operating system control over virtual- to-physical address translations at the granularity of aligned chunks of 4096 (212) bytes. Such a chunk is called a page.

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/3-1.png" width="60%" height="60%"></p>
<p align="center">Fig 3-1.</p>

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/3-2.png" width="60%" height="60%"></p>
<p align="center">Fig 3-2.</p>

## 3.2 Kernel address space

The kernel has its own page table. When a process enters the kernel, xv6 switches to the kernel page table, and when the kernel returns to user space, it switches to the page table of the user process. The memory of the kernel is private.

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/3-3.png" width="60%" height="60%"></p>
<p align="center">Fig 3-3.</p>

QEMU simulates a computer that includes I/O devices such as a disk interface. QEMU exposes the device interfaces to software as memory-mapped control registers that sit below 0x80000000 in physical memory. The kernel can interact with the devices by reading/writing these memory locations.

The kernel uses an identity mapping for most virtual addresses; that is, most of the kernel’s address space is “direct-mapped.” For example, the kernel itself is located at KERNBASE in both the virtual address space and in physical memory. This direct mapping simplifies kernel code that needs to both read or write a page (with its virtual address) and also manipulate a PTE that refers to the same page (with its physical address). 

## 3.6 Process address space

Each process has a separate page table, and when xv6 switches between processes, it also changes page tables.

<p align="center"><img src="/assets/img/xv6-riscv-book-rev0/3-4.png" width="60%" height="60%"></p>
<p align="center">Fig 3-4.</p>
