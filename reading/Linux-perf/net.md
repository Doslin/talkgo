# 网络性能篇

## 34 | 关于 Linux 网络，你必须知道这些

#### 网络模型

1. OSI七层模型
2. TCP/IP模型（LInux中使用）
   1. 应用层，负责向用户提供一组应用程序，比如 HTTP、FTP、DNS 等。
   2. 传输层，负责端到端的通信，比如 TCP、UDP 等。
   3. 网络层，负责网络包的封装、寻址和路由，比如 IP、ICMP 等。
   4. 网络接口层，负责网络包在物理网络中的传输，比如 MAC 寻址、错误侦测以及通过网卡传输网络帧等。

#### Linux网络协议栈

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/linux-perf/net1.png" style="float:left; width:600px;height:500 px"/>

#### Linux网络收发流程

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/linux-perf/net2.png" style="float:left; width:600px;height:500 px"/>

#### 指标

- 带宽，表示链路的最大传输速率，单位通常为 b/s （比特 / 秒）。
- 吞吐量，表示单位时间内成功传输的数据量，单位通常为 b/s（比特 / 秒）或者 B/s（字节 / 秒）。
- 延时，表示从网络请求发出后，一直到收到远端响应，所需要的时间延迟。
- PPS，是 Packet Per Second（包 / 秒）的缩写，表示以网络包为单位的传输速率。
- 除此，网络的可用性（网络能否正常通信）、并发连接数（TCP 连接数量）、丢包率（丢包百分比）、重传率（重新传输的网络包比例）等也是常用的性能指标。

| 指标           | 获取方式                    | 备注                   |
| -------------- | --------------------------- | ---------------------- |
| 网络配置和状态 | ifconfig或ip                | 推荐ip                 |
| 套接字信息     | `netstat -nlp`或`ss -ltnp`  | head -n 3              |
| 协议栈统计信息 | `netstat -s`或`ss -s`       |                        |
| 吞吐和PPS      | `sar -n DEV 1`              |                        |
| 带宽           | `ethtool eth0 | grep Speed` | Gb/s 或者 Mb/s，b是bit |
| 连通性和延迟   | `ping`                      |                        |

## 35 | 基础篇：C10K 和 C1000K 回顾

#### C10K问题

1. 怎样在一个线程内处理多个请求，也就是要在一个线程内响应多个网络 I/O。
2. 怎么更节省资源地处理客户请求，也就是要用更少的线程来服务这些请求。

#### How

#### 	I/O 模型优化

I/O事件的通知方式：

1. 水平触发：只要文件描述符可以非阻塞地执行 I/O ，就会触发通知。也就是说，应用程序可以随时检查文件描述符的状态，然后再根据状态，进行 I/O 操作。
2. 边缘触发：只有在文件描述符的状态发生改变（也就是 I/O 请求达到）时，才发送一次通知。这时候，应用程序需要尽可能多地执行 I/O，直到无法继续读写，才可以停止。如果 I/O 没执行完，或者因为某种原因没来得及处理，那么这次通知也就丢失了。

##### 第一种，使用非阻塞 I/O 和水平触发通知，比如使用 select 或者 poll。

I/O 是非阻塞的，一个线程中就可以同时监控一批套接字的文件描述符，这样就达到了单线程处理多请求的目的。

- 优点：对应用程序比较友好，它的 API 非常简单。
- 缺点：
  1. 使用 select 和 poll 时，需要对这些文件描述符列表进行轮询，这样，请求数多的时候就会比较耗时。
  2. select 使用固定长度的位相量，表示文件描述符的集合，因此会有最大描述符数量的限制。

##### 第二种，使用非阻塞 I/O 和边缘触发通知，比如 epoll。

用以解决水平触发的问题

- 优点：
  1. epoll 使用红黑树，在内核中管理文件描述符的集合
  2. epoll 使用事件驱动的机制，只关注有 I/O 事件发生的文件描述符，不需要轮询扫描整个集合。
- 缺点：边缘触发只在文件描述符可读或可写事件发生时才通知，那么应用程序就需要尽可能多地执行 I/O，并要处理更多的异常事件。且实现难度高

##### 第三种，使用异步 I/O（Asynchronous I/O，简称为 AIO）

异步 I/O 允许应用程序同时发起很多 I/O 操作，而不用等待这些操作完成。而在 I/O 完成后，系统会用事件通知（比如信号或者回调函数）的方式，告诉应用程序

#### 工程模型优化

##### 第一种，主进程 + 多个 worker 子进程，这也是最常用的一种模型。

- 主进程执行 bind() + listen() 后，创建多个子进程；
- 然后，在每个子进程中，都通过 accept() 或 epoll_wait() ，来处理相同的套接字。

惊群的问题：当网络 I/O 事件发生时，多个进程被同时唤醒，但实际上只有一个进程来响应这个事件，其他被唤醒的进程都会重新休眠。（可解决）

##### 第二种，监听到相同端口的多进程模型。

所有的进程都监听相同的接口，并且开启 SO_REUSEPORT 选项，由内核负责将请求负载均衡到这些监听进程中去。内核确保了只有一个进程被唤醒，就不会出现惊群问题了

#### 应用层的网络协议优化思路

- 使用长连接取代短连接，可以显著降低 TCP 建立连接的成本。在每秒请求次数较多时，这样做的效果非常明显。
- 使用内存等方式，来缓存不常变化的数据，可以降低网络 I/O 次数，同时加快应用程序的响应速度。
- 使用 Protocol Buffer 等序列化的方式，压缩网络 I/O 的数据量，可以提高应用程序的吞吐。
- 使用 DNS 缓存、预取、HTTPDNS 等方式，减少 DNS 解析的延迟，也可以提升网络 I/O 的整体速度。

#### C1000K问题

硬件替代软件功能

#### C10MK问题

DPDK，XDP方案

当然，实际上，我们不需要单机处理，可以采用集群的方式



## 36 | 套路篇：怎么评估系统的网络性能？

#### 网络基准测试

需要通过性能测试来确定这些指标的基准值。且弄清楚，你要评估的网络性能，究竟属于协议栈的哪一层？

#### 转发性能

针对的是网络接口层和网络层。

重点指标：PPS

工具：pktgen

#### TCP/UDP 性能

iperf 和 netperf 都是最常用的网络性能测试工具，测试 TCP 和 UDP 的吞吐量

重点指标：TCP 和 UDP 的吞吐量，带宽等

工具：iperf, netperf

#### http性能

重点指标：HTTP 服务的每秒请求数、请求延迟、吞吐量以及请求延迟的分布情况等

工具：ab, wrk(更接近实际业务)，jmeter(商业版，wrk升级版)

## 37 | 案例篇：DNS 解析时快时慢，我该怎么办？

DNS（Domain Name System），即域名系统，主要提供域名和 IP 地址之间映射关系的查询服务。



查看DNS配置：`cat /etc/resolv.conf`



DNS 服务通过资源记录的方式，来管理所有数据，它支持 A、CNAME、MX、NS、PTR 等多种类型的记录。

- A 记录，用来把域名转换成 IP 地址；
- CNAME 记录，用来创建别名；
- 而 NS 记录，则表示该域名对应的域名服务器地址。



工具：

1.  `nslookup`: 简要查看域名解析
2. `dig +trace +nodnssec time.geekbang.org`: 查看域名解析整个调用链

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/linux-perf/net3.png"/>

#### 案例解析

##### DNS 解析失败

>1. `nslookup time.geekbang.org`超时失败
>2. `ping -c3 114.114.114.114`却成功
>3. `nslookup -debug time.geekbang.org`，发现nslookup 连接环回地址（127.0.0.1 和 ::1）的 53 端口失败
>4. `cat /etc/resolv.conf`发现确实没配置DNS服务，所以查询了环回地址

##### DNS 解析不稳定

>1. `time nslookup time.geekbang.org`发现解析不稳定，发现存在过慢情况，甚至超时
>2. 怀疑，DNS 服务器本身有问题、客户端到 DNS 服务器的网络延迟比较大、DNS 请求或者响应包，在某些情况下被链路中的网络设备弄丢了。
>3. `ping -c3 8.8.8.8`查看所用DNS服务的网络情况，发现较慢且存在丢包。断定为期间的网络问题

## 38 | 案例篇：怎么使用 tcpdump 和 Wireshark 分析网络流量？

#### 案例分析

1. 使用`ping -c3 geektime.org`发现，发现3次请求平均30ms，但是总时间搞到11000ms。
2. 怀疑域名解析有问题，`time nslookup geektime.org`发现域名解析也很快
3. 使用tcpdump工具，`tcpdump -nn udp port 53 or host 35.190.27.188`发现期间有PTR请求存在5s延迟
4. 通过禁用PTR解决。

#### tcpdump快速上手

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/blog/net4.png"/>

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/blog/net5.png"/>

#### 使用wireshark更加简单易懂

一般使用tcpdump的`-w`选项将网络包保存下来，再使用wireshark查看。

## 39 | 案例篇：怎么缓解 DDoS 攻击带来的性能下降问题？

#### 案例分析

1. 访问nginx服务超时了，之前都是正常的
2. 使用`sar -n DEV 1`命令，发现PPS为2000多，而BPS则为1174KB，每个包只有50KB
3. 使用`tcpdump -i eth0 -n tcp port 80`，发现大量syn包。判定为SYN Flood的DDOS攻击
4. `netstat -n -p | grep SYN_REC | wc -l`，发现有193个，进一步证明

#### 如何防御？

内核调优：

1. `net.ipv4.tcp_max_syn_backlog = 256`调大
2. `net.ipv4.tcp_synack_retries = 1`减小
3. `net.ipv4.tcp_syncookies = 1`开启

交给专业的网络设备防火墙等：

如在C10M中描述，使用DPDK、XDP等避免过长的协议栈直接判断过滤掉恶意流量

## 40 | 案例篇：网络请求延迟变大了，我该怎么办？

#### 案例分析

1. 发现网络延迟变高，由于服务器关闭了ICMP。使用`hping3 -c 3 -S -p 80 192.168.0.30`测试tcp延迟，发现延迟确实太大
2. 使用 traceroute，确认路由是否正确，并查看路由中每一跳网关的延迟。发现并无问题
3. 使用`tcpdump -nn tcp port 8080 -w nginx.pcap`，下载到本地，用wireshark分析
4. 分析返现客户端有40ms延迟，这正是延迟相应机制的体现。
5. 使用`strace`，发现客户端只设置了`TCP_NODELAY`，确实开启了延迟访问
6. 而这是只是客户端行为，不应该影响服务端。接着分析服务端
7. 观察wireshark，第二个分组没跟前一个分组（697 号）一起发送，而是等到客户端对第一个分组的 ACK 后（1173 号）才发送。这是Nagle 算法，与延迟算法结合造成了延迟
8. 通过查看nginx配置，发现`tcp_nodelay off;`，原因得以确认。

#### 总结

在发现网络延迟增大的情况后，你可以先从路由、网络包的收发、网络包的处理，再到应用程序等，从各个层级分析网络延迟，等到找出网络延迟的来源层级后，再深入定位瓶颈所在。

## 41-42 | 案例篇：如何优化 NAT 性能？

SystemTap：用以动态跟踪内核，它把用户提供的脚本，转换为内核模块来执行，用来监测和跟踪内核的行为。（监控很有用）

分析 NAT 性能问题时，可以先从内核连接跟踪模块 conntrack 角度来分析，比如用 systemtap、perf、netstat 等工具，以及 proc 文件系统中的内核选项，来分析网络协议栈的行为；然后，通过内核选项调优、切换到无状态 NAT、使用 DPDK 等方式，进行实际优化。

## 43 | 套路篇：网络性能优化的几个思路

#### 网络有很多层，对不同测进行基准测试

1. 网络接口层和网络层：每秒可处理的网络包数 PPS，就是它们最重要的性能指标（特别是在小包的情况下）。使用`pktgen` ，来测试 PPS 的性能
2. 传输层：吞吐量（BPS）、连接数以及延迟，就是最重要的性能指标。iperf 或 netperf测试。网络包的大小，会直接影响这些指标的值。
3. 应用层：吞吐量（BPS）、每秒请求数以及延迟等指标。wrk、ab

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/blog/net6.png"/>

<img src="https://raw.githubusercontent.com/erenming/image-pool/master/blog/net7.png"/>

#### 套接字方面思路

- 增大每个套接字的缓冲区大小 net.core.optmem_max；
- 增大套接字接收缓冲区大小 net.core.rmem_max 和发送缓冲区大小 net.core.wmem_max；
- 增大 TCP 接收缓冲区大小 net.ipv4.tcp_rmem 和发送缓冲区大小 net.ipv4.tcp_wmem。

#### 传输层优化思路

<img src="https://blog-1300816757.cos.ap-shanghai.myqcloud.com/img/net8.png"/>

#### 网络层

第一种，从路由和转发的角度出发，调整下面的内核选项

第二种，从分片的角度出发，最主要的是调整 MTU（Maximum Transmission Unit）的大小。

第三种，从 ICMP 的角度出发，为了避免 ICMP 主机探测、ICMP Flood 等各种网络问题，你可以通过内核选项，来限制 ICMP 的行为。

#### 链路层

主要是优化网络包的收发、网络功能卸载以及网卡选项。