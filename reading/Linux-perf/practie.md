# 实战举例

## 46 | 案例篇：为什么应用容器化后，启动慢了很多

### 案例分析

1. 访问容器化后的java web应用，curl 返回了 “Connection reset by peer” 的错误
2. `docker logs -f tomcat`查看日志，返现只打印了环境变量
3. 客户端模拟请求，。发现curl 终于给出了我们想要的结果 “Hello, wolrd!”。但是，随后又出现了 “Empty reply from server”。
4. 查看容器日志，Tomcat 在启动 24s 后完成初始化然后退出到shell。显然是应用退出了
5. `docker inspect tomcat -f '{{json .State}}' | jq`，发现退出原因确实是OOM
6. **当 OOM 发生时，系统会把相关的 OOM 信息，记录到日志中**，查看发现匿名内存超过了512M
7. `docker exec tomcat java -XX:+PrintFlagsFinal -version | grep HeapSize`，执行java命令查看内存配置情况，最大堆内存则是 1.95GB。
8. 虽然设置了内存限制512M，但是从容器内部看到的限制，却并不是 512M。
9. `docker exec tomcat free -m`看到的显然还是主机内存情况。

## 47-48 | 案例篇：服务器总是时不时丢包，我该怎么办？

`hping3 -c 10 -S -p 80 192.168.0.30`，发现存在丢包现象。如何分析排查？应该根据网络协议栈逐层分析

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/linux-perfpr1.png"/>

#### 排查链路层

1. 进入容器内，排查是否因为栈溢出导致。执行`netstat -i`，发现虚拟网卡没有丢包
2. 排查是否配置了tc规则，因为它的信息不回记录在网卡中。`tc -s qdisc show dev eth0`， 发现eth0 上面配置了一个网络模拟排队规则（qdisc netem），并且配置了丢包率为 30%（loss 30%）
3. `删除 tc 中的 netem 模块`，重新执行hping3命令，发现还是有50%的丢包, RTT依然很大。

#### 排查网络层&传输层

1. 执行`netstat -s`查看各种协议的收发情况。发现只有 TCP 协议发生了丢包和重传。

   ```shell
   TcpExt: 
     11 resets received for embryonic SYN_RECV sockets //半连接重置数 
     0 packet headers predicted 
     TCPTimeouts: 7 //超时数 
     TCPSynRetrans: 4 //SYN重传数 
     ...
   ```

2. 可以看出，TCP 协议有多次超时和失败重试，并且主要错误是半连接重置。换句话说，主要的失败，都是三次握手失败。你也可以抓包，然后用wireshark查看
3. 除了各种协议丢包，IPtable和内核中的跟踪机制也会导致丢包
4. 排查跟踪机制，`sysctl net.netfilter.nf_conntrack_max`, `sysctl net.netfilter.nf_conntrack_count`。连接跟踪数只有 182，剔除假设
5. 排查iptable，`iptables -t filter -nvL`查看filter的统计信息
6. 发现，两条 DROP 规则的统计数值不是 0，它们分别在 INPUT 和 OUTPUT 链中。且都配置statistic 模块，进行随机 30% 的丢包。
7. 删除DROP规则。重新hping3已正常，但是curl一下Nginx服务，发现超时了。很奇怪了
8. 使用`tcpdump`, `tcpdump -i eth0 -nn port 80`再执行curl抓包后，使用wireshark查看。
9. 没有抓取到 curl 发来的 HTTP GET 请求，重新执行`netstat -i`，发现RX-DRP有344个。
10. 为啥hping3正常，而GET丢包？仔细观察mtu为100，而hping3发的SYN包，HTTP请求本质上是TCP包较大，被丢弃了(可配置的)
11. 修改mtu大小，恢复正常

## 49 | 案例篇：内核线程 CPU 利用率太高，我该怎么办？

### 何为内核线程？

1. 0号进程，系统创建的第一个进程，初始化 1 号和 2 号进程后，演变为空闲任务。
2. 1 号进程为 init 进程，通常是 systemd 进程，在用户态运行，用来管理其他用户态进程。
3. 2 号进程为 kthreadd 进程，在内核态运行，用来管理内核线程。

### 如何查找？

1. 查看2号进程的子进程，`ps -f --ppid 2 -p 2`
2. 括号过滤，`ps -ef | grep "\[.*\]"`

### 常见性能问题的内核线程

- kswapd0：用于内存回收。在 Swap 变高 案例中，我曾介绍过它的工作原理。
- kworker：用于执行内核工作队列，分为绑定 CPU （名称格式为 kworker/CPU86330）和未绑定 CPU（名称格式为 kworker/uPOOL86330）两类。
- migration：在负载均衡过程中，把进程迁移到 CPU 上。每个 CPU 都有一个 migration 内核线程。
- jbd2/sda1-8：jbd 是 Journaling Block Device 的缩写，用来为文件系统提供日志功能，以保证数据的完整性；名称中的 sda1-8，表示磁盘分区名称和设备号。每个使用了 ext4 文件系统的磁盘分区，都会有一个 jbd2 内核线程。
- pdflush：用于将内存中的脏页（被修改过，但还未写入磁盘的文件页）写入磁盘（已经在 3.10 中合并入了 kworker 中）。

### 如何排查(同样适用于普通进程)

1. 使用perf工具记录，并使用`FlameGraph`火焰图，方便地查看

### FlameGraph火焰图介绍

1. **横轴表示采样数和采样比例**。一个函数占用的横轴越宽，就代表它的执行时间越长。同一层的多个函数，则是按照字母来排序。
2. **纵轴表示调用栈，由下往上根据调用关系逐个展开**。换句话说，上下相邻的两个函数中，下面的函数，是上面函数的父函数。这样，调用栈越深，纵轴就越高。

### 分类

- on-CPU 火焰图：表示 CPU 的繁忙情况，用在 CPU 使用率比较高的场景中。
- off-CPU 火焰图：表示 CPU 等待 I/O、锁等各种资源的阻塞情况。
- 内存火焰图：表示内存的分配和释放情况。
- 热 / 冷火焰图：表示将 on-CPU 和 off-CPU 结合在一起综合展示。
- 差分火焰图：表示两个火焰图的差分情况，红色表示增长，蓝色表示衰减。差分火焰图常用来比较不同场景和不同时期的火焰图，以便分析系统变化前后对性能的影响情况。

## 50-51 | 案例篇：动态追踪怎么用？

perf技术，也称为动态追踪技术

### What

动态追踪技术，通过探针机制，来采集内核或者应用程序的运行信息，从而可以不用修改内核和应用程序的代码，就获得丰富的信息，帮你分析、定位想要排查的问题。

为了追踪内核或用户空间的事件，Dtrace 和 SystemTap 都会把用户传入的追踪处理函数（一般称为 Action），关联到被称为探针的检测点上。这些探针，实际上也就是各种动态追踪技术所依赖的事件源。

#### 动态追踪的事件源

- 动态追踪所使用的事件源，可以分为静态探针、动态探针以及硬件事件等三类
- 硬件事件通常由性能监控计数器 PMC（Performance Monitoring Counter）产生，包括了各种硬件的性能情况，比如 CPU 的缓存、指令周期、分支预测等等。
- 静态探针，是指事先在代码中定义好，并编译到应用程序或者内核中的探针。
- 动态探针，则是指没有事先在代码中定义，但却可以在运行时动态添加的探针，比如函数的调用和返回等。

---

### 动态跟踪机制

#### ftrace

ftrace 通过 debugfs（或者 tracefs），为用户空间提供接口。

直接使用`trace-cmd`命令，其封装了ftrace的操作步骤，可方便地查看调用关系

1. 如查看`ls`命令，`do_sys_open`函数的调用情况。可执行命令`
   1. `trace-cmd record -p function_graph -g do_sys_open -O funcgraph-proc ls`
   2. `trace-cmd report`

通常来说，使用perf和火焰图，找到热点函数，再使用ftrace来查看调用情况

#### perf

功能：

1. 查找应用程序或者内核中的热点函数，从而定位性能瓶颈。
2. perf 可以用来分析 CPU cache、CPU 迁移、分支预测、指令周期等各种硬件事件；
3. perf 也可以只对感兴趣的事件进行动态追踪。

#### 案例分析：查看内核函数`do_sys_open`

1. 添加探针，`perf probe --add do_sys_open`
2. 采样，`perf record -e probe:do_sys_open -aR sleep 10`
3. 查看结果，`perf script`
4. 查看参数情况，可以用`perf trace`（取代strace, ptrace首选工具）

#### 案例分析：用户空间的库函数

1. 添加探针，`perf probe -x /bin/bash 'readline%return +0($retval):string’`
2. 采样记录，`perf record -e probe_bash:readline__return -aR sleep 5`
3. 查看结果，`perf script`
4. 删除探针，`perf probe --del probe_bash:readline__return`

#### eBPF 和 BCC

eBPF: 可以通过 C 语言自由扩展（这些扩展通过 LLVM 转换为 BPF 字节码后，加载到内核中执行）

BBC: eBPF的封装版本，BCC 把 eBPF 中的各种事件源（比如 kprobe、uprobe、tracepoint 等）和数据操作（称为 Maps），也都转换成了 Python 接口（也支持 lua）。

#### SystemTap

也是一种可以通过脚本进行自由扩展的动态追踪技术。eBPF出现之前，适用于3.X等老版本

#### sysdig

 主要用于容器的动态追踪， 大集成。表示为 sysdig = strace + tcpdump + htop + iftop + lsof + docker inspect。

### 工具适用总结

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/linux-perfpr2.png"/>

## 52 | 案例篇：服务吞吐量下降很厉害，怎么分析？

1.内核连接数限制 nf_conntrack.
2.php程序的工作进程数量
3.半链接队列偏小,导致高并发时的丢包.
4.系统分配的临时端口号范围.
5.系统的端口复用参数配置.

一环接一环，建议看原文。https://time.geekbang.org/column/article/87342

## 53 | 套路篇：系统监控的综合思路

USE原则：

- 使用率，表示资源用于服务的时间或容量百分比。100% 的使用率，表示容量已经用尽或者全部时间都用于服务。
- 饱和度，表示资源的繁忙程度，通常与等待队列的长度相关。100% 的饱和度，表示资源无法接受更多的请求。
- 错误数表示发生错误的事件个数。错误数越多，表明系统的问题越严重。

一个完整的监控系统通常由数据采集、数据存储、数据查询和处理、告警以及可视化展示等多个模块组成

## 54 | 套路篇：应用监控的一般思路

RED原则：

- 请求率，Rate (R): The number of requests per second.
- 错误率，Errors (E): The number of failed requests.
- 响应时间，Duration (D): The amount of time to process a request.

核心指标：

1. 请求数、错误率和响应时间

重要指标

1. 是应用进程的资源使用情况，比如进程占用的 CPU、内存、磁盘 I/O、网络等。
2. 是应用程序之间调用情况，比如调用频率、错误数、延时等。
3. 是应用程序内部核心逻辑的运行情况，比如关键环节的耗时以及执行过程中的错误等。这需要应用主动暴露

#### 日志监控

我们还需要对这些指标的上下文信息进行监控，而日志正是这些上下文的最佳来源。

与指标对比：

- 指标是特定时间段的数值型测量数据，通常以时间序列的方式处理，适合于实时监控。
- 而日志则完全不同，日志都是某个时间点的字符串消息，通常需要对搜索引擎进行索引后，才能进行查询和汇总分析。

通常通过 ELK 技术栈来进行收集、索引和图形化展示。

## 55 | 套路篇：分析性能问题的一般步骤

#### 系统资源瓶颈的角度

USE 法是最为有效的方法，即从使用率、饱和度以及错误数这三个方面，来分析

#### 应用程序瓶颈的角度

- 资源瓶颈跟系统资源瓶颈，本质是一样的。

- 依赖服务瓶颈，你可以使用全链路跟踪系统进行定位。
- 应用自身的问题，你可以通过系统调用、热点函数，或者应用自身的指标监控以及日志监控等，进行分析定位。

#### 注意

系统是应用的运行环境，系统的瓶颈会导致应用的性能下降；而应用的不合理设计，也会引发系统资源的瓶颈。

## 答疑篇

### 通过性能工具定位到了具体的内核函数，如何快速了解其含义？

- 最好的方法，就是去查询所用内核版本的源代码。推荐 https://elixir.bootlin.com 这个网站
- 对于 eBPF 来说，除了可以通过内核源码来了解，我更推荐你从 BPF Compiler Collection (BCC) 这个项目开始。BCC 提供了很多短小的示例，可以帮你快速了解 eBPF 的工作原理，并熟悉 eBPF 程序的开发思路。



## 书单推荐

- 《深入理解计算机系统》（已看过，还需深入lab）
- 《Linux 程序设计》和《UNIX 环境高级编程》
- 《深入 Linux 内核架构》
- 《性能之巅：洞悉系统、企业与云计算》(权威工具书，收藏)