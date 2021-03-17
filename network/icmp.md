## ICMP - ping 背后发生了什么？

### 1. ICMP 是什么？

```markdown
ICMP(Internet Control Message Protocol), 互联网控制报文协议。ICMP 的主要功能包括：确认 IP 包是否成功到达目标地址、报告发送过程中 IP 包被废弃的原因、改善网络设置等。也就是说，在 IP 通信中，如果某个 IP 包因为某种原因未能到达目标地址，那么这个具体的原因将由 ICMP 负责通知。

example:
1. 主机A  ---> ARP 包 --->  主机B
2. 主机A 和 主机B 之间需要经过 路由器1 和 路由器2，但是由于 路由器2 的原因，ARP 包 未能到达 主机B，此时 路由器2 会向 主机A 发送一个 ICMP 目标不可达的包，表示这次发送没有成功。
3. 主机A 收到该 ICMP 包以后分解其首部和数据域后，得知具体原因。
```

### 2. ICMP 包的格式是什么？

*ICMP 报文是封装在 IP 包里面的，它工作在网络层，是 IP 协议的助手。*

+ 查询报文类型：用于诊断的查询信息；（回送应答:0、回送请求:8）

+ 差错报文类型：通知出错原因的错误消息；（目标不可达:3、原点抑制:4、重定向或改变路由:5、超时:11）

![](/Users/caiqingjing/project/private/leetcode-practice/statics/icmp_headers.jpg)

### 3. 查询报文类型

```markdown
回送消息（ICMP Echo Request Message）：
用于进行通信的主机或路由器之间，判断所发送的数据包是否已经成功到达对端的一种消息。`ping` 就是利用这个消息实现的。

回送请求（ICMP Echo Reply Message）：
回应是否城固接收到数据包的一种消息。
```

![](/Users/caiqingjing/project/private/leetcode-practice/statics/icmp_req_ack.jpg)

```markdown
标识符：用以区分是哪个应用程序发的 ICMP 包。
序号：序列号从 0 开始，每发送一次新的回送请求就会加 1，用来确认网络包是否有丢失。
选项数据：`ping` 会存放发送请求的时间值，用来说明路程的长短。
```

### 4. 差错报文类型

**目标不可达消息：**

```markdown
IP 路由器无法将 IP 数据包发送给目标地址时，会给发送端主机返回一个目标不可达的 ICMP 消息，并在这个消息中显示不可达的具体原因，原因记录在 ICMP 包头的 `代码` 字段。`代码` 值有：

0 - 网络不可达（Network Unreachable）
exp: 当路由器中的路由表匹配不到接收方 IP 的网络号时（网络号表示的是 A类、B类、C类）没太懂。

1 - 主机不可达（Host Unreachable）
exp: 当路由表中没有该主机的信息，或者该主机没有连接到网络。

2 - 协议不可达（Protocol Unreachable）
exp: 当主机使用 TCP 协议访问对端主机，但是对端主机的防火墙已经禁止 TCP 协议访问。

3 - 端口不可达（Port Unreachable）
exp: 当主机访问对端主机的 8080 端口时，防火墙没有限制，但是对端主机并未监听 8080 端口。

4 - 需要进行分片但是设置了不分片（Fragmentation needed but no frag）
exp: 端主机发送方 发送了 IP 数据报时，将 IP 首部的分片禁止标志位设置为 1。途中的路由器遇到超过 MTU 大小的数据包时，不进行分片直接进行丢弃。
```

**原点抑制消息：**

```markdown
在使用低速广域线路的情况下，连接 WAN 的路由器可能会遇到网络拥堵的问题。ICMP 原点抑制消息的目的就是为了缓和这种拥堵情况。当路由器向低速线路发送数据时，其发送队列的缓存变为 0 而无法发送出去时，可以向 IP 包的源地址发送一个 ICMP 原点抑制消息。

收到这个消息的主机借此了解在整个线路的某一处发生了拥堵的情况，从而增大 IP 包的传输间隔，减少网络拥堵的情况。

然而这种 ICMP 可能会引起不公平的网络通信，一般不被使用。
```

**重定向消息**

```markdown
如果路由器发现发送端主机使用了 `不是最优` 的路径发送数据，那么它会返回 ICMP 重定向消息给这个主机。在这个消息中包含了 `最合适的路由信息和源数据`。这主要发生在路由器持有更好的路由信息的情况下。路由器会发送重定向消息给发送端，让他下一次发送给另外一个路由器。
```

**超时消息**

```markdown
IP 包中有一个字段叫做 TTL(Time To Live, 生存周期)，它的值随着每经过一次路由器就会减 1，减到 0 时该 IP 包会被丢弃。此时路由器将会发送一个 ICMP 超时消息给发送端主机，并通知该包被抛弃。TTL 可以避免在路由控制遇到问题发生循环状况时，避免 IP 包无休止地在网络上转发的问题。可以设置一个较小的 TTL 值。
```

### 4. ping - 查询报文类型的使用

```markdown
1. 主机A  --->  `ICMP 回送请求消息数据包`  --->   主机B
2. ICMP 包中主要的两个字段：类型、序号。每发出一个请求数据包，`序号都会自动加 1`，为了能计算往返时间 RTT，它会在报文的数据部分插入发送时间。
3. IP 层将以 192.268.1.2 作为目的地址，本机 IP 地址作为源地址，协议字段设置为 1 表示是 ICMP 协议，在加上一些其它的控制信息，构建一个 IP 数据包。
4. 加上 MAC 头，如果在本地 ARP 映射表中查找出 IP 地址 192.268.1.2 多对应的 MAC 地址，则可以直接使用。如果没有，则需要发送 ARP 协议查询 MAC 地址，获取 MAC 地址后，由数据链路层构建一个数据帧。再将他们发送出现。
```

![](/Users/caiqingjing/project/private/leetcode-practice/statics/icmp_mac_req.jpg)

```markdown
1. 主机 B 收到这个数据帧后，先检查它的目的 MAC 地址，和本机的 MAC 地址对比，如果符合则接收，否则就丢弃。
2. 接收后检查该数据帧，将 IP 数据包从帧中提取出来，交给本机的 IP 层。IP 层检查后拿有用的信息给到 ICMP 协议。
3. 主机 B 会构建一个 `ICMP 回送响应消息数据包`, 回送响应数据包的类型字段为 0，序号为接收到的请求数据包中的序号，然后再发送出去给主机 A。
4. 在规定的时间内，源主机没有收到 `ICMP 回送响应消息数据包`，说明目标主机不可达。此时源主机会检查，用当前时刻减去该数据包最初从源主机上发出的时刻，就是 ICMP 的数据延时。
```

![](/Users/caiqingjing/project/private/leetcode-practice/statics/icmp_mac_ack.jpg)





























































