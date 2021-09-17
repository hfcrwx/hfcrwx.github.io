---
title: "muduo"
published: true
---

## Intro ##

    构建一个分布式服务器，可借鉴的muduo代码

**URL**

    https://github.com/chenshuo/muduo
    https://github.com/chenshuo/muduo-tutorial

**AsyncLogging**

    muduo-tutorial/src/echo.cc

**ProtobufCodec**

    examples/protobuf/codec/

**ThreadPool, setHighWaterMarkCallback, setWriteCompleteCallback**

    examples/sudoku/server_prod.cc

**PingPong**

    examples/pingpong/server.cc
    examples/pingpong/client.cc //EventLoopThreadPool, throughput
    
    examples/ace/logging/server.cc
    
    examples/sudoku/batch.cc //QPS

**CountDownLatch**

    examples/memcached/client/bench.cc //EventLoopThreadPool, QPS, pingpong
    examples/protobuf/rpcbench/client.cc
    examples/wordcount/hasher.cc

**outstandings_**

    examples/protobuf/rpcbalancer/balancer.cc
    examples/protobuf/rpcbalancer/balancer_raw.cc
    muduo/net/protorpc/RpcChannel.cc
         CallMethod
         onRpcMessage

// wire format
//
// Field     Length  Content
//
// size      4-byte  M+N+4
// tag       M-byte  could be "PB", "MP", etc.
// payload   N-byte
// checksum  4-byte  adler32 of tag+payload

message PbMessage
{
  required MessageType type = 1;
  required fixed64 id = 2;

  optional string typeName = 3;
  optional bytes request = 4;

  optional bytes response = 5;

  optional ErrorCode error = 6;
}

http://chenshuo.com/practical-network-programming/

# 网络编程实践

## 1

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\1.网络编程概要.mkv_000005.865.jpg)

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\1.网络编程概要.mkv_000022.988.jpg)

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\1.网络编程概要.mkv_000059.001.jpg)

UNP 1e 1990

APU

TCP/IP v1-v3



UNP 2e 1998

​	v1: XPI已经淘汰

​	v2: 和网络编程关系不大，主要讲进程间通讯。多进程和多线程的并发编程

​	v3：应用，作者去世

v1：强调的不够：

​	1.消息格式的处理：非阻塞IO下正确处理TCP分包

​	2.并发模型稍显陈旧



UNP 3e 2003

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\1.网络编程概要.mkv_000305.337.jpg)

避免基于猜测、猜想的优化

网络编程有很多以讹传讹的讲究，如：尽量减少动态内存分配，使用STL更是罪大恶极

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\1.网络编程概要.mkv_000438.590.jpg)

以太网层：frame 帧

IP层：packet 分组（和IP分片是两个事情，一般不用管IP分片）

传输层：segment 分节

应用层：message 消息

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\1.网络编程概要.mkv_000510.655.jpg)

2.有人说TCP不可靠，收到的数据不完整：主要涉及TCP连接断开的时机与条件，close太早的话，有可能导致协议栈发送rst分节，将连接重置，数据自然就收不完了。在阻塞编程中可以用SO_LINGER这个选项，在非阻塞编程中这个选项没用，需要从应用层的协议上入手解决。

4.C struct

​	1.考虑对齐，修改全局对齐方式，pack=1，导致第三方lib core dump：破坏ABI二进制接口

​	2.高度的不可扩展，如果增加一个字段，所有客户端、服务端都要升级。非C语言，需要手工维护pack on pack的代码

5.TCP客户端往本机的服务端发起连接时，如果服务端没有启来，在一定条件下客户端会和自己建立连接

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\1.网络编程概要.mkv_001017.888.jpg)

## 2

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000004.708.jpg)

验证TCP的有效带宽

atom -> e6400：118MB/s

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000240.157.jpg)

本机测试

atom -> atom：dd：580MB/s（双核）

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000343.822.jpg)

从磁盘读文件（第一次）：115MB/s

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000508.681.jpg)

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000522.512.jpg)

这是磁盘性能。

第二次测（数据已经缓存到内存里了，双核，两个进程：nc nc）：1074MB/s

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000546.300.jpg)

nc加pv显示带宽（450，更慢一点，双核，四个进程：dd nc nc pv，dd本身也消耗资源）

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000723.525.jpg)

top看每个进程占的cpu

atom -> e350：118MB/s

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000859.309.jpg)

加pv

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_000949.973.jpg)

注意：pv用的2进制的兆字节，不是10进制，准确应该用Mib

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_001001.600.jpg)

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\2.一个TCP的简单实验.mkv_001342.893.jpg)

比较内核态和用户态的操作

580：6个过程

1074：4个过程

如果只有TCP（2个过程），大约是580的3倍



172.29.233.89

```
[root@iZ2ze7qslbwa07f03lfmegZ ~]# nc -l 5001 > /dev/null
```

172.29.233.88

```
[root@iZ2ze7qslbwa07f03lfmehZ ~]# dd if=/dev/zero bs=1MB count=1000 | nc 172.29.233.89 5001
1000+0 records in
1000+0 records out
1000000000 bytes (1.0 GB) copied, 3.16583 s, 316 MB/s
```

本机测试

```
[root@iZ2ze7qslbwa07f03lfmehZ ~]# dd if=/dev/zero bs=1MB count=10000 | nc localhost 5001
10000+0 records in
10000+0 records out
10000000000 bytes (10 GB) copied, 9.45878 s, 1.1 GB/s
```

wget http://ftp-archive.freebsd.org/pub/FreeBSD-Archive/old-releases/amd64/ISO-IMAGES/8.2/FreeBSD-8.2-RELEASE-amd64-memstick.img

## 3

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\3.课程内容大纲.mkv_000005.928.jpg)

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\3.课程内容大纲.mkv_000149.147.jpg)

proxy：阻塞简单；非阻塞若两端带宽不匹配，复杂

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\3.课程内容大纲.mkv_000409.876.jpg)

数据交换从小到大。

分布式计算框架hadoop spark

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\3.课程内容大纲.mkv_20210910_113359.610.jpg)

## 4

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\4.回顾基础的Sockets API.mkv_20210910_114106.775.jpg)

https://en.wikipedia.org/wiki/Ttcp

https://en.wikipedia.org/wiki/Traceroute

现在比较流行Iperf

https://en.wikipedia.org/wiki/Iperf

![](D:\projects\hfcrwx.github.io\_posts\pic\pnp\4.回顾基础的Sockets API.mkv_20210910_114534.130.jpg)

带宽：只管消息量，

单核CPU是否能占满TCP千兆网带宽

吞吐量：应用层面

延迟：平均延迟，百分位数延迟

资源使用率

额外开销

​	加密：只有开销

​	压缩：先压缩后加密，反过来压缩起不到作用。

​		原1s。CPU压缩需要0.5s，若压缩率是50%，压缩后IO需要0.5s。用非阻塞IO，考虑到压缩和网络IO是可以重叠的，每次少量压缩发送，需要0.6-0.7s的IO，提高了拷贝文件的效率，代价是cpu使用率上升了一些。

![](pic/pnp/4.回顾基础的Sockets API.mkv_20210910_122612.955.jpg)

![](pic/pnp/4.回顾基础的Sockets API.mkv_20210910_123021.526.jpg)

客户端收到响应后再发下一次请求，比nc慢

![](pic/pnp/4.回顾基础的Sockets API.mkv_20210910_123727.259.jpg)

这种情况下，非阻塞IO降低性能，多一步等待。去掉服务器的关闭连接的代码，可以直接支持并发。

https://github.com/chenshuo/recipes

https://github.com/chenshuo/muduo-examples-in-go

## 5

void transmit(const Options& opt)

void receive(const Options& opt)

:tabe common.h

char data[0]; //数组长度是运行时决定的

TCP_NODELAY不等TCP的ACK

## 6

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_160436.165.jpg)

cd recipes/tpc && make bin/ttcp

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_170300.604.jpg)

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_171517.798.jpg)

发送包大小：2^10, 2^11, ..., 2^16

9（机器cs组合） * 7（包大小种类） * 16（语言cs组合数）

nc和延迟无关，ttcp和延迟有关

atom -> e6400

-l 65535

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_172233.908.jpg)

-l 1024

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_172333.700.jpg)

-l 2048

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_172527.902.jpg)

-l 4096

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_172609.231.jpg)

-l 8192

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_172729.272.jpg)

-l 16834

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_172801.904.jpg)

-l 32768

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_172917.017.jpg)

-l 65536

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_173015.273.jpg)

-l 128000

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_173049.738.jpg)

-l 256000

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_173200.741.jpg)

测试时运行时间不能太短，TCP slow start 慢启动

-l 512000 -n 4096

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_173508.909.jpg)

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_173623.094.jpg)

消息越小，传输延迟的效用越大

atom -> atom

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_174118.450.jpg)

-l 4096

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_174332.012.jpg)

本机，消息大小4k的情况下，吞吐量超过了千兆网

-l 8192

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_174446.477.jpg)

-l 16384

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_174630.214.jpg)

-l 32768

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_174703.414.jpg)

-l 65536

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_174752.287.jpg)

-l 102400

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_174839.112.jpg)

-l 1024000

![](pic/pnp/6.使用TTCP进行网络传输性能测试.mkv_20210910_174931.192.jpg)

作为进程间通讯，即使在本机，TCP也是相当的强

## 7

![](pic/pnp/7.阻塞IO下的TTCP实验.mkv_20210910_175418.836.jpg)

阻塞IO有可能会一直阻塞过去

atom -> atom

20M的时候阻塞掉了

![](pic/pnp/7.阻塞IO下的TTCP实验.mkv_20210910_180310.603.jpg)

netstat

![](pic/pnp/7.阻塞IO下的TTCP实验.mkv_20210910_180430.948.jpg)

客户端发送缓冲区满了。服务端接收到部分数据，进行send，服务端等待客户端的recv，但客户端必须等到自己send完。服务端阻塞在send，又导致服务端不能recv，导致客户端阻塞在send。

![](pic/pnp/7.阻塞IO下的TTCP实验.mkv_20210910_182249.059.jpg)

20M是本机测试的结果，如果用网络上的两台机器，会显著小于这个数字（2M）

![](pic/pnp/7.阻塞IO下的TTCP实验.mkv_20210910_182415.068.jpg)

关键在于服务端没有完整的读到客户端的请求。服务端收到4K数据就发给客户端，但客户端没法收。

解决问题在于协议设计上。

![](pic/pnp/7.阻塞IO下的TTCP实验.mkv_20210910_183606.406.jpg)

## 8

![](pic/pnp/8.TCP自连接.mkv_20210910_183704.959.jpg)

netstat -ltnp

```
[root@iZ2ze7qslbwa07f03lfmehZ python]# python self-connect.py 22
connected ('127.0.0.1', 48600) ('127.0.0.1', 22)
^CTraceback (most recent call last):
  File "self-connect.py", line 17, in <module>
    time.sleep(60*60)
KeyboardInterrupt
[root@iZ2ze7qslbwa07f03lfmehZ python]# python self-connect.py 48600
connected ('127.0.0.1', 48600) ('127.0.0.1', 48600)
```

本机端口号

```
[root@iZ2ze7qslbwa07f03lfmehZ python]# sysctl -A |grep range
net.ipv4.ip_local_port_range = 32768    60999
```

本机，在端口范围之内，且没有使用的端口，都会发生自连接

```
[root@iZ2ze7qslbwa07f03lfmehZ python]# python self-connect.py 36000
connected ('::1', 36000, 0, 0) ('::1', 36000, 0, 0)
```

netstat -tpn | grep 36000

bool isSelfConnection(const Socket& sock)

## 9

![](pic/pnp/9.扩展练习.mkv_20210910_204415.165.jpg)

client，server，同一个随机数种子，生成随机数验证TCP有效性

![](pic/pnp/9.扩展练习.mkv_20210910_210704.192.jpg)

pipeline：如果延迟比较大，可以有效利用带宽

## 10

![](pic/pnp/10.时钟概述.mkv_20210910_212803.018.jpg)

UDP

chenshuo.com/pnp

![](pic/pnp/10.时钟概述.mkv_20210911_123329.810.jpg)

![](pic/pnp/10.时钟概述.mkv_20210911_123412.210.jpg)

计时

精度最高的是美国的原子钟，10^-15

时钟=振荡器+计数器（精确）

改进振荡器

![](pic/pnp/10.时钟概述.mkv_20210911_124245.935.jpg)

14.318M

![](pic/pnp/10.时钟概述.mkv_20210911_124401.687.jpg)

ppm = 10^-6

ppb = 10^-9

电脑里用的是Clock XO

## 11

![](pic/pnp/11.时钟精确度和校准.mkv_20210911_125300.740.jpg)

Timestamp: T*，不能相加，只能相减

Time interval: int，可以相加、相减

(T1+T2)/2 = T1 + (T2-T1)/2

![](pic/pnp/11.时钟精确度和校准.mkv_20210911_130832.432.jpg)

L: 热胀冷缩等，需要调整，影响频率

![](pic/pnp/11.时钟精确度和校准.mkv_20210911_131610.902.jpg)

Resolution: 分辨率

NTP最好部署在FreeBSD上

## 12

![](pic/pnp/12.网络时间同步.mkv_20210911_133648.063.jpg)

1.Daytime，Time精确到s

2.往返时间

NTP精确度: 0.25ns

![](pic/pnp/12.网络时间同步.mkv_20210911_135045.980.jpg)

NTP的复杂度不在网络编程上

![](pic/pnp/12.网络时间同步.mkv_20210911_140159.528.jpg)

![](pic/pnp/12.网络时间同步.mkv_20210911_140214.080.jpg)

![](pic/pnp/12.网络时间同步.mkv_20210911_141459.620.jpg)

![](pic/pnp/12.网络时间同步.mkv_20210911_142458.506.jpg)

## 13

![](pic/pnp/13.Roundtrip代码分析.mkv_20210911_142702.034.jpg)

UDP不可靠，不能用单线程阻塞IO，发送请求后等待回复

消息格式收发都是 8B+8B，为了对称性，对TCP有意义，以太网的最小帧长是64B。UDP会填充到最小帧长，意义不大。

![](pic/pnp/13.Roundtrip代码分析.mkv_20210911_144903.891.jpg)

UPD server 1个socket可以服务多个client

yum install ntp

systemctl start ntpd

ntptime

ntpq -pn

C-z

bg

## 14

![](pic/pnp/14.其他测试方案.mkv_20210911_182357.495.jpg)

千兆以太网单程延迟100us

![](pic/pnp/14.其他测试方案.mkv_20210911_184540.424.jpg)

ntpdate -u 172.29.233.89
ntpdate cn.pool.ntp.org

## 15

![](pic/pnp/15.UDP vs TCP.mkv_20210911_223923.194.jpg)

UDP设计比TCP要晚。

UDP：NAT穿透

​	NTP

TCP：一个socket，一个线程读，一个线程写，是可以的。

## 16

![](pic/pnp/16.扩展知识.mkv_20210911_225606.117.jpg)

UTC原子时

GMT天文时

UTC = TAI + Leap Seconds

![](pic/pnp/16.扩展知识.mkv_20210911_231645.447.jpg)

![](pic/pnp/16.扩展知识.mkv_20210911_232751.765.jpg)

## 17

![](pic/pnp/17.如何正确使用TCP.mkv_20210911_232848.372.jpg)

![](pic/pnp/17.如何正确使用TCP.mkv_20210911_233018.756.jpg)

socket, stdin, stdout

![](pic/pnp/17.如何正确使用TCP.mkv_20210911_233521.203.jpg)

![](pic/pnp/17.如何正确使用TCP.mkv_20210911_233803.858.jpg)

难度: 建立TCP连接的难度 < 销毁TCP连接的难度

​	  TCP server建立连接的难度 < TCP client建立连接的难度

​	  接收TCP数据的难度 < 发送TCP数据的难度

recipes/tpc/bin/sender.cc

![](pic/pnp/17.如何正确使用TCP.mkv_20210911_235152.197.jpg)

正确的做法：shutdown write导致对方的read返回0，对方close，我们的socket read也返回0，我们也close

## 18

![](pic/pnp/18.TCP使用的注意事项.mkv_20210912_000911.337.jpg)

![](pic/pnp/18.TCP使用的注意事项.mkv_20210912_002402.885.jpg)

tpc/bin/nodelay.cc

tpc/bin/nodelay_server.cc

![](pic/pnp/18.TCP使用的注意事项.mkv_20210912_085310.757.jpg)

![](pic/pnp/18.TCP使用的注意事项.mkv_20210912_085405.077.jpg)

![](pic/pnp/18.TCP使用的注意事项.mkv_20210912_090118.568.jpg)

![](pic/pnp/18.TCP使用的注意事项.mkv_20210912_091026.051.jpg)

## 19

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_091641.736.jpg)

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_092014.158.jpg)

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_092152.541.jpg)

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_092227.877.jpg)

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_092701.418.jpg)

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_092813.442.jpg)

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_093148.840.jpg)

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_093259.287.jpg)

![](pic/pnp/19.多个版本的Netcat概览.mkv_20210912_093444.230.jpg)

并发连接数少的情况下，thread-per-connection性能比IO-multiplexing好，节约了等待IO就绪事件，epoll_wait()

## 20

![](pic/pnp/20.第一个Netcat的实现.mkv_20210912_093836.108.jpg)

Go: goroutine, channel, select

tpc/bin/netcat.cc

阻塞IO自动节流限速

nc atom 1234 < /dev/zero

nc atom 1234 | slow_reader

## 21

![](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_103841.922.jpg)

thread-per-connection适合连接数不多，线程比较廉价的情况

IO复用：一个线程处理多个fd，复用的不是IO而是线程thread of control

python/netcat.py

tpc/bin/chargen.cc

![](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121313.493.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121330.341](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121330.341.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121344.101](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121344.101.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121348.301](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121348.301.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121410.628](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121410.628.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121622.443](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121622.443.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121636.579](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121636.579.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121805.722](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121805.722.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121818.866](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121818.866.jpg)

![21.IO-multiplexing方式实现Netcat.mkv_20210912_121824.298](pic/pnp/21.IO-multiplexing方式实现Netcat.mkv_20210912_121824.298.jpg)

阻塞IO如果和IO复用配合的话，一旦真的发送阻塞，就会把别的socket上的事件挡住了。

## 22

![](pic/pnp/22.使用非阻塞IO 1.mkv_20210912_122118.416.jpg)

python/netcat-nonblock.py

short write

![](pic/pnp/22.使用非阻塞IO 1.mkv_20210912_123439.521.jpg)

![](pic/pnp/22.使用非阻塞IO 1.mkv_20210912_124521.011.jpg)

## 23

![](pic/pnp/23.使用非阻塞IO 2.mkv_20210912_125224.135.jpg)

非阻塞读由应用程序处理，写由网络库处理

![](pic/pnp/23.使用非阻塞IO 2.mkv_20210912_130237.266.jpg)

对方接收数据缓慢

![](pic/pnp/23.使用非阻塞IO 2.mkv_20210912_131839.961.jpg)

![](pic/pnp/23.使用非阻塞IO 2.mkv_20210912_131912.809.jpg)

ET: write, accept

LT: read

![](pic/pnp/23.使用非阻塞IO 2.mkv_20210912_132756.892.jpg)

## 24

![24.进程监控概述.mkv_000005.706](pic/pnp/24.进程监控概述.mkv_000005.706.jpg)

![24.进程监控概述.mkv_000052.688](pic/pnp/24.进程监控概述.mkv_000052.688.jpg)

bin/procman 1 3000

examples/procmon/dummyload.cc

bin/dummyload c 80 2

top -H -p

## 25

![](pic/pnp/25.实现前要考虑的问题.mkv_000005.575.jpg)

python + matplotlib

![](pic/pnp/25.实现前要考虑的问题.mkv_000540.811.jpg)

## 26

examples/procmon/procmon.cc

void tick() // 采样

![](pic/pnp/26.procmon代码解析.mkv_000541.147.jpg)

## 27

![](pic/pnp/27.dummyload实现原理和代码解析.mkv_000006.344.jpg)

![](pic/pnp/27.dummyload实现原理和代码解析.mkv_000522.999.jpg)

## 28

bin/procmon 1 3000

bin/procmon 3671 2345

ab -k -n 100000 http://localhost:3000/

ab -k -n 1000000 http://localhost:3000/

ab -k -n 500000 -c 5 http://localhost:3000/

ab -n 100000 http://localhost:3000/

htop

![](pic/pnp/28.procmon性能测试.mkv_000033.838.jpg)

![28.procmon性能测试.mkv_000044.731](pic/pnp/28.procmon性能测试.mkv_000044.731.jpg)

![28.procmon性能测试.mkv_000135.675](pic/pnp/28.procmon性能测试.mkv_000135.675.jpg)

![28.procmon性能测试.mkv_000136.674](pic/pnp/28.procmon性能测试.mkv_000136.674.jpg)

![28.procmon性能测试.mkv_000159.293](pic/pnp/28.procmon性能测试.mkv_000159.293.jpg)

![28.procmon性能测试.mkv_000757.228](pic/pnp/28.procmon性能测试.mkv_000757.228.jpg)

## 29

![](pic/pnp/29.知识扩展和总结.mkv_000241.495.jpg)

![](pic/pnp/29.知识扩展和总结.mkv_000353.123.jpg)

![](pic/pnp/29.知识扩展和总结.mkv_000715.947.jpg)

![](pic/pnp/29.知识扩展和总结.mkv_001038.584.jpg)

![](pic/pnp/29.知识扩展和总结.mkv_001607.062.jpg)

## 30

![](pic/pnp/30.功能描述.mkv_000003.131.jpg)

![](pic/pnp/30.功能描述.mkv_000031.942.jpg)

![](pic/pnp/30.功能描述.mkv_000129.637.jpg)

隔离度：内存密集型 > CPU密集型 > IO密集型

![](pic/pnp/30.功能描述.mkv_000704.828.jpg)

![](pic/pnp/30.功能描述.mkv_001246.551.jpg)

![](pic/pnp/30.功能描述.mkv_001501.164.jpg)

shutdown实现最难

memcached-1.4.22/t

![](pic/pnp/30.功能描述.mkv_001743.820.jpg)

![](pic/pnp/30.功能描述.mkv_001959.144.jpg)

## 31

![](pic/pnp/31.数据结构设计与分析.mkv_000006.412.jpg)

![](pic/pnp/31.数据结构设计与分析.mkv_000413.328.jpg)

![](pic/pnp/31.数据结构设计与分析.mkv_000946.057.jpg)

![](pic/pnp/31.数据结构设计与分析.mkv_001313.054.jpg)

![](pic/pnp/31.数据结构设计与分析.mkv_001651.582.jpg)

![](pic/pnp/31.数据结构设计与分析.mkv_002233.738.jpg)

![](pic/pnp/31.数据结构设计与分析.mkv_002236.644.jpg)

阅读版本memcached 1.2.8

![](pic/pnp/31.数据结构设计与分析.mkv_002911.180.jpg)

## 32

muduo/examples/memcached/server/footprint_test.cc

footprint 占资源的大小

tcmalloc

./memcached_footprint 1000000

perf record ./memcached_footprint 1000000

perf report

## 33

![](pic/pnp/33.网络IO模型与代码解读.mkv_000004.943.jpg)

![](pic/pnp/33.网络IO模型与代码解读.mkv_000145.281.jpg)

## 34

## 35

![](pic/pnp/35.性能测试 2.mkv_000903.784.jpg)

## 36

![](pic/pnp/36.性能分析.mkv_000006.234.jpg)

![](pic/pnp/36.性能分析.mkv_000025.144.jpg)

google-pprof --pdf bin/memcached_debug_tcmalloc http://localhost:11212 > set.pdf

sumatra pdf

## 37

![](pic/pnp/37.定制数据结构以减小内存使用.mkv_000129.243.jpg)

![](pic/pnp/37.定制数据结构以减小内存使用.mkv_000238.392.jpg)

![](pic/pnp/37.定制数据结构以减小内存使用.mkv_000516.607.jpg)

https://gist.github.com/chenshuo/612177b4caf0c1ad7064

## 38

![](pic/pnp/38.数独求解服务简介.mkv_000006.492.jpg)

![38.数独求解服务简介.mkv_000015.868](pic/pnp/38.数独求解服务简介.mkv_000015.868.jpg)

![](pic/pnp/38.数独求解服务简介.mkv_000042.095.jpg)

recipes/sudoku/test1

![](pic/pnp/38.数独求解服务简介.mkv_000446.923.jpg)

## 39

![](pic/pnp/39.并发模型和测试工具.mkv_000005.908.jpg)

5.有并发，没并行

![](pic/pnp/39.并发模型和测试工具.mkv_000827.266.jpg)

![](pic/pnp/39.并发模型和测试工具.mkv_000831.108.jpg)

## 40

## 41

![](pic/pnp/41.内置性能监控.mkv_000006.107.jpg)

![](pic/pnp/41.内置性能监控.mkv_000758.400.jpg)

![](pic/pnp/41.内置性能监控.mkv_001210.860.jpg)

## 42

## 43

bin/sudoku_client_pipeline sudoku17 10.0.0.49 //测量延迟低

bin/sudoku_loadtest sudoku17 10.0.0.49 1000 //测量延迟高

原因：两者延迟计算方式不同

sudoku_client_pipeline 发数据均匀，请求返回后再发下一个

sudoku_loadtest 一次发多个，请求的累积，多个请求都返回再计算



bin/sudoku_client_pipeline sudoku17 10.0.0.49 10

bin/sudoku_client_pipeline sudoku17 10.0.0.49 1 10

client server 都要观察cpu

如果client cpu跑满了，而server cpu很空闲，实际是压力测试器自己的性能，不是server的性能

bin/sudoku_client_pipeline sudoku17 10.0.0.49 10 1

10个连接，IO开销略大一些，client cpu 高一些

less p0003

gnuplot

plot 'p0003' using 1:2 with boxes, 'p0003' using 1:3 with lines axes x1y2

set y2tics 20 nomirror

replot

set xrange[0:2000]

replot



bin/sudoku_solver_hybrid 4 0 -n



head p0002

fg



bin/sudoku_solver_hybrid 4 4 -n



sudoku_client_pipeline 测的是极限情况下

sudoku_loadtest 在某种合理情况下测

## 44

给定RPS，测量延迟分布

bin/sudoku_solver_hybrid 4 0 -n

server: top

10.0.0.49:9982/sudoku/stats

bin/sudoku_loadtest sudoku17 10.0.0.49 100

client和server测的延迟数值差很多，原因是网络部分的延迟server没有算进去



mv r0100 r-e4-t0-r100-c1

gnuplot

plot 'r-e4-t0-r100-c1' using 1:2 with boxes



bin/sudoku_loadtest sudoku17 10.0.0.49 1000

mv r0060 r-e4-t0-r1000-c1

plot 'r-e4-t0-r100-c1' using 1:2 with boxes, 'r-e4-t0-r1000-c1' using 1:2 with boxes

plot 'r-e4-t0-r1000-c1' using 1:($2/1000) with boxes, 'r-e4-t0-r100-c1' using 1:($2/100) with boxes



mv r0025 r-e4-t0-r10000-c1

plot 'r-e4-t0-r1000-c1' using 1:($2/1000) with boxes, 'r-e4-t0-r100-c1' using 1:($2/100) with boxes, 'r-e4-t0-r10000-c1' using 1:($2/10000) with boxes

只在IO线程做计算，不好。

bin/sudoku_solver_hybrid 4 4 -n

plot 'r-e4-t4-r100-c1' using 1:2 with boxes, 'r-e4-t0-r100-c1' using 1:2 with boxes

用了线程池，延迟的分布更集中。

## 45

bin/sudoku_loadtest sudoku17 10.0.0.49 20000

数据堆积在客户端，客户端内存不断增加

in-fly数据不断增加

recipes/tpc/bin/sudoku_stress.cc

./sudoku_stress 127.0.0.1

./sudoku_stress 127.0.0.1 200000

./sudoku_stress 127.0.0.1 200000 -r

10.0.0.37:9982/pprof/memstats

10.0.0.37:9982/pprof/releasefreememory



bin/sudoku_solver_hybrid 0 2

10.0.0.37:9982/pprof/memstats

![](pic/pnp/45.过载保护.mkv_001450.848.jpg)

muduo/examples/sudoku/server_prod.cc

highWaterMark: 5*1024 * 1024(5w条100Byte的消息未读)

![](pic/pnp/45.过载保护.mkv_002354.315.jpg)

请求过多，返回“busy”

发送回复数据量过多，关闭socket

## 46

![](pic/pnp/46.负载均衡.mkv_000005.749.jpg)

![](pic/pnp/46.负载均衡.mkv_001717.534.jpg)

## 47

muduo/examples/fastcgi/fastcgi_test.cc

nginx 1.6.2

nginx -p muduo/examples/fastcgi

psg nginx

netstat -tpna | grep 19981

curl -v http://localhost:10080/sudoku/0000...

./fastcgi_test 19981 2 > /dev/null

ab 只能测https1.0的长连接，不支持chunked。1.1的长连接用weighttp测。

weighttp -n 100000 -c 1 -k localhost:10080/sudoku/0000...

nginx -p muduo/examples/fastcgi -s stop

nginx -p muduo/examples/fastcgi -s reload

![](pic/pnp/40.批处理模型及疑似内存泄露.mkv_001404.121.jpg)

## 48

![](pic/pnp/48.如何进一步适应生产环境.mkv_000006.539.jpg)

合理的延迟 -> RPS/core

RPS(daily peak) / (RPS/core) = core

core / (core/pc) = pc

![](pic/pnp/48.如何进一步适应生产环境.mkv_000936.872.jpg)

## 49

## 50

## 51

![](pic/pnp/51.苏迪曼杯羽毛球比赛.mkv_000006.571.jpg)

![51.苏迪曼杯羽毛球比赛.mkv_000023.591](pic/pnp/51.苏迪曼杯羽毛球比赛.mkv_000023.591.jpg)

![51.苏迪曼杯羽毛球比赛.mkv_000640.297](pic/pnp/51.苏迪曼杯羽毛球比赛.mkv_000640.297.jpg)

## 52

![](pic/pnp/52.记分系统设计.mkv_000006.395.jpg)

![](pic/pnp/52.记分系统设计.mkv_001516.029.jpg)

## 53

![](pic/pnp/53.聊天服务器.mkv_000006.538.jpg)

网络编程3个重要例子：

echo、chat、proxy

## 54

diff -u server.cc server_threaded.cc

fg

message数据拷贝

vim -p server*

## 55

![](pic/pnp/55.hub服务器[new!].mkv_000006.794.jpg)

相当于不止一个聊天室

## 56

![](pic/pnp/56.设计难点[new!].mkv_000054.852.jpg)

实验，用C-z把一个client暂停起来，server内存会不断增长。

## 57

![](pic/pnp/57. TCP relay功能描述及Python实现.mkv_000005.232.jpg)

![](pic/pnp/57. TCP relay功能描述及Python实现.mkv_000020.349.jpg)

![](pic/pnp/57. TCP relay功能描述及Python实现.mkv_000119.219.jpg)

![](pic/pnp/57. TCP relay功能描述及Python实现.mkv_000329.313.jpg)

![](pic/pnp/57. TCP relay功能描述及Python实现.mkv_000356.732.jpg)

## 58

![](pic/pnp/58. TCP半关连接.mkv_000223.497.jpg)

![](pic/pnp/58. TCP半关连接.mkv_000500.820.jpg)

## 59

![](pic/pnp/59. 非阻塞TCP relay实现.mkv_000005.566.jpg)

![](pic/pnp/59. 非阻塞TCP relay实现.mkv_000221.413.jpg)

![59. 非阻塞TCP relay实现.mkv_000439.235](pic/pnp/59. 非阻塞TCP relay实现.mkv_000439.235.jpg)

## 60

虚拟内存

//conn->stopRead();

![](pic/pnp/60. 源码及运行.mkv_000646.886.jpg)

server:

nc -l -p 3000

relay:

bin/tcprelay 127.0.0.1 3000 2000

client:

nc localhost 2000



pv /dev/zero | nc localhost 2000

nc -l -p 3000 > /dev/null



nc localhost 2000 < /dev/zero

nc -l -p 3000 | pv > /dev/null

nc -l -p 3000 | pv -L 1m > /dev/null

## 61

nc localhost 2000 < /dev/zero

nc -l -p 3000 > /dev/null



gdb bin/tcprelay 13409

bt

info threads

p this->startRead()

writeCompleteCallback_, highWaterMarkCallback_都是延迟回调

连接有相互关联的情况下，回调的顺序

![](pic/pnp/61. 竞态条件及修复.mkv_000928.346.jpg)

## 62

![](pic/pnp/62. SOCKS4a服务器实现.mkv_000237.701.jpg)

## 63

https://en.wikipedia.org/wiki/Transport_Layer_Security#Basic_TLS_handshake

状态机

https://golang.org/src/crypto/tls/

![](pic/pnp/63. 非阻塞IO之外的选择.mkv_000429.161.jpg)

https://hpbn.co/transport-layer-security-tls/

https://golang.org/src/crypto/tls/handshake_server.go

## 64

![](pic/pnp/64. 用 GO 语言实现 TCP relay.mkv_000055.455.jpg)

![](pic/pnp/64. 用 GO 语言实现 TCP relay.mkv_000748.608.jpg)

```go
func (srv *Server) Serve(l net.Listener) error {
```

```go
		rw, err := l.Accept()
		if err != nil {
```

https://golang.org/src/net/tcpsock.go

```go
// CloseWrite shuts down the writing side of the TCP connection.
// Most callers should just use Close.
func (c *TCPConn) CloseWrite() error {
	if !c.ok() {
		return syscall.EINVAL
	}
	if err := c.fd.closeWrite(); err != nil {
		return &OpError{Op: "close", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
	}
	return nil
}
```

https://groups.google.com/forum/#!topic/golang-dev/cq-Y0vDXdwg

netstat -tpn

C-z fg

C-d

strace -tt nc -l -p 3000

## 65

![](pic/pnp/65. 事件驱动与多线程的取舍.mkv_000725.602.jpg)

## 66

![](pic/pnp/66. 第七层以外的实现方式.mkv_000258.172.jpg)

![66. 第七层以外的实现方式.mkv_000617.521](pic/pnp/66. 第七层以外的实现方式.mkv_000617.521.jpg)

![66. 第七层以外的实现方式.mkv_000804.842](pic/pnp/66. 第七层以外的实现方式.mkv_000804.842.jpg)

https://research.google/pubs/pub44824/

## 67

![](pic/pnp/67. 正确理解TCP的可靠性.mkv_000633.955.jpg)

https://www.zhihu.com/question/25016042/answer/29798924

TCP可靠：不重，不漏，不乱（没有说不改）

![](pic/pnp/67. 正确理解TCP的可靠性.mkv_000636.966.jpg)

## 68

![](pic/pnp/68. Muduo与C++11.mkv_000250.333.jpg)

![68. Muduo与C++11.mkv_000251.828](pic/pnp/68. Muduo与C++11.mkv_000251.828.jpg)

![68. Muduo与C++11.mkv_000829.033](pic/pnp/68. Muduo与C++11.mkv_000829.033.jpg)

## 69

![](pic/pnp/69. N皇后问题及单机求解方法.mkv_000005.327.jpg)

![69. N皇后问题及单机求解方法.mkv_000020.179](pic/pnp/69. N皇后问题及单机求解方法.mkv_000020.179.jpg)

![69. N皇后问题及单机求解方法.mkv_000142.237](pic/pnp/69. N皇后问题及单机求解方法.mkv_000142.237.jpg)

![69. N皇后问题及单机求解方法.mkv_000410.405](pic/pnp/69. N皇后问题及单机求解方法.mkv_000410.405.jpg)

![69. N皇后问题及单机求解方法.mkv_000446.494](pic/pnp/69. N皇后问题及单机求解方法.mkv_000446.494.jpg)

![69. N皇后问题及单机求解方法.mkv_000628.279](pic/pnp/69. N皇后问题及单机求解方法.mkv_000628.279.jpg)

![69. N皇后问题及单机求解方法.mkv_000656.391](pic/pnp/69. N皇后问题及单机求解方法.mkv_000656.391.jpg)

![69. N皇后问题及单机求解方法.mkv_000922.583](pic/pnp/69. N皇后问题及单机求解方法.mkv_000922.583.jpg)

https://oeis.org/A000170

## 70

![](pic/pnp/70. 并行算法与MapReduce.mkv_000211.513.jpg)

![70. 并行算法与MapReduce.mkv_000505.086](pic/pnp/70. 并行算法与MapReduce.mkv_000505.086.jpg)

![70. 并行算法与MapReduce.mkv_000506.054](pic/pnp/70. 并行算法与MapReduce.mkv_000506.054.jpg)

![70. 并行算法与MapReduce.mkv_000539.527](pic/pnp/70. 并行算法与MapReduce.mkv_000539.527.jpg)

## 71

![](pic/pnp/71. RPC简介与接口定义.mkv_000059.811.jpg)

![71. RPC简介与接口定义.mkv_000247.936](pic/pnp/71. RPC简介与接口定义.mkv_000247.936.jpg)

![71. RPC简介与接口定义.mkv_000555.009](pic/pnp/71. RPC简介与接口定义.mkv_000555.009.jpg)

![71. RPC简介与接口定义.mkv_000601.760](pic/pnp/71. RPC简介与接口定义.mkv_000601.760.jpg)

## 72

## 73

## 74

![](pic/pnp/74. RPC 负载均衡.mkv_000031.650.jpg)

![74. RPC 负载均衡.mkv_000226.278](pic/pnp/74. RPC 负载均衡.mkv_000226.278.jpg)

![74. RPC 负载均衡.mkv_000406.774](pic/pnp/74. RPC 负载均衡.mkv_000406.774.jpg)

每次选出pending response最少的server来发请求。

## 75

![](pic/pnp/75. 多机求平均数和中位数的算法.mkv_000005.781.jpg)

![75. 多机求平均数和中位数的算法.mkv_000050.583](pic/pnp/75. 多机求平均数和中位数的算法.mkv_000050.583.jpg)

![75. 多机求平均数和中位数的算法.mkv_000054.089](pic/pnp/75. 多机求平均数和中位数的算法.mkv_000054.089.jpg)

![75. 多机求平均数和中位数的算法.mkv_000337.071](pic/pnp/75. 多机求平均数和中位数的算法.mkv_000337.071.jpg)

![75. 多机求平均数和中位数的算法.mkv_000737.218](pic/pnp/75. 多机求平均数和中位数的算法.mkv_000737.218.jpg)

![75. 多机求平均数和中位数的算法.mkv_000856.708](pic/pnp/75. 多机求平均数和中位数的算法.mkv_000856.708.jpg)

![75. 多机求平均数和中位数的算法.mkv_001018.217](pic/pnp/75. 多机求平均数和中位数的算法.mkv_001018.217.jpg)

## 76

![](pic/pnp/76. 代码实现及运行实例.mkv_001220.418.jpg)

## 77

![](pic/pnp/77. 实现RCP框架：服务端.mkv_000005.476.jpg)

![77. 实现RCP框架：服务端.mkv_000153.212](pic/pnp/77. 实现RCP框架：服务端.mkv_000153.212.jpg)

![77. 实现RCP框架：服务端.mkv_000247.536](pic/pnp/77. 实现RCP框架：服务端.mkv_000247.536.jpg)

![77. 实现RCP框架：服务端.mkv_000507.159](pic/pnp/77. 实现RCP框架：服务端.mkv_000507.159.jpg)

![77. 实现RCP框架：服务端.mkv_000642.699](pic/pnp/77. 实现RCP框架：服务端.mkv_000642.699.jpg)

![77. 实现RCP框架：服务端.mkv_000741.298](pic/pnp/77. 实现RCP框架：服务端.mkv_000741.298.jpg)

![77. 实现RCP框架：服务端.mkv_001125.847](pic/pnp/77. 实现RCP框架：服务端.mkv_001125.847.jpg)

## 78

![](pic/pnp/78. 实现RPC框架：客户端.mkv_000005.576.jpg)

![78. 实现RPC框架：客户端.mkv_000247.985](pic/pnp/78. 实现RPC框架：客户端.mkv_000247.985.jpg)

![78. 实现RPC框架：客户端.mkv_000249.315](pic/pnp/78. 实现RPC框架：客户端.mkv_000249.315.jpg)

![78. 实现RPC框架：客户端.mkv_000339.263](pic/pnp/78. 实现RPC框架：客户端.mkv_000339.263.jpg)

![78. 实现RPC框架：客户端.mkv_000501.850](pic/pnp/78. 实现RPC框架：客户端.mkv_000501.850.jpg)

![78. 实现RPC框架：客户端.mkv_000554.556](pic/pnp/78. 实现RPC框架：客户端.mkv_000554.556.jpg)

![78. 实现RPC框架：客户端.mkv_000708.085](pic/pnp/78. 实现RPC框架：客户端.mkv_000708.085.jpg)

![78. 实现RPC框架：客户端.mkv_001319.563](pic/pnp/78. 实现RPC框架：客户端.mkv_001319.563.jpg)

## 79

![](pic/pnp/79. 单词计数及按频度排序，单机算法.mkv_000005.983.jpg)

![79. 单词计数及按频度排序，单机算法.mkv_000020.019](pic/pnp/79. 单词计数及按频度排序，单机算法.mkv_000020.019.jpg)

![79. 单词计数及按频度排序，单机算法.mkv_000250.373](pic/pnp/79. 单词计数及按频度排序，单机算法.mkv_000250.373.jpg)

effective stl

![](pic/pnp/79. 单词计数及按频度排序，单机算法.mkv_000711.664.jpg)

![79. 单词计数及按频度排序，单机算法.mkv_001115.144](pic/pnp/79. 单词计数及按频度排序，单机算法.mkv_001115.144.jpg)

https://www.cnblogs.com/baiyanhuang/archive/2012/11/11/2764914.html

## 80

recipes/topk

:set nowrap

![](pic/pnp/80. 单机版代码阅读.mkv_002724.648.jpg)

## 81

![](pic/pnp/81. 多机单词计数算法与代码.mkv_000009.336.jpg)

![81. 多机单词计数算法与代码.mkv_000057.513](pic/pnp/81. 多机单词计数算法与代码.mkv_000057.513.jpg)

![81. 多机单词计数算法与代码.mkv_000337.746](pic/pnp/81. 多机单词计数算法与代码.mkv_000337.746.jpg)

## 83

http://chenshuo.com/tcpipv2/

http://www.kohala.com/start/tcpipiv2.html

http://chenshuo.com/tcpipv2/calltree/init.html

https://github.com/chenshuo/tcpipv2

![](pic/pnp/83. 复活《TCPIP 详解第2卷》讲的4.4BSD协议栈.mkv_000004.561.jpg)

![83. 复活《TCPIP 详解第2卷》讲的4.4BSD协议栈.mkv_000505.007](pic/pnp/83. 复活《TCPIP 详解第2卷》讲的4.4BSD协议栈.mkv_000505.007.jpg)

## 84

https://www.zhihu.com/question/53747085/answer/2046814026

git show bd21010098

git blame Channel.h 

git show e2167e82

git blame Channel.h e2167e82

![](pic/pnp/84. 课程总结.mkv_000006.130.jpg)

![84. 课程总结.mkv_000445.674](pic/pnp/84. 课程总结.mkv_000445.674.jpg)



