# 操作系统期末试题

## 一、判断题

请判断下列说法的正误。正确打“√”，错误打“×”。

1. RM（Rate Monotonic）实时调度算法采用静态优先级，周期越短的周期任务优先级越高。
2. EDF（Earliest Deadline First）算法在整个系统运行期间为每个任务固定一个不变的优先级。
3. 在多处理器系统中，仅在本 CPU 上关闭中断，不能保证其他 CPU 不会同时进入同一临界区。
4. 同一进程内的多个线程通常共享地址空间和打开文件表，但每个线程仍需要各自的执行上下文。
5. 管道（pipe）的读操作只会复制数据而不会消耗数据，因此多个读者可以反复读到同一批已写入字节。
6. 文件描述符表把用户态看到的整数 fd 与内核中的打开文件对象关联起来。
7. `dup` 或 `fork` 后，若多个 fd 指向同一个打开文件对象，关闭其中任意一个 fd 都会立即释放底层文件对象。
8. 目录可以看作一种特殊文件，其内容描述“文件名到文件元数据位置”的映射关系。
9. 在基于 inode 的文件系统中，文件名通常保存在 inode 自身内部，而不是保存在目录项中。
10. 块缓存可以减少重复访问块设备的开销，也有助于把文件系统对块的读写集中管理。
11. 对用户缓冲区执行 `read` / `write` 时，内核通常需要依据当前进程页表完成地址翻译或拷贝。
12. 信号量初值为 1 时可用于互斥；初值为 0 时也可用于表达两个执行流之间的先后同步。
13. 条件变量的 `wait` 操作通常需要与互斥锁配合，等待前释放锁，被唤醒后再重新获得锁。
14. 管程通过条件变量队列保证等待者最终会被唤醒，因此进入管程时不需要互斥锁保护共享数据。
15. 死锁产生的必要条件包括互斥、持有并等待、不可抢占和循环等待。
16. 若资源分配图中存在环路，且每类资源都只有一个实例，则系统必然处于死锁状态。
17. 银行家算法的核心目标是在资源分配前判断系统是否仍可保持安全状态，从而避免进入不安全状态。
18. DMA 方式下，设备控制器可以直接在设备和内存之间搬运数据，但 CPU 仍需要参与初始化和完成处理。
19. 中断方式一定比轮询方式更适合所有设备访问场景，因此现代操作系统不再需要轮询。
20. 将设备抽象为文件接口后，`read` / `write` 的统一调用路径可以屏蔽部分设备差异，但驱动内部仍需处理具体设备协议。

## 二、选择题

1. 某设备需要进行大块数据传输，目标是减少 CPU 逐字节搬运数据的开销。操作系统更适合优先采用：

   A. 反复轮询设备状态并由 CPU 搬运全部数据
   B. 通过 DMA 让设备控制器和内存之间直接传输数据
   C. 把设备数据先写入目录项，再由用户程序读取目录项
   D. 禁止所有中断直到传输完成

2. 某进程执行 `read(fd, buf, len)` 读取普通文件。内核通过 fd 找到打开文件对象后，下一步最合理的是：

   A. 根据文件对象中的 inode 和当前偏移定位要读的文件数据块
   B. 直接把 fd 当作磁盘块号读取块设备
   C. 跳过文件系统结构，直接顺序扫描整个磁盘
   D. 先关闭该 fd，再重新分配一个新的 fd

3. 关于管道（pipe）的读写语义，下列说法正确的是：

   A. 管道读端直接读取磁盘 inode 中的数据块
   B. 管道写满时，写者应等待或让出 CPU，而不是覆盖尚未读出的数据
   C. 管道空时，读者一定立即返回错误，不能等待
   D. 管道的读端和写端必须属于同一个进程

4. 关于条件变量，下列说法正确的是：

   A. 条件变量本身保存资源数量，因此可以完全替代信号量
   B. 线程执行 `wait` 时通常应在持有相关互斥锁的情况下调用
   C. `signal` 必须唤醒所有等待者
   D. 条件变量不需要等待队列

5. 下列哪一项最准确地描述了“死锁避免”与“死锁检测”的区别？

   A. 死锁避免在分配资源前检查安全性，死锁检测在系统运行后检查是否已有死锁
   B. 死锁避免只适用于文件系统，死锁检测只适用于线程同步
   C. 死锁检测不需要任何资源分配信息
   D. 死锁避免一定比死锁检测开销更低

6. 在基于 inode 的文件系统中，一级间接索引块的主要作用是：

   A. 直接保存文件名
   B. 保存一组数据块地址，从而扩大单个文件可表示的大小
   C. 保存进程的打开文件表
   D. 保存管道缓冲区中的字节

7. 关于线程与进程的关系，下列说法正确的是：

   A. 同一进程中的线程不能共享全局变量
   B. 线程切换一定要重建整个进程地址空间
   C. 线程可共享进程资源，但需要独立保存自己的执行位置和栈等上下文
   D. 线程没有并发执行的意义，只是进程的别名

## 三、计算题

### 1. 索引式文件大小与访问路径

某文件系统采用 inode 保存文件的数据块索引。一个 inode 中包含：

- 4 个直接地址项；
- 1 个一级间接地址项；
- 1 个二级间接地址项。

每个地址项大小为 4 字节，数据块和索引块大小均为 1 KiB。

1. 计算一个一级间接索引块最多能保存多少个数据块地址。
2. 计算该 inode 能表示的单个文件最大大小，要求写出计算过程。
3. 若读取该文件中逻辑块号为 300 的数据块（逻辑块号从 0 开始），说明需要经过哪一级索引，并给出访问索引块和数据块的大致过程。

### 2. 银行家算法

系统中有 4 个进程 `P0-P3`，3 类资源 `A/B/C`。当前可用资源向量为：

```text
Available = (1, 1, 1)
```

当前资源分配矩阵 `Allocation` 与最大需求矩阵 `Max` 如下：

| 进程 | Allocation(A,B,C) | Max(A,B,C) |
| --- | --- | --- |
| P0 | (0,1,0) | (2,1,1) |
| P1 | (2,0,0) | (3,1,1) |
| P2 | (1,0,1) | (1,1,2) |
| P3 | (0,1,1) | (1,1,3) |

1. 计算各进程的 `Need = Max - Allocation`。
2. 判断当前状态是否安全；若安全，给出一个安全序列。
3. 若此时 `P1` 请求资源 `(1,0,0)`，请判断是否可以立即分配，并说明理由。

## 四、简答题

### 1. 文件描述符与文件对象

`uCore`：

```c
uint64 sys_write(int fd, uint64 va, uint64 len)
{
    struct proc *p = curr_proc();
    struct file *f = p->files[fd];
    switch (f->type) {
    case FD_STDIO:
        return console_write(va, len);
    case FD_PIPE:
        return pipewrite(f->pipe, va, len);
    case FD_INODE:
        return inodewrite(f, va, len);
    }
}
```

`rCore`：

```rust
pub trait File: Send + Sync {
    fn readable(&self) -> bool;
    fn writable(&self) -> bool;
    fn read(&self, buf: UserBuffer) -> usize;
    fn write(&self, buf: UserBuffer) -> usize;
}

pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize {
    let token = current_user_token();
    let process = current_process();
    let inner = process.inner_exclusive_access();
    if let Some(file) = &inner.fd_table[fd] {
        if !file.writable() { return -1; }
        let file = file.clone();
        drop(inner);
        file.write(UserBuffer::new(translated_byte_buffer(token, buf, len))) as isize
    } else {
        -1
    }
}
```

回答下列问题：

1. 这两段代码如何体现“文件描述符只是索引，真正的 I/O 逻辑由内核中的文件对象完成”？
2. 为什么同一个 `sys_write` 可以同时支持标准输出、管道和普通文件？
3. 为什么在调用具体文件对象的 `write` 前，需要先处理用户缓冲区地址？

### 2. 管道使用

Shell 执行命令：

```sh
cat input.txt | grep keyword
```

可以抽象为如下过程：

```c
pipe(p);                 // p[0] 为读端，p[1] 为写端

if (fork() == 0) {        // 子进程 A：执行 cat
    close(1);
    dup(p[1]);            // 让标准输出指向管道写端
    close(p[0]);
    close(p[1]);
    exec("cat", ...);
}

if (fork() == 0) {        // 子进程 B：执行 grep
    close(0);
    dup(p[0]);            // 让标准输入指向管道读端
    close(p[0]);
    close(p[1]);
    exec("grep", ...);
}

close(p[0]);
close(p[1]);
wait(...);
wait(...);
```

相关内核代码片段如下。

`uCore`：

```c
int pipewrite(struct pipe *pi, uint64 addr, int n)
{
    while (w < n) {
        if (pi->readopen == 0) {
            return -1;
        }
        if (pi->nwrite == pi->nread + PIPESIZE) {
            yield();
        } else {
            copyin(p->pagetable, &pi->data[pi->nwrite % PIPESIZE], addr + w, size);
            pi->nwrite += size;
            w += size;
        }
    }
    return w;
}

int piperead(struct pipe *pi, uint64 addr, int n)
{
    while (pi->nread == pi->nwrite) {
        if (pi->writeopen)
            yield();
        else
            return -1;
    }
    copyout(p->pagetable, addr + r, &pi->data[pi->nread % PIPESIZE], size);
    pi->nread += size;
    return r;
}
```

`rCore`：

```rust
pub fn make_pipe() -> (Arc<Pipe>, Arc<Pipe>) {
    let buffer = Arc::new(unsafe { UPSafeCell::new(PipeRingBuffer::new()) });
    let read_end = Arc::new(Pipe::read_end_with_buffer(buffer.clone()));
    let write_end = Arc::new(Pipe::write_end_with_buffer(buffer.clone()));
    buffer.exclusive_access().set_write_end(&write_end);
    (read_end, write_end)
}

fn read(&self, buf: UserBuffer) -> usize {
    loop {
        let mut ring_buffer = self.buffer.exclusive_access();
        if ring_buffer.available_read() == 0 {
            if ring_buffer.all_write_ends_closed() { return already_read; }
            drop(ring_buffer);
            suspend_current_and_run_next();
            continue;
        }
        /* 从环形缓冲区读出字节 */
    }
}
```

回答下列问题：

1. 对 `cat` 子进程和 `grep` 子进程来说，标准输入、标准输出分别应当指向哪里？`close + dup` 在这里起什么作用？
2. 为什么父进程和两个子进程都要关闭自己不再使用的管道端？如果父进程忘记关闭管道写端，`grep` 可能出现什么现象？
3. 结合 `uCore` 或 `rCore` 任一代码片段说明：当 `grep` 先读而 `cat` 尚未写入时，内核为什么不应让它忙等？当所有写端关闭后，读端继续读空管道时应如何处理？