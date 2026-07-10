---
marp: true
theme: default
paginate: true
_paginate: false
header: ''
footer: ''
backgroundColor: white

size: 16:9
style: |
  section {
    font-family: "Noto Sans CJK SC", "Microsoft YaHei", "PingFang SC", sans-serif;
    font-size: 28px;
    line-height: 1.35;
    padding: 48px 60px;
  }
  h1 {
    font-size: 48px;
    color: #1f2937;
  }
  h2 {
    font-size: 38px;
    color: #1f2937;
  }
  h3 {
    font-size: 31px;
    color: #374151;
  }
  table {
    font-size: 22px;
    width: 100%;
  }
  th {
    background: #eef2ff;
  }
  td, th {
    padding: 8px 10px;
    vertical-align: top;
  }
  code {
    font-size: 0.9em;
  }
  pre {
    font-size: 21px;
  }
  .ok {
    color: #047857;
    font-weight: 700;
  }
  .bad {
    color: #b91c1c;
    font-weight: 700;
  }
  .answer {
    color: #1d4ed8;
    font-weight: 700;
  }
  .small {
    font-size: 23px;
  }
  .tiny {
    font-size: 20px;
  }
math: mathjax
---

# 进程间通信 IPC

## 关键知识点与考点

### UNIX IPC 使用、pipe 程序特点、IPC 设计实现与系统关系

---

# 本讲主线

## 五个核心问题

1. 为什么需要 IPC？
2. UNIX 中 pipe、fork、dup、exec 如何组合？
3. 管道、消息队列、共享内存、信号分别适合什么场景？
4. IPC 在内核中如何设计和实现？
5. IPC 与文件系统、虚拟内存、进程有什么关系？

---

# IPC 是什么？

IPC：Inter Process Communication，进程间通信。

定义：

> 进程间通过数据交换或事件通知进行交互的行为。

为什么需要：

- 单个程序功能有限。
- 多进程协作完成复杂任务。
- 进程之间相对隔离，需要受控交互。
- Shell 管道、客户端/服务端、后台服务都依赖 IPC。

---

# IPC 机制总览

| IPC 机制 | 数据/事件形态 | 通信方式 | 典型用途 |
|---|---|---|---|
| 管道 Pipe | 字节流 | 内核中转 | 父子/兄弟进程流水线 |
| 命名管道 FIFO | 字节流 | 内核中转 | 无亲缘关系进程 |
| 消息队列 | 带类型消息 | 内核队列 | 结构化消息传递 |
| 共享内存 | 共享物理页 | 直接通信 | 大量数据共享 |
| 信号 Signal | 异步事件编号 | 内核通知 | 终止、暂停、用户事件 |
| Socket | 字节流/报文 | 内核网络栈 | 本机或跨机器通信 |
| 文件 | 持久字节序列 | 文件系统中转 | 持久化交换 |

---

# 直接通信与间接通信

| 类型 | 说明 | 例子 |
|---|---|---|
| 直接通信 | 多进程共享同一片数据区，少量内核参与 | 共享内存 |
| 间接通信 | 通过内核对象中转数据或事件 | pipe、消息队列、socket、signal |

直接通信的优点：

- 数据传递快，少拷贝。

直接通信的问题：

- 同步必须自己解决。
- 需要互斥锁、信号量、条件变量等协调。

---

# 阻塞、非阻塞与缓冲

通信接口常见语义：

| 维度 | 形式 |
|---|---|
| 发送 | 阻塞发送 / 非阻塞发送 |
| 接收 | 阻塞接收 / 非阻塞接收 |
| 缓冲 | 无限容量 / 有限容量 / 0 容量 |

管道是有限容量缓冲：

- 管道空：阻塞读通常等待写者写入。
- 管道满：阻塞写通常等待读者读出。
- 非阻塞模式下，可能返回错误，如 `EAGAIN`。

---

# UNIX IPC 的特点

UNIX 把很多 IPC 对象抽象为文件或 fd。

核心系统调用组合：

| 系统调用 | 作用 |
|---|---|
| `pipe(p)` | 创建管道，返回读端 `p[0]` 和写端 `p[1]` |
| `fork()` | 创建子进程，继承 fd 表 |
| `dup/dup2()` | 复制 fd，用于重定向 stdin/stdout |
| `exec()` | 执行新程序，但保留已设置好的 fd |
| `close()` | 关闭不用的 fd，影响 EOF 和引用计数 |
| `wait()` | 父进程等待子进程退出 |

---

# UNIX 用户态 pipe1

单进程内管道读写：

```c
int fds[2];
char buf[100];
int n;

pipe(fds);
write(fds[1], "this is pipe1\n", 14);
n = read(fds[0], buf, sizeof(buf));
write(1, buf, n);
```

特点：

- `pipe(fds)` 返回两个 fd。
- `fds[1]` 写端，`fds[0]` 读端。
- `write(1, ...)` 写到标准输出。

---

# UNIX 用户态 pipe2

父子进程通信：

```c
int fds[2];
char buf[100];
pipe(fds);

if (fork() == 0) {
    write(fds[1], "this is pipe2\n", 14);
} else {
    int n = read(fds[0], buf, sizeof(buf));
    write(1, buf, n);
}
```

关键点：

- `fork` 后父子继承同一管道的两个 fd。
- 子进程写，父进程读。
- 实际程序应关闭不用的一端，避免 EOF 判断出错。

---

# UNIX 重定向 redirect

```c
if (fork() == 0) {
    close(1);
    open("output.txt", O_WRONLY | O_CREAT);
    char *argv[] = {"echo", "this", "is", "redirected", "echo", 0};
    execv("echo", argv);
}
```

原理：

- fd 1 默认是标准输出。
- `close(1)` 释放 fd 1。
- `open` 通常选择最小可用 fd，因此新文件拿到 fd 1。
- `exec` 保留 fd，`echo` 不知道自己输出到了文件。

---

# Shell 管道：ls | wc -l

核心模式：

```c
int p[2];
pipe(p);

if (fork() == 0) {
    dup2(p[1], 1);
    close(p[0]);
    close(p[1]);
    execvp("ls", ...);
}

if (fork() == 0) {
    dup2(p[0], 0);
    close(p[0]);
    close(p[1]);
    execvp("wc", ...);
}
```

---

# Shell 管道：fd 连接关系

```text
ls stdout fd 1
  -> pipe write end
  -> 内核管道缓冲区
  -> pipe read end
  -> wc stdin fd 0
```

关键点：

- `dup2(p[1], 1)` 把 `ls` 标准输出接到管道写端。
- `dup2(p[0], 0)` 把 `wc` 标准输入接到管道读端。
- `exec` 后新程序继承这些 fd 绑定。

---

# 为什么必须 close？

每个进程要关闭自己不用的管道端。

原因：

- 管道 EOF 取决于所有写端是否关闭。
- 如果父进程忘记关闭 `p[1]`：
  - `wc` 读完 `ls` 的输出后继续等待。
  - 内核认为仍有写端打开。
  - `wc` 迟迟读不到 EOF，可能阻塞。

重点：

> close 不只是“省 fd”，还影响管道引用计数和 EOF 语义。

---

# 管道 Pipe

管道是一种匿名 IPC 机制。

特征：

- 内核维护一段缓冲区。
- 有读端和写端。
- 通过两个 fd 表示。
- 字节流通信，不保留应用层消息边界。
- 常用于有亲缘关系的进程。
- 匿名管道通常单向。

判断题：

| 说法 | 判断 |
|---|:---:|
| 匿名管道提供字节流通信，一般不保留应用层消息边界。 | <span class="ok">√</span> |

---

# 管道读写语义

| 情况 | 阻塞模式行为 |
|---|---|
| 管道为空且写端仍打开 | 读者睡眠等待 |
| 管道为空且所有写端关闭 | 读返回 0，表示 EOF |
| 管道满且读端仍打开 | 写者睡眠等待 |
| 所有读端关闭后继续写 | 触发 `SIGPIPE` 或返回 `EPIPE` |

判断题：

| 说法 | 判断 |
|---|:---:|
| 管道读端全部关闭后，往写端继续写会收到 SIGPIPE 或错误返回。 | <span class="ok">√</span> |

---

# 管道实现

内核中通常需要：

- 管道缓冲区，如环形缓冲区。
- 读端 file 对象。
- 写端 file 对象。
- 引用计数：读端数量、写端数量。
- 读等待队列。
- 写等待队列。
- 锁：保护缓冲区状态。

读空时：

```text
进程状态 -> sleeping
挂入 pipe 读等待队列
调用调度器
```

写入数据后唤醒读者。

---

# rCore/uCore 风格管道设计

管道可实现为文件对象：

```text
File trait
  readable()
  writable()
  read(UserBuffer)
  write(UserBuffer)
```

管道结构：

```text
Pipe read end
Pipe write end
  -> shared PipeRingBuffer
       arr[]
       head
       tail
       status
       write_end weak ref
```

好处：

- `read/write` 统一支持普通文件、标准输入输出、管道。
- fd 表里都放“文件对象”。

---

# sys_pipe 的关键点

`sys_pipe(pipe: *mut usize)`：

- 创建管道读端和写端 file 对象。
- 为当前进程分配两个空闲 fd。
- 将读端 fd 和写端 fd 写回用户数组。

难点：

- `pipe` 参数是用户虚拟地址。
- 内核要检查地址是否合法。
- 需要通过当前进程页表安全写回两个 fd。

IPC 与虚拟内存关系在这里非常明显：

> 系统调用传入的用户指针必须经过地址翻译与权限检查。

---

# 消息队列

消息队列是内核维护的结构化消息队列。

特征：

- 每条消息有类型 `mtype`。
- 消息正文是字节序列。
- 可按类型选择性接收。
- 由内核中转，不要求共享用户地址空间。

常见系统调用：

```c
msgget(key, flags);
msgsnd(msgid, buf, size, flags);
msgrcv(msgid, buf, size, type, flags);
msgctl(msgid, cmd, buf);
```

---

# 共享内存

共享内存把同一物理内存区域映射到多个进程地址空间。

系统调用：

```c
shmget(key, size, flags);
shmat(shmid, shmaddr, flags);
shmdt(shmaddr);
shmctl(shmid, cmd, buf);
```

优点：

- 避免用户态和内核态之间反复复制大量数据。
- 通常是最快 IPC 方式之一。

不足：

- 只提供共享数据通道。
- 同步互斥必须另行实现。

---

# 共享内存与虚拟内存

共享内存的本质：

```text
进程 A 虚拟页 ----\
                  -> 同一组物理页帧
进程 B 虚拟页 ----/
```

关键点：

- 两个进程的虚拟地址可以不同。
- 页表项指向相同物理页帧。
- 权限可分别设置为只读/可写。
- fork、mmap、shmat 都会涉及地址空间和页表修改。

重点：

> 共享内存快，但多个进程访问共享数据仍然需要锁、信号量等同步机制。

---

# 信号 Signal

信号是进程间异步通知机制，也可看作软件中断。

用途：

- `Ctrl+C` 产生 `SIGINT`。
- `kill(pid, sig)` 发送信号。
- 管道读端关闭后写端写入可触发 `SIGPIPE`。
- 非法访存可能导致 `SIGSEGV`。

特点：

- 适合传递少量事件信息。
- 不适合传输大量数据。

---

# 信号处理方式

进程收到信号后可选择：

| 方式 | 含义 |
|---|---|
| 忽略 | 当作没发生 |
| 捕获 | 执行用户注册的 signal handler |
| 默认处理 | 由内核执行默认动作，如终止、停止、继续 |

常见 API：

```c
sigaction(sig, &new_action, &old_action);
kill(pid, sig);
sigreturn();
```

---

# 信号实现机制

典型流程：

```text
进程注册 signal handler
  -> 内核记录在进程控制块中
某事件发送 signal
  -> 内核标记 pending signal
返回用户态前
  -> 内核发现待处理信号
  -> 修改用户态上下文
  -> 让返回地址指向 signal handler
handler 执行结束
  -> sigreturn 恢复原上下文
```

重点：

> 信号处理依赖内核安全修改用户态上下文，不能完全由用户态随意伪造。

---

# Socket

socket 支持：

- 同一台机器上的进程通信。
- 跨机器网络通信。
- 字节流或报文式通信。

判断题：

| 说法 | 判断 |
|---|:---:|
| socket 既可用于同一台机器上的进程通信，也可用于跨机器网络通信。 | <span class="ok">√</span> |

与 pipe 区别：

- pipe 常用于有亲缘关系的本机进程。
- socket 更通用，能跨主机。

---

# IPC 与文件系统

UNIX 把许多 IPC 对象做成“文件样”的对象。

| IPC | 与文件系统/文件抽象关系 |
|---|---|
| 管道 | fd 指向 Pipe file object，支持 read/write |
| 命名管道 FIFO | 文件系统中有名字的管道节点 |
| socket | fd 指向 socket 对象 |
| 标准输入输出 | fd 0/1/2 指向 stdio file object |
| 普通文件 | 也可作为间接 IPC 介质 |

核心：

> fd + read/write 统一了文件、管道、终端、socket 等对象。

---

# IPC 与进程

IPC 对象通常挂在进程资源上。

```text
Process / Task
  -> fd table
       fd -> File object
              Pipe / Stdin / Stdout / Inode / Socket
  -> signal actions
  -> pending signals
  -> address space mappings
       shared memory
```

`fork` 的影响：

- 子进程继承 fd 表。
- 管道、socket、普通文件引用计数增加。
- 信号处理配置和屏蔽状态可能继承。
- 共享内存映射可继承。

---

# IPC 与虚拟内存

IPC 与虚拟内存的交点：

| 场景 | 虚拟内存参与方式 |
|---|---|
| `read/write` pipe | 用户 buffer 是虚拟地址，内核需翻译和拷贝 |
| 消息队列 | `msgsnd/msgrcv` 需要 copy_from_user/copy_to_user |
| 共享内存 | 多个页表映射同一物理页 |
| signal handler | 内核修改用户态上下文和用户栈 |
| mmap 文件/共享区 | 文件页或共享页映射到进程地址空间 |

核心：

> 间接 IPC 主要靠内核拷贝；共享内存主要靠页表共享映射。

---

# IPC 与同步互斥

IPC 只解决“如何交换数据或通知事件”，不总是解决并发正确性。

尤其是共享内存：

- 多进程可同时读写同一物理页。
- 会产生数据竞争。
- 需要同步机制：
  - 互斥锁
  - 信号量
  - 条件变量
  - futex
  - 文件锁

判断题：

| 说法 | 判断 |
|---|:---:|
| 多个进程通过共享内存通信时，仍然需要同步机制避免数据竞争。 | <span class="ok">√</span> |

---

# 高频判断题汇总

| 说法 | 判断 | 解释 |
|---|:---:|---|
| 管道读端全部关闭后，继续写会收到 SIGPIPE 或错误返回。 | <span class="ok">√</span> | 无读者时写端出错。 |
| 共享内存最快，因为内核不用在用户态和内核态之间反复拷贝数据。 | <span class="ok">√</span> | 同一物理页映射到多个进程。 |
| 匿名管道是字节流，一般不保留应用层消息边界。 | <span class="ok">√</span> | 多次写可能被一次读组合。 |
| 消息队列要求双方共享用户地址空间。 | <span class="bad">×</span> | 内核维护队列并中转。 |

---

# 高频判断题汇总

| 说法 | 判断 | 解释 |
|---|:---:|---|
| socket 可用于本机进程通信，也可用于跨机器网络通信。 | <span class="ok">√</span> | UNIX domain / Internet socket。 |
| 信号适合传递少量事件信息，不适合大量数据。 | <span class="ok">√</span> | 信号本质是异步通知。 |
| 共享内存通信不需要任何同步机制。 | <span class="bad">×</span> | 共享数据仍会竞争。 |
| 管道读操作只复制不消耗数据，多个读者可反复读同一批字节。 | <span class="bad">×</span> | pipe 读会消耗字节，多个读者竞争读取。 |

---

# 综合题：ls | wc -l

必答点：

1. `pipe(p)` 创建内核管道对象和两个 file 端点。
2. `fork` 后子进程继承 fd，指向同一管道 file 对象。
3. `dup2(p[1], 1)` 让 `ls` 标准输出写入管道。
4. `dup2(p[0], 0)` 让 `wc` 标准输入来自管道。
5. 所有不需要的 fd 都要关闭。
6. 父进程忘关写端会导致 `wc` 等不到 EOF。
7. 读端先运行且管道为空时，读者睡眠等待。

---

# 选择题：pipe 语义

正确说法：

> 管道写满时，写者应等待或让出 CPU，而不是覆盖尚未读出的数据。

错误陷阱：

- 管道数据不是磁盘 inode 数据块。
- 管道空时阻塞读可以等待，不一定立即返回错误。
- 管道读端和写端不必属于同一个进程。
- 管道是内核缓冲区，不是两个用户地址空间直接相连。

---

# IPC 机制选择

| 需求 | 推荐机制 |
|---|---|
| Shell 流水线 | 匿名管道 |
| 无亲缘关系本机简单字节流 | 命名管道 / UNIX socket |
| 结构化小消息 | 消息队列 |
| 大量数据共享 | 共享内存 + 同步 |
| 异步通知 | 信号 |
| 跨机器通信 | Socket |
| 持久化交换 | 文件 |

---

# 总结：IPC

- IPC 是进程之间交换数据或事件通知的机制。
- UNIX 把 pipe、socket、stdio 等统一到 fd + read/write 抽象里。
- pipe 是内核字节流缓冲区，读消耗数据，空读/满写可阻塞。
- `fork + pipe + dup2 + close + exec` 是 Shell 管道的关键组合。
- 消息队列由内核中转结构化消息。
- 共享内存最快，但必须配合同步。
- 信号是异步通知，适合少量事件，不适合大量数据。

---

# 总结：关系

## 与文件系统

- IPC 对象常作为 file object 挂到 fd 表中。
- read/write 统一访问普通文件、管道、终端、socket。

## 与虚拟内存

- 用户 buffer 要查页表安全拷贝。
- 共享内存通过页表映射同一物理页。

## 与进程

- fork 继承 fd 和部分 IPC 状态。
- 阻塞 IPC 会改变进程状态并触发调度。
- 信号状态和处理函数属于进程管理的一部分。

---

# 参考材料

本讲义综合整理自：

- `os-lectures/lec10/p1-ipcoverview.md`
- `os-lectures/lec10/p2-ipclabs.md`
- `os-lectures/lec1/p5-tryunix.md`
- `os-lectures/lec1/examples/linux-c/pipe1.c`
- `os-lectures/lec1/examples/linux-c/pipe2.c`
- `os-lectures/lec1/examples/linux-c/redirect.c`
- `os-lectures/os-knowledge-point.md`
- `final-exam-vol2-discuss.md`

---

