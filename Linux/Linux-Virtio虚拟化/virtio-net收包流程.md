# virtio-net收包流程


## 后端

```c
virtio_net_receive
    -->virtio_net_do_receive
    	-->virtio_net_receive_rcu
    		-->virtqueue_pop
    			-->virtqueue_split_pop
    				-->vring_split_desc_read	/* 获取可用的desc */
    				-->virtqueue_map_desc	/* 将获取的desc的虚拟机物理地址转换为qemu进程虚拟地址 */
    				-->virtqueue_alloc_element	/* 将获取的信息组成elem */
    		-->iov_from_buf	/* 将buf拷贝到elem的iovec中 */
    		-->virtqueue_fill	/* 将uelem写入到vq->vring.used */
    		-->virtio_notify	/* 中断注入，通知VM有报文待接收 */
```

### virtio_net_receive

`virtio_net_receive` 函数把数据拷贝到虚拟机的 virtio 网卡接收队列，然后向虚拟机注入一个中断，虚拟机便感知到有网络数据报文。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static ssize_t virtio_net_receive(NetClientState *nc, const uint8_t *buf,
                                  size_t size)
{
    VirtIONet *n = qemu_get_nic_opaque(nc);
    /* 是否使用rsc协议接收数据包 */
    if ((n->rsc4_enabled || n->rsc6_enabled)) {
        return virtio_net_rsc_receive(nc, buf, size);
    } else {
        return virtio_net_do_receive(nc, buf, size);
    }
}
```

#### virtio_net_do_receive

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static ssize_t virtio_net_do_receive(NetClientState *nc, const uint8_t *buf,
                                  size_t size)
{
    RCU_READ_LOCK_GUARD();

    return virtio_net_receive_rcu(nc, buf, size, false);
}

```

##### virtio_net_receive_rcu

`virtio_net_receive_rcu` 函数用于将从网络中接收的数据填充到接收队列的输入缓冲区中。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static ssize_t virtio_net_receive_rcu(NetClientState *nc, const uint8_t *buf,
                                      size_t size, bool no_rss)
{
    VirtIONet *n = qemu_get_nic_opaque(nc);
    VirtIONetQueue *q = virtio_net_get_subqueue(nc);
    VirtIODevice *vdev = VIRTIO_DEVICE(n);
    VirtQueueElement *elems[VIRTQUEUE_MAX_SIZE];
    size_t lens[VIRTQUEUE_MAX_SIZE];
    struct iovec mhdr_sg[VIRTQUEUE_MAX_SIZE];
    struct virtio_net_hdr_mrg_rxbuf mhdr;
    unsigned mhdr_cnt = 0;
    size_t offset, i, guest_offset, j;
    ssize_t err;

    /* 根据 vm running 状态，queue 和设备状态判断 virtio-net 是否可以收包 */
    if (!virtio_net_can_receive(nc)) {
        return -1;
    }

    if (!no_rss && n->rss_data.enabled && n->rss_data.enabled_software_rss) {
        int index = virtio_net_process_rss(nc, buf, size);
        if (index >= 0) {
            NetClientState *nc2 = qemu_get_subqueue(n->nic, index);
            return virtio_net_receive_rcu(nc2, buf, size, true);
        }
    }

    /* hdr_len refers to the header we supply to the guest */
    if (!virtio_net_has_buffers(q, size + n->guest_hdr_len - n->host_hdr_len)) {
        return 0;
    }

    /* 对网络数据包进行过滤 */
    if (!receive_filter(n, buf, size))
        return size;

    offset = i = 0;

    while (offset < size) {
        VirtQueueElement *elem;
        int len, total;
        const struct iovec *sg;

        total = 0;

        if (i == VIRTQUEUE_MAX_SIZE) {
            virtio_error(vdev, "virtio-net unexpected long buffer chain");
            err = size;
            goto err;
        }

        /* 从vring中取出一个请求，将信息传递给elem域中 */
        elem = virtqueue_pop(q->rx_vq, sizeof(VirtQueueElement));
        if (!elem) {
            if (i) {
                virtio_error(vdev, "virtio-net unexpected empty queue: "
                             "i %zd mergeable %d offset %zd, size %zd, "
                             "guest hdr len %zd, host hdr len %zd "
                             "guest features 0x%" PRIx64,
                             i, n->mergeable_rx_bufs, offset, size,
                             n->guest_hdr_len, n->host_hdr_len,
                             vdev->guest_features);
            }
            err = -1;
            goto err;
        }

        if (elem->in_num < 1) {
            virtio_error(vdev,
                         "virtio-net receive queue contains no in buffers");
            virtqueue_detach_element(q->rx_vq, elem, 0);
            g_free(elem);
            err = -1;
            goto err;
        }

        sg = elem->in_sg;
        if (i == 0) {
            assert(offset == 0);
            if (n->mergeable_rx_bufs) {
                mhdr_cnt = iov_copy(mhdr_sg, ARRAY_SIZE(mhdr_sg),
                                    sg, elem->in_num,
                                    offsetof(typeof(mhdr), num_buffers),
                                    sizeof(mhdr.num_buffers));
            }

            receive_header(n, sg, elem->in_num, buf, size);
            if (n->rss_data.populate_hash) {
                offset = sizeof(mhdr);
                iov_from_buf(sg, elem->in_num, offset,
                             buf + offset, n->host_hdr_len - sizeof(mhdr));
            }
            offset = n->host_hdr_len;
            total += n->guest_hdr_len;
            guest_offset = n->guest_hdr_len;
        } else {
            guest_offset = 0;
        }

        /* copy in packet.  ugh */
        len = iov_from_buf(sg, elem->in_num, guest_offset,
                           buf + offset, size - offset);
        total += len;
        offset += len;
        /* If buffers can't be merged, at this point we
         * must have consumed the complete packet.
         * Otherwise, drop it. */
        if (!n->mergeable_rx_bufs && offset < size) {
            virtqueue_unpop(q->rx_vq, elem, total);
            g_free(elem);
            err = size;
            goto err;
        }

        elems[i] = elem;
        lens[i] = total;
        i++;
    }

    if (mhdr_cnt) {
        virtio_stw_p(vdev, &mhdr.num_buffers, i);
        iov_from_buf(mhdr_sg, mhdr_cnt,
                     0,
                     &mhdr.num_buffers, sizeof mhdr.num_buffers);
    }

    for (j = 0; j < i; j++) {
        /* signal other side */
        /* 数据写完后，撤销映射，更新 VRingAvail.ring[] 的相关字段 */
        virtqueue_fill(q->rx_vq, elems[j], lens[j], j);
        g_free(elems[j]);
    }

    /* 更新 VRingUsed.ring[] 的 idx，表明可以回收 */
    virtqueue_flush(q->rx_vq, i);
    /* 注入中断，通知前端虚拟机 */
    virtio_notify(vdev, q->rx_vq);

    return size;

err:
    for (j = 0; j < i; j++) {
        virtqueue_detach_element(q->rx_vq, elems[j], lens[j]);
        g_free(elems[j]);
    }

    return err;
}
```

###### virtqueue_pop

`virtqueue_pop` 函数用于从虚拟队列 `vq` 中弹出一个元素。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
void *virtqueue_pop(VirtQueue *vq, size_t sz)
{
    if (virtio_device_disabled(vq->vdev)) {
        return NULL;
    }

    /* 判断虚拟队列采用的结构是分离式结构（split）还是压缩式结构（packed）实现 */
    if (virtio_vdev_has_feature(vq->vdev, VIRTIO_F_RING_PACKED)) {
        return virtqueue_packed_pop(vq, sz);
    } else {
        return virtqueue_split_pop(vq, sz);
    }
}
```

###### iov_from_buf

`iov_from_buf ` 函数负责将报文拷贝到 buffer 中，实现向 Guest OS 物理地址写入数据

`源码位置`[/include/qemu/iov.h](https://github1s.com/qemu/qemu/blob/HEAD/include/qemu/iov.h)

```c
static inline size_t
iov_from_buf(const struct iovec *iov, unsigned int iov_cnt,
             size_t offset, const void *buf, size_t bytes)
{
    if (__builtin_constant_p(bytes) && iov_cnt &&
        offset <= iov[0].iov_len && bytes <= iov[0].iov_len - offset) {
        memcpy(iov[0].iov_base + offset, buf, bytes);
        return bytes;
    } else {
        return iov_from_buf_full(iov, iov_cnt, offset, buf, bytes);
    }
}
```

###### virtqueue_fill

`virtqueue_fill` 函数用于填充虚拟队列的元素。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
/* Called within rcu_read_lock().  */
void virtqueue_fill(VirtQueue *vq, const VirtQueueElement *elem,
                    unsigned int len, unsigned int idx)
{
    trace_virtqueue_fill(vq, elem, len, idx);

    virtqueue_unmap_sg(vq, elem, len);

    if (virtio_device_disabled(vq->vdev)) {
        return;
    }

    if (virtio_vdev_has_feature(vq->vdev, VIRTIO_F_RING_PACKED)) {
        virtqueue_packed_fill(vq, elem, len, idx);
    } else {
        virtqueue_split_fill(vq, elem, len, idx);
    }
}
```

###### virtio_notify

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
void virtio_notify(VirtIODevice *vdev, VirtQueue *vq)
{
    WITH_RCU_READ_LOCK_GUARD() {
        if (!virtio_should_notify(vdev, vq)) {
            return;
        }
    }

    trace_virtio_notify(vdev, vq);
    virtio_irq(vq);
}
```



## 前端

```c
vring_interrupt
    -->vq->vq.callback(&vq->vq)		/* 在virtnet_probe->init_vqs->virtnet_find_vqs->中被注册为skb_recv_done */
/* 接收数据中断回调函数 */
-->skb_recv_done
    -->virtqueue_napi_schedule
    	-->__napi_schedule
    		-->____napi_schedule
    			-->list_add_tail	/* 把 NAPI 加入到本地cpu的 softnet_data 的 poll_list链表头 */
    				-->__raise_softirq_irqoff	/* 调用收包软中断 */
/* 执行收包软中断函数 */
    virtnet_poll
    	-->virtnet_receive
    		-->receive_buf	/* 接收到的数据转换成skb */
    			/* 根据接收类型XDP_PASS、XDP_TX等对 virtqueue 中的数据进行不同的处理 */
    			-->receive_mergeable(dev, vi, rq, buf, ctx, len, xdp_xmit,stats);or
				-->receive_big(dev, vi, rq, buf, len, stats) or 
                -->receive_small(dev, vi, rq, buf, ctx, len, xdp_xmit, stats)
                -->napi_gro_receive	/* 把skb上传到上层协议栈 */
    				...
    				-->netif_receive_skb
    					-->netif_receive_skb_internal
    						-->__netif_receive_skb	/* 传输skb给网络层，交由协议栈处理 */
    		-->schedule_delayed_work	/* 延时x节拍数，接收数据 */
    			-->refill_work
    				-->try_fill_recv
    	-->virtqueue_napi_complete
    	
```

### vring_interrupt

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
/**
 * vring_interrupt - notify a virtqueue on an interrupt
 * @irq: the IRQ number (ignored)
 * @_vq: the struct virtqueue to notify
 *
 * Calls the callback function of @_vq to process the virtqueue
 * notification.
 */
irqreturn_t vring_interrupt(int irq, void *_vq)
{
	struct vring_virtqueue *vq = to_vvq(_vq);

	if (!more_used(vq)) {
		pr_debug("virtqueue interrupt with no work for %p\n", vq);
		return IRQ_NONE;
	}

	if (unlikely(vq->broken)) {
#ifdef CONFIG_VIRTIO_HARDEN_NOTIFICATION
		dev_warn_once(&vq->vq.vdev->dev,
			      "virtio vring IRQ raised before DRIVER_OK");
		return IRQ_NONE;
#else
		return IRQ_HANDLED;
#endif
	}

	/* Just a hint for performance: so it's ok that this can be racy! */
	if (vq->event)
		vq->event_triggered = true;

	pr_debug("virtqueue callback for %p (%p)\n", vq, vq->vq.callback);
    /* 回调函数，一般在 xxx_probe 函数中注册 */
	if (vq->vq.callback)
		vq->vq.callback(&vq->vq);

	return IRQ_HANDLED;
}
EXPORT_SYMBOL_GPL(vring_interrupt);
```

#### skb_recv_done

接收数据包回调函数。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void skb_recv_done(struct virtqueue *rvq)
{
	struct virtnet_info *vi = rvq->vdev->priv;
	struct receive_queue *rq = &vi->rq[vq2rxq(rvq)];

	virtqueue_napi_schedule(&rq->napi, rvq);
}
```

##### virtqueue_napi_schedule

函数 `virtqueue_napi_schedule` 用于将 NAPI 设备的 napi 结构体加入到系统中轮询执行的链表中。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtqueue_napi_schedule(struct napi_struct *napi,
				    struct virtqueue *vq)
{
	if (napi_schedule_prep(napi)) {
		virtqueue_disable_cb(vq);
		__napi_schedule(napi);
	}
}
```

## virtnet_poll

`virtnet_poll` 函数在 `NAPI（New API）`机制下执行。NAPI 是一种用于网络设备驱动的软件架构，其目的是减少网络中断对系统性能的影响。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static int virtnet_poll(struct napi_struct *napi, int budget)
{
	struct receive_queue *rq =
		container_of(napi, struct receive_queue, napi);
	struct virtnet_info *vi = rq->vq->vdev->priv;
	struct send_queue *sq;
	unsigned int received;
	unsigned int xdp_xmit = 0;

    /* 清空接收队列的缓冲区 */
	virtnet_poll_cleantx(rq);
	
    /* 接收数据包，转发至网络层 */
	received = virtnet_receive(rq, budget, &xdp_xmit);

	/* Out of packets? */
	if (received < budget)	/* 接收的报文小于 budget，说明报文已经接收完成 */
        /*  */
		virtqueue_napi_complete(napi, rq->vq, received);

	if (xdp_xmit & VIRTIO_XDP_REDIR)
		xdp_do_flush();

	if (xdp_xmit & VIRTIO_XDP_TX) {
		sq = virtnet_xdp_get_sq(vi);
		if (virtqueue_kick_prepare(sq->vq) && virtqueue_notify(sq->vq)) {
			u64_stats_update_begin(&sq->stats.syncp);
			sq->stats.kicks++;
			u64_stats_update_end(&sq->stats.syncp);
		}
		virtnet_xdp_put_sq(vi, sq);
	}

	return received;
}
```

### virtnet_receive

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static int virtnet_receive(struct receive_queue *rq, int budget,
			   unsigned int *xdp_xmit)
{
	struct virtnet_info *vi = rq->vq->vdev->priv;
	struct virtnet_rq_stats stats = {};
	unsigned int len;
	void *buf;
	int i;

	if (!vi->big_packets || vi->mergeable_rx_bufs) {
		void *ctx;

		while (stats.packets < budget &&
               /* 从rx queue的used ring里取一个描述符，并返回其关联的收到的skb报文。 */
		       (buf = virtqueue_get_buf_ctx(rq->vq, &len, &ctx))) {
            /* 根据描述符信息填充skb，并将skb投递至TCP/IP协议栈。 */
			receive_buf(vi, rq, buf, len, ctx, xdp_xmit, &stats);
			stats.packets++;
		}
	} else {
		while (stats.packets < budget &&
		       (buf = virtqueue_get_buf(rq->vq, &len)) != NULL) {
			receive_buf(vi, rq, buf, len, NULL, xdp_xmit, &stats);
			stats.packets++;
		}
	}

	if (rq->vq->num_free > min((unsigned int)budget, virtqueue_get_vring_size(rq->vq)) / 2) {
        /* 往 rx buffer 的 avail ring 里填充用于接收新报文的 empty buffer */
		if (!try_fill_recv(vi, rq, GFP_ATOMIC)) {
			spin_lock(&vi->refill_lock);
			if (vi->refill_enabled)
				schedule_delayed_work(&vi->refill, 0);
			spin_unlock(&vi->refill_lock);
		}
	}

	u64_stats_update_begin(&rq->stats.syncp);
	for (i = 0; i < VIRTNET_RQ_STATS_LEN; i++) {
		size_t offset = virtnet_rq_stats_desc[i].offset;
		u64 *item;

		item = (u64 *)((u8 *)&rq->stats + offset);
		*item += *(u64 *)((u8 *)&stats + offset);
	}
	u64_stats_update_end(&rq->stats.syncp);

	return stats.packets;
}
```

#### receive_buf

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void receive_buf(struct virtnet_info *vi, struct receive_queue *rq,
			void *buf, unsigned int len, void **ctx,
			unsigned int *xdp_xmit,
			struct virtnet_rq_stats *stats)
{
	struct net_device *dev = vi->dev;
	struct sk_buff *skb;
	struct virtio_net_hdr_mrg_rxbuf *hdr;

	if (unlikely(len < vi->hdr_len + ETH_HLEN)) {
		pr_debug("%s: short packet %i\n", dev->name, len);
		dev->stats.rx_length_errors++;
		if (vi->mergeable_rx_bufs) {
			put_page(virt_to_head_page(buf));
		} else if (vi->big_packets) {
			give_pages(rq, buf);
		} else {
			put_page(virt_to_head_page(buf));
		}
		return;
	}

	if (vi->mergeable_rx_bufs)
		skb = receive_mergeable(dev, vi, rq, buf, ctx, len, xdp_xmit,
					stats);
	else if (vi->big_packets)
		skb = receive_big(dev, vi, rq, buf, len, stats);
	else
		skb = receive_small(dev, vi, rq, buf, ctx, len, xdp_xmit, stats);

	if (unlikely(!skb))
		return;

	hdr = skb_vnet_hdr(skb);
	if (dev->features & NETIF_F_RXHASH && vi->has_rss_hash_report)
		virtio_skb_set_hash((const struct virtio_net_hdr_v1_hash *)hdr, skb);

	if (hdr->hdr.flags & VIRTIO_NET_HDR_F_DATA_VALID)
		skb->ip_summed = CHECKSUM_UNNECESSARY;

	if (virtio_net_hdr_to_skb(skb, &hdr->hdr,
				  virtio_is_little_endian(vi->vdev))) {
		net_warn_ratelimited("%s: bad gso: type: %u, size: %u\n",
				     dev->name, hdr->hdr.gso_type,
				     hdr->hdr.gso_size);
		goto frame_err;
	}

	skb_record_rx_queue(skb, vq2rxq(rq->vq));
	skb->protocol = eth_type_trans(skb, dev);
	pr_debug("Receiving skb proto 0x%04x len %i type %i\n",
		 ntohs(skb->protocol), skb->len, skb->pkt_type);

    /* 将报文送至网络协议栈。 */
	napi_gro_receive(&rq->napi, skb);
	return;

frame_err:
	dev->stats.rx_frame_errors++;
	dev_kfree_skb(skb);
}
```

##### napi_gro_receive

`napi_gro_receive()` 对 `skbuf` 进行合并处理，根据处理的结果，决定 `skbuf` 是否继续送给上次协议栈，或是直接丢弃。

`源码位置`[/net/core/gro.c](https://github1s.com/torvalds/linux/blob/HEAD/net/core/gro.c)

```c
gro_result_t napi_gro_receive(struct napi_struct *napi, struct sk_buff *skb)
{
	gro_result_t ret;

	skb_mark_napi_id(skb, napi);
	trace_napi_gro_receive_entry(skb);

	skb_gro_reset_offset(skb, 0);

	ret = napi_skb_finish(napi, skb, dev_gro_receive(napi, skb));
	trace_napi_gro_receive_exit(ret);

	return ret;
}
EXPORT_SYMBOL(napi_gro_receive);
```

#### try_fill_recv

`try_fill_recv` 函数会将所有可用的描述符均加入 `receive_queue` 的 `avail ring` ，供后端驱动使用。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
/*
 * Returns false if we couldn't fill entirely (OOM).
 *
 * Normally run in the receive path, but can also be run from ndo_open
 * before we're receiving packets, or from refill_work which is
 * careful to disable receiving (using napi_disable).
 */
static bool try_fill_recv(struct virtnet_info *vi, struct receive_queue *rq,
			  gfp_t gfp)
{
	int err;
	bool oom;

	do {
		if (vi->mergeable_rx_bufs)
			err = add_recvbuf_mergeable(vi, rq, gfp);
		else if (vi->big_packets)
			err = add_recvbuf_big(vi, rq, gfp);
		else
			err = add_recvbuf_small(vi, rq, gfp);

		oom = err == -ENOMEM;
		if (err)
			break;
	} while (rq->vq->num_free);
	if (virtqueue_kick_prepare(rq->vq) && virtqueue_notify(rq->vq)) {
		unsigned long flags;

		flags = u64_stats_update_begin_irqsave(&rq->stats.syncp);
		rq->stats.kicks++;
		u64_stats_update_end_irqrestore(&rq->stats.syncp, flags);
	}

	return !oom;
}
```



### virtqueue_napi_complete

`源码位置`[/net/core/gro.c](https://github1s.com/torvalds/linux/blob/HEAD/net/core/gro.c)

```c
static void virtqueue_napi_complete(struct napi_struct *napi,
				    struct virtqueue *vq, int processed)
{
	int opaque;

	opaque = virtqueue_enable_cb_prepare(vq);
    /* 判断是否处理了所有报文 */
	if (napi_complete_done(napi, processed)) {
        /* 检查队列是否还有数据需要处理 */
		if (unlikely(virtqueue_poll(vq, opaque)))
			virtqueue_napi_schedule(napi, vq);
	} else {
		virtqueue_disable_cb(vq);
	}
}
```








# 参考

[Linux virtio-net driver - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/540370469)

[virtio-net 实现机制【二】（图文并茂） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/545258186)
