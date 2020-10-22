# README.md

------------------------------

### 0x00 前言
`PortForward` 是使用 Golang 进行开发的端口转发工具，解决在某些场景下 `内外网无法互通`的问题。

`PortForward` 的功能如下：

	1. 支持 tcp/udp 协议层的端口转发
	2. 支持级联
	3. 支持正向/反向连接模式
	4. 支持多通道
	5. 支持 ipv6

本文对 `PortForward` 进行了详细的介绍。

目录：

1. 使用说明
2. 工作场景
3. 源码结构
4. 逻辑结构
5. 端口转发的实现
6. udp的knock报文
7. udp的超时设置
8. listen-listen的超时设置
9. 多通路的实现
10. issue
11. contributions

### 0x01 使用说明
**1.使用**  

	Usage:
	  ./portforward [proto] [sock1] [sock2]
	Option:
	  proto      the port forward with protocol(tcp/udp)
	  sock       format: [method:address:port]
	  method     the sock mode(listen/conn)
	Example:
	  tcp conn:192.168.1.1:3389 conn:192.168.1.10:23333
	  udp listen:192.168.1.3:5353 conn:8.8.8.8:53
	  tcp listen:[fe80::1%lo0]:8888 conn:[fe80::1%lo0]:7777

	version: 0.5.0(build-20201022)

**2.编译**  

	Golang 1.12及以上
	GO111MODULE=on
	
	git clone https://github.com/knownsec/Portforward.git
	./build.sh


### 0x02 工作场景
这里列举了一些 `PortForward` 的工作场景，如下：

**2.1 简单模式**  
<div align="center">
<img src="./Images/simple_forward.png" width="500">
</br>[图1.简单转发模式]
</div>

**2.2 受限主机转发**  
<div align="center">
<img src="./Images/restricted_forward.png" width="500">
</br>[图2.受限主机转发模式图]
</div> 

**2.3 级联端口转发**  
<div align="center">
<img src="./Images/mutil_forward.png" width="500">
</br>[图3.级联端口转发]
</div> 


### 0x03 源码结构

	.
	├── CHANGELOG
	├── Images        // images resource
	├── README.md
	├── build.sh      // compile script
	├── forward.go    // portforward main logic
	├── go.mod
	├── log.go        // log module
	├── main.go       // main, parse arguments
	├── tcp.go        // tcp layer
	└── udp.go        // udp layer


### 0x04 逻辑结构
`PortForward` 支持 `TCP` , `UDP` 协议层的端口转发，代码抽象后逻辑结构框架如下：
<div align="center">
<img src="./Images/portforward_framework.png" width="500">
</br>[图4.整体框架]
</div>


### 0x05 端口转发的实现
端口转发程序作为网络传输的中间人，无非就是将两端的 socket 对象进行联通，数据就可以通过这条「链路」进行传输了。

按照这样的思路，我们从需求开始分析和抽象，可以这么认为：无论是作为 `tcp` 还是 `udp` 运行，无论是作为 `connect` 还是 `listen` 运行，最终都将获得两个 socket，其中一个连向原服务，另一个与客户端连接；最终将这两端的 socket 联通即可实现端口转发。

在 Golang 中我们采用了 `io.Copy()` 来联通两个 socket，但是 `io.Copy` 必须要求对象实现了 `io.Reader/io.Writer` 接口，`tcp` 的 socket 可以直接支持，而 `udp` 的 socket 需要我们进行封装。


### 0x06 udp的knock报文
在 `udp` 的 `connect` 模式下，我们在连接服务器成功后，立即发送了一个 `knock` 报文，如下：

	conn, err := net.DialTimeout("udp", ...
	_, err = conn.Write([]byte("\x00"))

其作用是通知远程 `udp` 服务器我们已经连上了(`udp` 创建连接后，仅在本地操作系统层进行了注册，只有当发送一个报文到对端后，远程服务器才能感知到新连接)，当我们在 `udp` 的 `conn-conn` 模式下运行时，这个报文是必须的。


### 0x07 udp的超时设置
在 `udp` 的实现中，我们为所有的 `udp` 连接 socket 对象都设置了超时时间(`tcp` 中不需要)，这是因为在 `udp` 中，socket 对象无法感知对端退出，如果不设置超时时间，将会一直在 `conn.Read()` 阻塞下去。

我们设置了 `udp` 超时时间为 60 秒，当 60 秒无数据传输，本次建立的虚拟通信链路将销毁，端口转发程序将重新创建新的通信链路。


### 0x08 listen-listen的超时设置
对于 `listen-listen` 模式，需要等待两端的客户端都连上端口转发程序后，才能将两个 socket 进行联通。

为此我们在此处设置了 120 秒的超时时间，也就是说当其中一端有客户端连接后，另一端在 120 秒内没有连接，我们就销毁这个未成功建立的通信链路；用户重新连接即可。

>如果没有这个超时，可能某些场景遗留了某个连接，将造成后续的通信链路错位。


### 0x09 多通路的实现
多通路可以支持同时发起多个连接，这里我们以 `tcp` 作为例子来说明。为了处理这种情况，我们的处理方式是：

1. `listen-conn`: 每当 listen 服务器接收到新连接后，与远端创建新的连接，并将两个 socket 进行联通。
2. `listen-listen`: (好像没有实际场景)两端的 listen 服务器接收到新连接后，将两个 socket 进行联通。
3. `conn-conn`: 创建 sock1 的连接，当 sock1 端收到数据，创建与 sock2 的连接，将两个 socket 进行联通；随后继续创建 sock1 连接(预留)。

>我们在 `udp` 中也加入了多通路的支持，和 `tcp` 基本类似，但由于 `udp` 是无连接的，我们不能像 `tcp` 直接联通两个 socket 对象。我们在 `udp listen` 服务器中维护了一个临时表，使用 `ip:port` 作为标志，以描述各个通信链路的联通情况，依据此进行流量的分发。


### 0x0A issue
**1.udp的映射表未清空**  
udp多通路中的映射表没有对无效数据进行清理，长时间运行可能造成内存占用

**2.http服务的 host 字段影响**  
当转发 http 服务时，由于常见的客户端在访问时自动将 `host` 设置为访问目标(端口转发程序的地址)，我们直接对流量进行转发，那么 `host` 字段一定是错误的，某些 http 服务器对该字段进行了校验，所以无法正常转发。

>比如端口转发 `www.baidu.com`

**3.不支持多端口通信**  
这里当然无法处理多端口通信的服务，我们仅做了一个端口的转发，多个端口的情况无法直接处理，比如 `FTP` 服务，其分别使用 20 端口(数据传输)和 21 端口(命令传输)。

但还是要单独说一句，对于大多数 `udp` 服务的实现，是 `udp` 服务器收到了一个新报文，对此进行处理后，然后使用新的 socket 将响应数据发给对端，比如 `TFTP` 服务：

	1.client 发起请求 [get aaa]
	  src: 55123 => dst: 69
	2.tftp 收到报文，进行处理，返回响应：
	  src: 61234 => dst: 55123

这种多端口通信我们也是无法处理的，因为在目前多通路的实现下，上述流程中第 2 步过后，对于端口转发程序来说，后续的报文无法确定转发给 69 端口还是 61234 端口。


### 0x0B contributions

[r0oike@knownsec 404](https://github.com/r0oike)  
[fenix@knownsec 404](https://github.com/13ph03nix)  
[0x7F@knownsec 404](https://github.com/0x7Fancy)  

</br>

------------------------------
References:  
内网渗透之端口转发、映射、代理: <https://xz.aliyun.com/t/6349>  
内网渗透之端口转发与代理工具总结: <https://www.freebuf.com/articles/web/170970.html>  
sensepost/reGeorg: <https://github.com/sensepost/reGeorg>  
idlefire/ew: <https://github.com/idlefire/ew>  

knownsec404  
2020.10.22  