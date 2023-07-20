# Linux网络子系统

网络上的数据的发送与接收均由网络设备完成。网络设备的驱动程序管理网络设备，如地址设置、修改传输参数及维护流量和错误统计，同时也驱动网络设备进行数据的收发。

![image-20221208134806261](image/Linux%E7%BD%91%E7%BB%9C%E5%AD%90%E7%B3%BB%E7%BB%9F/image-20221208134806261.png)

Linux内核将网络设备驱动程序划分为4个层次，分别为网络协议接口层、网路设备接口层、提供实际功能的设备驱动功能层、网络设备与媒介层。

![image-20230407160749619](image/Linux%E7%BD%91%E7%BB%9C%E5%AD%90%E7%B3%BB%E7%BB%9F/image-20230407160749619.png)

## 网络协议接口层

网络协议接口层向网络协议栈提供统一的数据包收发接口，不论上层是`ARP`协议还是`IP`协议，发送数据通过`dev_queue_xmit`等接口，接收数据通过`netif_rx`等接口。该层的实现独立于具体的网络协议和网络设备，使网络协议和网络设备之间解耦。

函数位于 [**[\net\core\dev.c]([dev.c « core « net - kernel/git/stable/linux.git - Linux kernel stable tree](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/net/core/dev.c?h=v5.10.158))**]

## 网络设备接口层

网络设备接口层向协议接口层提供统一的用于描述具体网络设备属性和操作的结构体`net_device`，用来描述一个具体的网络设备信息，实现不同硬件的统一。实际上，网络设备接口层从宏观上规划了具体操作硬件的设备驱动功能层的结构。

结构体位于 [[**\include\linux\netdevice.h**]([netdevice.h « linux « include - kernel/git/stable/linux.git - Linux kernel stable tree](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/netdevice.h?h=v5.10.158))]

## 设备驱动功能层

各函数是网络设备接口层 `net_device` 数据结构的具体成员，是驱使网络设备硬件完成相应动作的程序用来负责驱动网络设备硬件来完成各个功能，它通过 `hard_start_xmit()` 函数启动发送操作， 并通过网络设备上的中断触发接收操作。对于具体的设备xxx，工程师应该编写相应的设备驱动功能层的函数，这些函数形如`xxx_open()`、`xxx_stop()`、`xxx_tx()`、`xxx_hard_header()`、`xxx_get_stats()`和`xxx_tx_timeout()`等函数。

由于网络数据包的接收可由中断引发， 设备驱动功能层中的另一个主体是中断处理函数， 它负责读取硬件上接收到的数据包并传送给上层协议， 因此可能包含 `xxx_interrupt()` 和`xxx_rx()` 函数， 前者完成中断类型判断等基本工作， 后者则需完成数据包的生成及将其递交给上层等复杂工作。

## 网络设备与媒介层

网络设备与媒介层直接对应于实际的硬件设备。用来负责完成数据包发送和接收的物理实体, 设备驱动功能层的函数都在这物理上驱动的。



Linux网络子系统



![image-20221208155853528](image/Linux%E7%BD%91%E7%BB%9C%E5%AD%90%E7%B3%BB%E7%BB%9F/image-20221208155853528.png)

Linux内核只涉及OSI模型中的3层，其中L2、L3、L4、分别对应OSI模型中的数据链路层、网络层、传输层，本质上，Linux内核主要任务是完成将接收到的数据包从L2（网络设备驱动）传输给L3（网络层，通常为IPv4/IPv6）。若数据包目的地为当前设备，Linux内核网络栈将传递给L4（传输层，应用到TCP/IP协议侦听套接字），若数据包需要转发，将其交还给L2进行传输，对于本地所生成出战的数据包，从L4依次传递给L3和L2，最后由网络设备驱动进行传输。



中断驱动模式下，若网络数据负载很高的情况下，数据的接收发送效率会下降，故，采用`NAPI`

网络设备驱动主要任务为接收目的地为当前主机的数据包，将他传输给网络层，最后再将其传输给传输层；







```c
dev_queue_xmit()
	->__dev_queue_xmit()
		->dev_hard_start_xmit()
			->xmit_one()
				->netdev_start_xmit()
					->__netdev_start_xmit()
    					->ops->ndo_start_xmit(skb, dev);
```

# Linux网络发包

![img](image/Linux%E7%BD%91%E7%BB%9C%E5%AD%90%E7%B3%BB%E7%BB%9F/0e29ce387f45ddc5cd241eb93e332df7.png)

<center>数据包发送流程</center>

# Linux网络收包

![img](image/Linux%E7%BD%91%E7%BB%9C%E5%AD%90%E7%B3%BB%E7%BB%9F/7ecb2e70a9c2ed546a5c19fd443b54a2-1689834989179-3.png)

<center>数据包接收流程</center>



![image-20230720192605186](image/Linux%E7%BD%91%E7%BB%9C%E5%AD%90%E7%B3%BB%E7%BB%9F/image-20230720192605186.png)







# 参考

[Linux 网络子系统 - 陈富林 - 博客园 (cnblogs.com)](https://www.cnblogs.com/chenfulin5/p/6073755.html)

[[转载\]Linux TCP/IP 协议栈源码分析 - Better-zyy - 博客园 (cnblogs.com)](https://www.cnblogs.com/better-zyy/archive/2012/03/16/2400811.html)

[Linux 网络栈接收数据（RX）：原理及内核实现（2022） (arthurchiao.art)](http://arthurchiao.art/blog/linux-net-stack-implementation-rx-zh/)

[Linux 网络设备驱动开发（一） —— linux内核网络分层结构-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2154372)
