# 内存/IO性能篇

## 15 | 基础篇：Linux内存是怎么工作的？

#### 操作系统内存管理

> 属于多级缓存管理机制CPU->MMU->VIRTUAL->DRAM，详细可查阅csapp第九章

#### 回收机制

>1. 回收缓存，比如使用 LRU（Least Recently Used）算法，回收最近使用最少的内存页面；
>2. 回收不常访问的内存，把不常用的内存通过交换分区直接写到磁盘中；
>3. 杀死进程，内存紧张时系统还会通过 OOM（Out of Memory），直接杀掉占用大量内存的进程。

#### 如何查看

1. `free`命令

| total  | used                 | free   | shared | buff/cache   | available  |
| ------ | -------------------- | ------ | ------ | ------------ | ---------- |
| 总内存 | 已使用，包括共享内存 | 未使用 | 共享   | 缓存和缓冲区 | 新进程可用 |

2. `top`命令

   | VIRT               | RES                              | SHR            | %MEM                                 |
   | ------------------ | -------------------------------- | -------------- | ------------------------------------ |
   | 进程虚拟内存的大小 | 常驻内存的大小，不包括Swap和共享 | 共享内存的大小 | 进程使用物理内存占系统总内存的百分比 |

   

## 16 | 基础篇：怎么理解内存中的Buffer和Cache？

作为磁盘到CPU的桥梁

#### Buffer:

- 对数据磁盘的缓存，读写
- 场景：直接磁盘IO，例如存储系统操作磁盘，虚拟容器绕过文件系统直接操作磁盘

#### Cache:

- 对文件系统的缓存，读写
- 场景：普通文件的读写，因为此时是文件系统与磁盘交互

## 17 | 案例篇：如何利用系统缓存优化程序的运行效率？

#### 缓存命中率：

作为衡量缓存使用的好坏情况，命中率越高，表示使用缓存带来的收益越高，应用程序的性能也就越好。

#### 工具

1. bcc软件包：查看进程的缓存命中情况
   1. cachestat：系统整体缓存读写命中情况
   2. cachetop 提供了每个进程的缓存命中情况。
2. pcstat:  指定文件在内存中的缓存大小。

#### 案例分析：

1. 发现问价程序读取文件速度很慢
2. cachetop发现命中率确有100%
3. 由于cachetop是只计算普通io，并没有计入直接IO的。因此怀疑程序绕过了缓存，使用了直接读
4. 通过`stace -p (pgrep app)`命令，发现代码里确实存在直接读

## 18 | 案例篇：内存泄漏了，我该如何定位和处理？

#### 案例分析

1. 采集指标，观察内存指标的变化情况
2. 使用memleak确认内存泄露点

## 19-20 | 案例篇：为什么系统的Swap变高了？

#### 概念

Swap 说白了就是把一块磁盘空间或者一个本地文件，当成内存来使用，包括换入换出

- 换出，就是把进程暂时不用的内存数据存储到磁盘中，并释放这些数据占用的内存。
- 换入，则是在进程再次访问这些内存的时候，把它们从磁盘读到内存中来。

操作系统对剩余内存评估来将文件页、匿名页等内存换出换入。

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/linux-perf/swap.png"/>

现如今的云计算中，基本都不开启swap了

## 21 | 套路篇：如何“快准狠”找到系统内存的问题？

#### 内存指标

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/linux-perf/mem-metrics.png" style="float:left; width:600px;height:500 px"/>

#### 指标->工具

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/linux-perf/mem-mt.png"  style="float:left; width:600px;height:500 px" />

#### 工具->指标

<img src="https://blog-1300816757.cos.ap-shanghai.myqcloud.com/img/mem-tm.png" style="float:left; width:600px;height:500 px" />

#### roadmap

<img src="https://blog-1300816757.cos.ap-shanghai.myqcloud.com/img/roadmap-mem.png"/>

#### 常见优化思路

1. 禁止swap
2. 减少内存分配，使用内存池、大页等
3. 尽量使用缓存和缓冲区来访问数据