---
title: perf的基本用法
date: 2020-12-18 16:01:01
tags:
---
# perf

## 架构图总览

![img](https://raw.githubusercontent.com/junixcn/images/master/perf_events_map.png)







## Events

事件主要有哪些



**hardware events**：CPU performance monitoring counters

**software events**: 基于kernel counters的低水平事件，比如cpu迁移、minor faults、major faults等等

**kernel tracepoint events**：编码嵌入在内核中的内核级别静态测试点

**User statically-defined tracing(USDT)**: 用户级别的静态测试点

**Dynamic Traceing**：动态软件测试点，可以在任何地方创建。内核态使用kprobes框架，用户态使用uprobes工具

**Timed Profiling**：perf-record -F *Hz*可以按照指定的频率进行监测，这个常被用于监测CPU使用率以及创建定时的中断事件



PMU：

Most processors nowadays have special, on‐chip hardware that monitors micro architectural events like elapsed cycles, cache hits, cache miss etc.It is a subsystem which helps in analyzing how an application or operating systems are performing on the processor.

```shell
The Performance Monitoring Events can be broadly categorized in two types
• Hardware
Ex: CPU‐Cycles, Instructions, Cache References
• Software
Ex: Page Fault, Context Switch, etc
```





## page fault

Linux 内核给每个进程都提供了一个独立的虚拟地址空间，并且这个地址空间是连续的。这样，进程就可以很方便地访问内存，更确切地说是访问虚拟内存。虚拟地址空间的内部又被分为内核空间和用户空间两部分。并不是所有的虚拟内存都会分配物理内存，只有那些实际使用的虚拟内存才分配物理内存，并且分配后的物理内存，是通过内存映射来管理的。

内存映射，其实就是将虚拟内存地址映射到物理内存地址。为了完成内存映射，内核为每个进程都维护了一张页表，记录虚拟地址与物理地址的映射关系。页表实际上存储在 CPU 的内存管理单元 MMU 中。而当进程访问的虚拟地址在页表中查不到时，系统会产生一个缺页异常，进入内核空间分配物理内存、更新进程页表，最后再返回用户空间，恢复进程的运行，这是一个次缺页异常(minor page fault)。minor page fault 也称为 soft page fault, 指需要访问的内存不在虚拟地址空间，但是在物理内存中，只需要MMU建立物理内存和虚拟地址空间的映射关系即可。

major page fault指需要访问的内存不在虚拟地址空间，也不在物理内存中，进入内核空间分配物理内存，更新进程页表，还需要swap从磁盘中读取数据换入物理内存中。

 

​    当进程访问它的虚拟地址空间中的PAGE时，如果这个PAGE目前还不在物理内存中，此时CPU是不能干活的，Linux会产生一个hard page fault中断。系统需要从慢速设备（如磁盘）将对应的数据PAGE读入物理内存，并建立物理内存地址与虚拟地址空间PAGE的映射关系。然后进程才能访问这部分虚拟地址空间的内存。

page fault 又分为几种，major page fault、 minor page fault、 invalid(segment fault)。

**major page fault** 也称为 hard page fault, 指需要访问的内存不在虚拟地址空间，也不在物理内存中，需要从慢速设备载入。从swap 回到物理内存也是 hard page fault。

**minor page fault** 也称为 soft page fault, 指需要访问的内存不在虚拟地址空间，但是在物理内存中，只需要MMU建立物理内存和虚拟地址空间的映射关系即可。

1. 当一个进程在调用 malloc 获取虚拟空间地址后，首次访问该地址会发生一次soft page fault。
2. 通常是多个进程访问同一个共享内存中的数据，可能某些进程还没有建立起映射关系，所以访问时会出现soft page fault

invalid fault 也称为 **segment fault**，指进程需要访问的内存地址不在它的虚拟地址空间范围内，属于越界访问，内核会报 segment fault错误。



## linux内核映像文件分类

### zImage

zImage是ARM Linux常用的一种压缩映像文件，不超过512KB。

### bzImage

big zImage，和zImage一样都是gzip压缩的。

### uImage

u-boot专用的映像文件，它是在zImage上加上一个长度为0x40的“头部”，包含了这个映像文件的类型、加载位置、生成时间、大小等信息。如果直接从zImage的0x40位置开始加载，其和zImage就没有区别。

### vmlinuz

可引导、压缩的内核。“vm”代表“virtual memory”。Linux支持虚拟内存。vmlinuz是可执行的linux内核。

### vmlinux

未压缩的linux内核，vmlinuz是vmlinux的压缩文件。

### initrd-xxx.img

initrd是initial ramdisk的缩写，initrd一般被用来临时的引导硬件到实际内核vmlinuz能够接管并继续引导的状态





## perf详解

### 1. 简介

perf工具是基于linux内核提供的**perf_event**接口工作的。

### 2. 命令行

```shell
root@ubuntu:~# perf -h

 usage: perf [--version] [--help] [OPTIONS] COMMAND [ARGS]

 The most commonly used perf commands are:
   annotate        Read perf.data (created by perf record) and display annotated code
   archive         Create archive with object files with build-ids found in perf.data file
   bench           General framework for benchmark suites
   buildid-cache   Manage build-id cache.
   buildid-list    List the buildids in a perf.data file
   c2c             Shared Data C2C/HITM Analyzer.
   config          Get and set variables in a configuration file.
   data            Data file related processing
   diff            Read perf.data files and display the differential profile
   evlist          List the event names in a perf.data file
   ftrace          simple wrapper for kernel's ftrace functionality
   inject          Filter to augment the events stream with additional information
   kallsyms        Searches running kernel for symbols
   kmem            Tool to trace/measure kernel memory properties
   kvm             Tool to trace/measure kvm guest os
   list            List all symbolic event types
   lock            Analyze lock events
   mem             Profile memory accesses
   record          Run a command and record its profile into perf.data
   report          Read perf.data (created by perf record) and display the profile
   sched           Tool to trace/measure scheduler properties (latencies)
   script          Read perf.data (created by perf record) and display trace output
   stat            Run a command and gather performance counter statistics
   test            Runs sanity tests.
   timechart       Tool to visualize total system behavior during a workload
   top             System profiling tool.
   version         display the version of perf binary
   probe           Define new dynamic tracepoints
   trace           strace inspired tool

 See 'perf help COMMAND' for more information on a specific command.
```



#### 子功能表

|               |                                                              |
| ------------- | ------------------------------------------------------------ |
| annotate      | perf annotate用于解析由perf record记录的数据文件perf.data并将代码注解显示。如果源代码开启了debug符号，则源码和汇编一起解析。如果源码未开启debug，则解析汇编代码 |
| archive       | 根据数据文件记录的build-id，将所有被采样到的elf文件打包。利用此压缩包，可以再任何机器上分析数据文件中记录的采样数据。 |
| bench         | perf中内置的benchmark。子系统：调度器和IPC机制、内存管理、NUMA调度、futex压力基准、epoll压力基准等 |
| buildid-cache | 管理perf的buildid缓存，每个elf文件都有一个独一无二的buildid。buildid被perf用来关联性能数据与elf文件。 |
| buildid-list  | 列出perf.data文件中的buildid                                 |
| c2c           | 用于调试cache to cache的false sharing问题，用于Shared Data C2C/HITM分析，可以追踪cacheline竞争问题 |
| config        | perf config用于读取和配置 .perfconfig配置文件                |
| diff          | 对比两个数据文件的差异。能够给出每个符号（函数）在热点分析上的具体差异。 |
| evlist        | 列出数据文件perf.data中所有性能事件                          |
| ftrace        | 是内核ftrace功能的简化封装，可以跟踪指定进程的内核函数调用栈 |
| inject        | 该工具读取perf record工具记录的事件流，并将其定向到标准输出  |
| kallsyms      | 查找运行中的内核符号                                         |
| kmem          | 针对内核内存（slab）子系统进行追踪测量的工具                 |
| kvm           | 用于测试kvm客户机的性能参数                                  |
| list          | 列出event事件                                                |
| lock          | 分析内核锁统计信息                                           |
| mem           | 测试内存存取性能数据                                         |
| record        | 运行一个命令，并将其数据保存到perf.data中。随后，可以使用perf report进行分析 |
| report        | 显示perf数据                                                 |
| sched         | 分析调度器性能                                               |
| script        | 执行测试脚本                                                 |
| stat          | perf stat能完整统计应用整个生命周期的信息                    |
| test          | 用于sanity test                                              |
| timechart     | 生成图标                                                     |
| top           | 类似linux的top命令，查看整体性能                             |
| version       | 查看版本信息                                                 |
| probe         | 动态监测点                                                   |
| trace         | 跟踪系统调用                                                 |
|               |                                                              |



#### 2.1 annotate

```shell
annotate中文意思：
vi. 注释；给…作注释或评注
vt. 注释；作注解
```

perf annotate用于解析由perf record记录的数据文件perf.data并将代码注解显示。如果源代码开启了debug符号，则源码和汇编一起解析。如果源码未开启debug，则解析汇编代码。

##### 用法

```shell
 Usage: perf annotate [<options>]

    -C, --cpu <cpu>       list of cpus to profile
    -d, --dsos <dso[,dso...]>
                          only consider symbols in these dsos
    -D, --dump-raw-trace  dump raw trace in ASCII
    -f, --force           don't complain, do it
    -i, --input <file>    input file name
    -k, --vmlinux <file>  vmlinux pathname
    -l, --print-line      print matching source lines (may be slow)
    -M, --disassembler-style <disassembler style>
                          Specify disassembler style (e.g. -M intel for intel syntax)
    -m, --modules         load module symbols - WARNING: use only with -k and LIVE kernel
    -n, --show-nr-samples
                          Show a column with the number of samples
    -P, --full-paths      Don't shorten the displayed pathnames
    -q, --quiet           do now show any message
    -s, --symbol <symbol>
                          symbol to annotate
    -v, --verbose         be more verbose (show symbol address, etc)
        --asm-raw         Display raw encoding of assembly instructions (default)
        --group           Show event group information together
        --group           Show event group information together
        --gtk             Use the GTK interface
        --ignore-vmlinux  don't load vmlinux even if found
        --objdump <path>  objdump binary to use for disassembly and annotations
        --percent-type <local-period>
                          Set percent type local/global-period/hits
        --show-total-period
                          Show a column with the sum of periods
        --skip-missing    Skip symbols that cannot be annotated
        --source          Interleave source code with assembly code (default)
        --stdio           Use the stdio interface
        --stdio-color <mode>
                          'always' (default), 'never' or 'auto' only applicable to --stdio mode
        --stdio2          Use the stdio interface
        --symfs <directory>
                          Look for files with symbols relative to this directory
        --tui             Use the TUI interface
```



##### **举例**

实验`perf annotate -i perf.data -C0`，其结果：

![image-20201209142209833](https://raw.githubusercontent.com/junixcn/images/master/image-20201209142209833.png)



#### 2.2 archive

根据数据文件记录的build-id，将所有被采样到的elf文件打包。利用此压缩包，可以再任何机器上分析数据文件中记录的采样数据。

该命令需要perf buildid-list --with-hits配合使用。

##### 用法

```shell
perf archive [file]
```

##### 举例

**没搞清楚是怎么用的，总是报错**

https://linux-perf-users.vger.kernel.narkive.com/gjAAds7D/perf-archive-is-not-a-perf-command

照网上上面这个例子，给cflags加上buildid和fno-xxx参数，还是不行

```shell
root@ubuntu:test# make
gcc -g -Wl,--build-id -fno-omit-frame-pointer -o t1 test.c
root@ubuntu:test# perf record -e cpu-clock ./t1 
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.017 MB perf.data (207 samples) ]
root@ubuntu:test# perf buildid-list -i perf.data 
8b5069415e14c65b746661feb0b23246a1d44ea7 [kernel.kallsyms]
e22e5fae1bd7e9508834fdfce490ba5b12f6bcf6 /root/test/t1
cbbbd6f042b731b98a8df7ecc2de408198cf3506 [vdso]
root@ubuntu:test# perf archive perf.data 
perf: 'archive' is not a perf-command. See 'perf --help'.
root@ubuntu:test# perf archive 
perf: 'archive' is not a perf-command. See 'perf --help'.
root@ubuntu:test# perf buildid-list -i perf.data -H
8b5069415e14c65b746661feb0b23246a1d44ea7 /proc/kcore
e22e5fae1bd7e9508834fdfce490ba5b12f6bcf6 /root/test/t1
```



#### 2.3 bench

除了调度器之外，很多时候人们都需要衡量自己的工作对系统性能的影响。benchmark 是衡量性能的标准方法，对于同一个目标，如果能够有一个大家都承认的 benchmark，将非常有助于”提高内核性能”这项工作

benchmark：基准测试

##### 用法

```shell
root@ubuntu:~# perf bench -h
# benchmark：基准
 Usage: perf bench [<common options>] <collection> <benchmark> [<options>]

    -f, --format <default|simple>
                          Specify the output formatting style
    -r, --repeat <n>      Specify amount of times to repeat the run

 #使用方法
 # perf bench [<common options>] <subsystem> <suite> [<options>]
 # subsystem子系统包括有sched、mem、numa、futex、epoll以及all选项；
```

| 子系统 | 说明                |
| ------ | ------------------- |
| sched  | 测试调度器和IPC机制 |
| mem    | 测试内存性能        |
| numa   | NUMA内存和调度      |
| futex  | futex压力测试       |
| epoll  | epoll压力测试       |
| all    | 所有benchmark子系统 |



##### 举例：

##### **2.3.1 sched**

+ **sched message** 是从经典的测试程序 hackbench 移植而来，用来衡量调度器的性能，overhead 以及可扩展性。该 benchmark 启动 N 个 reader/sender 进程或线程对，通过 IPC(socket 或者 pipe) 进行并发的读写。一般人们将 N 不断加大来衡量调度器的可扩展性。Sched message 的用法及用途和 hackbench 一样



+ **sched pipe** 从 Ingo Molnar 的 pipe-test-1m.c 移植而来。当初 Ingo 的原始程序是为了测试不同的调度器的性能和公平性的。其工作原理很简单，两个进程互相通过 pipe 拼命地发 1000000 个整数，进程 A 发给 B，同时 B 发给 A。。。因为 A 和 B 互相依赖，因此假如调度器不公平，对 A 比 B 好，那么 A 和 B 整体所需要的时间就会更长。



本地虚拟机和树莓派4B数据对比

![虚拟机和树莓派4B进行对比](https://raw.githubusercontent.com/junixcn/images/master/image-20201211155615950.png)



##### **2.3.2 mem**

这个是 perf bench 的作者 Hitoshi Mitake 自己写的一个执行 memcpy 的 benchmark。该测试衡量一个拷贝 1M 数据的 memcpy() 函数所花费的时间。我尚不明白该 benchmark 的使用场景。。。或许是一个例子，告诉人们如何利用 perf bench 框架开发更多的 benchmark 吧。

* **memcpy**

  用于评估简单的内存复制性能

* **memset**

  用于简单评估内存写性能



![image-20201211162923972](https://raw.githubusercontent.com/junixcn/images/master/image-20201211162923972.png)



##### **2.3.3 numa**

NUMA(Non Uniform Memory Access)即非一致内存访问架构，市面上主要有X86_64(JASPER)和MIPS64(XLP)体系。

**测试：**

`perf bench numa mem`



**NUMA架构介绍**

**1. SMP vs AMP**

- [SMP(Symmetric Multiprocessing)](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)， 即对称多处理器架构，是目前最常见的多处理器计算机架构。
- [AMP(Asymmetric Multiprocessing)](https://en.wikipedia.org/wiki/Asymmetric_multiprocessing)， 即非对称多处理器架构，则是与SMP相对的概念。

那么两者之间的主要区别是什么呢？ 总结下来有这么几点，

1. SMP的多个处理器都是同构的，使用相同架构的CPU；而AMP的多个处理器则可能是异构的。
2. SMP的多个处理器共享同一内存地址空间；而AMP的每个处理器则拥有自己独立的地址空间。
3. SMP的多个处理器操通常共享一个操作系统的实例；而AMP的每个处理器可以有或者没有运行操作系统， 运行操作系统的CPU也是在运行多个独立的实例。
4. SMP的多处理器之间可以通过共享内存来协同通信；而AMP则需要提供一种处理器间的通信机制。

现今主流的x86多处理器服务器都是SMP架构的， 而很多嵌入式系统则是AMP架构的

**2. NUMA vs UMA**

[NUMA(Non-Uniform Memory Access)](https://en.wikipedia.org/wiki/Non-uniform_memory_access) 非均匀内存访问架构是指多处理器系统中，内存的访问时间是依赖于处理器和内存之间的相对位置的。 这种设计里存在和处理器相对近的内存，通常被称作本地内存；还有和处理器相对远的内存， 通常被称为非本地内存。

[UMA(Uniform Memory Access)](https://en.wikipedia.org/wiki/Uniform_memory_access) 均匀内存访问架构则是与NUMA相反，所以处理器对共享内存的访问距离和时间是相同的。

由此可知，不论是NUMA还是UMA都是SMP架构的一种设计和实现上的选择。

阅读文档时，也常常能看到**ccNUMA(Cache Coherent NUMA)**，即缓存一致性NUMA架构。 这种架构主要是在NUMA架构之上保证了多处理器之间的缓存一致性。降低了系统程序的编写难度。

x86多处理器发展历史上，早期的多核和多处理器系统都是UMA架构的。这种架构下， 多个CPU通过同一个北桥(North Bridge)芯片与内存链接。北桥芯片里集成了内存控制器(Memory Controller)，



参考：https://houmin.cc/posts/b893097a/



##### **2.3.4 futex**

Futex 是Fast Userspace muTexes的缩写。

Futex按英文翻译过来就是快速用户空间互斥体。其设计思想其实 不难理解，在传统的Unix系统中，System V IPC(inter process communication)，如 semaphores, msgqueues, sockets还有文件锁机制(flock())等进程间同步机制都是对一个内核对象操作来完成的，这个内核对象对要同步的进程都是可见的，其提供了共享 的状态信息和原子操作。当进程间要同步的时候必须要通过系统调用(如semop())在内核中完成。可是经研究发现，很多同步是无竞争的，即某个进程进入 互斥区，到再从某个互斥区出来这段时间，常常是没有进程也要进这个互斥区或者请求同一同步变量的。但是在这种情况下，这个进程也要陷入内核去看看有没有人 和它竞争，退出的时侯还要陷入内核去看看有没有进程等待在同一同步变量上。这些不必要的系统调用(或者说内核陷入)造成了大量的性能开销。为了解决这个问 题，Futex就应运而生，Futex是一种用户态和内核态混合的同步机制。首先，同步的进程间通过mmap共享一段内存，futex变量就位于这段共享 的内存中且操作是原子的，当进程尝试进入互斥区或者退出互斥区的时候，先去查看共享内存中的futex变量，如果没有竞争发生，则只修改futex,而不 用再执行系统调用了。当通过访问futex变量告诉进程有竞争发生，则还是得执行系统调用去完成相应的处理(wait 或者 wake up)。简单的说，futex就是通过在用户态的检查，（motivation）如果了解到没有竞争就不用陷入内核了，大大提高了low-contention时候的效率。 Linux从2.5.7开始支持Futex。

Futex是一种用户态和内核态混合机制，所以需要两个部分合作完成，linux上提供了sys_futex系统调用，对进程竞争情况下的同步处理提供支持。

所有的futex同步操作都应该从用户空间开始，首先创建一个futex同步变量，也就是位于共享内存的一个整型计数器。

当进程尝试持有锁或者要进入互斥区的时候，对futex执行"down"操作，即原子性的给futex同步变量减1。如果同步变量变为0，则没有竞争发生， 进程照常执行。

如果同步变量是个负数，则意味着有竞争发生，需要调用futex系统调用的futex_wait操作休眠当前进程。

当进程释放锁或 者要离开互斥区的时候，对futex进行"up"操作，即原子性的给futex同步变量加1。如果同步变量由0变成1，则没有竞争发生，进程照常执行。

如果加之前同步变量是负数，则意味着有竞争发生，需要调用futex系统调用的futex_wake操作唤醒一个或者多个等待进程。

|               |                                          |
| ------------- | ---------------------------------------- |
| hash          | 评估哈希表性能                           |
| wake          | Suite for evaluating wake calls          |
| wake-parallel | Suite for evaluating parallel wake calls |
| requeue       | Suite for evaluating requeue calls       |
| lock-pi       | Suite for evaluating futex lock_pi calls |

![futex对比](https://raw.githubusercontent.com/junixcn/images/master/image-20201211174849540.png)



##### **2.3.5 epoll**

select的改进者；

|      |                                                  |
| ---- | ------------------------------------------------ |
| wait | Suite for evaluating concurrent epoll_wait calls |
| ctl  | Suite for evaluating multiple epoll_ctl calls    |



![image-20201211180637168](https://raw.githubusercontent.com/junixcn/images/master/image-20201211180637168.png)



#### 2.4 buildid-cache

管理perf的buildid缓存，每个elf文件都有一个独一无二的buildid。buildid被perf用来关联性能数据与elf文件。

##### 用法

```shell
[root@localhost jrg]# perf buildid-cache -h

 Usage: perf buildid-cache [<options>]

    -a, --add <file list>
                          file(s) to add
    -f, --force           don't complain, do it
    -k, --kcore <file>    kcore file to add
    -l, --list            list all cached files
    -M, --missing <file>  to find missing build ids in the cache
    -p, --purge <file list>
                          file(s) to remove (remove old caches too)
    -P, --purge-all       purge all cached files
    -r, --remove <file list>
                          file(s) to remove
    -u, --update <file list>
                          file(s) to update
    -v, --verbose         be more verbose
        --target-ns <n>   target pid for namespace context
```





#### 2.5 buildid-list

列出perf.data中的buildids。

##### 用法

```shell
[root@localhost jrg]# perf buildid-list -h

 Usage: perf buildid-list [<options>]

    -f, --force           don't complain, do it
    -H, --with-hits       Show only DSOs with hits
    -i, --input <file>    input file name
    -k, --kernel          Show current kernel build id
    -v, --verbose         be more verbose
```



##### 举例

```shell
root@ubuntu:~# perf buildid-list -i perf.data 
8b5069415e14c65b746661feb0b23246a1d44ea7 [kernel.kallsyms]
cbbbd6f042b731b98a8df7ecc2de408198cf3506 [vdso]
705933c4b146d0227e659a71b02f8fc187f20029 /usr/lib/x86_64-linux-gnu/libglib-2.0.so.0.6400.3
```



**何为build id?**

是二进制文件的头部位和section内容计算出来的160位SHA-1算法值。

The build ID is a 160-bit SHA1 string computed over the elf header bits and section contents in the file. It is bundled in the elf file as an entry in the notes section.

```shell
+----------------+
|     namesz     |   32-bit, size of "name" field
+----------------+
|     descsz     |   32-bit, size of "desc" field
+----------------+
|      type      |   32-bit, vendor specific "type"
+----------------+
|      name      |   "namesz" bytes, null-terminated string
+----------------+
|      desc      |   "descsz" bytes, binary data
+----------------+
```

In GCC, you can enable build IDs with the `-Wl,--build-id` which passes the `--build-id` flag to the linker. You can then read it back by dumping the notes section of the resulting elf file with `readelf -n`

**build id有何用？**

* 当调试设备时，给出了一堆debug符号信息时，可以用于定位指定build id的二进制文件符号信息
* 用于区别二进制文件

**build id举例：**

```shell
root@ubuntu:test# make clean
root@ubuntu:test# make
gcc -g -Wl,--build-id -o t1 test.c
root@ubuntu:test# ls
Makefile  perf.data  t1  test.c
root@ubuntu:test# readelf -n t1 

Displaying notes found in: .note.gnu.property
  Owner                Data size 	Description
  GNU                  0x00000010	NT_GNU_PROPERTY_TYPE_0
      Properties: x86 feature: IBT, SHSTK

Displaying notes found in: .note.gnu.build-id
  Owner                Data size 	Description
  GNU                  0x00000014	NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: 104006ed448e657d6a4160f3718a72231df54b06

Displaying notes found in: .note.ABI-tag
  Owner                Data size 	Description
  GNU                  0x00000010	NT_GNU_ABI_TAG (ABI version tag)
    OS: Linux, ABI: 3.2.0
root@ubuntu:test# rm perf.data 
root@ubuntu:test# perf record -e cpu-clock ./t1 
now into main
now foo1 over
now foo2 over
now main over
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.018 MB perf.data (222 samples) ]
root@ubuntu:test# perf report
root@ubuntu:test# perf archive
perf: 'archive' is not a perf-command. See 'perf --help'.
root@ubuntu:test# perf buildid-list -i perf.data 
8b5069415e14c65b746661feb0b23246a1d44ea7 [kernel.kallsyms]
104006ed448e657d6a4160f3718a72231df54b06 /root/test/t1
cbbbd6f042b731b98a8df7ecc2de408198cf3506 [vdso]
```



#### 2.6 c2c

CPU缓存机制参考：https://www.cnblogs.com/jokerjason/p/10711022.html

perf c2c的使用主要参考博客：https://joemario.github.io/blog/2016/09/01/c2c-blog/

shared data c2c/HITM分析。

cache to cache：

缓存的命中率，是 CPU 性能的一个关键性能指标。我们知道，CPU 里面有好几级缓存（Cache），每一级缓存都比后面一级缓存访问速度快。最后一级缓存叫 LLC（Last Level Cache）；LLC 的后面就是内存。

当 CPU 需要访问一块数据或者指令时，它会首先查看最靠近的一级缓存（L1）；如果数据存在，那么就是缓存命中（Cache Hit），否则就是不命中（Cache Miss），需要继续查询下一级缓存。



c2c用来检测cache共享命中失败，一个处理器修改了某个cache line中的数据，另一个处理器访问该cache line数据时需要refresh该cache line， perf c2c命令就用来调试这里问题。



At a high level, “perf c2c” will show you:
\* The cachelines where false sharing was detected.
\* The readers and writers to those cachelines, and the offsets where those accesses occurred.
\* The pid, tid, instruction addr, function name, binary object name for those readers and writers.
\* The source file and line number for each reader and writer.
\* The average load latency for the loads to those cachelines.
\* Which numa nodes the samples a cacheline came from and which cpus were involved



**HITM：**

Notice the term “HITM”, which stands for a load that hit in a modified cacheline.这就是false sharing产生的地方。

**Remote HITMs：**

meaning across numa nodes, are the most expensive - especially when there are lots of readers and writers.



perf c2c report输出含义：

```shell
 1  =================================================
 2              Trace Event Information
 3  =================================================
 4    Total records                     :     329219  << Total loads and stores sampled.
 5    Locked Load/Store Operations      :      14654
 6    Load Operations                   :      69679  << Total loads
 7    Loads - uncacheable               :          0
 8    Loads - IO                        :          0
 9    Loads - Miss                      :       3972
10    Loads - no mapping                :          0
11    Load Fill Buffer Hit              :      11958
12    Load L1D hit                      :      17235  << loads that hit in the L1 cache.
13    Load L2D hit                      :         21
14    Load LLC hit                      :      14219  << loads that hit in the last level cache (LLC).
15    Load Local HITM                   :       3402  << loads that hit in a modified cache on the same numa node (local HITM).
16    Load Remote HITM                  :      12757  << loads that hit in a modified cache on a remote numa node (remote HITM).
17    Load Remote HIT                   :       5295
18    Load Local DRAM                   :        976  << loads that hit in the local node's main memory.
19    Load Remote DRAM                  :       3246  << loads that hit in a remote node's main memory.
20    Load MESI State Exclusive         :       4222 
21    Load MESI State Shared            :          0
22    Load LLC Misses                   :      22274  << loads not found in any local node caches.
23    LLC Misses to Local DRAM          :        4.4% << % hitting in local node's main memory.
24    LLC Misses to Remote DRAM         :       14.6% << % hitting in a remote node's main memory.
25    LLC Misses to Remote cache (HIT)  :       23.8% << % hitting in a clean cache in a remote node.
26    LLC Misses to Remote cache (HITM) :       57.3% << % hitting in remote modified cache. (most expensive - false sharing)
27    Store Operations                  :     259539  << store instruction sample count
28    Store - uncacheable               :          0
29    Store - no mapping                :         11
30    Store L1D Hit                     :     256696  << stores that got L1 cache when requested.
31    Store L1D Miss                    :       2832  << stores that couldn't get the L1 cache when requested (L1 miss).
32    No Page Map Rejects               :       2376
33    Unable to parse data source       :          1
```

**shared data cache line table**表格显示了fasle sharing发生的地方，它列表是按照拥有的HITMs数量从高到低排列显示的，

**Shared Cache Line Distribution Pareto**是最重要的表，它展示了每个冲突竞争的cacheline的详细信息。

 

##### **举例**

参考文章：https://www.bookstack.cn/read/perf-little-book/posts-check-cache-false-sharing.md

对比下数据：

* test.c

```c
#include <omp.h>
#define N 100000000
#define THRAED_NUM 8
int values[N];
int main(void)
{
    int sum[THRAED_NUM];
    #pragma omp parallel for
    for (int i = 0; i < THRAED_NUM; i++)
    {
        //int local_sum;
        for (int j = 0; j < N; j++)
        {
            //local_sum += values[j] >> i;
            sum[i] += values[j] >> i;
        }
        //sum[i] = local_sum;
    }
    return 0;
}
```

编译：

```shell
gcc -fopenmp -g test.c -o test
```

数据：

```shell

[root@localhost test]# gcc -fopenmp -g test.c -o test
[root@localhost test]# perf c2c record ./test
[ perf record: Woken up 139 times to write data ]
[ perf record: Captured and wrote 36.329 MB perf.data (424292 samples) ]
[root@localhost test]# perf c2c report -i perf.data --stdio

=================================================
            Trace Event Information
=================================================
  Total records                     :     424292
  Locked Load/Store Operations      :        129
  Load Operations                   :     182641
  Loads - uncacheable               :          5
  Loads - IO                        :          0
  Loads - Miss                      :          2
  Loads - no mapping                :          0
  Load Fill Buffer Hit              :     153672
  Load L1D hit                      :      26947
  Load L2D hit                      :        278
  Load LLC hit                      :       1263
  Load Local HITM                   :         78
  Load Remote HITM                  :        456
  Load Remote HIT                   :          6
  Load Local DRAM                   :         10
  Load Remote DRAM                  :        464
  Load MESI State Exclusive         :        468
  Load MESI State Shared            :          6
  Load LLC Misses                   :        936
  LLC Misses to Local DRAM          :        1.1%
  LLC Misses to Remote DRAM         :       49.6%
  LLC Misses to Remote cache (HIT)  :        0.6%
  LLC Misses to Remote cache (HITM) :       48.7%
  Store Operations                  :     241651
  Store - uncacheable               :          0
  Store - no mapping                :         32
  Store L1D Hit                     :     212012
  Store L1D Miss                    :      29607
  No Page Map Rejects               :       5684
  Unable to parse data source       :          0

=================================================
    Global Shared Cache Line Event Information
=================================================
  Total Shared Cache Lines          :         35
  Load HITs on shared lines         :     180894
  Fill Buffer Hits on shared lines  :     153200
  L1D hits on shared lines          :      26135
  L2D hits on shared lines          :        240
  LLC hits on shared lines          :        859
  Locked Access on shared lines     :          4
  Store HITs on shared lines        :     219944
  Store L1D hits on shared lines    :     194838
  Total Merged records              :     220478

```

![image-20201214154611317](https://raw.githubusercontent.com/junixcn/images/master/image-20201214154611317.png)

![image-20201214154718036](https://raw.githubusercontent.com/junixcn/images/master/image-20201214154718036.png)

数据中显示，程序的第15行，有问题；



* test.c

```c
#include <omp.h>
#define N 100000000
#define THRAED_NUM 8
int values[N];
int main(void)
{
    int sum[THRAED_NUM];
    #pragma omp parallel for
    for (int i = 0; i < THRAED_NUM; i++)
    {
        int local_sum;
        for (int j = 0; j < N; j++)
        {
            local_sum += values[j] >> i;
            //sum[i] += values[j] >> i;
        }
        sum[i] = local_sum;
    }
    return 0;
}
```

数据：

```shell
[root@localhost test]# perf c2c record ./test
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 1.741 MB perf.data (20209 samples) ]

=================================================
            Trace Event Information
=================================================
  Total records                     :      20052
  Locked Load/Store Operations      :         28
  Load Operations                   :       9740
  Loads - uncacheable               :          0
  Loads - IO                        :          0
  Loads - Miss                      :          0
  Loads - no mapping                :          1
  Load Fill Buffer Hit              :        102
  Load L1D hit                      :       9517
  Load L2D hit                      :         14
  Load LLC hit                      :         99
  Load Local HITM                   :          4
  Load Remote HITM                  :          3
  Load Remote HIT                   :          0
  Load Local DRAM                   :          2
  Load Remote DRAM                  :          5
  Load MESI State Exclusive         :          7
  Load MESI State Shared            :          0
  Load LLC Misses                   :         10
  LLC Misses to Local DRAM          :       20.0%
  LLC Misses to Remote DRAM         :       50.0%
  LLC Misses to Remote cache (HIT)  :        0.0%
  LLC Misses to Remote cache (HITM) :       30.0%
  Store Operations                  :      10312
  Store - uncacheable               :          0
  Store - no mapping                :         26
  Store L1D Hit                     :      10107
  Store L1D Miss                    :        179
  No Page Map Rejects               :        397
  Unable to parse data source       :          0

=================================================
    Global Shared Cache Line Event Information
=================================================
  Total Shared Cache Lines          :          7
  Load HITs on shared lines         :          8
  Fill Buffer Hits on shared lines  :          1
  L1D hits on shared lines          :          0
  L2D hits on shared lines          :          0
  LLC hits on shared lines          :          4
  Locked Access on shared lines     :          0
  Store HITs on shared lines        :          0
  Store L1D hits on shared lines    :          0
  Total Merged records              :          7

=================================================
                 c2c details
=================================================
  Events                            : cpu/mem-loads,ldlat=30/P
                                    : cpu/mem-stores/P
  Cachelines sort on                : Total HITMs
  Cacheline data grouping           : offset,iaddr

=================================================

```

![image-20201214155321846](https://raw.githubusercontent.com/junixcn/images/master/image-20201214155321846.png)



cache to cache这部分还是不理解，需要找计算机的专业书籍来看看；



#### **2.7 config**

perf config用于读取和配置 .perfconfig配置文件， name = value 格式；

##### 用法

```shell
 Usage: perf config [<file-option>] [options] [section.name[=value] ...]

    -l, --list            show current config variables
        --system          use system config file
        --user            use user config file
```



#### 2.8 data

perf data可以将perf数据文件格式转换为其他格式（目前只支持ctf格式）；

##### 用法

```shell
Usage: perf data [<common options>] <command> [<options>]
```

##### 举例

```shell
root@ubuntu:~# perf data convert --to-ctf
No conversion support compiled in. perf should be compiled with environment variables LIBBABELTRACE=1 and LIBBABELTRACE_DIR=/path/to/libbabeltrace/
```



#### 2.9 diff

This command displays the performance difference amongst two or more perf.data files captured via perf record.

用于比较多个perf.data数据之间的不同。如果没有输入参数，默认比较perf.data.old和perf.data文件；

##### 用法

```shell
Usage: perf diff [<options>] [old_file] [new_file]

    -b, --baseline-only   Show only items with match in baseline
    -C, --comms <comm[,comm...]>
                          only consider symbols in these comms
    -c, --compute <delta,delta-abs,ratio,wdiff:w1,w2 (default delta-abs),cycles>
                          Entries differential computation selection
    -d, --dsos <dso[,dso...]>
                          only consider symbols in these dsos
    -D, --dump-raw-trace  dump raw trace in ASCII
    -f, --force           don't complain, do it
    -F, --formula         Show formula.
    -m, --modules         load module symbols - WARNING: use only with -k and LIVE kernel
    -o, --order <n>       Specify compute sorting.
    -p, --period          Show period values.
    -q, --quiet           Do not show any message
    -s, --sort <key[,key2...]>
                          sort by key(s): pid, comm, dso, symbol, parent, cpu, srcline, ... Please refer the man page for the complete list.
    -S, --symbols <symbol[,symbol...]>
                          only consider these symbols
    -t, --field-separator <separator>
                          separator for columns, no spaces will be added between columns '.' is reserved.
    -v, --verbose         be more verbose (show symbol address, etc)
        --cpu <cpu>       list of cpus to profile
        --kallsyms <file>
                          kallsyms pathname
        --percentage <relative|absolute>
                          How to display percentage of filtered entries
        --pid <pid[,pid...]>
                          only consider symbols in these pids
        --symfs <directory>
                          Look for files with symbols relative to this directory
        --tid <tid[,tid...]>
                          only consider symbols in these tids
        --time <str>      Time span (time percent or absolute timestamp)
```



##### 举例

```shell
[root@localhost test]# ls
perf.data  perf.data.old  test  test.c
[root@localhost test]# perf diff perf.data.old perf.data --cpu 0
# Event 'cpu/mem-loads,ldlat=30/P'
#
# Baseline  Delta Abs  Shared Object      Symbol
# ........  .........  .................  ............................
#
    77.78%             [kernel.kallsyms]  [k] find_busiest_group
    11.11%             [kernel.kallsyms]  [k] __update_load_avg_cfs_rq
    11.11%             [kernel.kallsyms]  [k] account_user_time

# Event 'cpu/mem-stores/P'
#
# Baseline  Delta Abs  Shared Object      Symbol
# ........  .........  .................  ..........................
#
    54.13%             [kernel.kallsyms]  [k] tick_sched_timer
    37.15%             [kernel.kallsyms]  [k] repeat_nmi
     4.59%             [kernel.kallsyms]  [k] __native_set_fixmap
     3.36%             [kernel.kallsyms]  [k] finish_task_switch
     0.39%             [kernel.kallsyms]  [k] perf_event_nmi_handler
     0.29%             [kernel.kallsyms]  [k] nmi_handle
     0.09%             [kernel.kallsyms]  [k] native_apic_mem_write

```



#### 2.10 evlist

列出数据文件perf.data中所有性能事件。

##### 用法

```shell
 Usage: perf evlist [<options>]

    -f, --force           don't complain, do it
    -F, --freq            Show the sample frequency
    -g, --group           Show event group information
    -i, --input <file>    Input file name
    -v, --verbose         Show all event attr details
        --trace-fields    Show tracepoint fields
```



##### 举例

```shell
[root@localhost test]# perf evlist -i perf.data
cpu/mem-loads,ldlat=30/P
cpu/mem-stores/P
```



#### 2.11 ftrace

是内核ftrace功能的简化封装，可以跟踪指定进程的内核函数调用栈。目前perf ftrace只支持单线程跟踪，仅仅是从ftrace管道中读取数据输出到标准输出。

##### 用法

```shell
 Usage: perf ftrace [<options>] [<command>]
    or: perf ftrace [<options>] -- <command> [<options>]

    -a, --all-cpus        system-wide collection from all CPUs
    -C, --cpu <cpu>       list of cpus to monitor
    -D, --graph-depth <n>
                          Max depth for function graph tracer
    -G, --graph-funcs <func>
                          Set graph filter on given functions
    -g, --nograph-funcs <func>
                          Set nograph filter on given functions
    -N, --notrace-funcs <func>
                          do not trace given functions
    -p, --pid <pid>       trace on existing process id
    -T, --trace-funcs <func>
                          trace given functions only
    -t, --tracer <tracer>
                          tracer to use: function_graph(default) or function
    -v, --verbose         be more verbose
```

##### 举例

```shell
[root@localhost jrg]# perf ftrace -g -p 120058
  2)               |  switch_mm_irqs_off() {
  2)   0.327 us    |    load_new_mm_cr3();
  2)   2.808 us    |  }
 ------------------------------------------
  2)    <idle>-0    =>  <...>-122943
 ------------------------------------------

  2)               |  finish_task_switch() {
  2)   ==========> |
  2)               |    smp_irq_work_interrupt() {
  2)               |      irq_enter() {
  2)   0.052 us    |        rcu_irq_enter();
  2)   0.096 us    |        irqtime_account_irq();
  2)   1.029 us    |      }
  2)               |      __wake_up() {
  2)               |        __wake_up_common_lock() {
  2)   0.128 us    |          _raw_spin_lock_irqsave();
  2)   0.057 us    |          __wake_up_common();
  2)   0.048 us    |          _raw_spin_unlock_irqrestore();
  2)   1.842 us    |        }
  2)   2.550 us    |      }
  2)               |      irq_exit() {
  2)   0.080 us    |        irqtime_account_irq();
  2)   0.037 us    |        idle_cpu();
  2)   0.035 us    |        rcu_irq_exit();
  2)   1.136 us    |      }
  2)   6.958 us    |    }
  2)   <========== |
  2)   8.031 us    |  }
  2)   0.047 us    |  finish_wait();
  2)               |  mutex_lock() {
  2)               |    _cond_resched() {
  2)   0.033 us    |      rcu_all_qs();
  2)   0.361 us    |    }
  2)   0.766 us    |  }
  2)   0.031 us    |  generic_pipe_buf_confirm();
  2)               |  _cond_resched() {
  2)   0.032 us    |    rcu_all_qs();
  2)   0.360 us    |  }
  2)   0.034 us    |  anon_pipe_buf_release();
  2)   0.049 us    |  mutex_unlock();
  2)               |  __wake_up_sync_key() {
  2)               |    __wake_up_common_lock() {
  2)   0.035 us    |      _raw_spin_lock_irqsave();
  2)   0.036 us    |      __wake_up_common();
  2)   0.045 us    |      _raw_spin_unlock_irqrestore();
  2)   1.069 us    |    }
  2)   1.397 us    |  }
  2)   0.042 us    |  kill_fasync();
  2)               |  touch_atime() {
  2)               |    atime_needs_update() {
  2)               |      current_time() {
  2)   0.034 us    |        ktime_get_coarse_real_ts64();
  2)   0.042 us    |        timespec64_trunc();
  2)   0.740 us    |      }
  2)   1.277 us    |    }
  2)   0.163 us    |    __sb_start_write();
  2)   0.116 us    |    __mnt_want_write();
```



#### 2.12 inject

该工具读取perf record工具记录的事件流，并将其定向到标准输出。在被分析代码中的任何一点，都可以向事件流中注入其它事件。

##### 用法

```shell

 Usage: perf inject [<options>]

    -b, --build-ids       Inject build-ids into the output stream
    -f, --force           don't complain, do it
    -i, --input <file>    input file name
    -j, --jit             merge jitdump files into perf.data file
    -o, --output <file>   output file name
    -s, --sched-stat      Merge sched-stat and sched-switch for getting events where and how long tasks slept
    -v, --verbose         be more verbose (show build ids, etc)
        --itrace[=<opts>]
                          Instruction Tracing options
                                i:                      synthesize instructions events
                                b:                      synthesize branches events
                                c:                      synthesize branches events (calls only)
                                r:                      synthesize branches events (returns only)
                                x:                      synthesize transactions events
                                w:                      synthesize ptwrite events
                                p:                      synthesize power events
                                e:                      synthesize error events
                                d:                      create a debug log
                                g[len]:                 synthesize a call chain (use with i or x)
                                l[len]:                 synthesize last branch entries (use with i or x)
                                sNUMBER:                skip initial number of events
                                PERIOD[ns|us|ms|i|t]:   specify period to sample stream
                                concatenate multiple options. Default is ibxwpe or cewp

        --kallsyms <file>
                          kallsyms pathname
        --strip           strip non-synthesized events (use with --itrace)
```



##### 举例

```shell
[root@localhost test]# ls
perf.data  perf.data.old  test  test.c
[root@localhost test]# perf inject -i perf.data --jit -o perf.data.jitted
[root@localhost test]# ls
perf.data  perf.data.jitted  perf.data.old  test  test.c
```



#### 2.13 kallsyms

查找运行中的内核符号；

##### 用法

```shell
root@ubuntu:python# perf kallsyms -h

 Usage: perf kallsyms [<options>] symbol_name

    -v, --verbose         be more verbose (show counter open errors, etc)
```

##### 举例

```shellroot@ubuntu:python# perf kallsyms -v vmw_cmdbuf_header_submit
root@ubuntu:python# perf kallsyms -v vmw_cmdbuf_header_submit
vmw_cmdbuf_header_submit: [vmwgfx] /lib/modules/5.4.0-56-generic/kernel/drivers/gpu/drm/vmwgfx/vmwgfx.ko 0xffffffffc040c300-0xffffffffc040c3a7 (0x1b380-0x1b427)
```



#### 2.14 kmem

针对内核内存（slab）子系统进行追踪测量的工具。

比如内存分配/释放等。可以用来研究程序在哪里分配了大量内存，或者在什么地方产生碎片之类的和内存管理相关的问题。

perf kmem和perf lock实际上都是perf tracepoint的子类，等同于perf record -e kmem:*和perf record -e lock:*。

但是这些工具在内部队员是数据进行了慧聪和分析，因此统计报表更具可读性。

perf kmem record：抓取命令的内核slab分配器事件

perf kmem stat：生成内核slab分配器统计信息

##### 用法

```shell
 Usage: perf kmem [<options>] {record|stat}

    -f, --force           don't complain, do it
    -i, --input <file>    input file name
    -l, --line <num>      show n lines
    -s, --sort <key[,key2...]>
                          sort by keys: ptr, callsite, bytes, hit, pingpong, frag, page, order, migtype, gfp
    -v, --verbose         be more verbose (show symbol address, etc)
        --alloc           show per-allocation statistics
        --caller          show per-callsite statistics
        --live            Show live page stat
        --page            Analyze page allocator
        --raw-ip          show raw ip instead of symbol
        --slab            Analyze slab allocator
        --time <str>      Time span of interest (start,stop)

```

##### 举例

![image-20201216134713881](https://raw.githubusercontent.com/junixcn/images/master/image-20201216134713881.png)



#### 2.15 kvm

用于测试kvm客户机的性能参数。

##### 用法

```shell
 Usage: perf kvm [<options>] {top|record|report|diff|buildid-list|stat}

    -i, --input <file>    Input file name
    -o, --output <file>   Output file name
    -v, --verbose         be more verbose (show counter open errors, etc)
        --guest           Collect guest os data
        --guestkallsyms <file>
                          file saving guest os /proc/kallsyms
        --guestmodules <file>
                          file saving guest os /proc/modules
        --guestmount <directory>
                          guest mount directory under which every guest os instance has a subdir
        --guestvmlinux <file>
                          file saving guest os vmlinux
        --host            Collect host os data
```

##### 举例

```shell
perf kvm --host record
perf kvm --host report

Samples: 41K of event 'cycles', Event count (approx.): 8303321915
Overhead  Command          Shared Object               Symbol
  21.84%  qemu-system-x86  [kernel.kallsyms]           [k] vmx_vmexit
  18.92%  swapper          [kernel.kallsyms]           [k] intel_idle
   3.23%  qemu-system-x86  [kernel.kallsyms]           [k] do_syscall_64
   2.15%  qemu-system-x86  [kernel.kallsyms]           [k] native_write_msr
   1.49%  qemu-system-x86  [kernel.kallsyms]           [k] syscall_return_via_sysret
   1.39%  qemu-system-x86  [kernel.kallsyms]           [k] kvm_arch_vcpu_ioctl_run
   1.25%  qemu-system-x86  [kernel.kallsyms]           [k] entry_SYSCALL_64
   0.96%  qemu-system-x86  [kernel.kallsyms]           [k] kvm_put_guest_fpu
   0.90%  qemu-system-x86  [kernel.kallsyms]           [k] kvm_on_user_return
   0.86%  qemu-system-x86  [kernel.kallsyms]           [k] vcpu_enter_guest
   0.79%  qemu-system-x86  [kernel.kallsyms]           [k] native_write_msr_safe
   0.78%  qemu-system-x86  [kernel.kallsyms]           [k] vmx_vcpu_run.part.75
   0.75%  qemu-system-x86  [kernel.kallsyms]           [k] native_set_debugreg
   0.71%  qemu-system-x86  [kernel.kallsyms]           [k] __fget
   0.60%  qemu-system-x86  [kernel.kallsyms]           [k] __x86_indirect_thunk_rax
   0.55%  qemu-system-x86  qemu-system-x86_64          [.] object_dynamic_cast_assert
   0.55%  qemu-system-x86  [kernel.kallsyms]           [k] native_load_gdt
   0.52%  qemu-system-x86  [kernel.kallsyms]           [k] kvm_vcpu_ioctl
   0.51%  qemu-system-x86  [kernel.kallsyms]           [k] __audit_syscall_exit
   0.49%  qemu-system-x86  [kernel.kallsyms]           [k] vmx_prepare_switch_to_guest
   0.47%  qemu-system-x86  [kernel.kallsyms]           [k] __srcu_read_lock
...
```





#### 2.16 list

显示所有的能够触发perf采样点的事件。比如cpu-clock，task-clock，contex-switches等等。

在 2.6.35 版本的内核中，该列表已经相当的长，但无论有多少，我们可以将它们划分为三类：

Hardware Event 是由 PMU 硬件产生的事件，比如 cache 命中，当您需要了解程序对硬件特性的使用情况时，便需要对这些事件进行采样；

Software Event 是内核软件产生的事件，比如进程切换，tick 数等 ;

Tracepoint event 是内核中的静态 tracepoint 所触发的事件，这些 tracepoint 用来判断程序运行期间内核的行为细节，比如 slab 分配器的分配次数等。

##### 举例

```shell
[root@localhost jrg]# perf list

List of pre-defined events (to be used in -e):

  branch-instructions OR branches                    [Hardware event]
  branch-misses                                      [Hardware event]
  bus-cycles                                         [Hardware event]
  cache-misses                                       [Hardware event]
  cache-references                                   [Hardware event]
  cpu-cycles OR cycles                               [Hardware event]
  instructions                                       [Hardware event]
  ref-cycles                                         [Hardware event]

  alignment-faults                                   [Software event]
  bpf-output                                         [Software event]
  context-switches OR cs                             [Software event]
  cpu-clock                                          [Software event]
  cpu-migrations OR migrations                       [Software event]
  dummy                                              [Software event]
  emulation-faults                                   [Software event]
  major-faults                                       [Software event]
  minor-faults                                       [Software event]
  page-faults OR faults                              [Software event]
  task-clock                                         [Software event]

  duration_time                                      [Tool event]

  L1-dcache-load-misses                              [Hardware cache event]
  L1-dcache-loads                                    [Hardware cache event]
  L1-dcache-stores                                   [Hardware cache event]
  L1-icache-load-misses                              [Hardware cache event]
  LLC-load-misses                                    [Hardware cache event]
  LLC-loads                                          [Hardware cache event]
  LLC-store-misses                                   [Hardware cache event]
```



#### 2.17 lock

要使用此功能，内核需要编译选项的支持：**CONFIG_LOCKDEP、CONFIG_LOCK_STAT**

分析内核锁统计信息。

锁是内核用于同步的方法，一旦加了锁，其他加锁的内核执行路径就必须等待，降低了并行。同时，如果加锁不正确还会造成死锁。

因此对于内核锁进行分析是一项重要的调优工作。

##### 用法

```shell
 Usage: perf lock [<options>] {record|report|script|info}

    -D, --dump-raw-trace  dump raw trace in ASCII
    -f, --force           don't complain, do it
    -i, --input <file>    input file name
    -v, --verbose         be more verbose (show symbol address, etc)
```



#### 2.18 mem

测试内存存取性能数据。

##### 用法

```shell
 Usage: perf mem [<options>] {record|report}

    -C, --cpu <cpu>       list of cpus to profile
    -D, --dump-raw-samples
                          dump raw samples in ASCII
    -f, --force           don't complain, do it
    -i, --input <file>    input file name
    -p, --phys-data       Record/Report sample physical addresses
    -t, --type <type>     memory operations(load,store) Default load,store
    -U, --hide-unresolved
                          Only display entries resolved to a symbol
    -x, --field-separator <separator>
                          separator for columns, no spaces will be added between columns '.' is reserved.
```

##### 举例

![image-20201217172725747](https://raw.githubusercontent.com/junixcn/images/master/image-20201217172725747.png)



#### 2.19 record

运行一个命令，并将其数据保存到perf.data中。随后，可以使用perf report进行分析。

perf record和perf report可以更精确的分析一个应用，perf record可以精确到函数级别。并且在函数里面混合显示汇编语言和代码。

```shell
perf record [-e <EVENT> | --event=EVENT] [-a] <command>
perf record [-e <EVENT> | --event=EVENT] [-a] — <command> [<options>]
```



#### 2.20 report

读取perf record创建的数据文件，并给出热点分析结果。

top适合对整体性能分析，stat适合单个程序，report则可以分析更细粒度，具体到代码指令。

##### 用法

```shell
-i 导入的数据文件名称，如果没有则默认为perf.data

-g 生成函数调用关系图，**此时内核要打开CONFIG_KALLSYMS；用户空间库或者执行文件需要带符号信息(not stripped)，编译选项需要加上-g。**

--sort 从更高层面显示分类统计信息，比如： pid, comm, dso, symbol, parent, cpu,socket, srcline, weight, local_weight.
```



#### 2.21 sched

调度器的好坏直接影响一个系统的整体运行效率。在这个领域，内核黑客们常会发生争执，一个重要原因是对于不同的调度器，每个人给出的评测报告都各不相同，甚至常常有相反的结论。因此一个权威的统一的评测工具将对结束这种争论有益。Perf sched 便是这种尝试。

##### 用法

子命令：

**record**：统计数据

**latency**：输出每个任务的延迟数据

**map**：显示上下文切换的映射

**replay**： 仿真perf.data数据，可重复仿真运行测试性能

**script**：同perf script功能

**timehist**：提供scheduling事件分析报告

```shell
 Usage: perf sched [<options>] {record|latency|map|replay|script|timehist}

    -D, --dump-raw-trace  dump raw trace in ASCII
    -f, --force           don't complain, do it
    -i, --input <file>    input file name
    -v, --verbose         be more verbose (show symbol address, etc)
```

用户一般使用’ perf sched record ’收集调度相关的数据，然后就可以用’ perf sched latency ’查看诸如调度延迟等和调度器相关的统计数据。

其他几个命令也同样读取 record 收集到的数据并从其他不同的角度来展示这些数据

##### 举例

```shell
[root@localhost jrg]# perf sched record sleep 1
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 8.557 MB perf.data (49023 samples) ]



perf sched latency:

 -----------------------------------------------------------------------------------------------------------------
  Task                  |   Runtime ms  | Switches | Average delay ms | Maximum delay ms | Maximum delay at       |
 -----------------------------------------------------------------------------------------------------------------
  ksoftirqd/14:96       |      0.451 ms |        3 | avg:    2.315 ms | max:    3.981 ms | max at: 280823.049244 s
  kworker/53:0-mm:1635514 |      0.008 ms |        1 | avg:    0.293 ms | max:    0.293 ms | max at: 280823.408523 s
  kworker/49:3-mm:1693592 |      0.008 ms |        1 | avg:    0.188 ms | max:    0.188 ms | max at: 280823.134432 s
  kworker/31:0-mm:1181706 |      0.023 ms |        2 | avg:    0.065 ms | max:    0.105 ms | max at: 280823.183359 s
  kworker/41:0-ev:1674483 |      0.007 ms |        1 | avg:    0.062 ms | max:    0.062 ms | max at: 280823.256325 s
  tail:1881449          |      1.235 ms |        1 | avg:    0.061 ms | max:    0.061 ms | max at: 280823.136554 s
  kworker/40:2-ev:933115 |      0.027 ms |        2 | avg:    0.059 ms | max:    0.106 ms | max at: 280823.881309 s
  kworker/38:3-ev:1698336 |      0.019 ms |        3 | avg:    0.053 ms | max:    0.141 ms | max at: 280823.600360 s
  kworker/32:0-mm:1784542 |      0.007 ms |        1 | avg:    0.051 ms | max:    0.051 ms | max at: 280823.366285 s
  kworker/48:1-ev:1877889 |      0.052 ms |        4 | avg:    0.041 ms | max:    0.110 ms | max at: 280823.744327 s
  QuorumPeer[myid:76988 |      0.084 ms |        1 | avg:    0.034 ms | max:    0.034 ms | max at: 280823.441806 s
  who:1881451           |      1.315 ms |        1 | avg:    0.031 ms | max:    0.031 ms | max at: 280823.139770 s
  kworker/4:2-mm_:1866327 |      0.007 ms |        1 | avg:    0.031 ms | max:    0.031 ms | max at: 280823.281280 s
  kworker/29:2-mm:1823927 |      0.013 ms |        1 | avg:    0.030 ms | max:    0.030 ms | max at: 280823.263316 s
  SPICE Worker:(5)      |      7.424 ms |      172 | avg:    0.026 ms | max:    0.048 ms | max at: 280823.087379 s
  gvfsd-fuse:(2)        |      0.075 ms |        2 | avg:    0.025 ms | max:    0.045 ms | max at: 280823.139171 s
  bash:(2)              |      1.604 ms |        9 | avg:    0.023 ms | max:    0.036 ms | max at: 280823.254885 s
  kworker/u113:2-:1880846 |      0.115 ms |        6 | avg:    0.022 ms | max:    0.036 ms | max at: 280823.254836 s
  kworker/35:0-mm:1862481 |      0.017 ms |        2 | avg:    0.022 ms | max:    0.038 ms | max at: 280823.263328 s
  rngd:1631             |      0.165 ms |       14 | avg:    0.022 ms | max:    0.045 ms | max at: 280823.805026 s
...

上面latency结果中各个 column 的含义如下：
 Task: 进程的名字和 pid 
 Runtime: 实际运行时间
 Switches: 进程切换的次数
 Average delay: 平均的调度延迟
 Maximum delay: 最大延迟
这里最值得人们关注的是 Maximum delay，一般从这里可以看到对交互性影响最大的特性：调度延迟，如果调度延迟比较大，那么用户就会感受到视频或者音频断断续续的。


```



`perf sched map`结果：

![image-20201217173900465](https://raw.githubusercontent.com/junixcn/images/master/image-20201217173900465.png)

星号表示调度事件发生所在的 CPU。

点号表示该 CPU 正在 IDLE。

Map 的好处在于提供了一个的总的视图，将成百上千的调度事件进行总结，显示了系统任务在 CPU 之间的分布，假如有不好的调度迁移，比如一个任务没有被及时迁移到 idle 的 CPU 却被迁移到其他忙碌的 CPU，类似这种调度器的问题可以从 map 的报告中一眼看出来。



`perf sched replay`:

Perf replay 这个工具更是专门为调度器开发人员所设计，它试图重放 perf.data 文件中所记录的调度场景。很多情况下，一般用户假如发现调度器的奇怪行为，他们也无法准确说明发生该情形的场景，或者一些测试场景不容易再次重现，或者仅仅是出于“偷懒”的目的，使用 perf replay，perf 将模拟 perf.data 中的场景，无需开发人员花费很多的时间去重现过去，这尤其利于调试过程，因为需要一而再，再而三地重复新的修改是否能改善原始的调度场景所发现的问题。

```shell
[root@localhost jrg]# perf sched replay
run measurement overhead: 61 nsecs
sleep measurement overhead: 52656 nsecs
the run test took 1000030 nsecs
the sleep test took 1053803 nsecs
nr_run_events:        24218
nr_sleep_events:      26210
nr_wakeup_events:     12036
target-less wakeups:  16
multi-target wakeups: 53
task      0 (             swapper:         0), nr_events: 34269
task      1 (             swapper:         1), nr_events: 1
task      2 (                  :2:         2), nr_events: 1
task      3 (                 :10:        10), nr_events: 1
task      4 (                :100:       100), nr_events: 1
task      5 (               :9687:      9687), nr_events: 1
task      6 (              :10004:     10004), nr_events: 1
task      7 (           gsd-power:     10049), nr_events: 1
task      8 (           gsd-power:     10053), nr_events: 1
task      9 (           gsd-power:     10085), nr_events: 1
task     10 (              :10006:     10006), nr_events: 1
task     11 (     gsd-print-notif:     10019), nr_events: 1
task     12 (     gsd-print-notif:     10023), nr_events: 1
```





#### 2.22 script

根据脚本来分析perf.data数据。可以编写perl和python脚本来辅助分析。自带的脚本有：

```shell
[root@localhost jrg]# perf script -l
List of available trace scripts:
  event_analyzing_sample               analyze all perf samples
  compaction-times [-h] [-u] [-p|-pv] [-t | [-m] [-fs] [-ms]] [pid|pid-range|comm-regex] display time taken by mm compaction
  mem-phys-addr                        resolve physical address samples
  stackcollapse                        produce callgraphs in short form for scripting use
  netdev-times [tx] [rx] [dev=] [debug] display a process of packet and processing time
  net_dropmonitor                      display a table of dropped frames
  syscall-counts [comm]                system-wide syscall counts
  sched-migration                      sched migration overview
  export-to-sqlite [database name] [columns] [calls] export perf data to a sqlite3 database
  powerpc-hcalls
  export-to-postgresql [database name] [columns] [calls] export perf data to a postgresql database
  sctop [comm] [interval]              syscall top
  syscall-counts-by-pid [comm]         system-wide syscall counts, by pid
  failed-syscalls-by-pid [comm]        system-wide failed syscalls, by pid
  futex-contention                     futext contention measurement
  intel-pt-events                      print Intel PT Power Events and PTWRITE
  rw-by-file <comm>                    r/w activity for a program, by file
  failed-syscalls [comm]               system-wide failed syscalls
  rwtop [interval]                     system-wide r/w top
  wakeup-latency                       system-wide min/max/avg wakeup latency
  rw-by-pid                            system-wide r/w activity
```

`perf script  record <script> <command>`用于记录，对应的`perf script  report <script> <command>`则用于显示数据。

`perf script <script> <required-script-args> <command>`实时记录事件，不会把数据保存到硬盘。

##### 举例

```shell
[root@localhost jrg]# perf script report event_analyzing_sample
In trace_begin:

comm=bytearray(b'perf\x00)\x00\x000\x00\x00\x00\x00\x00\x00\x00') common_callchain=[] common_comm=:1848848 common_cpu=7 common_ns=128968792 common_pid=1848848 common_s=277782 pid=1848855 prio=120 success=1 target_cpu=11
comm=bytearray(b'kworker/u113:2\x00\x00') common_callchain=[] common_comm=ls common_cpu=11 common_ns=131027987 common_pid=1848855 common_s=277782 pid=1845831 prio=120 success=1 target_cpu=36
In trace_end:

There is 680264 records in gen_events table
Statistics about the general events grouped by thread/symbol/dso:


            comm   number        histogram
==========================================
         swapper   380530     ###################
 qemu-system-x86   191107     ##################
            perf    94123     #################
......
```





#### 2.23 stat

perf stat用于运行指令，并分析其统计结果。虽然perf top也可以指定pid，但是必须先启动应用才能查看信息。

perf stat能完整统计应用整个生命周期的信息。



##### 用法

```shell
Usage: perf stat [<options>] [<command>]
    -a, --all-cpus        system-wide collection from all CPUs
    -A, --no-aggr         disable CPU count aggregation
    -B, --big-num         print large numbers with thousands' separators
    -C, --cpu <cpu>       list of cpus to monitor in system-wide
    -D, --delay <n>       ms to wait before starting measurement after program start
    -d, --detailed        detailed run - start a lot of events
    -e, --event <event>   event selector. use 'perf list' to list available events
    -G, --cgroup <name>   monitor event in cgroup name only
    -g, --group           put the counters into a counter group
    -I, --interval-print <n>
                          print counts at regular interval in ms (overhead is possible for values <= 100ms)
    -i, --no-inherit      child tasks do not inherit counters
    -M, --metrics <metric/metric group list>
                          monitor specified metrics or metric groups (separated by ,)
    -n, --null            null run - dont start any counters
    -o, --output <file>   output file name
    -p, --pid <pid>       stat events on existing process id  //指定进程pid
    -r, --repeat <n>      repeat command and print average + stddev (max: 100, forever: 0)
    -S, --sync            call sync() before starting a run
    -t, --tid <tid>       stat events on existing thread id
    -T, --transaction     hardware transaction statistics
    -v, --verbose         be more verbose (show counter open errors, etc)
    -x, --field-separator <separator>
                          print counts with custom separator
        --append          append to the output file
        --filter <filter>
                          event filter
        --interval-clear  clear screen in between new interval
        --interval-count <n>
                          print counts for fixed number of times
        --log-fd <n>      log output to fd, instead of stderr
        --metric-only     Only print computed metrics. No raw values
        --no-merge        Do not merge identical named events
        --per-core        aggregate counts per physical processor core
        --per-die         aggregate counts per processor die
        --per-socket      aggregate counts per processor socket
        --per-thread      aggregate counts per thread
        --post <command>  command to run after to the measured command
        --pre <command>   command to run prior to the measured command
        --scale           Use --no-scale to disable counter scaling for multiplexing
        --smi-cost        measure SMI cost
        --table           display details about each run (only with -r option)
        --timeout <n>     stop workload and print counts after a timeout period in ms (>= 10ms)
        --topdown         measure topdown level 1 statistics
```

##### 举例

```shell
[root@localhost jrg]# perf stat ls
 Performance counter stats for 'ls':

              0.98 msec task-clock                #    0.726 CPUs utilized
                 0      context-switches          #    0.000 K/sec
                 0      cpu-migrations            #    0.000 K/sec
                96      page-faults               #    0.098 M/sec
         2,427,546      cycles                    #    2.486 GHz
         1,963,341      instructions              #    0.81  insn per cycle
           391,211      branches                  #  400.601 M/sec
            14,916      branch-misses             #    3.81% of all branches

       0.001345674 seconds time elapsed

       0.000683000 seconds user
       0.000683000 seconds sys


[root@localhost jrg]# perf stat
^C
 Performance counter stats for 'system wide':

         58,817.62 msec cpu-clock                 #   55.475 CPUs utilized
            32,551      context-switches          #    0.553 K/sec
                62      cpu-migrations            #    0.001 K/sec
               749      page-faults               #    0.013 K/sec
     2,901,510,136      cycles                    #    0.049 GHz
     1,016,253,852      instructions              #    0.35  insn per cycle
       202,174,764      branches                  #    3.437 M/sec
        11,997,035      branch-misses             #    5.93% of all branches

       1.060261691 seconds time elapsed


cpu-clock：任务真正占用的处理器时间，单位为ms。CPUs utilized = task-clock / time elapsed，CPU的占用率。
context-switches：程序在运行过程中上下文的切换次数。
CPU-migrations：程序在运行过程中发生的处理器迁移次数。Linux为了维持多个处理器的负载均衡，在特定条件下会将某个任务从一个CPU迁移到另一个CPU。
CPU迁移和上下文切换：发生上下文切换不一定会发生CPU迁移，而发生CPU迁移时肯定会发生上下文切换。发生上下文切换有可能只是把上下文从当前CPU中换出，下一次调度器还是将进程安排在这个CPU上执行。
page-faults：缺页异常的次数。当应用程序请求的页面尚未建立、请求的页面不在内存中，或者请求的页面虽然在内存中，但物理地址和虚拟地址的映射关系尚未建立时，都会触发一次缺页异常。另外TLB不命中，页面访问权限不匹配等情况也会触发缺页异常。
cycles：消耗的处理器周期数。如果把被ls使用的cpu cycles看成是一个处理器的，那么它的主频为2.486GHz。可以用cycles / task-clock算出。
instructions：执行了多少条指令。IPC为平均每个cpu cycle执行了多少条指令。
branches：遇到的分支指令数。
branch-misses是预测错误的分支指令数。
```



统计更多选项

```shell
[root@localhost jrg]# perf stat -e task-clock,context-switches,cpu-migrations,page-faults,cycles,instructions,branches,branch-misses,L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses,dTLB-loads,dTLB-load-misses
^C
 Performance counter stats for 'system wide':

         70,320.23 msec task-clock                #   55.846 CPUs utilized
            39,186      context-switches          #    0.557 K/sec
                87      cpu-migrations            #    0.001 K/sec
               741      page-faults               #    0.011 K/sec
     5,744,216,840      cycles                    #    0.082 GHz                      (21.45%)
     1,460,300,936      instructions              #    0.25  insn per cycle           (30.52%)
       276,595,324      branches                  #    3.933 M/sec                    (35.96%)
        20,787,859      branch-misses             #    7.52% of all branches          (40.15%)
       429,103,243      L1-dcache-loads           #    6.102 M/sec                    (23.70%)
        46,528,058      L1-dcache-load-misses     #   10.84% of all L1-dcache hits    (20.50%)
        23,797,515      LLC-loads                 #    0.338 M/sec                    (19.81%)
           398,316      LLC-load-misses           #    1.67% of all LL-cache hits     (24.26%)
       442,296,850      dTLB-loads                #    6.290 M/sec                    (19.25%)
         5,169,611      dTLB-load-misses          #    1.17% of all dTLB cache hits   (18.79%)

       1.259180585 seconds time elapsed
```



#### 2.24 test

用于sanity test。perf自带了不少测试用例，可用于测试。

##### 用法

```shell
root@ubuntu:~# perf test -h

 Usage: perf test [<options>] [{list <test-name-fragment>|[<test-name-fragments>|<test-numbers>]}]

    -F, --dont-fork       Do not fork for testcase
    -s, --skip <tests>    tests to skip
    -v, --verbose         be more verbose (show symbol address, etc)
```

##### 举例

perf list查看有哪些测试用例

```shell
root@ubuntu:~# perf test list
 1: vmlinux symtab matches kallsyms
 2: Detect openat syscall event
 3: Detect openat syscall event on all cpus
 4: Read samples using the mmap interface
 5: Test data source output
 6: Parse event definition strings
 7: Simple expression parser
 8: PERF_RECORD_* events & perf_sample fields
 9: Parse perf pmu format
10: DSO data read
11: DSO data cache
12: DSO data reopen
...
```

以test 51为例

```shell
root@ubuntu:~# perf test list 51
51: Print cpu map

root@ubuntu:~# perf test 51
51: Print cpu map                              : Ok

root@ubuntu:~# perf test Print
51: Print cpu map                              : Ok
53: is_printable_array                         : Ok
54: Print bitmap                               : Ok
57: unit_number__scnprintf                     : Ok

root@ubuntu:~# perf test -F -v Print
51: Print cpu map                              :
--- start ---
---- end ----
Print cpu map: Ok
53: is_printable_array                         :
--- start ---
---- end ----
is_printable_array: Ok
54: Print bitmap                               :
--- start ---
bitmap: 1
bitmap: 1,5
bitmap: 1,3,5,7,9,11,13,15,17,19,21-40
bitmap: 2-5
bitmap: 1,3-6,8-10,24,35-37
bitmap: 1,3-6,8-10,24,35-37
bitmap: 1-10,12-20,22-30,32-40
---- end ----
Print bitmap: Ok
57: unit_number__scnprintf                     :
--- start ---
n 1, str '1B', buf '1B'
n 10240, str '10K', buf '10K'
n 20971520, str '20M', buf '20M'
n 32212254720, str '30G', buf '30G'
n 0, str '0B', buf '0B'
---- end ----
unit_number__scnprintf: Ok
```



#### 2.25 timechart

将统计信息转换为图形显示模式的工具。

主要两种用途：

`perf timechart record <command>`，记录系统级事件，默认只记录scheduler和CPU事件（诸如进程切换，运行时间，cpu电源状态等）。不过也可以用来记录IO活动。

`perf timechart`可以将perf.data中的数据以图表形式显示。

##### 用法

record参数

```shell
-P 只记录power相关events
-T 只记录task相关events
-I 只记录IO相关events
-g 记录调用关系

[root@localhost test]# perf timechart record -h
 Usage: perf timechart record [<options>]
    -g, --callchain       record callchain
    -I, --io-only         record only IO data
```

timechart参数

```shell
[root@localhost test]# perf timechart -h
 Usage: perf timechart [<options>] {record}
    -f, --force           don't complain, do it
    -i, --input <file>    input file name
    -n, --proc-num <n>    min. number of tasks to print
    -o, --output <file>   output file name
    -p, --process <process>
                          process selector. Pass a pid or process name.
    -t, --topology        sort CPUs according to topology
    -w, --width <n>       page width		//调整输出图宽度
        --highlight <duration or task name>
                          highlight tasks. Pass duration in ns or process name.
        --io-merge-dist <time>
                          merge events that are merge-dist us apart
        --io-min-time <time>
                          all IO faster than min-time will visually appear longer
        --io-skip-eagain  skip EAGAIN errors
        --symfs <directory>
                          Look for files with symbols relative to this directory
```

##### 举例

命令`perf timechart record git pull`

![output](https://raw.githubusercontent.com/junixcn/images/master/output.svg)



#### 2.26 top

实时显示性能统计数据。

```shell
amples: 2K of event 'cpu-clock:pppH', 4000 Hz, Event count (approx.): 414561842 lost: 0/0 drop: 0/0
Overhead  Shared Object                 Symbol
  37.18%  [kernel]                      [k] __lock_text_start                                                                   
  12.92%  [kernel]                      [k] vmw_cmdbuf_header_submit                                                           
  12.03%  [kernel]                      [k] clear_page_orig                                                                     
   1.93%  [kernel]                      [k] finish_task_switch                                                                   
   0.61%  libc-2.31.so                  [.] __memmove_avx_unaligned_erms
   0.58%  [kernel]                      [k] do_syscall_64
   0.55%  [kernel]                      [k] exit_to_usermode_loop
   0.50%  [kernel]                      [k] mpt_put_msg_frame
   0.46%  [kernel]                      [k] rmqueue
   0.46%  [kernel]                      [k] number
   0.43%  [kernel]                      [k] kallsyms_expand_symbol.constprop.0
   0.43%  perf                          [.] rb_next
   0.42%  libslang.so.2.3.2             [.] SLsmg_write_chars
   0.38%  [kernel]                      [k] arch_local_irq_enable
   0.33%  libc-2.31.so                  [.] malloc
   0.33%  [kernel]                      [k] vsnprintf
   0.31%  [kernel]                      [k] memcg_kmem_get_cache
   0.29%  [kernel]                      [k] memset_orig
   0.28%  libc-2.31.so                  [.] read
   0.28%  perf                          [.] __symbols__insert.constprop.0
   0.26%  libpixman-1.so.0.38.4         [.] 0x000000000006e0ff
   0.26%  [kernel]                      [k] native_write_msr
   0.24%  [kernel]                      [k] __fget
   0.23%  libpixman-1.so.0.38.4         [.] 0x000000000008cac5
   0.23%  libpixman-1.so.0.38.4         [.] 0x000000000008cadb
   0.23%  perf                          [.] rust_demangle_callback
   ...
```

第一列：符号引发的性能事件的比例，指占用的cpu周期比例。

第二列：符号所在的DSO(Dynamic Shared Object)，可以是应用程序、内核、动态链接库、模块。

第三列：DSO的类型。[.]表示此符号属于用户态的ELF文件，包括可执行文件与动态链接库；[k]表述此符号属于内核或模块。

第四列：符号名。有些符号不能解析为函数名，只能用地址表示

##### 用法

```shell
Usage: perf top [<options>]

    -a, --all-cpus        system-wide collection from all CPUs
    -b, --branch-any      sample any taken branches
    -c, --count <n>       event period to sample
    -C, --cpu <cpu>       list of cpus to monitor
    -d, --delay <n>       number of seconds to delay between refreshes
    -D, --dump-symtab     dump the symbol table used for profiling
    -E, --entries <n>     display this many functions
    -e, --event <event>   event selector. use 'perf list' to list available events  //指定event
    -f, --count-filter <n>
                          only display functions with more events than this
    -F, --freq <freq or 'max'>
                          profile at this frequency
    -g                    enables call-graph recording and display  //得到函数调用关系图
    -i, --no-inherit      child tasks do not inherit counters
    -j, --branch-filter <branch filter mask>
                          branch stack filter modes
    -K, --hide_kernel_symbols
                          hide kernel symbols
    -k, --vmlinux <file>  vmlinux pathname
    -M, --disassembler-style <disassembler style>
                          Specify disassembler style (e.g. -M intel for intel syntax)
    -m, --mmap-pages <pages>
                          number of mmap data pages
    -n, --show-nr-samples
                          Show a column with the number of samples
    -p, --pid <pid>       profile events on existing process id
    -r, --realtime <n>    collect data with this RT SCHED_FIFO priority
    -s, --sort <key[,key2...]>
                          sort by key(s): pid, comm, dso, symbol, parent, cpu, srcline, ... Please refer the man page for the complete list.
    -t, --tid <tid>       profile events on existing thread id
    -U, --hide_user_symbols
                          hide user symbols
    -u, --uid <user>      user to profile
    -v, --verbose         be more verbose (show counter open errors, etc)
    -w, --column-widths <width[,width...]>
                          don't try to adjust column width, use these fixed values
    -z, --zero            zero history across updates
        --asm-raw         Display raw encoding of assembly instructions (default)
        --call-graph <record_mode[,record_size],print_type,threshold[,print_limit],order,sort_key[,branch]>
                          setup and enables call-graph (stack chain/backtrace):

                                record_mode:    call graph recording mode (fp|dwarf|lbr)
                                record_size:    if record_mode is 'dwarf', max size of stack recording (<bytes>)
                                                default: 8192 (bytes)
                                print_type:     call graph printing style (graph|flat|fractal|folded|none)
                                threshold:      minimum call graph inclusion threshold (<percent>)
                                print_limit:    maximum number of call graph entry (<number>)
                                order:          call graph order (caller|callee)
                                sort_key:       call graph sort key (function|address)
                                branch:         include last branch info to call graph (branch)
                                value:          call graph value (percent|period|count)

                                Default: fp,graph,0.5,caller,function
        --children        Accumulate callchains of children and show total overhead as well
        --comms <comm[,comm...]>
                          only consider symbols in these comms
        --demangle-kernel
                          Enable kernel symbol demangling
        --dsos <dso[,dso...]>
                          only consider symbols in these dsos
        --fields <key[,keys...]>
                          output field(s): overhead, period, sample plus all of sort keys
        --force           don't complain, do it
        --group           put the counters into a counter group
        --hierarchy       Show entries in a hierarchy
        --ignore-callees <regex>
                          ignore callees of these functions in call graphs
        --ignore-vmlinux  don't load vmlinux even if found
        --kallsyms <file>
                          kallsyms pathname
        --max-stack <n>   Set the maximum stack depth when parsing the callchain. Default: kernel.perf_event_max_stack or 127
        --namespaces      Record namespaces events
        --no-bpf-event    do not record bpf events
        --num-thread-synthesize <n>
                          number of thread to run event synthesize
        --objdump <path>  objdump binary to use for disassembly and annotations
        --overwrite       Use a backward ring buffer, default: no
        --percent-limit <percent>
                          Don't show entries under that percent
        --percentage <relative|absolute>
                          How to display percentage of filtered entries
        --proc-map-timeout <n>
                          per thread proc mmap processing timeout in ms
        --raw-trace       Show raw trace event output (do not use print fmt or plugins)
        --show-total-period
                          Show a column with the sum of periods
        --source          Interleave source code with assembly code (default)
        --stdio           Use the stdio interface
        --sym-annotate <symbol name>
                          symbol to annotate
        --symbols <symbol[,symbol...]>
                          only consider these symbols
        --tui             Use the TUI interface
```



##### 举例

写一个死循环函数t1，perf top查看性能；

示例：

```c
#include "stdlib.h"
#include "stdio.h"

void longa() 
{ 
    int i,j; 
    for(i = 0; i < 1000000; i++) 
        j=i; //am I silly or crazy? I feel boring and desperate. 
} 
 
void foo2() 
{ 
    int i; 
    for(i=0 ; i < 10; i++) 
        longa(); 
} 
 
void foo1() 
{ 
    int i; 
    for(i = 0; i< 100; i++) 
        longa(); 
} 
 
int main(int argc, char *argv[]) 
{ 
    int i = 0;
    foo1(); 
    foo2(); 
    while(1)
    {
        foo1(); 
        foo2(); 
        i++;
    }
    return 0;
}
```



t1的main函数占了96%的cpu周期，性能全部耗在t1上。实际环境中没这么容易找到问题，可结合-e参数指定event来判断

```shell
amples: 80K of event 'cpu-clock:pppH', 4000 Hz, Event count (approx.): 7881621045 lost: 0/0 drop: 0/0
Overhead  Shared Object                  Symbol
  96.05%  t1                             [.] main
   0.56%  [kernel]                       [k] __softirqentry_text_start
   0.55%  [kernel]                       [k] clear_page_orig
   0.17%  [kernel]                       [k] vmw_cmdbuf_header_submit
   0.16%  [kernel]                       [k] __lock_text_start
   0.06%  ld-2.31.so                     [.] do_lookup_x
   0.04%  [kernel]                       [k] mpt_put_msg_frame
   0.04%  [kernel]                       [k] exit_to_usermode_loop
   0.04%  [kernel]                       [k] do_syscall_64
```



`perf top --call-graph graph`

```shell
Samples: 25K of event 'cpu-clock:pppH', 4000 Hz, Event count (approx.): 6058000000 lost: 0/0 drop: 0/0
  Children      Self  Shared Object                 Symbol
-   94.37%    94.25%  t1                            [.] longa
     94.25% __libc_start_mai
      - main
         - 85.62% foo1
              longa
         - 8.63% foo2
              longa
-   59.45%     0.00%  libc-2.31.so                  [.] __libc_start_main
     __libc_start_main
   - main
      - 54.05% foo1
           longa
      - 5.40% foo2
           longa
-   59.45%     0.00%  t1                            [.] main
   - main
      - 54.05% foo1
           longa
      - 5.40% foo2
           longa
-   54.05%     0.00%  t1                            [.] foo1
     foo1
     longa
-    5.40%     0.00%  t1                            [.] foo2
     foo2
     longa
+    1.82%     0.07%  [kernel]                      [k] do_syscall_64
...
```



#### 2.27 version

查看当前perf版本

```shell
root@pi:~# perf version
perf version 5.4.73
```



#### 2.28 probe

动态插入采样监测点。

内核需要编译支持CONFIG_DEBUG_INFO



#### 2.29 trace

追踪系统调用。strace工具

##### 用法

```shell
Usage: perf trace [<options>] [<command>]
    or: perf trace [<options>] -- <command> [<options>]
    or: perf trace record [<options>] [<command>]
    or: perf trace record [<options>] -- <command> [<options>]

    -a, --all-cpus        system-wide collection from all CPUs
    -C, --cpu <cpu>       list of cpus to monitor
    -D, --delay <n>       ms to wait before starting measurement after program start
    -e, --event <event>   event/syscall selector. use 'perf list' to list available events
    -f, --force           don't complain, do it
    -F, --pf <all|maj|min>
                          Trace pagefaults
    -G, --cgroup <name>   monitor event in cgroup name only
    -i, --input <file>    Analyze events in file
    -m, --mmap-pages <pages>
                          number of mmap data pages
    -o, --output <file>   output file name
    -p, --pid <pid>       trace events on existing process id
    -s, --summary         Show only syscall summary with statistics
    -S, --with-summary    Show all syscalls and summary with statistics
    -t, --tid <tid>       trace events on existing thread id
    -T, --time            Show full timestamp, not time relative to first start
    -u, --uid <user>      user to profile
    -v, --verbose         be more verbose
        --call-graph <record_mode[,record_size]>
                          setup and enables call-graph (stack chain/backtrace):

                                record_mode:    call graph recording mode (fp|dwarf|lbr)
                                record_size:    if record_mode is 'dwarf', max size of stack recording (<bytes>)
                                                default: 8192 (bytes)

                                Default: fp
        --comm            show the thread COMM next to its id
        --duration <float>
                          show only events with duration > N.M ms
        --expr <expr>     list of syscalls/events to trace
        --failure         Show only syscalls that failed
        --filter-pids <CSV list of pids>
                          pids to filter (by the kernel)
        --kernel-syscall-graph
                          Show the kernel callchains on the syscall exit path
        --map-dump <BPF map>
                          BPF map to periodically dump
        --max-events <n>  Set the maximum number of events to print, exit after that is reached.
        --max-stack <n>   Set the maximum stack depth when parsing the callchain, anything beyond the specified depth will be ignored. Default: kernel.perf_event_max_stack or 127
        --min-stack <n>   Set the minimum stack depth when parsing the callchain, anything below the specified depth will be ignored.
        --no-inherit      child tasks do not inherit counters
        --print-sample    print the PERF_RECORD_SAMPLE PERF_SAMPLE_ info, for debugging
        --proc-map-timeout <n>
                          per thread proc mmap processing timeout in ms
        --sched           show blocking scheduler events
        --sort-events     Sort batch of events before processing, use if getting out of order events
        --syscalls        Trace syscalls
        --tool_stats      show tool stats
```

##### 举例

```shell
[root@localhost jrg]# perf trace
     0.000 ( 1.008 ms): qemu-system-x8/2231208 futex(uaddr: 0x55d15469b5a8, op: WAIT_BITSET|PRIVATE_FLAG|CLOCK_REALTIME, utime: 0x7fb6395fd630, val3: MATCH_ANY) = -1 ETIMEDOUT (Connection timed out)
     1.013 ( 1.006 ms): qemu-system-x8/2231208 futex(uaddr: 0x55d15469b5a8, op: WAIT_BITSET|PRIVATE_FLAG|CLOCK_REALTIME, utime: 0x7fb6395fd630, val3: MATCH_ANY) = -1 ETIMEDOUT (Connection timed out)
     2.024 (         ): qemu-system-x8/2231208 futex(uaddr: 0x55d15469b5a8, op: WAIT_BITSET|PRIVATE_FLAG|CLOCK_REALTIME, utime: 0x7fb6395fd630, val3: MATCH_ANY) ...
18446744073709.520 ( 1.020 ms): qemu-system-x8/2342046 futex(uaddr: 0x55f9bd192208, op: WAIT_BITSET|PRIVATE_FLAG|CLOCK_REALTIME, utime: 0x7f69d65fd630, val3: MATCH_ANY) = -1 ETIMEDOUT (Connection timed out)
     1.004 ( 0.995 ms): qemu-system-x8/2342046 futex(uaddr: 0x55f9bd192208, op: WAIT_BITSET|PRIVATE_FLAG|CLOCK_REALTIME, utime: 0x7f69d65fd630, val3: MATCH_ANY) = -1 ETIMEDOUT (Connection timed out)
     2.004 (         ): qemu-system-x8/2342046 futex(uaddr: 0x55f9bd192208, op: WAIT_BITSET|PRIVATE_FLAG|CLOCK_REALTIME, utime: 0x7f69d65fd630, val3: MATCH_ANY) ...
     0.920 (         ): qemu-system-x8/2364252 ppoll(ufds: 0x562288f9fa70, nfds: 27, tsp: 0x7ffe1f525420, sigsetsize: 8) ...
     0.942 ( 0.988 ms): qemu-system-x8/2364252 ppoll(ufds: 0x562288f9fa70, nfds: 27, tsp: 0x7ffe1f525420, sigsetsize: 8) = 0 (Timeout)
     1.951 ( 0.006 ms): qemu-system-x8/2364252 ppoll(ufds: 0x562288f9fa70, nfds: 27, tsp: 0x7ffe1f525420, sigsetsize: 8) = 0 (Timeout)
     1.971 (         ): qemu-system-x8/2364252 ppoll(ufds: 0x562288f9fa70, nfds: 27, tsp: 0x7ffe1f525420, sigsetsize: 8) ...
     0.525 (         ): qemu-system-x8/11141 ioctl(fd: 17<anon_inode:kvm-vcpu:1>, cmd: KVM_RUN)                 ...
     0.539 ( 0.997 ms): qemu-system-x8/11141 ioctl(fd: 17<anon_inode:kvm-vcpu:1>, cmd: KVM_RUN)                    = 0
     1.543 ( 0.023 ms): qemu-system-x8/11141 ioctl(fd: 17<anon_inode:kvm-vcpu:1>, cmd: KVM_RUN)                    = 0
     1.569 ( 0.955 ms): qemu-system-x8/11141 ioctl(fd: 17<anon_inode:kvm-vcpu:1>, cmd: KVM_RUN)                    = 0
     2.533 ( 0.010 ms): qemu-system-x8/11141 ioctl(fd: 17<anon_inode:kvm-vcpu:1>, cmd: KVM_RUN)                    = 0
     2.545 (         ): qemu-system-x8/11141 ioctl(fd: 17<anon_inode:kvm-vcpu:1>, cmd: KVM_RUN)                 ...
     0.385 (         ): qemu-system-x8/11140 ioctl(fd: 16<anon_inode:kvm-vcpu:0>, cmd: KVM_RUN)                 ...
     0.397 (         ): qemu-system-x8/11140 ioctl(fd: 16<anon_inode:kvm-vcpu:0>, cmd: KVM_RUN)                 ...
     0.405 (         ): qemu-system-x8/11140 ioctl(fd: 16<anon_inode:kvm-vcpu:0>, cmd: KVM_RUN)                 ...
     0.413 (         ): qemu-system-x8/11140 ioctl(fd: 16<anon_inode:kvm-vcpu:0>, cmd: KVM_RUN)                 ...

```





参考文档：

https://www.cnblogs.com/arnoldlu/p/6241297.html

https://www.ibm.com/developerworks/cn/linux/l-cn-perf1/

https://www.ibm.com/developerworks/cn/linux/l-cn-perf2/

perf的man手册

























