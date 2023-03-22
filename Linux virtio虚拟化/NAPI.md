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





# 参考

[virtio-net 实现机制【二】（图文并茂） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/545258186)