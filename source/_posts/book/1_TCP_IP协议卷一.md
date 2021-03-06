---
title: TCP/IP协议卷一阅读笔记
date: 2018-06-13 15:58:34
tags:
- 计算机网络
categories:
- 读书笔记
---
## 零 · 概述

### I · OSI参考模型

| 名称   | 例子                                                         |
| ------ | ------------------------------------------------------------ |
| 应用层 | 指定完成某些用户初始化任务的方法，应用协议通常由应用开发者设计实现。FTP、Skype等 |
| 表示层 | 指定数据表示格式和转化规则的方法，如转码，加密等             |
| 会话层 | 指定由多个连接组成一个通信会话的方法，包括关闭连接、重启连接、session管理等类似 |
| 传输层 | 指定计算机系统间多个程序之间连接的方法，TCP、UDP             |
| 网络层 | 指定经过不同链路层网络间多跳通信的方法，描述了抽象的分组格式，IP |
| 链路层 | 指定经过单一链路通信的方法，多个系统共享同一介质的控制协议，以太网、Wi-Fi |
| 物理层 | 指定连接器、数据速率和如何在介质上编码                       |

> 数据在网络中传输的过程经过每一层都会附加相应的头部，上层不关心下层的协议，直接将完整的数据报当作本层的数据内容并附加协议头部

### II · 端口号

端口号是16位的非负整数(0 ~ 65535)，用于识别主机上的不同服务。主机间的服务通信都是通过端对端进行的，每个端口都标识了一种服务，如SSH(22)、FTP(20+21)、Telnet(23)、DNS(53)等等。

标准的端口号由Internet号码分配机构（IANA）分配，熟知端口（0 ~ 1023）、注册端口（1024 ~ 49151）和动态/私有端口（49152 ~ 65535）。

### III · DNS

DNS是一个分布式数据库，提供主机名和IP地址间的映射关系，它是一个应用层协议，运行依赖其他协议。



### IV · 广播和组播

- 广播

   广播是指将报文发送到网络中所有可能的接收者。

- 组播

   组播是为了减少广播开销，可以只向那些对它感兴趣的接收方发送数据。

   

## 壹 · Internet地址结构

### I · IP地址表示

​	IPv4地址采用点分四组或点分10进制表示(165.195.130.107)，采用32位表示，每8位一组共四组，每组最大值为全1即255，所以IPv4的表示范围是0.0.0.0 ~ 255.255.255.255。

|  点分四组表示   |             二进制表示              |
| :-------------: | :---------------------------------: |
|     0.0.0.0     | 00000000 00000000 00000000 00000000 |
|     1.2.3.4     | 00000001 00000010 00000011 00000100 |
|   10.0.0.255    | 00001010 00000000 00000000 11111111 |
| 165.195.130.107 | 10100101 11000011 10000010 01101011 |
| 255.255.255.255 | 11111111 11111111 11111111 11111111 |

​	IPv6地址长度128位，是IPv4的4倍，通常采用块或字段的4个十六进制数为一组，每组由:分隔。IPv6也有一些简化的表示规则：

 1. 一个块中开头的0不必书写。如 5f05:2000:80ad:5800:0058:0800:2023:1d71可写为5f05:2000:80ad:5800:58:800:2023:1d71。

	2. 全0的块可以省略，用::替代。如 0:0:0:0:0:0:0:1 简写为::1，2001:0db8:0:0:0:0:0:2简写为2001:0db8::2，为了避免歧义，::只能出现一次。

	3. IPv6中嵌入IPv4可使用混合符号形式，紧接着IPv4的部分地址块为ffff，其余部分使用点分四组格式。

    如 ::ffff:10.0.0.1就是IPv4的10.0.0.1，被称为IPv4映射的IPv6地址。

	4. IPv6地址低32位通常采用点分四组格式。如 ::0102:f001 相当于 ::1.2.240.1，被称为IPv4兼容的IPv6地址，但与映射不同。

	5. 每个块开头的0必须压缩。

	6. ::省略只能用于影响最大的地方，即压缩最多的0。

	7. a到f的十六进制用小写表示。

### II · IP地址结构

#### 1. 地址分类

| 类别 |      网络号      |          地址范围           | 网络数 | 主机号 | 主机数 |           私有网段            |
| :--: | :--------------: | :-------------------------: | :----: | :----: | :----: | :---------------------------: |
|  A   |     7位（0）     |  0.0.0.0 ~ 127.255.255.255  |  2^7   |  24位  |  2^24  |   10.0.0.0 ~ 10.255.255.255   |
|  B   |    14位（10）    | 128.0.0.0 ~ 191.255.255.255 |  2^14  |  16位  |  2^16  |  172.16.0.0 ~ 172.31.255.255  |
|  C   |   21位（110）    | 192.0.0.0 ~ 223.255.255.255 |  2^21  |  8位   |  2^8   | 192.168.0.0 ~ 192.168.255.255 |
|  D   | 28位（1110）组播 | 224.0.0.0 ~ 239.255.255.255 |  N/A   |  N/A   |  N/A   |                               |
|  E   | 28位（1111）保留 | 240.0.0.0 ~ 255.255.255.255 |  N/A   |  N/A   |  N/A   |                               |

#### 2. 子网寻址

​	为了方便站点管理自己的网络地址，划分子网由站点管理员自己分配，将基础地址中的主机号进一步划分为一个子网号和一个主机号。

##### 2.1 子网掩码

​	子网掩码是由主机或路由器使用的分配位，用来确定如何从一台主机对应IP地址中获得网络和子网信息。IP子网掩码与IP地址位数相同，前缀为1。将IP地址和掩码做与运算就得到了该IP地址的网络段，确定其在哪个子网。

​	在同一站点的不同部分，可将不同长度的子网掩码应用于相同网络号，目前大多数主机、路由器和路由协议支持**可变长度子网掩码（VLSM）**。如128.32.0.0/16: /24、 /25 和 /26分别表示不同前缀长度的掩码，这样/24拥有8位主机号（256台主机）、/25有7位主机号、/26有6位主机号，通过这样的方式改变了网络的拓扑结构。

##### 2.2 广播地址

​	每个IPv4子网中，主机号为全1时作为子网的广播地址，使用这种地址作为目的的数据报被称之为**定向广播**，这种广播可作为一个单独的数据报通过Internet路由直达目标子网，再作为一组广播数据报发送给子网内所有主机。

> 255.255.255.255是一个特殊地址，被用作本地网络广播，不会被路由器转发。

##### 2.3 IPv6地址和接口标识符

​	IPv6地址使用特殊前缀表示一个地址范围，一个IPv6地址范围是其可用的网络规模，如节点本地（同一个计算机中通信）、链路本地（同一个链路或IPv6前缀的节点）或全球性。

​	链路本地IPv6地址使用**接口标识符（IID）**作为一个单播IPv6地址的分配基础。除了地址以二进制000开始之外，IID在所有情况下都作为一个IPv6地址的低序位，必须在同一网络中有唯一前缀。通常IID为64位，由一个网络接口相关的链路层MAC地址形成（EUI-64格式）。

> IEEE标准中，EUI表示扩展唯一标识符。前24位是组唯一标识符（OUI），后40位是由组织分配的扩展标识符

### III · 特殊地址

- IPv4

|        前缀        |                    特殊用途                    |
| :----------------: | :--------------------------------------------: |
|     0.0.0.0/8      |      本地网络中的主机，仅作为源IP地址使用      |
|     10.0.0.0/8     | 专用网络（内联网）的地址，不会出现在Internet中 |
|    127.0.0.0/8     |          本来环回地址，通常127.0.0.1           |
|   169.254.0.0/16   |   链路本地地址，只用于一条链路，通常自动分配   |
|   172.16.0.0/12    |            专用网络（内联网）的地址            |
|    192.0.0.0/24    |                  IETF协议分配                  |
|    192.0.2.0/24    |          批准用于文档中TEST-NET-1地址          |
|   192.88.99.0/24   |              用于6to4（任播地址）              |
|   192.168.0.0/16   |            专用网络（内联网）的地址            |
|   198.18.0.0/15    |               用于基准和性能测试               |
|  198.51.100.0/24   |                 TEST-NET-2地址                 |
|   203.0.113.0/24   |                 TEST-NET-3地址                 |
|    224.0.0.0/4     |       IPv4组播地址，仅作为目的IP地址使用       |
|    240.0.0.0/4     |                    保留空间                    |
| 255.255.255.255/32 |           本地网络（受限的）广播地址           |

- IPv6

|     前缀      |                     特殊用途                     |
| :-----------: | :----------------------------------------------: |
|     ::/0      |             默认路由条目，不用于寻址             |
|    ::/128     |          未指定地址，可作为源IP地址使用          |
|    ::1/128    |                   IPv6环回地址                   |
| ::ffff:0:0/96 | IPv4地址映射，不会出现在分组头部，只用于内部主机 |
|   2001::/32   |                    Teredo地址                    |
| 2001:10::/28  |        ORCHI（覆盖可路由加密散列标识符）         |
| 2001:db8::/32 |             用于文档和实例的地址范围             |
|   2002::/16   |              6to4隧道中继的6to4地址              |
|   fc00::/7    |                唯一的本地单播地址                |
|   fe80::/10   |                 链路本地单播地址                 |
|   ff00::/8    |        IPv6组播地址，仅作为目的IP地址使用        |



## 贰 · 链路层

### I · 以太网

​	由于多个站点共享同一网络，所以每个以太网接口需要实现一种分布式算法，来控制数据发送。这种特定的方法称为**带冲突检测的载波监听多路访问（CSMA/CD）**它协调哪些计算机可访问共享介质，同时不需要其他特殊协议或同步。

​	采用CSMA/CD，一个站首先检测目前网络上正在发送的信号，并在网络空闲时发送自己的帧。如与其他站同时发送，发生重叠的电信号被检测为一次碰撞，每个站点等待一个随机时间后再次尝试。在任何给定的时间内，网络中只能有一个帧传输，这样的访问方式称之为**介质访问控制（MAC）**协议。

​	一个交换式以太网包含一个或多个站，每个站使用一条专线连接到交换机端口，大多数情况下，交换式以太网以全双工方式运行，并且不需要CSMA/CD算法。

#### 1. 以太网帧格式

|        字段        | 大小（字节） |                        作用                        |
| :----------------: | :----------: | :------------------------------------------------: |
|        前导        |      7       |                        同步                        |
|  帧开始符（SFD）   |      1       |             标明下一字节为目的MAC字段              |
| 目的MAC地址（DST） |      6       |                   指明帧的接收者                   |
|  源MAC地址（SRC）  |      6       |                   指明帧的发送者                   |
|     长度或类型     |      2       |             帧数据的字段长度或协议类型             |
|     数据和填充     |   46~1500    |                     高层的数据                     |
|     帧校验序列     |      4       | 判断是否传输错误的一种方法，如果发现错误，丢弃此帧 |

- 帧校验序列/循环冗余校验

  循环冗余校验（CRC）字段位于尾部，32位，通过一系列计算方法算出值若与FCS字段中数据不同，则说明帧可能在传输过程中受损，通常被丢弃。

- 帧大小

  以太网帧最小64字节，要求数据区长度最小为48字节。传统以太网帧最大长度为1518（4字节CRC和14字节头部），最大传输单元MTU则问1500字节。

#### 2. 通讯传输

- 全双工

  允许数据在两个方向上同时传输，即A => B的同时 B => A

- 半双工

  同一时间段内，只允许单方向通讯


### II · 网桥和交换机

> 网桥也称之为桥接器，是连接局域网的存储转发设备，用它可以使用完全具有相同或相似体系结构网络系统的连接，扩展网络的距离和范围，提高网络的性能、可靠性和安全性。

交换机本质上是高性能网桥，用于连接多个物理的链路层网络。

### III · 无线局域网---IEEE 802.11（Wi-Fi）

802.11帧有3种类型：管理帧、控制帧和数据帧。

- 管理帧

  用于创建、维持、终止站和接入点之间的连接。也被用于确定是否采用加密、传输网络名称、支持哪种传输速率、采用的时间数据库等，这些帧提供接入的必要信息。

- 控制帧：RTS/CTS和ACK

  控制帧和帧确认被用于一种流量控制方式。

- 数据帧、分片和聚合

  网络中大多数是数据帧，支持帧分片与聚合。

### IV · 点到点协议

> PPP表示点到点协议，它是一个协议集合，而不是一个单一的协议。
>
> 它支持建立连接的基本方法**链路控制协议（LCP）**和一系列NCP协议。

PPP是为在点对点连接上传输多协议数据包提供了一个标准方法。

- PPP具有动态分配IP地址的能力，允许在连接时刻协商IP地址
- PPP支持多种网络协议，如TCP/IP、NetBEUI、NWLINK等
- PPP具有错误检测及纠错功能，支持数据压缩
- PPP具有身份验证功能
-  PPP可以用于多种类型的物理介质上，包括串口线、电话线、移动电话和光纤（例如SDH），PPP也用于Internet接入。

### VI · 隧道

​	不同网络间建立一条虚拟的链路进行通信的方法称之为**隧道**，虚拟专用网络（VPN）就提供了这种服务，隧道在高层分组中携带低层数据。



## 叁 · 地址解析协议ARP

> 网络间传输数据报需要IP地址和有效硬件地址，地址解析协议ARP提供了IP地址到硬件地址的映射。

### I · 工作流程

主机A IP地址为192.168.1.1，MAC地址为0A-11-22-33-44-01

主机B IP地址为192.168.1.2，MAC地址为0A-11-22-33-44-02

当主机A与主机B通信时，地址解析协议可以将主机B的IP地址解析成B的MAC地址

1. 根据主机A路由表内容，确定访问主机B的转发IP地址为192.168.1.2，主机A查看自己本地ARP缓存，试图寻址主机B的MAC地址
2. 若未找到192.168.1.2对应的MAC地址，则发送ARP请求帧广播到该网段所有主机，询问192.168.1.2对应的MAC地址。网络中的主机接收到ARP请求并检查是否与自己的IP地址相同，不同则丢到该ARP请求。
3. 主机B确定ARP请求中的IP地址与自己匹配，则将主机A的IP地址和MAC地址映射添加到本地ARP缓存。
4. 主机B将自己的MAC地址封装到ARP应答直接回复主机A。
5. 主机A收到主机B的ARP应答，更新自己的ARP缓存。

### II · ARP帧格式



|       字段       | 大小（字节） |            作用             |
| :--------------: | :----------: | :-------------------------: |
|    以太网头部    |      14      |         以太网头部          |
|     硬件类型     |      2       | 指明硬件地址类型，以太网为1 |
|     协议类型     |      2       |   指明映射协议的地址类型    |
|     硬件大小     |      1       |      硬件地址的字节数       |
|     协议大小     |      1       |      协议地址的字节数       |
|        Op        |      2       |     指明该ARP的操作类型     |
| 发送方的硬件地址 |      6       |       如以太网MAC地址       |
| 发送方的协议地址 |      4       |         如IPv4地址          |
|   目的硬件地址   |      6       |        ARP请求为全1         |
|   目的协议地址   |      4       |                             |

- Op字段

  ARP请求（1）、ARP应答（2）、RARP请求（3）、RARP应答（4）

### III · ARP缓存

​	缓存超时通常与ARP缓存中的每个条目相关（也允许设置永不超时），大多情况完整的条目超时20分钟，不完整的条目超时3分钟。



## 肆 · IP协议

​	IP是TCP/IP协议族的核心协议，所有TCP、UDP、ICMP和IGMP数据都通过IP数据报传输，IP提供了一种尽最大努力交付、无连接的服务。

- IPv4数据报

|    字段    |     大小      |                   作用                   |
| :--------: | :-----------: | :--------------------------------------: |
|    版本    |      4位      |          指明协议版本，IPv4为4           |
|    IHL     |      4位      | IPv4头部的32位字的数量，最大15*32=60字节 |
|   DS字段   |      6位      |                                          |
|    ECN     |      2位      |                                          |
|   总长度   |     16位      |    IPv4数据报的总长度，最大65535字节     |
|    标识    |     16位      |      帮助标识由IPv4主机发送的数据报      |
|    标志    |      3位      |                                          |
|  分片偏移  |     13位      |                                          |
|   生存期   |      8位      |     一个数据报可经过路由器的数量上限     |
|    协议    |      8位      |      协议类型，如UDP（17）TCP（6）       |
| 头部校验和 |     16位      |       计算IPv4头部，但不检查IP数据       |
|  源IP地址  |     32位      |                                          |
| 目的IP地址 |     32位      |                                          |
|    选项    |  最大40字节   |                                          |
|   IP数据   | 最大65515字节 |                                          |

- IPv6头部

|    字段    | 大小  |                   作用                   |
| :--------: | :---: | :--------------------------------------: |
|    版本    |  4位  |          指明协议版本，IPv6为6           |
|   DS字段   |  6位  |                                          |
|    ECN     |  2位  |                                          |
|   流标签   | 20位  |                                          |
|  负载长度  | 16位  | IPv6头部不支持分片，负载长度被限制为64K  |
| 下一个头部 |  8位  | 表明IPv6头部之后其他扩展头部的存在和类型 |
|  跳数限制  |  8位  |                                          |
|  源IP地址  | 128位 |                                          |
| 目的IP地址 | 128位 |                                          |



### I · IP头部

​	正常IPv4头部大小为20字节。IPv6头部40字节，但没有任何选项，有扩展头部。网络字节序为大端存储，与很多PC的小端存储相反。

- Internet头部长度（IHL）

  IPv4限制为60字节，IPv6没有改字段，头部长度固定60字节。

- 总长度

  IPv4最大数据报长度65535字节，很多链路层不能携带这么大的数据报，而且主机不需要接收大于576字节的IPv4数据报，很多UDP协议传输数据的应用程序限制使用512字节的数据。

  IPv6头部不支持分片，其长度有负载长度获得，限制为64K，IPv6还支持一个超长数据报选项，最高达到4GB

- 生存期

  数据报可经过路由器的数量上限，当该字段为0时，数据报被丢弃，并使用一个ICMP消息通知发送方。

- 协议

  IPv4头部协议字段标识数据报所用的协议类型，TCP为6、UDP为17，IPv6中使用下一个头部标识

- 头部校验和

  计算IPv4头部数据有效性，但并不监测IP数据，IPv6不存在校验。

- 目的地址和源地址

  IPv4使用32位，IPv6使用128位。

### II · IP转发

​	主机发送IP数据报，如果目的地是直连的主机或共享网络，IP数据报直接发送到目的地，否则主机将数据报发送到路由器，由该路由器将数据报交付到目的地。

#### 1. 路由表（转发表）

​	IP数据报转发时通过路由表配置信息进行转发，通常包含以下字段：

- 目的地：32位字段（IPv6为128位），用于与掩码操作相匹配
- 掩码：32位（IPv6为128位），与目的地做与操作查询网段
- 下一跳：下一个路由器或主机IP地址，数据报将被转发到该地址
- 接口：包含IP层使用的标识，以确定将数据报发送到下一跳网络接口

#### 2. 转发行为

​	主机或路由器的IP层向下一跳发送数据报时，首先检查数据报中的目的IP地址，并在路由表中使用该IP地址执行**最长前缀匹配算法**：

 	1. 检索路由表，将该IP地址与每一条的掩码做与运算，并与同一转发条目的目的地做比较，若满足则匹配，匹配的位数越多，说明约好。
		2. 选择最匹配的条目，并将下一跳地址设置。



## 伍 · DHCP

### I · 动态主机配置协议

​	DHCP是一种流行的C/S协议，用于为主机指定配置信息，获取IP地址、子网掩码、路由器IP地址、DNS服务器IP地址。DHCP使用租用的概念协商配置信息可用，客户机可向DHCP服务器请求续订租约，以延长配置可用。

​	DHCP有两个主要部分：地址管理和配置数据交付。地址管理用于IP地址的动态分配并提供地址租用，配置数据交付包括DHCP协议的消息格式和状态机。

​	DHCP提供三种地址分配：自动分配、动态分配和手动分配。动态分配是从服务器配置池中获取一个可撤销的IP地址，自动分配从服务器配置池中获取一个不可撤销的IP地址，手动分配则不从服务器配置池中获取IP地址。

#### 1. 地址池和租用

​	在动态分配中，DHCP客户机请求分配一个IP地址，服务器从可用的地址池中选择一个地址作为响应，通常情况这个地址池是专门为DHCP用途而分配的一个连续IP地址范围。

​	分配给客户机的地址只在一段时间内有效，我们称之为**租用期**。租用期是DHCP服务器的一个重要配置参数，其时间的长短应根据网络不同而选择，客户机会在租用期过半时尝试续订租约。

#### 2. DHCP协议操作

​	当一台新的客户机连接到网络时，它首先发现可用的DHCP服务器，以及它们能够提供的地址，然后客户机决定使用哪台服务器和哪个地址，并向提供地址的服务器发出请求（同时通知所有的服务器），除非服务器在此期间将该地址分配出去，否则它通过确认将地址分配给请求的客户机z

DHCP交换过程：

1. 客户机广播DHCPDISCOVER消息
2. 接收到请求的每台服务器都会响应一个DHCPOFFER消息（包含所要配置的信息）
3. 客户机收到一个或多个DHCPOFFER响应，确定自己想要的并广播一个包括服务器标识符选项的DHCPREQUEST消息（包含自己选中的信息）
4. 服务器收到与自己相关的REQUEST消息，并发送DHCPACK消息（服务器无法分配则发送DHCPPNAK消息），与自己不相关的服务器则丢弃该REQUEST信息
5. 客户机收到DHCPACK消息和其他相关的配置信息，会检测地址是否被使用，如果客户机确定该地址被使用，则不使用该地址，并向服务器发送DHCPDECLINE消息，通知该地址不能使用，经过一段时延后，客户机可重试，如果客户机在租约到期前发起该地址，发送一个DHCPRELEASE消息
6. 客户机在已有一个IP地址并更新租约时，会跳过DISCOVER和OFFER过程，直接REQUEST



## 陆 · 防火墙和网络地址转换

### I · 防火墙

​	最为常见的两种防火墙是***代理防火墙***和***包过滤防火墙***，它们的区别主要是所操作的协议栈层次以及由此决定的IP地址和端口号的使用。包过滤防火墙是一个互联网路由器，能够丢弃特定条件的数据包，代理防火墙则是一个多宿主的服务器主机，通常不会在IP协议层路由IP数据报。

#### 1. 包过滤防火墙

​	包过滤防火墙作为一个路由器可选择性的丢弃数据包，一般可配置为丢弃或转发数据包头中符合特定标准的数据包，这些标准被称之为过滤。过滤一般包括IP地址或者选项、ICMP报文类型以及确定端口号的TCP或UDP服务。

#### 2. 代理防火墙

​	代理防火墙不是真正意义上的路由器，本质是一个或多个应用层网关的主机，该主机拥有多个网络接口，能够在应用层中继两个连接/关联之间特定类型的流量，不会做IP转发。



### II · 网络地址转换NAT

​	NAT本质上是一种允许在互联网的不同地方重复使用相同IP集的机制。NAT必须重写每个数据包的寻址信息，以便私有地址空间的系统和Internet正常通信。NAT的工作原理是重写通过路由器的数据包识别信息。

#### 1. 传统NAT

- 基本NAT

  基本NAT只执行IP地址的重写，本质上就是将私有地址改写为一个公共地址，但并不会减少公网IP的数量。

- NAPT

  NAPT使用传输层标识符（TCP和UDP端口、ICMP查询标识符）来确定一个特定的数据包和内部哪台主机关联，这样可以大大减少公网IP的数量。



## 柒 · ICMP

​	IP协议本身没有为终端提供方法来发现哪些IP数据报发送失败，也没有提供直接的方式来获取诊断信息，使用ICMP和IP结合可以提供诊断和控制信息。ICMP使用IP协议传输，但也不是传输层协议，位于网络层与传输层之间

### I · ICMP报文

​	ICMP报文分为两大类：有关IP数据报传递的ICMP报文（差错报文）、有关信息采集和配置的ICMP报文（查询或信息类报文）。

- ICMPv4报文类型

| 类型 |  正式名称  |          用途          |
| :--: | :--------: | :--------------------: |
|  0   |  回显应答  | 回显ping应答，返回数据 |
|  3   | 目的不可达 |   不可达的主机/协议    |
|  4   |  源端抑制  |    表示拥塞（弃用）    |
|  5   |   重定向   |  表示应用使用的路由器  |
|  8   |    回显    | 回显ping请求，返回数据 |
|  9   | 路由器通告 | 指示路由器地址/优先级  |
|  10  | 路由器请求 |     请求路由器通告     |
|  11  |    超时    |        资源耗尽        |
|  12  |  参数问题  |  有问题的数据包或头部  |

- ICMPv6报文类型

| 类型 |   正式名称   |           描述           |
| :--: | :----------: | :----------------------: |
|  1   |  目的不可达  | 不可达的主机、端口、协议 |
|  2   |  数据包太大  |         需要分片         |
|  3   |     超时     |    跳数用尽或重组超时    |
|  4   |   参数问题   |    畸形的数据包或头部    |
| 127  | 差错报文扩充 |   为更多的差错报文保留   |
| 128  |   回显请求   |         ping请求         |
| 129  |   回显应答   |         ping应答         |



## 捌 · UDP和IP分片

​	UDP是一种保留消息边界的简单的面向数据报的传输层协议，它不提供差错纠正、队列管理、重复消除、流量控制和拥塞控制，仅提供差错监测，但并不保证数据可达。

### I · UDP头部

|    字段    |   大小   |                        作用                         |
| :--------: | :------: | :-------------------------------------------------: |
|  源端口号  |  2字节   |             可选，若不要求回复可设置为0             |
| 目的端口号 |  2字节   |            用来帮助分离从IP层进入的数据             |
|    长度    |  2字节   | UDP头部和数据的总长度，冗余字段，IP中带有数据报长度 |
|   校验和   |  2字节   |    端到端的校验，计算IP头部的源和目的IP地址得到     |
|  负载数据  | 长度可变 |                      UPD数据区                      |

- 检验和

  - UDP数据报长度可以是奇数，而校验和算法只相加16位字，所有在奇数长度的情况下，UDP的处理过程是在奇数长度的数据报尾部追加一个值为0的填充字节，但该字节不会被发送出去，仅作校验使用
  - UDP计算校验和时，包含了IPv4的一个12字节伪头部或IPv640字节的伪头部，它和填充字节一样不会被发送出去，仅作校验使用

  > IPv6没有校验数据，所以要求UDP必须使用校验和



### II · IP分片

​	链路层通常对可传输的数据帧的最大长度有一个限制，为了保证IP数据报与链路层一致，引入了IP**分片**和**重组**，当IP层接收到一个数据报时，它会判断从本地哪个接口发送以及要求的MTU，对于较大的数据报，采用分片处理。IPv4分片可以在源和目的间任意的中间路由进行，IPv6仅允许在源主机进行分片。

​	IP数据报分片后，只有到达目的后才会重组，一是为了减轻网络路由转发的负担，二是同一数据报的不同分片可能经过不同路由到达。

- 重组超时

		一个数据报的任一分片到达时，IP层就得启动一个计时器（并且收到新分片时也不会重置），当超时后，目标机器回复一个ICMPv4超时消息，通知发送方数据已经丢失。



## 玖 · 域名系统DNS

​	DNS是一个分布式的C/S网络数据库，TCP/IP应用使用它来完成主机名称和IP地址之间的映射。

### I · DNS命名空间

​	DNS命名空间是一棵域名树，位于顶部的树根未命名，树的最高层就是顶级域名（TLD），包括通用顶级域名（gTLD），国家代码顶级域名（ccTLD）、国际化国家代码顶级域名（IDN ccTLD）。

​	gTLD分为几类：通用、通用限制和赞助。通用gTLD是开放的，可无限制使用，其他通用限制和赞助有一定的规则：edu用于教育、mil和gov用于美国的军事和政府机构、int用于国际组织机构等。

### II · 缓存

​	大部分DNS服务器利用缓存的信息来应答查询请求，每个DNS记录都有自己的TTL用以控制其缓存时间，并且缓存对解析成功以及不成功的记录一视同仁（错误解析也会记录缓存）。

### III · DNS协议

​	DNS协议由两个主要部分组成：执行DNS查询/响应协议、名称服务器交换数据库记录协议。DNS使用递归查询的方式请求主机名映射地址。

#### 1. DNS消息格式

​	基本的DNS消息以固定的12字节开头，其后跟随4个可变长度的区域：查询、回答、授权记录和额外记录。

|                      字段                      |                            作用                             |
| :--------------------------------------------: | :---------------------------------------------------------: |
|                 事务ID（16位）                 |        由客户端设置，服务器返回，用于匹配响应和查询         |
|                   QR（1位）                    |                    0表示查询，1表示响应                     |
|                 OpCode（4位）                  |             0为标准，4为通知，5为更新，其它弃用             |
|                   AA（1位）                    |               表示授权回答（与缓存回答相对）                |
|                   TC（1位）                    |      表示截断的消息，使用UDP时，该字段表示超过512字节       |
|                   RD（1位）                    |                  表示该服务器期望递归查询                   |
|                   RA（1位）                    |        表示该服务器支持递归可用，根服务器一般不支持         |
|                   Z   (1位)                    |                      目前必须为0，保留                      |
|                   AD（1位）                    |                         信息已授权                          |
|                   CD（1位）                    |                        禁用安全检查                         |
|                  RCODE（4位）                  |           0表示没有差错，3表示名称差错或不存在域            |
|   QDCOUNT/ZOCOUNT<br />（查询数/区域数）16位   |                                                             |
| ANCOUNT/PRCOUNT<br />（回答数/先决条件数）16位 |                                                             |
|   NSCOUNT/UPCOUNT<br />（授权记录数/更新数）   |                                                             |
|      ARCOUNT/ADCOUNT<br />（额外信息数）       |                                                             |
|          区段:问题,回答,授权,额外信息          | 区段(用于DNSUPDATE):区 域,先决条件,更新,额外信息(可 变长度) |

> 为了避免数据冗余，在使用数据标签时会有一中压缩机制，使用偏移量的形式降低冗余字符串。



#### 2. TCP或UDP

​	对于TCP和UDP，DNS的端口号为53。DNS消息通常封装在UDP/IPv4数据报中，长度限制为512字节，当解析器发出一个查询消息，而返回的响应消息中TC位字段被设置（“被截断”）时，其真实响应消息长度超过512字节，服务器只返回前面的512字节，该解析器可能会再次使用TCP发送请求消息，这样就允许超过512字节的消息传输。

​	当一个区域的辅助名称服务器开启时，通常从该区域的主名称服务器执行区域传输，区域传输由计时器或DNS NOTIFY消息引起。完全区域传输时使用TCP（消息很大），增量区域传输时，只有更新条目被传输，优先使用UDP，但若消息过大，依旧使用TCP传输。

> 当使用UDP传输时，解析器和服务器软件都必须执行自己的超时和重传，建议起始超时至少4秒，随后指数增长，Linux通过/etc/resolv.conf文件（timeout和attempts）来改变重传超时参数。



#### 3. 资源记录类型

|  类型   |                          描述和目的                          |
| :-----: | :----------------------------------------------------------: |
|    A    |                         IPv4地址记录                         |
|   NS    |           名称服务器，提供区域授权名称服务器的名称           |
|  CNAME  |  规范名称；将一个名称映射为另一个（提供一种形式的名称别名）  |
|   SOA   | 授权开始，为区域提供授权信息（名称服务器，email，序列号，区域传输计时器） |
|   PTR   |                指针，提供地址到规范名称的映射                |
|   MX    |        邮件交换器，为一个域提供电子邮件处理主机的名称        |
|   TXT   |                      文本，提供各种信息                      |
|  AAAA   |                         IPv6地址记录                         |
|   SRV   |                服务器选择，通用服务的传输终点                |
|  NAPTR  |               名称授权指针，支持交替的名称空间               |
|   OPT   |        伪PR，支持更大的数据报、标签、EDNS0中的返回码         |
|  IXFR   |                         增量区域传输                         |
|  AXFR   |                  完全区域传输，通过TCP运载                   |
| （ANY） |                         请求任意记录                         |

- 地址（A, AAAA）和名称服务器记录

  DNS中最重要的记录是（A,AAAA）记录和名称服务器（NS）记录。A记录包含32位IPv4地址，AAAA记录包含128位的IPv6地址，NS记录包含授权DNS服务器名称，该服务器包含一个特定区域的信息。

  ```shell
  # 使用dig命令可以查询DNS记录
  dig +nostats -t ANY www.baidu.com
  
  ; <<>> DiG 9.8.3-P1 <<>> +nostats -t ANY www.baidu.com
  ;; global options: +cmd
  ;; Got answer:
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60094
  ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 5, ADDITIONAL: 5
  
  ;; QUESTION SECTION:
  ;www.baidu.com.			IN	ANY
  
  ;; ANSWER SECTION:
  www.baidu.com.		1074	IN	CNAME	www.a.shifen.com.
  
  ;; AUTHORITY SECTION:
  baidu.com.		50334	IN	NS	dns.baidu.com.
  baidu.com.		50334	IN	NS	ns4.baidu.com.
  baidu.com.		50334	IN	NS	ns7.baidu.com.
  baidu.com.		50334	IN	NS	ns2.baidu.com.
  baidu.com.		50334	IN	NS	ns3.baidu.com.
  
  ;; ADDITIONAL SECTION:
  dns.baidu.com.		50453	IN	A	202.108.22.220
  ns2.baidu.com.		50453	IN	A	61.135.165.235
  ns3.baidu.com.		50453	IN	A	220.181.37.10
  ns4.baidu.com.		50453	IN	A	220.181.38.10
  ns7.baidu.com.		50335	IN	A	180.76.76.92
  
  # 清楚DNS缓存记录
  ifconfig flushall
  ```

- 规范名称（CNAME）记录

  CNAME记录代表规范名称的记录，用于将单一域名的别名引入到其他DNS命名系统中。通常情况下，建立一个域名到IP地址的A记录，然后通过CNAME到这个域名，这样在IP更换时仅修改A记录即可。

- 权威（SOA）记录

  DNS中，每个区域有一个授权记录，使用授权启动（SOA）的PR类型，记录了DNS名称空间和服务器之间的授权联系，用于识别主机的名称、提供官方永久性数据库、负责方的email地址、区域更新参数和默认TTL。



## 拾 · 传输控制协议（TCP）

> TCP提供了一种面向连接的、可靠的字节流传输服务。

### I · TCP头部和封装

|       字段        |                             作用                             |
| :---------------: | :----------------------------------------------------------: |
|  源端口（16位）   |                      发送方使用的端口号                      |
| 目的端口（16位）  |                      接收方使用的端口号                      |
|  序列号（32位）   | 标识发送端到接收端数据流的一个字节，代表包含该序列号的报文数据的第一个字节 |
|  确认号（32位）   | 也称ACK号，该确认号的发送方期待接收下一个序列号（成功接收的序列号+1） |
|  头部长度（4位）  |   TCP头部长度，TCP限制最大60字节，即40字节选项，20字节头部   |
|    保留（4位）    |                             保留                             |
|    CWR（1位）     |              拥塞窗口（发送方降低它的发送速率）              |
|    ECE（1位）     |         ECN回显（发送方接收到了一个更早的拥塞通告）          |
|    URG（1位）     |                   紧急（紧急指针字段有效）                   |
|    ACK（1位）     |                    确认（确认号字段有效）                    |
|    PSH（1位）     |          推送（接收方应尽快给应用程序传送这个数据）          |
|    RST（1位）     |             重置连接（连接取消，经常是因为错误）             |
|    SYN（1位）     |                用于初始化一个连接的同步序列号                |
|    FIN（1位）     |            该报文段的发送方已经结束向对方发送数据            |
| 窗口大小（16位）  |          TCP流量控制由窗口大小字段控制，由ACK号指定          |
| TCP校验和（16位） |                    强制使用，确保数据可用                    |
| 紧急指针（16位）  | 一个必须要加到报文段的序列号字段上的正偏移，以产生紧急数据的最后字节序列号 |
|  选项（可变量）   |                                                              |



### II · TCP的连接与终止

​	一个TCP连接由一个四元组构成，两个IP地址和两个端口号，更准确的是一对端点或套接字构成，其中通信的每一端都由一对（IP地址，端口号）唯一标识。

​	一个TCP连接通常分为3个阶段：启动、数据传输和关闭。

- 连接建立：三次握手

   1. 主动开启者（通常为客户端）发送一个SYN报文段，并指明连接端口号和客户端初始序列号（ISN(c)），通常客户端还会借此发送一个或多个选项。客户端发送的这个SYN报文称作段1。
   2. 服务器也发送自己的SYN报文段作为响应，并包含它的初始序列号(ISN(s))，称之为段2。为了确认客户端SYN，服务器将其包含的ISN(c)+1作为ACK返回，因此每发送一个SYN，序列号就会自动加1，如果出现丢失的情况，该SYN段将会重传。
   3. 为了确认服务器的SYN，客户端将ISN(s)+1作为ACK返回给服务器，这个段称之为段3。

- 连接关闭：四次挥手

   1. 连接的主动关闭方发送一个FIN段，并指明自己当前的序列号K，还包含一个ACk段用于确认对方最近一次发送的数据。
   2. 连接的被动关闭方将K+1作为响应的ACK值，表明成功接收到发送方的FIN。
   3. 接着被动关闭方转换为主动关闭方，并发送自己的FIN携带序列号L。
   4. 为了完成关闭连接，最后发送的报文段包含一个ACK用于确认上一个FIN，如果该FIN丢失，那么发送方将重新传输直到接收到一个ACK确认为止。

   


#### 1. TCP半关闭

​	TCP支持半关闭操作，场景如“我已经完成了数据的发送工作，并发送一个FIN给对方，但是我仍然希望接收到来自对方的数据直到它发送一个FIN给我”。

1. 连接的主动关闭方发送一个FIN段，并指明自己当前的序列号K，还包含一个ACk段用于确认对方最近一次发送的数据。
2. 连接的被动关闭方将K+1作为响应的ACK值，表明成功接收到发送方的FIN。
3. **此时主动关闭方已经处于半关闭状态，但仍然会正常接收对方发送过来的数据，直到对方的FIN**。
4. 接着被动关闭方转换为主动关闭方，并发送自己的FIN携带序列号L。
5. 为了完成关闭连接，最后发送的报文段包含一个ACK用于确认上一个FIN，如果该FIN丢失，那么发送方将重新传输直到接收到一个ACK确认为止。

#### 2. 初始化序列号

​	当一个连接打开时，任何拥有合适的IP地址、端口号、符号逻辑的序列号以及正确校验和的报文段都将被对方接收，这样会导致TCP报文段排序混乱的情况，为了解决这些问题，初始序列号尤为重要。

​	在发送用于建立连接的SYN之前，通信双方会选择一个初始序列号，初始序列号会随时间而改变，因此每个连接都拥有不同的初始序列号。TCP报文只有同时具备4元组与当前活动窗口的序列号才会确定对方是正确的，但这样会导致TCP的脆弱性，任何人都可以伪造TCP报文随时关闭连接，所以是初始序列号相对难猜是很好的保证手段。现代系统通常采用半随机的方法选择初始序列号。

> Linux系统采用一个相对复杂的过程来选择它的初始序列号。它采用基于时钟的方案，并针对每个连接为时钟设置随机的偏移量，随机偏移量是在连接标识（即4元组）的基础上利用加密散列函数得到的，散列函数输入每5分钟改变一次。

#### 3. 连接建立超时

​	一些系统可以配置发送初始SYN的次数，但通常选择一个相对较小的数字5。在Linux系统中，系统配置变量net.ipv4.tcp_syn_retries表示了在一次主动打开请求中尝试重新发送SYN报文的最大次数，变量net.ipv4.tcp_synack_retreis表示了响应对方一个主动打开请求时尝试重新发送SYN+ACK报文的最大次数。

> 每次重试使用指数回退的方式



### III · TCP选项

- 最大段大小选项
   最大段大小指TCP协议所允许的从对方接收到的最大报文段，该大小只记录TCP数据的字节数不包括TCP头部和IP头部。当建立一条TCP连接时，通信的每一方都要在SYN报文段的MSS选项中说明自己允许的最大段大小，该值默认为536字节，这样就正好组成一个576（20+20+536）字节的IPv4数据报。

- 选择确认选项

   由于采用累积ACK确认，TCP不能判断是否接收到之前的数据，由于接收的数据是无序的，所以接收到数据的序列号也是不连续的，这种情况下，TCP接收方的数据队列会出现空洞情况。TCP选择确认（SACK）能够很好的防止空洞的出现。

   通过允许选择确认选项，通信双方了解到自身具备发布SACK信息的能力。当接收到乱序的数据时，提供一个SACK选项来描述这些乱序数据，从而帮助有效重传。SACK信息保存于SACK选项中，包含了接收方已经成功接收的数据块的序列号范围。

- 窗口缩放选项

   窗口缩放选项能够有效的将TCP窗口广告字段从16位增加至30位。TCP头部不需要改变窗口广告字段大小，使用另一个选项作为这16位的比例因子，是窗口字段值有效左移，这样就可以扩充至原来的$2^s$倍（s为比例因子范围0-14）。该选项只能出现于一个SYN报文段中，因此连接建立后比例因子是双向绑定的。通信双方都需要在SYN报文段中包含该选项，若未成功协商，则比例因子被设置为0。

- 时间戳选项与防回绕序列号

   时间戳选项要求发送方在每一个报文段中添加2个4字节的时间戳数值，接收方会在确认中反映这些数值，允许发送方针对每个接收的ACK估算TCP连接往返时间，利用该估算时间设置重传超时。

- 用户超时选项

   用户超时数值指明了TCP发送方在确认对方未能成功接收数据之前愿意等待该数据ACK确认的时间。用户超时选项的数值是建议性的：

   1. 当TCP连接达到3次重传阈值时通知应用程序
   2. 当大于100秒时应该关闭连接

   ```
   # 设置用户超时的方法
   USER_TIMEOUT = min(U_LIMIT, max(ADV_UTO, REMOTE_UTO, L_LIMIT))
   # ADV_UTP 是本端告知远端通信方用户超时选项数值
   # REMOTE_UTO 是远端通信方告知的用户超时选项数值
   # U_LIMIT 是本地系统对用户超时选项设置的数值上界 L_LIMIT是下界
   ```

- 认证选项

   当发送数据时，TCP会根据共享秘钥生成一个通信秘钥，并根据一个特殊的加密算法计算散列值，接收者拥有相同的密钥，也能够产生通信密钥。借助通信密钥，接收者可以确认达到的报文段是否在传输过程中被篡改，设置该选项是为了针对各种TCP欺骗攻击提供有力的防护措施。



### IV · TCP路径最大传输单元发现

​	TCP常规的路径最大传输单元发现如下：在建立一个连接时，TCP使用对外接口的最大传输单元的最小值，或根据通信对方声明的最大段大小来选择发送方的最大段大小(SMSS)。路径最大传输单元发现不允许TCP发送有超过另一方所声明的最大段大小行为，若对方未指明最大段大小值，发送方默认采用536字节。

> 一条连接的两个方向的路径最大传输单元是不同的。

​	一旦为发送方的最大段大小选定了初始值，TCP通过这条连接发送的所有IPv4数据报都会对DF字段进行设置。如果接收到PTB消息，TCP就会减少段的大小，然后用修改过的段大小进行重传，若PTB消息已包含了下一跳推荐的最大传输单元，段大小的数值可以设置为下一跳最大传输单元的数值减去IP与TCP头部的大小。



### V · TCP状态转换

#### 1. TCP状态转换图

- CLOSED状态（初始状态）
  - open(p) ，转移到 LISTEN状态，用于监听连接，被动打开
  - open(a) 发送SYN，转移到SYN_SENT状态
- LISTEN状态
  - 发送SYN， 转移到SYN_SENT
  - 接收SYN，发送SYN+ACK 转移到SYN_RCVD状态
- SYN_SENT状态
  - close或超时，转移到CLOSED状态
  - 接收SYN，发送SYN+ACK，转移到SYN_RCVD状态
  - 接收SYN+ACK，发送SYN， 转移到ESTABLISHED状态
- SYN_RCVD状态
  - 接收ACK，转移到ESTABLISHED状态
  - 发送FIN(close)，转移到FIN_WAIT_1状态
  - 接收RST，转移到LISTEN状态
- ESTABLISHED状态
  - 接收FIN，发送ACK，转移到CLOSE_WAIT状态
  - 发送FIN(close)，转移到FIN_WAIT_1状态
- CLOSE_WAIT状态（被动关闭）
  - 发送FIN(close)，转移到LAST_ACK状态
- LAST_ACK状态（被动关闭）
  - 接收ACK，转移到CLOSED状态
- FIN_WAIT_1状态（主动关闭）
  - 接收ACK，转移到FIN_WAIT_2状态
  - 接收FIN+ACK，发送ACK，转移到TIME_WAIT状态
  - 接收FIN，发送ACK， 转移到CLOSING状态
- FIN_WAIT_2状态（主动关闭）
  - 接收FIN，发送ACK，转移到TIME_WAIT状态

- CLOSEING状态（主动关闭）
  - 接收ACK，转移到TIME_WAIT状态
- TIME_WAIT状态(2MSL) （主动关闭）
  - 主动关闭，转移到CLOSED状态

   

#### 2. TIME_WAIT状态

​	TIME_WAIT状态也称为2MSL等待状态，该状态中，TCP将会等待两倍于**最大段生存期(MSL)**的时间，也称为加倍等待。每个实现都必须为最大段生存期选择一个数值，它代表任何报文在被丢弃前在网络中被允许存在的最长时间。

> Linux系统中，net.ipv4.tcp_fin_timeout记录了2MSL（单位为秒）。
>
> 当TCP执行一个主动关闭并发送最终ACK时，连接必须处于TIME_WAIT状态并持续2MSL时间，这样就能够让TCP重新发送最终ACK以避免出现丢失情况。重发最终ACK是因为通信对方在未收到最终ACK情况下会持续重传FIN。

​	当TCP处于等待状态，通信双方将该连接定义为不可重新使用，只有2MSL等待结束时，或一条新连接使用的初始序列号超过了连接之前的实例所使用的最高序列号时，或者允许使用时间戳选项来区分之前连接实例的报文段，这条连接才能被再次使用。

> 2MSL存在强约束，如一个端口号处于2MSL等待状态，那么该端口就不能被再次使用。伯克利套接字API中，SO_REUSERADDR选项可绕开这种约束限制。

- 静默时间

  如果一台与处于TIME_WAIT状态下的连接相关联的主机崩溃，然后再MSL内重新启动，并且使用相同的IP地址与端口号，那么该连接在主机崩溃之前产生的延迟报文段被认为属于主机重启后创建的新连接，并且不会考虑新连接是如何选择初始序列号的。

  为了防止上述情况，在崩溃或重启后TCP协议应该当在创建新连接之前等待相当于一个MSL的时间，该段时间被称之为**静默时间**。

#### 3. FIN_WAIT_2状态

​	在FIN_WAIT_2状态，TCP通信段会一直等待接收另一方的FIN（关闭操作），否则将会一直保持这种状态，而对方也会处于CLOSE_WAIT状态，并且用于维持，直到应用程序关闭它。

​	若主动关闭的应用程序执行的是完全关闭操作，而不是半关闭（还希望接收数据），那么就会设置一个计时器，如果计时器超时时连接时空的，那么TCP连接就会转移到CLOSED状态。

> Linux系统中，net.ipv4.tcp_fin_timeout的数值来设置计时器秒数，默认60s。 



#### 4. CLOSE_WAIT状态

​	被动关闭的一方在接收到FIN后会转移到CLOSE_WAIT状态，此时被动关闭方应该关闭连接发送FIN，已转移到LAST_ACK状态，但由于某些原因，如被动关闭方崩溃，还要数据没有发送完毕等待，导致FIN未发送，这样会使自己长时间处于CLOSE_WAIT状态，对方处于FIN_WAIT_2状态，浪费资源。



### VI · 报文段重置

> 当发现一个到达的报文段对于相关连接不正确，TCP就会发送一个重置报文段。

#### 1. 针对不存在端口的连接请求

​	通常情况下，当一个连接请求到达本地却没有相关端口监听时会产生一个重置报文段。UDP协议规定，当一个数据报到达一个不能使用的目的端口时会产生一个ICMP目的不可达的小小，TCP协议使用重置报文段来完成相关工作。

```bash
[root@~]$ telnet localhost 9999
Trying 127.0.0.1...
telnet: connect to address 127.0.0.1: Connection refused
```

#### 2. 终止一条连接

​	终止一条连接的正常方法是由通信一方发送一个FIN，这种方法也称为**有序释放**，因为FIN是在之前所有排队数据都已发送后才会发送，通常不会丢失数据。然而在任何时刻，可以通过发送一个重置报文段替代FIN来终止一条连接，这种方式称为**终止释放**。

终止一条连接为应用程序提供两大特性：

 	1. 任何排队的数据都将被抛弃，一个重置报文段会被立即发送出去。
	2. 重置报文段的接收方会说明通信另一端采用了终止方式，而不是一次正常关闭。

> 重置报文段不会让通信另一端做出任何响应（不会被确认），通常会有“（connection reset by peer）连接被另一端重置”的错误提示。

#### 3. 半开连接

​	如果在未通知另一端的情况下关闭或终止连接，TCP连接则处于**半开**状态，通常发生在通信一方主机崩溃的情况下，只要不尝试通过半开连接传输数据，正常工作的一段不会检测另一端已经崩溃。

#### 4. 时间等待错误

​	在TIME_WAIT的2MSL状态下，若接收到来自这条连接的一些报文段，或重置报文段，那么该状态会被破坏，这种情况称之为**时间等待错误**。因此许多系统规定，处于TIME_WAIT状态时不对重置报文段做出反应。



### VII · TCP服务器选项

#### 1. TCP端口号

```bash
[root@~]$ netstat -a -n -t
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:20001           0.0.0.0:*               LISTEN    
tcp6       0      0 :::8080                 :::*                    LISTEN 
tcp        0      0 127.0.0.1:6234          127.0.0.1:33333         ESTABLISHED
# -a 选项报告所有网络节点
# -n 选项以点分十进制形式打印IP地址，包括端口号
# -t 选项用于只选择TCP节点
```

处于LISTEN状态的本地节点会独自运行，接收到SYN报文段，就会建立一个ESTABLISHED连接，并分配一个主机尚未使用的临时端口，进入ESTABLISHED状态后就可以传输数据了。

#### 2. 进入连接队列

​	一个并行的服务器会为每一个客户端分配一个新的进程或线程，这样负责监听的服务器能够始终准备着下一个请求到来，当高并发请求到来、操作系统忙于其他高优先级、受到攻击等等情况，来不及分配进程或线程，操作系统内部会有ESTABLISHED和SYN_RCVD状态的两个队列。

Linux系统中，有以下规则：

1. 当一个请求到达(SYN报文段)，将会检查系统范围的参数`net.ipv4.tcp_max_syn_backlog(默认为1000)`，如果处于SYN_RCVD状态的连接数超过这个阈值，进入的连接将被拒绝。
2. 每一个处于LISTEN状态下的节点都拥有一个固定长度的连接队列。队列中的连接已经被TCP完全接受(即三次握手已完成)，但未被应用程序接受。应用程序会对这一队列做出限制，通常称为未完成连接(backlog)。backlog的数目必须在0与一个系统指定的最大值`net.core.somaxconn(默认128)`之间。
3. 如果LISTEN节点的队列中仍然有空间分配给新的连接，TCP模块会应答SYN并完成连接，直到接收到三次握手的第3个报文段之后，应用程序才会知道新连接。当客户端的主动打开操作完成之后，会认为服务器已经做好接收数据的准备，但服务器上的应用程序此时可能还未收到关于新连接的通知，这样服务器的TCP模块会把接收到的数据存入队列中。
4. 如果队列中已没有足够的空间分配给新的连接，TCP将会延迟对SYN做出响应。Linux坚持在能力范围内不忽略进入的连接，若系统控制变量`net.ipv4.tcp_abort_on（通常关闭）`被设定，新进入的连接会被重置报文段重置。





## 拾壹 · TCP超时与重传

​	TCP协议提供可靠数据传输服务，当数据段或确认信息丢失，TCP启动重传操作，TCP拥有两套独立机制来完成重传，一是基于时间，二是基于确认信息的构成（比一更有效）。

​	TCP在发送数据时会设置一个计时器，若计时器超时仍未收到确认信息，则会引发相应的超时或基于计时器的重传操作，计时器超时称为**重传超时（RTO）**。另一种方式重传称为**快速重传**，通常发生在没有延时的情况下，若TCP累积确认无法返回新的ACK，或者当ACK包含的选择确认信息（SACK）表明出现失序报文段时，快速重传会推断出现丢包。

### I · 超时重传

​	若TCP先于RTT开始重传，会在网络中引起不必要的重复数据；反之，延迟至远大于RTT的间隔发送重传数据，整体网络利用率会随之下降，由于RTT的测量较为复杂，根据路由与网络资源的不同，会随时间而改变，TCP必须跟踪这些变化并做出调整来维持性能。

​	TCP在收到数据后会返回确认信息，可在该信息中携带一个字节数据(采用一个特殊序列号)来测量传输该确认信息所需要的时间，每个此类的测量结果称为**RTT样本**。TCP首先根据一段时间的样本值建立好估计值，然后基于估计值设置RTO。

> RTO设置恰当是保证TCP性能的关键。每个TCP连接的RTT独立估算。



### II · 快速重传

​	TCP发送端在观测到至少dupthresh(重传ACK阈值)个重复ACK后，重传可能丢失的数据分组，不用等待重传计时器超时，不采用SACK时，在接收到有效ACK前最多只能重传一个报文，采用SACK，ACK可包含额外信息，使得发送端在每个RTT时间内可以填补多个空缺。

> 重复ACK通常与网络拥塞有关，因此会触发TCP拥塞控制。

​	




## 拾贰 · TCP数据流与窗口管理

### I · 交互式通信

​	TCP不同情况网络数据传输存在一定差异，大多数TCP报文段都包含大批量数据（如web、文件共享、电子邮件），其他少数包含交互数据（如ssh、网络游戏），批量数据段通常较大1500字节或更大、交互数据则较小一般为几十字节用户数据。

​	SSH就是典型的交互式通信，每个交互按键通常会生成一个单独的数据包，因此每个输入字符会产生4个TCP数据段：客户端的键盘输入、服务端对输入确认、服务端生成回显、客户端确认回显。



### II · 延时确认

​	利用TCP的累积ACK字段可以允许TCP延迟一段时间发送ACK，以便相同方向的确认可以同时发送，这种方法经常用于批量数据传输，采用延时ACK可以减少ACK传输数目，一定程度减轻网络负载。

​	Linux使用动态调节算法，可以在每个报文返回一个ACK（快速确认）与传统时延ACK模式相互切换。Mac OS中`net.inet.tcp.delayed_ack`来设置延时ACK，禁用延时（0）、始终延时（1）、每隔一个包回复一个ACK（2）、自动检测确认时间（3）。



### III · Nagle算法

​	Nagle算法规定，所有在传数据未收到ACK时，小的报文段（长度小于SMSS）不能被发送，并在收到ACK后，TCP收集这些小数据并整合到一个报文段发送。这种方法迫使TCP遵循等待—只有等待接收到所在传输数据的ACK后才能继续发送。



### IV · 流量控制与窗口管理

#### 1. 滑动窗口

​	TCP连接的每一端都可收发数据，数据量通过一组窗口结构来维护，每个TCP活动连接的两端都维护一个发送窗口结构和接收窗口结构。TCP以字节为单位维护窗口结构，窗口大小字段相对ACK号有一个字节的偏移量，发送端计算其可用窗口。

#### 2. 零窗口与TCP持续计时器

​	TCP通过接收端的通告窗口来实现流量控制，当窗口值为0时，阻止发送端继续发送，直到窗口恢复到非零。当接收端重新获得可用空间时，给发送端发送一个窗口更新（不包含数据），不保证可靠，TCP必须自己处理。

​	若窗口更新的ACK丢失，那么通信双方将会处于等待状态，为了防止死锁，发送端采用一个持续计时器间歇性查询接收端，查看其窗口大小。持续计时器会触发窗口探测的传输，强制要求接收端返回ACK。与TCP重传计时器类似，TCP持续计时器采用指数时间退避来计算超时。

#### 3. 糊涂窗口综合征

​	基于窗口的流量控制机制，可能会出现**糊涂窗口综合征(SWS)**，交换数据段大小不是全长而是一些较小的数据段。TCP连接两端都可能导致SWS出现：接收端的通告窗口较小，或者发送端发送的数据段较小。

​	TCP无法提前预知某一端的行为，需要遵守以下规则：

1. 接收端不应通告小的窗口值。在窗口可增长至一个全长的报文段（即接收端MSS）或者接收端缓存空间的一半（两者间取较小值）之前，不能通告比当前窗口更大的窗口值。当应用程序处理接收到的数据后使得可用缓存增大、TCP接收端需要强制返回对窗口探测的响应，时使用该规则。

2. 发送端不应发送小的报文段，应由Nagle算法控制何时发送，并且满足下列情况之一：

   a. 全长的报文段可以发送

   b. 数据段长度>=接收端通告最大窗口值的一半，可以发送

   c. 满足一条即可：1）某一ACK不是目前期盼的（未经确认的在传数据），2）该链接禁用Nagle算法

#### 4. 大容量缓存与自动调优

​	在较新的操作系统中，支持窗口自动调优。

```bash
net.core.rmem_max
net.core.wmem_max
net.core.rmem_default
net.core.wmem_default
# 自动调优使用缓存的最小值、默认值和最大值
net.ipv4.tcp_rmem = 4096 87380 174760
net.ipv4.tcp_wmem = 4096 17380 174760
```



### V · 紧急机制

​	TCP头部有个位字段URG表示“紧急数据”。当发送端TCP收到这类操作时，会进入紧急模式的特殊状态，记录紧急数据的最后一个字节，用于设计紧急指针字段，随后发送端生成的每个TCP头部都包含该字段，直到应用程序停止紧急数据操作，并且所有序列号大于紧急指针的数据都经接收端确认。



## 拾叁 · TCP拥塞控制

​	TCP是提供系统间数据可靠传输服务的协议，流量控制基于ACK数据报中的窗口大小字段完成了对发送速率的调节，当发送速率大于接收速率时会有排队机制，但若一直持续下去，资源便会耗光，造成路由器无法处理达到的流量而丢弃称之为**网络拥塞**。

- TCP拥塞检测

  TCP首要机制是重传，包括超时重传和快速重传，若网络处于拥塞崩溃状态，TCP确认重传更多的数据，导致情况恶化。当拥塞出现时，应减缓TCP发送速率，然而在实际情况中却很难做到，对于发送端来说，无法判断拥塞状况已发生，在TCP中，丢包被常用作判断拥塞发生的指标。

- 减缓TCP发送

  ```bash
  # 反应网络传输能力的变量为拥塞窗口，cwnd
  # 发送端实际可用窗口W就是接收端通知窗口awnd和拥塞窗口cwnd的最小值
  W = min(cwnd, awnd)
  ```

### I · 经典算法

#### 1. 慢启动

​	当一个新的TCP连接建立时或检测到由重传超时（RTO）导致的丢包时，需要执行慢启动。慢启动的目的是在拥塞避免探测更多可用带宽之前得到cwnd值，以及建立ACK时钟。通常TCP在建立连接时执行慢启动，直到有丢包时，执行拥塞避免算法进入稳定状态。

 	1. 建立连接后初始化cwnd = 1，表明可以传一个MSS大小数据
	2. 每经过一个RTT，cwnd = cwnd * 2，呈指数递增（收到ACK为指数递增）
	3. 当cwnd >= sshthresh时，就会进入拥塞避免算法

#### 2. 拥塞避免

​	TCP开始于慢启动阶段，一旦到达慢启动阈值(sshthresh)，TCP会进入拥塞避免阶段，cwnd每次增长值近似于成功传输的数据报大小。每收到一个ACK，cwnd会有微弱的增长(次线性)，窗口中所有报文都确认后cwnd的增长大致为数据报大小(线性)，故cwnd随RTT线性增加。

#### 3. 慢启动和拥塞避免结合

1. 建立TCP连接，cwnd初始化为1，sshthresh初始化为16，发送一个数据包
2. 发送端收到一个ACK，cwnd+=1，然后可以发送2个数据包
3. 发送端收到两个ACK，cwnd+=2，然后可以发送4个数据包
4. 发送端收到四个ACK，cwnd+=4，然后可以发送8个数据包，这样cwnd随RTT指数增长
5. 当cwnd到达sshthresh时，慢启动算法结束，进入拥塞避免算法
6. cwnd按照RTT进行线性递增，每次+1，并在某一时刻达到拥塞
7. 出现拥塞后，sshthresh设置为当前cwnd的一半，重新执行慢启动算法，然后重复之前步骤

#### 4. 快速恢复

​	当TCP收到3个重复ACK，表明启动快速重传，这时需要执行快速恢复算法。

1. 设置cwnd = sshthresh + ACK个数*MSS
2. 重传丢失的数据包
3. 若只收到dup ACK，cwnd += 1，并在允许条件下发送一个报文段
4. 若收到新的ACK，设置cwnd = sshthresh，进入拥塞避免阶段
