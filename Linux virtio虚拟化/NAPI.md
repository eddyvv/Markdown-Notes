# NAPI（New API）

Linux 网络子系统中的 NAPI（New API）机制是一种用于处理网络中断的高效技术。它可以减少网络中断的数量，提高网络性能。

在传统的网络中断处理机制中，当网络设备接收到数据包时，它会触发一个中断，并通知内核处理该数据包。如果网络数据包的数量很大，那么这些中断将会成为系统的瓶颈。为了减少中断的数量，内核使用 NAPI 技术来将网络数据包缓存起来，等待一段时间后再一次性处理。这样就可以减少中断的数量，提高系统的性能。

`NAPI` 的核心思想就是不全部采用中断来读取网络数据，而是采用中断来唤醒数据接收服务程序，在接收服务程序中采用 `POLL` 的方法来轮询处理数据。

## NAPI接收网络包流程概述

传统的网络收包流程完全靠中断驱动，当网络包到达十分频繁时，就会频繁触发中断，进而影响系统的整体性能；

`NAPI` 方式的核心就是当有数据包到达时，集中处理网络包，之后再去处理其他事情；

`NAPI` 的处理流程是，当一些网络包到达触发中断时，内核处理完这些网络包之后，主动轮询 `poll` 网卡，主动去接收到来的网络包。如果一直有，就一直处理，等处理告一段落再返回，当再有下一批网络包到达时，再中断，再轮询 `poll`。这样就会大大减少中断的数量，提升网络处理的效率。

1. 在`xxx_probe`函数中使用`netif_napi_add`注册NAPI处理函数;
2. 接收中断来临，调用接收中断函数，中断函数中调用`napi_schedule`在下一个可用的处理器时间中运行NAPI处理接收数据；
3. NAPI处理函数被执行，接收并传递给上层协议栈进行处理。

```
IRQ中断
	-> napi_schedule
		->进入软中断
			->net_rx_action
				->napi_poll
					->驱动中注册的poll函数
```



## 相关函数

### napi_schedule - 网卡硬件中断用来触发软中断

`napi_schedule` 是 Linux 内核网络子系统中的一个函数，用于调度一个 NAPI 处理周期以在下一个可用的处理器时间中运行。NAPI 是 Linux 内核网络子系统提供的一种高效的网络数据包处理机制，它可以有效地减少网络数据包处理过程中的 CPU 开销，提高系统的网络性能。

`include/linux/netdevice.h`

```c
static inline void napi_schedule(struct napi_struct *n)
```

在网络设备驱动程序中，通常会在中断处理函数中调用 `napi_schedule` 函数，以调度一个 NAPI 处理周期以处理网络数据包。



### netif_napi_add- 驱动初始时向内核注册软软中断处理回调`poll`函数

`netif_napi_add`用于为一个网络设备注册一个 NAPI 处理程序。

`net/core/dev.c`

```c
void netif_napi_add(struct net_device *dev, struct napi_struct *napi,
		    int (*poll)(struct napi_struct *, int), int weight)
```

### napi_complete-数据包处理完成通知内核处理网络中断

`napi_complete`用于通知内核网络协议栈当前的 NAPI 处理状态。NAPI 处理程序通常在网络设备驱动程序中实现，它使用 `napi_schedule` 函数调度一个 NAPI 处理周期以处理网络数据包。在 NAPI 处理周期中，NAPI 处理程序会从网络设备接收网络数据包，并对其进行处理，直到达到预设的处理预算或者没有更多的网络数据包可处理。当 NAPI 处理程序完成处理网络数据包后，需要使用 `napi_complete` 函数通知内核网络协议栈当前的 NAPI 处理状态，以便内核网络协议栈可以继续处理网络数据包。





# 参考

[virtio-net 实现机制【二】（图文并茂） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/545258186)

[Linux网络协议栈：NAPI机制与处理流程分析（图解）_netif_napi_add_rtoax的博客-CSDN博客](https://blog.csdn.net/Rong_Toa/article/details/109401935)

[浅析 Linux 如何接收网络帧 | Shall We Code? (waynerv.com)](https://waynerv.com/posts/how-linux-process-input-frames/)