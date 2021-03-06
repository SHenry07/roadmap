# 平均负载 load average 

> 主要是为我们提供了**系统负载趋势** ,重点是在趋势上

平均负载是指单位时间内，系统处于**可运行状态和不可中断状态的平均进程数**，也就是**平均活跃进程数**，不仅包括了正在使用 CPU 的进程，还包括等待 CPU 和等待 I/O 的进程。

它和 CPU 使用率并没有直接关系。这里我先解释下，可运行状态和不可中断状态这俩词儿。

- 所谓可运行状态的进程，是指正在使用 CPU 或者正在等待 CPU 的进程，也就是我们常用 ps 命令看到的，处于 R 状态（Running 或 Runnable）的进程。
- 不可中断状态的进程则是正处于内核态关键流程中的进程，并且这些流程是不可打断的，比如最常见的是等待硬件设备的 I/O 响应，也就是我们在 ps 命令中看到的 D 状态（Uninterruptible Sleep，也称为 Disk Sleep）的进程。**不可中断状态实际上是系统对进程和硬件设备的一种保护机制**

## 可能造成负载升高的原因
1. CPU密集 2.io密集  3. 大量进程造成上下文切换

==总而言之, 平均负载只是告诉了我们系统当前的状态/发展趋势,实际生产中我觉得不必深究,毕竟现有的监控手段颗粒度都是比较细的==

# CPU

## CPU飙升的可能原因

1. 程序本身CPU密集
2. CPU上下文切换

## CPU的工作模式

1. 争CPU，cpu变换任务的时候进行CPU上下文切换(context switch)。

**CPU执行任务有4种方式：进程、线程、或者硬件中断**

2. 任务的时候，需要记录任务当前的状态和获取下一任务的信息和地址(指针)，这就是上下文的内容。因此，上下文是指某一时间点CPU寄存器(CPU register)和程序计数器(PC)的内容, 广义上还包括内存中进程的虚拟地址映射信息.
   
3. **CPU上下文**切换的过程：

   - 记录当前任务的上下文(即**CPU 寄存器和程序计数器(Program Counter，PC)**等所有的状态)
     - CPU 寄存器，是 CPU 内置的容量小、但速度极快的内存。而程序计数器，则是用来存储 CPU 正在执行的指令位置、或者即将执行的下一条指令位置。它们都是 CPU 在运行任何任务前，必须的依赖环境，因此也被叫做**CPU 上下文**

       > 保存下来的上下文,会存储到系统内核中, 并在任务重新调度执行时再次加载进来

     ![img](https://static001.geekbang.org/resource/image/98/5f/98ac9df2593a193d6a7f1767cd68eb5f.png)

   - 加载新任务的上下文到这些寄存器和程序计数器
   - 跳到新任务的程序计算器 所指的新位置，运算新任务/恢复其任务

## CPU的上下文切换

根据任务的不同，CPU 的上下文切换就可以分为几个不同的场景，也就是

- 进程上下文切换
- 线程上下文切换
- 中断上下文切换

分为两种类型

- 自愿上下文切换，是指**进程无法获取所需资源，导致的上下文切换**。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。
- 非自愿上下文切换，则是指**进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换**。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

### 进程上下文切换/ 进程上下文切换与系统调用有什么区别?

Dependencies/预备知识:

[system call](https://en.wikipedia.org/wiki/System_call)

> System calls in most Unix-like systems are processed in kernel mode, which is accomplished by changing the processor execution mode to a more privileged one, but no process context switch is necessary

[context switch](https://en.wikipedia.org/wiki/Context_switch)

> Some operating systems also require a context switch to move between user mode and kernel mode tasks. The process of context switching can have a negative impact on system performance

- **进程是由内核来管理和调度的**, 进程的切换只能发生在**内核态**
- 进程的上下文,不仅包括了
  - ==用户空间的资源==: **虚拟内存, 栈,全局变量,正文,数据等**, 
  - ==内核空间的资源==:**内核堆栈,寄存器等**
  
  > thread/process context has multiple parts, one, directly associated with execution and is held in the CPU and certain system tables in memory that the CPU uses (e.g. page tables), and the other, which is needed for the OS, for bookkeeping (think of the various IDs, handles, special OS-specific permissions, network connections and such).

进程上下文切换的过程:

1. 接收到切换信号,挂起程序, 记录当前进程的虚拟内存, 栈等用户资源

2. 记录当前进程的上下文(即**CPU 寄存器和程序计数器(Program Counter，PC == ALU 计算逻辑单元), 当前进程的内核状态**)

3. 加载新任务的上下文到这些寄存器和程序计数器

4. 刷新进程的虚拟内存和用户栈

5. 跳到新任务的程序计算器 所指的新位置，运算新任务/恢复其任务

   > 保存上下文和恢复上下文的过程 并不是"免费"的, 需要内核在CPU上运行才能完成
   >
   > ![img](https://static001.geekbang.org/resource/image/39/6b/395666667d77e718da63261be478a96b.png)

   我们知道， Linux 通过 TLB(Translation Lookaside Buffer)来管理虚拟内存到物理内存的映射关系。当虚拟内存更新后，TLB 也需要刷新，内存的访问也会随之变慢。特别是在多处理器系统上，缓存是被多个处理器共享的，刷新缓存不仅会影响当前处理器的进程，还会影响共享缓存的其他处理器的进程。

#### 进程调度的触发场景

- 时间片耗尽
- 系统资源不足(比如内存不足),需要等到资源满足后才可以运行
- 睡眠函数sleep主动挂起
- 保证高优先级的进程运行, 
- 硬件中断

#### system calls 系统调用(部分命令会触发中断)/ 特权模式切换

内核提供的程序接口，是应用程序和硬件设备之间的中间层

```
man 2 syscalls
```

- 文件操作类系统调用: 如打开，创建，读取，删除，修改文件
- 进程控制类系统调用: 如创建进程，设置或获取进程属性等
- 通信类系统调用: 创建进程间的通信连接，发送，接收信息，或其他的通信方式
- 设备管理类系统调用: 打开、关闭和操作设备
- 信息维护类系统调用：在用户程序和OS之间传递信息。例如，系统向用户程序传送当前时间、日期、操作系统版本号等。

Linux 按照特权等级，把进程的运行空间分为内核空间和用户空间，分别对应着下图中， CPU 特权等级的 Ring 0 和 Ring 3。

- 内核空间（Ring 0）具有最高权限，可以直接访问所有资源
- 用户空间（Ring 3）只能访问受限资源，不能直接访问内存等硬件设备，必须通过系统调用陷入到内核中，才能访问这些特权资源
  ![img](https://static001.geekbang.org/resource/image/4d/a7/4d3f622f272c49132ecb9760310ce1a7.png)

进程既可以在用户空间运行，又可以在内核空间中运行。进程在用户空间运行时，被称为**进程的用户态**，而陷入内核空间的时候，被称为**进程的内核态**。从用户态到内核态的转变，需要通过**系统调用**来完成。比如，当我们查看文件内容时，就需要多次系统调用来完成：首先调用 open() 打开文件，然后调用 read() 读取文件内容，并调用 write() 将内容写到标准输出，最后再调用 close() 关闭文件。那么，系统调用的过程有没有发生 CPU 上下文的切换呢？答案自然是肯定的。CPU 寄存器里原来用户态的指令位置，需要先保存起来。接着，为了执行内核态代码，CPU 寄存器需要更新为内核态指令的新位置。最后才是跳转到内核态运行内核任务。而系统调用结束后，CPU 寄存器需要恢复原来保存的用户态，然后再切换到用户空间，继续运行进程。所以，一次系统调用的过程，其实是**发生了两次 CPU 上下文切换**。不过，**需要注意的是，系统调用过程中，并不会涉及到虚拟内存等进程用户态的资源，也不会切换进程。**这跟我们通常所说的进程上下文切换是不一样的：

- 进程上下文切换，是指从一个进程切换到另一个进程运行。

- 而系统调用过程中一直是同一个进程在运行。

所以，系统调用过程通常称为**特权模式切换**，而不是上下文切换。但实际上，**系统调用过程中，CPU 的上下文切换还是无法避免的。**



##### 系统调用会引起CPU上下文切换吗？

会

##### 系统调用会引起进程上下文切换吗？

不一定

###### 要明确系统调用的目的/系统调用的函数的作用

解释: 调用系统调用就是为了切换进程,那自然就会引起context switch

例子1: 示例场景中有两个并发的进程,shell进程和hello进程,最开始只有shell进程在运行,即等待命令行上的输入. 当我们让它运行hello程序时,shell通过调用一个专门的函数,即**系统调用**,来执行我们的请求,系统调用会将控制权传递给操作系统, 操作系统保存shell进程的上下文,创建一个新的hell进程及其上下文,然后将控制权传给新的hello进程, hello进程终止后,操作系统恢复shell进程的上下文, 并将控制权传回给它.csappP12

例子二: 一个用于获取系统时间的 system call 

There's not much of context switch in here, only what's needed for the transition between the modes, user and kernel. [详见](https://github.com/Talk-Go-CSAPP-05/Discusion/issues/5#issuecomment-692029936)

## Context switch


### 线程上下文切换

线程与进程最大的区别在于，**线程是调度的基本单位，而进程则是资源拥有的基本单位**。说白了，所谓内核中的任务调度，实际上的调度对象是线程；而进程只是给线程提供了虚拟内存、全局变量等资源。所以，对于线程和进程，我们可以这么理解：

- 当进程只有一个线程时，可以认为进程就等于线程。
- 当进程拥有多个线程时，这些线程会共享相同的虚拟内存和全局变量等资源。这些资源在上下文切换时是不需要修改的。
- 另外，线程也有自己的私有数据，比如栈和寄存器等，这些在上下文切换时也是需要保存的。 
  这么一来，线程的上下文切换其实就可以分为两种情况：
  1. 前后两个线程属于不同进程。此时，因为资源不共享，所以切换过程就跟进程上下文切换是一样。
  2. 前后两个线程属于同一个进程。此时，因为虚拟内存是共享的，所以在切换时，虚拟内存这些资源就保持不动，只需要切换线程的私有数据、寄存器等不共享的数据。

### 中断上下文切换

为了快速响应硬件的事件，**中断处理会打断进程的正常调度和执行**，转而调用中断处理程序，响应设备事件。而**在打断其他进程时，就需要将进程当前的状态保存下来**，这样在中断结束后，进程仍然可以从原来的状态恢复运行。

跟进程上下文不同，中断上下文切换并不涉及到进程的用户态。所以，即便中断过程打断了一个正处在用户态的进程，也不需要保存和恢复这个进程的虚拟内存、全局变量等**用户态资源**。中断上下文，其实只包括内核态中断服务程序执行所必需的状态，包括 CPU 寄存器、内核堆栈、硬件中断参数等。对同一个 CPU 来说，中断处理比进程拥有更高的优先级，所以**中断上下文切换并不会与进程上下文切换同时发生**。

`watch -d cat /proc/interrupts` 提供了一个只读的中断使用情况

变化速度最快的是重调度中断（RES），这个中断类型表示，唤醒空闲状态的 CPU 来调度新的任务运行。这是多处理器系统（SMP）中，调度器用来分散任务到不同 CPU 的机制，通常也被称为处理器间中断（Inter-Processor Interrupts，IPI）。当它对应的值变化很快时，说明存在过多的任务要调度，CPU 资源可能紧张。

### 性能问题

Context switching itself has a cost in performance, due to running the [task scheduler](https://en.wikipedia.org/wiki/Scheduling_(computing)), TLB flushes, and indirectly due to sharing the [CPU cache](https://en.wikipedia.org/wiki/CPU_cache) between multiple tasks.[[4\]](https://en.wikipedia.org/wiki/Context_switch#cite_note-4) Switching between threads of a single process can be faster than between two separate processes, because threads share the same [virtual memory](https://en.wikipedia.org/wiki/Virtual_memory) maps, so a TLB flush is not necessary.[[5\]](https://en.wikipedia.org/wiki/Context_switch#cite_note-5)

# CPU使用率 --更直观的指标

单位时间内 CPU 使用情况的统计，以百分比的方式展示
> Linux 作为一个多任务操作系统，将每个 CPU 的时间划分为很短的时间片，再通过调度器轮流分配给各个任务使用，因此造成多任务同时运行的错觉。
> ​	为了维护 CPU 时间，Linux 通过事先定义的节拍率（内核中表示为 HZ），触发时间中断，并使用全局变量 Jiffies 记录了开机以来的节拍数。每发生一次时间中断，Jiffies 的值就加 1。
> ​	节拍率 HZ 是内核的可配选项，可以设置为 100、250、1000 等. 比如在我的系统中，节拍率设置成了 250，也就是每秒钟触发 250 次时间中断。
> ```
> $ grep 'CONFIG_HZ=' /boot/config-$(uname -r)`
> CONFIG_HZ=250
> ```
> 同时，正因为节拍率 HZ 是内核选项，所以用户空间程序并不能直接访问。为了方便用户空间程序，内核还提供了一个用户空间节拍率 USER_HZ，它总是固定为 100，也就是 1/100 秒。这样，用户空间程序并不需要关心内核中 HZ 被设置成了多少，因为它看到的总是固定值 USER_HZ。Linux 通过 /proc 虚拟文件系统，向用户空间提供了系统内部状态的信息，而 /proc/stat 提供的就是系统的 CPU 和任务统计信息。比方说，如果你只关注 CPU 的话，可以执行下面的命令：
> ```shell
> # 只保留各个CPU的数据
> $ cat /proc/stat | grep ^cpu
> cpu 280580 7407 286084 172900810 83602 0 583 0 0 0
> cpu0 144745 4181 176701 86423902 52076 0 301 0 0 0
> cpu1 135834 3226 109383 86476907 31525 0 282 0 0 0
> # CPU编号 # 其他列则表示不同场景下 CPU 的累加节拍数，它的单位是 USER_HZ，也就是 10 ms（1/100 秒）
> ```
> 有需要的时候，查询`man proc`

我们通常所说的 CPU 使用率，就是除了空闲时间外的其他时间占总 CPU 时间的百分比，用公式来表示就是：


$$
CPU使用率 = 1 - \frac{空闲}{总CPU时间}
$$

根据这个公式，我们就可以从 /proc/stat 中的数据，很容易地计算出 CPU 使用率。当然，也可以用每一个场景的 CPU 时间，除以总的 CPU 时间，计算出每个场景的 CPU 使用率。不过先不要着急计算，你能说出，直接用 /proc/stat 的数据，算的是什么时间段的 CPU 使用率吗？看到这里，你应该想起来了，这是开机以来的节拍数累加值，所以直接算出来的，是开机以来的平均 CPU 使用率，一般没啥参考价值。事实上，为了计算 CPU 使用率，性能工具一般都会取间隔一段时间（比如 3 秒）的两次值，作差后，再计算出这段时间内的平均 CPU 使用率，即这个公式，
$$
平均CPU使用率 = 1 - \frac{空闲new - 空闲old}{总CPU时间new-总CPU时间old}
$$
就是我们用各种性能工具所看到的 CPU 使用率的实际计算方法。

## 来详细讲讲man mpstat // top 中也一样

- user（通常缩写为 us），代表用户态 CPU 时间。注意，它不包括下面的 nice 时间，但包括了 guest 时间。
- nice（通常缩写为 ni），代表低优先级用户态 CPU 时间，也就是进程的 nice 值被调整为 1-19 之间时的 CPU 时间。这里注意，nice 可取值范围是 -20 到 19，数值越大，优先级反而越低。
- system（通常缩写为 sys），代表内核态 CPU 时间。idle（通常缩写为 id），代表空闲时间。注意，它不包括等待 I/O 的时间（iowait）。
- iowait（通常缩写为 wa），代表等待 I/O 的 CPU 时间。
- irq（通常缩写为 hi），代表处理硬中断的 CPU 时间。
- softirq（通常缩写为 si），代表处理软中断的 CPU 时间。
- steal（通常缩写为 st），代表当系统运行在虚拟机中的时候，被其他虚拟机占用的 CPU 时间。
- guest（通常缩写为 guest），代表通过虚拟化运行其他操作系统的时间，也就是运行虚拟机的 CPU 时间。
- guest_nice（通常缩写为 gnice），代表以低优先级运行虚拟机的时间。

用户（%user）、Nice（%nice）、系统（%system） 、等待 I/O（%iowait） 、中断（%irq）以及软中断（%softirq）这几种不同 CPU 的使用率。比如说：

- **%USER CPU 和 Nice CPU 高**，说明用户态进程占用了较多的 CPU，所以应该着重排查进程的性能问题。
- **%sys CPU 高**，说明内核态占用了较多的 CPU，所以应该着重排查内核线程或者系统调用的性能问题。
- **iowait CPU 高**，说明等待 I/O 的时间比较长，所以应该着重排查系统存储是不是出现了 I/O 问题。
- **irq/hi 和 softirq高**，说明软中断或硬中断的处理程序占用了较多的 CPU，所以应该着重排查内核中的中断服务程序。

> 碰到常规问题无法解释的 CPU 使用率情况时，首先要想到有可能是**短时应用导致**的问题，比如有可能是下面这两种情况。1. 应用里直接调用了其他二进制程序，这些程序通常运行时间比较短，通过 top 等工具也不容易发现。2. 应用本身在不停地崩溃重启，而启动过程的资源初始化，很可能会占用相当多的 CPU。

# 进程状态
**man top中关于Process Status字段的解释：**

```
20. S  --  Process Status
   The status of the task which can be one of:
     D = uninterruptible sleep // Disk Sleep的缩写
     R = running
     S = sleeping //  Interruptible Sleep的缩写
     T = stopped by job control signal
     t = stopped by debugger during trace
     Z = zombie
     // 补充
     I = idle 
     X = Dead  // 表示进程已经消亡
```
>  I 是 Idle 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。前面说了，硬件交互导致的不可中断进程用 D 表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用 Idle 正是为了区分这种情况。要注意，D 状态的进程会导致平均负载升高， I 状态的进程却不会。

**man ps中关于Process Status字段**

```shell
PROCESS STATE CODES
       Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to describe the state of a process:

               D    uninterruptible sleep (usually IO)
               R    running or runnable (on run queue)
               S    interruptible sleep (waiting for an event to complete)
               T    stopped by job control signal
               t    stopped by debugger during the tracing
               W    paging (not valid since the 2.6.xx kernel)
               X    dead (should never be seen)
               Z    defunct ("zombie") process, terminated but not reaped by its parent
               
        For BSD formats and when the stat keyword is used, additional characters may be displayed:

               <    high-priority (not nice to other users)
               N    low-priority (nice to other users)
               L    has pages locked into memory (for real-time and custom IO)
               s    is a session leader
               l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
               +    is in the foreground process group
s 表示这个进程是一个会话的领导进程，而 + 表示前台进程组。
```
#### 僵尸进程

正常情况下，当一个进程创建了子进程后，它应该通过系统调用 wait() 或者 waitpid() 等待子进程结束，回收子进程的资源；而子进程在结束时，会向它的父进程发送 SIGCHLD 信号，所以，父进程还可以注册 SIGCHLD 信号的处理函数，异步回收资源。

如果父进程没这么做，或是子进程执行太快，父进程还没来得及处理子进程状态，子进程就已经提前退出，那这时的子进程就会变成僵尸进程。通常，僵尸进程持续的时间都比较短，在父进程回收它的资源后就会消亡；或者在父进程退出后，由 init 进程回收后也会消亡。

# 软中断 softirq

中断其实是一种异步的事件处理机制，可以提高系统的并发处理能力。

为了减少对正常进程运行调度的影响，中断处理程序就需要尽可能快地运行。所以Linux 将中断处理过程分成了两个阶段，

1. 上半部用来快速处理中断，它在**中断禁止模式**下运行，主要处理跟硬件紧密相关的或时间敏感的工作。==硬中断==

   硬中断会切换存储的所有资源hardware context switching stores nearly all registers whether they are required or not.

2. 下半部用来延迟处理上半部未完成的工作，通常以**内核线程**的方式运行。==软中断==
   软中断只会加载存储他们需要的资源software context switching can be selective and store only those registers that need storin

   > 下半部中每个 CPU 都对应一个软中断内核线程，名字为 “ksoftirqd/CPU 编号”

> 不过要注意的是，软中断不只包括了刚刚所讲的硬件设备中断处理程序的下半部，一些内核自定义的事件也属于软中断，比如内核调度和 RCU 锁（Read-Copy Update 的缩写，RCU 是 Linux 内核中最常用的锁之一）等。

硬中断和软中断 分上下部分共同发生的有：硬件产生的,比如键盘、鼠标的输入，硬盘的写入读取、网卡有数据了,  

单纯的 软中断 常见的有 程序内的定时器、[文中提到的RCU锁]。

> 中断不是可以嵌套的吗？在中断处理程序中可以开中断以响应更高优先级的中断，为什么第二次中断会丢失？是指中断隐指令过程吗？？？
>
> 回复: 中断不会丢失的，因为有中断控制器，它会pending住所有外部中断。除非pending住的那个中断，CPU还没来得及处理，这时又来了一个同样的中断，这个中断才会丢失。还有就是自linux-2.6.3x开始就完全不支持中断嵌套了。
>
> 目前有三种方式实现中断下半部：工作队列，tasklet和软中断，软中断机制并不完全等同于中断下半部，很多人把所有下半部当成是软中断。"请问这部分怎么理解？麻烦老师解答一下
>
> 回复: 这儿区分的更细了，tasklet 也是基于软中断的，而工作队列则是用于可以睡眠的下半部处理过程

### 如何查看中断

- `/proc/softirqs` 提供了软中断的运行情况；

   TIMER（定时中断）、NET_RX（网络接收）NET_TX（网络发送）、SCHED（内核调度）、RCU（RCU 锁）

- `/proc/interrupts` 提供了硬中断的运行情况。

[一个案例](https://blog.huoding.com/2013/10/30/296)

#### 网络

观察网络收发的吞吐量（BPS，每秒收发的字节数），还可以观察网络收发的 PPS，即每秒收发的网络帧数。

```
  IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
  docker0   3560.00   7024.83    152.97    370.45    0.00      0.00      0.00      0.00
veth10db528 3560.00   7024.83    201.64    370.45    0.00      0.00      0.00   0.03
 lo        0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
 ens160   7031.00   3564.50    412.24    202.80      0.00      0.00      0.00      0.03
```



rxpck/s 和 txpck/s 分别表示每秒接收、发送的网络帧数，也就是  PPS

rxkB/s 和 txkB/s 分别表示每秒接收、发送的千字节数，也就是  BPS。

# CPU 缓存 通常是多级缓存

![img](https://static001.geekbang.org/resource/image/aa/33/aa08816b60e453b52b5fae5e63549e33.png)

CPU 缓存的是热点的内存数据, L1 和 L2 常用在单核中， L3 则用在多核中。从 L1 到 L3，三级缓存的大小依次增大，相应的，性能依次降低（当然比内存还是好得多）。而它们的命中率，衡量的是 CPU 缓存的复用情况，命中率越高，则表示性能越好。

<img src="../image/1e66612e0022cd6c17847f3ab6989007.png" alt="img" style="zoom: 35%;" />

# CPU总结

活学活用，**把性能指标和性能工具联系起来**

1. 第一个维度，**从 CPU 的性能指标出发**。也就是说，当你要查看某个性能指标时，要清楚知道哪些工具可以做到。
2. 第二个维度, **从工具出发** 也就是当你已经安装了某个工具后，要知道这个工具能提供哪些指标。

## 清楚性能指标的关联性,通晓每种性能指标的工作原理 逐步的降低寻找范围

![img](../image/7a445960a4bc0a58a02e1bc75648aa17.png)