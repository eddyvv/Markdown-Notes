# NAPI（轮询或中断）

中断的好处是响应快，数据量小的时候处理及时，速度快，但是当数据量增大时，短帧的时候会导致终端频繁发生，消耗大量的CPU处理时间在中断自身处理上。轮询恰好相反，响应没有中断及时，但是在处理大量数据的时候不需要消耗过多的 CPU 处理时间。

`NAPI` 的核心思想就是不全部采用中断来读取网络数据，而是采用中断来唤醒数据接收服务程序，在接收服务程序中采用 `POLL` 的方法来轮询处理数据。

## NAPI接收网络包流程概述

传统的网络收包流程完全靠中断驱动，当网络包到达十分频繁时，就会频繁触发中断，进而影响系统的整体性能；

`NAPI` 方式的核心就是当有数据包到达时，集中处理网络包，之后再去处理其他事情；

`NAPI` 的处理流程是，当一些网络包到达触发中断时，内核处理完这些网络包之后，主动轮询 `poll` 网卡，主动去接收到来的网络包。如果一直有，就一直处理，等处理告一段落再返回，当再有下一批网络包到达时，再中断，再轮询 `poll`。这样就会大大减少中断的数量，提升网络处理的效率。

在 `virtio-net` 前端驱动中，在 `probe` 过程中，会调用 `netif_napi_add_weight` 函数注册收包poll函数；

```c
static int virtnet_alloc_queues(struct virtnet_info *vi)
{
	int i;

	if (vi->has_cvq) {
		vi->ctrl = kzalloc(sizeof(*vi->ctrl), GFP_KERNEL);
		if (!vi->ctrl)
			goto err_ctrl;
	} else {
		vi->ctrl = NULL;
	}
	vi->sq = kcalloc(vi->max_queue_pairs, sizeof(*vi->sq), GFP_KERNEL);
	if (!vi->sq)
		goto err_sq;
	vi->rq = kcalloc(vi->max_queue_pairs, sizeof(*vi->rq), GFP_KERNEL);
	if (!vi->rq)
		goto err_rq;

	INIT_DELAYED_WORK(&vi->refill, refill_work);
	for (i = 0; i < vi->max_queue_pairs; i++) {
		vi->rq[i].pages = NULL;
        /* 注册NAPI收包poll函数 */
		netif_napi_add_weight(vi->dev, &vi->rq[i].napi, virtnet_poll,
				      napi_weight);
		netif_napi_add_tx_weight(vi->dev, &vi->sq[i].napi,
					 virtnet_poll_tx,
					 napi_tx ? napi_weight : 0);

		sg_init_table(vi->rq[i].sg, ARRAY_SIZE(vi->rq[i].sg));
		ewma_pkt_len_init(&vi->rq[i].mrg_avg_pkt_len);
		sg_init_table(vi->sq[i].sg, ARRAY_SIZE(vi->sq[i].sg));

		u64_stats_init(&vi->rq[i].stats.syncp);
		u64_stats_init(&vi->sq[i].stats.syncp);
	}

	return 0;

err_rq:
	kfree(vi->sq);
err_sq:
	kfree(vi->ctrl);
err_ctrl:
	return -ENOMEM;
}
```



## 相关函数

### napi_schedule

`napi_schedule` 是 Linux 内核网络子系统中的一个函数，用于调度一个 NAPI 处理周期以在下一个可用的处理器时间中运行。NAPI 是 Linux 内核网络子系统提供的一种高效的网络数据包处理机制，它可以有效地减少网络数据包处理过程中的 CPU 开销，提高系统的网络性能。

`include/linux/netdevice.h`

```c
static inline void napi_schedule(struct napi_struct *n)
```

在网络设备驱动程序中，通常会在中断处理函数中调用 `napi_schedule` 函数，以调度一个 NAPI 处理周期以处理网络数据包。

### netif_napi_add

`netif_napi_add`用于为一个网络设备注册一个 NAPI 处理程序。

`net/core/dev.c`

```c
void netif_napi_add(struct net_device *dev, struct napi_struct *napi,
		    int (*poll)(struct napi_struct *, int), int weight)
```

### napi_complete

`napi_complete`用于通知内核网络协议栈当前的 NAPI 处理状态。NAPI 处理程序通常在网络设备驱动程序中实现，它使用 `napi_schedule` 函数调度一个 NAPI 处理周期以处理网络数据包。在 NAPI 处理周期中，NAPI 处理程序会从网络设备接收网络数据包，并对其进行处理，直到达到预设的处理预算或者没有更多的网络数据包可处理。当 NAPI 处理程序完成处理网络数据包后，需要使用 `napi_complete` 函数通知内核网络协议栈当前的 NAPI 处理状态，以便内核网络协议栈可以继续处理网络数据包。



## NAPI接收数据一般流程

1. 在`xxx_probe`函数中使用`netif_napi_add`注册NAPI处理函数;
2. 接收中断来临，调用接收中断函数，中断函数中调用`napi_schedule`在下一个可用的处理器时间中运行NAPI处理接收数据；
3. NAPI处理函数被执行，接收并传递给上层协议栈进行处理。



# 参考

[virtio-net 实现机制【二】（图文并茂） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/545258186)