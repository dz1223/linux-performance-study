# clinux-performance-study

Linux性能优化学习笔记



## 平均负载

- 查看系统负载情况

  - ```shell
     $ uptime
     23:36:20 up 182 days, 12:31,  1 user,  load average: 0.00, 0.00, 0.00
     ```

  - ```shell
     $ top
     top - 23:37:06 up 182 days, 12:30,  1 user,  load average: 0.00, 0.00, 0.00
     Tasks: 158 total,   1 running, 157 sleeping,   0 stopped,   0 zombie
     Cpu(s):  0.2%us,  0.1%sy,  0.0%ni, 99.7%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
     Mem:  32880800k total, 10732044k used, 22148756k free,   175936k buffers
     Swap:  4194304k total,        0k used,  4194304k free,  3520344k cached
     ```

  - 最后三个数字，依次是过去**1分钟**、**5分钟**、**15分钟**的平均负载（Load Average）

- 简单来说，平均负载是

- 指单位时间内，系统处于**可运行状态**和**不可中断状态**的**平均进程数**，也就是**平均活跃进程数**。

- 你可以简单理解为，平均负载其实就是平均活跃进程数，即单位时间内的活跃进程数。

- 平均负载最理想的情况是等于 CPU个数。

  查询CPU个数：

  ```shell
  $ grep 'model name' /proc/cpuinfo | wc -l
  4
  ```

- 负载情况判断

  - 如果1分钟、5分钟、15分钟的三个值基本相同，或者相差不大，那就说明**系统负载很平稳**。
  - 但如果1分钟的值远小于15 分钟的值，就说明系统最近1分钟的负载在减少，而过去15分钟内却有很大的负载。
  - 反过来，如果1分钟的值远大于 15 分钟的值，就说明最近1分钟的负载在增加，这种增加有可能只是临时性的，也有可能还会持续增加下去，所以就需要持续观察。一旦1分钟的平均负载接近或超过了CPU的个数，就意味着系统正在发生过载的问题，这时就得分析调查是哪里导致的问题，并要想办法优化了。
  
- **当平均负载高于 CPU 数量70%的时候**，你就应该分析排查负载高的问题了。

- 平均负载是指单位时间内，处于可运行状态和不可中断状态的进程数。所以，它不仅包括了**正在使用 CPU** 的进程，还包括**等待 CPU** 和**等待 I/O** 的进程。

- 平均负载与CPU使用率

  - CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
  - I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高；
  - 大量等待 CPU 的进程调度也会导致平均负载升高，此时的CPU使用率也会比较高。

- 用 iostat、mpstat、pidstat 等工具，找出平均负载升高的根源

- stress 是一个 Linux 系统压力测试工具，这里我们用作异常进程模拟平均负载升高的场景。
[[stress工具使用指南](https://www.cnblogs.com/muahao/p/6346775.html)](https://www.cnblogs.com/muahao/p/6346775.html)
  

而 sysstat 包含了常用的 Linux 性能工具，用来监控和分析系统的性能。我们的案例会用到这个包的两个命令 mpstat 和 pidstat。

  - mpstat 是一个常用的多核 CPU 性能分析工具，用来实时查看每个 CPU 的性能指标，以及所有CPU的平均指标。
  - pidstat 是一个常用的进程性能分析工具，用来实时查看进程的 CPU、内存、I/O 以及上下文切换等性能指标。
  - 

`总结`

系统负载过高分析方法：

1. 通过`uptime`, `top`等命令观察系统平均负载情况，分析大致可能的引起系统负载过高的类型：CPU 密集型、I/O 密集型或其他。

   ```shell
   # -d 参数表示高亮显示变化的区域
   $ watch -d uptime
   ...,  load average: 1.00, 0.75, 0.39
   ```

2. 使用`mpstat`查看 CPU 使用率的变化情况。查找平均负载升高的原因，如下图说明，平均负载的升高正是由于 CPU 使用率为 100% 。

   ```shell
   # -P ALL 表示监控所有CPU，后面数字5表示间隔5秒后输出一组数据
   $ mpstat -P ALL 5
   Linux 4.15.0 (ubuntu) 09/22/18 _x86_64_ (2 CPU)
   13:30:06     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
   13:30:11     all   50.05    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   49.95
   13:30:11       0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
   13:30:11       1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
   ```

3. 使用`pidstat`确定到底哪个进程导致的CPU使用率过高/iowait过高？

   ```shell
   # 间隔5秒后输出一组数据
   $ pidstat -u 5 1
   13:37:07      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
   13:37:12        0      2962  100.00    0.00    0.00    0.00  100.00     1  stress
   ```

   

## CPU上下文切换

- CPU 寄存器，是 CPU 内置的容量小、但速度极快的内存。而程序计数器，则是用来存储 CPU 正在执行的指令位置、或者即将执行的下一条指令位置。它们都是 CPU 在运行任何任务前，必须的依赖环境，因此也被叫做 **CPU 上下文**。

  ![CPU上下文](https://static001.geekbang.org/resource/image/98/5f/98ac9df2593a193d6a7f1767cd68eb5f.png)

- **CPU 上下文切换**，就是先把前一个任务的 CPU 上下文（也就是 CPU 寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的新位置，运行新任务。

- 根据任务的不同，CPU 的上下文切换就可以分为几个不同的场景，也就是**进程上下文切换**、**线程上下文切换**以及**中断上下文切换**。

- Linux 按照特权等级，把进程的运行空间分为内核空间和用户空间，分别对应着下图中， CPU 特权等级的 Ring 0 和 Ring 3。

  - 内核空间（Ring 0）具有最高权限，可以直接访问所有资源；
  - 用户空间（Ring 3）只能访问受限资源，不能直接访问内存等硬件设备，必须通过系统调用陷入到内核中，才能访问这些特权资源。

  ![](https://static001.geekbang.org/resource/image/4d/a7/4d3f622f272c49132ecb9760310ce1a7.png)

- 系统调用，CPU 寄存器里原来用户态的指令位置，需要先保存起来。接着，为了执行内核态代码，CPU 寄存器需要更新为内核态指令的新位置。最后才是跳转到内核态运行内核任务。

  而系统调用结束后，CPU寄存器需要**恢复**原来保存的用户态，然后再切换到用户空间，继续运行进程。所以，一次系统调用的过程，其实是发生了两次 CPU 上下文切换。

- 进程上下文切换，是指从一个进程切换到另一个进程运行。

  ![](https://static001.geekbang.org/resource/image/39/6b/395666667d77e718da63261be478a96b.png)

- 系统调用过程中一直是同一个进程在运行。

- 线程与进程最大的区别在于，**线程是调度的基本单位，而进程则是资源拥有的基本单位**。

- **对同一个 CPU 来说，中断处理比进程拥有更高的优先级**

- 总结

  1. CPU 上下文切换，是保证 Linux 系统正常工作的核心功能之一，一般情况下不需要我们特别关注。
  2. 但过多的上下文切换，会把CPU时间消耗在寄存器、内核栈以及虚拟内存等数据的保存和恢复上，从而缩短进程真正运行的时间，导致系统的整体性能大幅下降。
  
-  查看系统的上下文切换情况

  ```shell
  # 每隔5秒输出1组数据
  [root@localhost ~]# vmstat 5
  procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu-----
   r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
   0  0 204780 1571232 166080 490732    0    0     0     3    1    1  1  0 99  0  0
   0  0 204780 1570952 166080 490736    0    0     0     0 1968 4030  0  0 100  0  0
   0  0 204780 1570968 166080 490736    0    0     0     6 1544 3585  0  0 100  0  0
   0  0 204780 1571068 166080 490748    0    0     0    18 1894 3945  0  0 100  0  0
  ```

- vmstat 只给出了系统总体的上下文切换情况，要想查看每个**进程**的详细情况，就需要使用我们前面提到过的 pidstat 了。给它加上 -w 选项，你就可以查看每个**进程上下文切换**的情况了。

  ```shell
  [root@localhost ~]# pidstat -w 5
  Linux 2.6.32-696.el6.x86_64 (localhost.localdomain) 	2020年01月02日 	_x86_64_	(4 CPU)
  
  17时43分41秒       PID   cswch/s nvcswch/s  Command
  17时43分46秒         3      0.60      0.00  migration/0
  17时43分46秒         6      0.20      0.00  watchdog/0
  17时43分46秒         7      0.60      0.00  migration/1
  17时43分46秒         9      0.40      0.00  ksoftirqd/1
  17时43分46秒        10      0.20      0.00  watchdog/1
  17时43分46秒        11      0.60      0.00  migration/2
  17时43分46秒        13      1.20      0.00  ksoftirqd/2
  17时43分46秒        14      0.20      0.00  watchdog/2
  17时43分46秒        15      0.40      0.00  migration/3
  ...
  ```

  这个结果中有两列内容是我们的重点关注对象。一个是 cswch ，表示每秒自愿上下文切换（voluntary context switches）的次数，另一个则是 nvcswch ，表示每秒非自愿上下文切换（non voluntary context switches）的次数。

  这两个概念你一定要牢牢记住，因为它们意味着不同的性能问题：

  - 所谓**自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换**。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。
  - 而**非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换**。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

- pidstat 默认显示进程的指标数据，加上 -t 参数后，才会输出线程的指标。

  ```shell
  [root@localhost ~]# pidstat -wt 1
  Linux 2.6.32-696.el6.x86_64 (localhost.localdomain) 	2020年01月02日 	_x86_64_	(4 CPU)
  
  18时04分09秒      TGID       TID   cswch/s nvcswch/s  Command
  18时04分10秒         4         -      1.92      0.00  ksoftirqd/0
  18时04分10秒         -         4      1.92      0.00  |__ksoftirqd/0
  18时04分10秒         9         -      5.77      0.00  ksoftirqd/1
  18时04分10秒         -         9      5.77      0.00  |__ksoftirqd/1
  18时04分10秒        13         -      2.88      0.00  ksoftirqd/2
  18时04分10秒         -        13      2.88      0.00  |__ksoftirqd/2
  18时04分10秒        17         -     16.35      0.00  ksoftirqd/3
  18时04分10秒         -        17     16.35      0.00  |__ksoftirqd/3
  18时04分10秒        19         -      0.96      0.00  events/0
  18时04分10秒         -        19      0.96      0.00  |__events/0
  18时04分10秒        20         -      1.92      0.00  events/1
  18时04分10秒         -        20      1.92      0.00  |__events/1
  18时04分10秒        21         -      0.96      0.00  events/2
  18时04分10秒         -        21      0.96      0.00  |__events/2
  18时04分10秒        22         -     10.58      0.00  events/3
  18时04分10秒         -        22     10.58      0.00  |__events/3
  18时04分10秒        49         -      1.92      0.00  kblockd/3
  18时04分10秒         -        49      1.92      0.00  |__kblockd/3
  18时04分10秒       526         -      3.85      0.00  jbd2/sda3-8
  18时04分10秒         -       526      3.85      0.00  |__jbd2/sda3-8
  18时04分10秒       785         -      0.96      0.00  vmmemctl
  18时04分10秒         -       785      0.96      0.00  |__vmmemctl
  18时04分10秒         -      3196  33426.92 210462.50  |__sysbench
  18时04分10秒         -      3197  33143.27 189980.77  |__sysbench
  18时04分10秒         -      3198  35290.38 193065.38  |__sysbench
  18时04分10秒         -      3199  34433.65 177601.92  |__sysbench
  18时04分10秒         -      3200  33533.65 199287.50  |__sysbench
  ```

  上述结果从可以看出，虽然 vmmemctl 进程（也就是主线程）的上下文切换次数看起来并不多，但它的子线程sysbench的上下文切换次数却有很多。看来，上下文切换罪魁祸首，还是过多的 sysbench 线程。

- 怎样才能知道中断发生的类型呢

  - 从 /proc/interrupts 这个只读文件中读取。/proc 实际上是 Linux 的一个虚拟文件系统，用于内核空间与用户空间之间的通信。/proc/interrupts 就是这种通信机制的一部分，提供了一个只读的中断使用情况。

  ```shell
  # -d 参数表示高亮显示变化的区域
  $ watch -d cat /proc/interrupts
             CPU0       CPU1
  ...
  RES:    2450431    5279697   Rescheduling interrupts
  ...
  ```

  观察一段时间，你可以发现，变化速度最快的是**重调度中断**（RES），这个中断类型表示，**唤醒空闲状态的 CPU 来调度新的任务运行**。这是多处理器系统（SMP）中，调度器用来分散任务到不同 CPU 的机制，通常也被称为**处理器间中断**（Inter-Processor Interrupts，IPI）。

- 总结

  - 碰到上下文切换次数过多的问题时，**我们可以借助 vmstat 、 pidstat 和 /proc/interrupts 等工具**，来辅助排查性能问题的根源。
  - 自愿上下文切换变多了，说明进程都在等待资源，有可能发生了 I/O 等其他问题；
  - 非自愿上下文切换变多了，说明进程都在被强制调度，也就是都在争抢 CPU，说明 CPU 的确成了瓶颈；
  - 中断次数变多了，说明 CPU 被中断处理程序占用，还需要通过查看 /proc/interrupts 文件来分析具体的中断类型。



## CPU使用率过高

-  CPU 使用率

   **CPU 使用率，就是除了空闲时间外的其他时间占总 CPU 时间的百分比**

  ![](https://static001.geekbang.org/resource/image/3e/09/3edcc7f908c7c1ddba4bbcccc0277c09.png)

  ![](https://static001.geekbang.org/resource/image/84/5a/8408bb45922afb2db09629a9a7eb1d5a.png)

- pidstat命令展示的CPU使用率指标

  -  用户态CPU使用率 （%usr）；
  -  内核态CPU使用率（%system）；

  - 运行虚拟机CPU使用率（%guest）；

  - 等待 CPU使用率（%wait）；

  - 以及总的CPU使用率（%CPU）。

- 性能分析工具给出的都是间隔一段时间的平均 CPU 使用率，所以要注意间隔时间的设置。

  -  top 默认使用 3 秒时间间隔
  -  ps 使用的却是进程的整个生命周期

- 查看 CPU 使用率

  - top 显示了系统总体的 CPU 和内存使用情况，以及各个进程的资源使用情况。不过top 默认显示的是所有 CPU 的平均值，这个时候你只需要按下数字 1 ，就可以切换到每个 CPU 的使用率了。
  - ps 则只显示了每个进程的资源使用情况。

- top 并没有细分进程的用户态CPU和内核态 CPU，那要怎么查看每个进程的详细情况呢？

  - pidstat 正是一个专门分析每个进程 CPU 使用情况的工具。

    - 用户态CPU使用率 （%usr）；
    
    - 内核态CPU使用率（%system）；
    - 运行虚拟机CPU使用率（%guest）；
    - 等待 CPU使用率（%wait）；
    - 以及总的CPU使用率（%CPU）。
    
  - pidstat

    ```shell
    # 每隔1秒输出一组数据，共输出5组
  $ pidstat 1 5
    15:56:02      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
    15:56:03        0     15006    0.00    0.99    0.00    0.00    0.99     1  dockerd
    
    ...
    
    Average:      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
    Average:        0     15006    0.00    0.99    0.00    0.00    0.99     -  dockerd
    ```

-  使用 perf 分析 CPU 性能问题

   ```shell
   $ perf top
   Samples: 833  of event 'cpu-clock', Event count (approx.): 97742399
   Overhead  Shared Object       Symbol
      7.28%  perf                [.] 0x00000000001f78a4
      4.72%  [kernel]            [k] vsnprintf
      4.32%  [kernel]            [k] module_get_kallsym
      3.65%  [kernel]            [k] _raw_spin_unlock_irqrestore
   ...
   ```

   - 第一列 Overhead ，是该符号的性能事件在所有采样中的比例，用百分比来表示。
   - 第二列 Shared ，是该函数或指令所在的动态共享对象（Dynamic Shared Object），如内核、进程名、动态链接库名、内核模块名等。
   - 第三列 Object ，是动态共享对象的类型。比如 [.] 表示用户空间的可执行程序、或者动态链接库，而 [k] 则表示内核空间。
   - 最后一列 Symbol 是符号名，也就是函数名。当函数名未知时，用十六进制的地址来表示。

-  perf top 虽然实时展示了系统的性能信息，但它的缺点是并不保存数据，也就无法用于离线或者后续的分析。而 perf record 则提供了保存数据的功能，保存后的数据，需要你用 perf report 解析展示。

-  `总结`

   CPU 使用率是最直观和最常用的系统性能指标，更是我们在排查性能问题时，通常会关注的第一个指标。

   - 用户 CPU 和 Nice CPU 高，说明用户态进程占用了较多的 CPU，所以应该着重排查**进程的性能问题**。
   - 系统 CPU 高，说明内核态占用了较多的 CPU，所以应该着重排查**内核线程**或者**系统调用**的性能问题。
   - I/O 等待 CPU 高，说明等待 I/O 的时间比较长，所以应该着重排查系统存储是不是出现了 **I/O 问题**。
   - 软中断和硬中断高，说明软中断或硬中断的处理程序占用了较多的 CPU，所以应该着重排查内核中的**中断服务**程序。

-  碰到常规问题无法解释的 CPU 使用率情况时，首先要想到有可能是短时应用导致的问题，比如有可能是下面这两种情况。

   - 第一，**应用里直接调用了其他二进制程序，这些程序通常运行时间比较短，通过 top 等工具也不容易发现**。
   - 第二，**应用本身在不停地崩溃重启，而启动过程的资源初始化，很可能会占用相当多的 CPU**。

   对于这类进程，我们可以用 pstree 或者 execsnoop 找到它们的父进程，再从父进程所在的应用入手，排查问题的根源。

   [centos 安装execsnoop cachestat等性能分析工具](https://www.jianshu.com/p/bf894464e86d)

## 不可中断进程和僵尸进程

-  进程状态

   -  **R** 是 Running 或 Runnable 的缩写，表示进程在 CPU 的就绪队列中，正在运行或者正在等待运行。
   -  **D** 是 Disk Sleep 的缩写，也就是不可中断状态睡眠（Uninterruptible Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断。
   -  **Z** 是 Zombie 的缩写，它表示僵尸进程，也就是进程实际上已经结束了，但是父进程还没有回收它的资源（比如进程的描述符、PID 等）。
   -  **S** 是 Interruptible Sleep 的缩写，也就是可中断状态睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入 R 状态。
   -  **I** 是 Idle 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。硬件交互导致的不可中断进程用 D 表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用 Idle 正是为了区分这种情况。要注意，D 状态的进程会导致平均负载升高， I 状态的进程却不会。
   
-  dstat 
dstat是一个新的性能工具，它吸收了 vmstat、iostat、ifstat 等几种工具的优点，可以同时观察系统的 CPU、磁盘 I/O、网络以及内存使用情况。

-  碰到 iowait 升高时，需要先用 dstat、pidstat 等工具，确认是不是磁盘 I/O 的问题，然后再找是哪些进程导致了 I/O。

-  等待 I/O 的进程一般是不可中断状态，所以用 ps 命令找到的 D 状态（即不可中断状态）的进程，多为可疑进程。

-  而僵尸进程的问题相对容易排查，使用 pstree 找出父进程后，去查看父进程的代码，检查 wait() / waitpid() 的调用，或是 SIGCHLD 信号处理函数的注册就行了。







   