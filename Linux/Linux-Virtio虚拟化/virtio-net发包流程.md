# virtio-net发包流程分析

![2020-11-23-virtionet](image/virtio-net%E5%8F%91%E5%8C%85%E6%B5%81%E7%A8%8B/2020-11-23-virtionet.4b41c976.png)



## 前端驱动部分
### 总流程

虚拟机中的进程发送网络包时，仍然通过文件系统和 `socket` 调用网络协议栈到达网络设备层。只不过此时不是到达普通的网络设备，而是 `virtio-net` 前端驱动， `virtio-net` 前端驱动作为网卡设备驱动层，接收IP层传输下来的二层网络数据包，发送网络包的流程最终将调用 `net_device_ops` 结构中的 `ndo_start_xmit`回调函数，在 `virtio-net` 驱动中，就是 `start_xmit` 函数

```c
start_xmit
    |---->free_old_xmit_skbs	/* 释放backend处理过的desc */
    |---->xmit_skb	/* 调用xmit_skb函数将网络包写入virtqueue */
    |	|---->sg_set_buf	/* 数据包头部填入scatterlist */
    |	|	|---->sg_set_page
    |	|---->skb_to_sgvec	/* 数据包填入scatterlist */
    |	|	|---->__skb_to_sgvec
    |	|		|---->sg_set_page
    |	|---->virtqueue_add_outbuf	/* 写入desc描述符表 */
    |	|	|---->virtqueue_add
    |	|		|---->virtqueue_add_split
    |---->skb_orphan	/* 孤立skb,因为此时对guest来说此skb的内容已发送 */
    |---->virtqueue_kick	/* 触发中断，通知host端 */
    	|---->virtqueue_notify
    		|---->vq->notify(_vq)
```
### start_xmit

当收到一包数据时，会调用客户机的 `virtio-net` 驱动中的 `start_xmit`。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static const struct net_device_ops virtnet_netdev = {
	.ndo_open            = virtnet_open,
	.ndo_stop   	     = virtnet_close,
	.ndo_start_xmit      = start_xmit,
	.ndo_validate_addr   = eth_validate_addr,
	.ndo_set_mac_address = virtnet_set_mac_address,
	.ndo_set_rx_mode     = virtnet_set_rx_mode,
	.ndo_get_stats64     = virtnet_stats,
	.ndo_vlan_rx_add_vid = virtnet_vlan_rx_add_vid,
	.ndo_vlan_rx_kill_vid = virtnet_vlan_rx_kill_vid,
	.ndo_bpf		= virtnet_xdp,
	.ndo_xdp_xmit		= virtnet_xdp_xmit,
	.ndo_features_check	= passthru_features_check,
	.ndo_get_phys_port_name	= virtnet_get_phys_port_name,
	.ndo_set_features	= virtnet_set_features,
	.ndo_tx_timeout		= virtnet_tx_timeout,
};
```

`start_xmit` 函数首先获取 `sk_buff`对应的 `send_queue`， `free_old_xmit_skbs` 释放 `backend` 处理过的 `desc`，然后调用 `xmit_skb` 发包，之后调用 `skb_orphan` 孤立 `skb` ,因为对于guest来说 `xmit_skb` 之后 `skb` 的内容已经是发送的了，最后调用 `virtqueue_kick` 通知Host。

`源码位置` [/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static netdev_tx_t start_xmit(struct sk_buff *skb, struct net_device *dev)
{
	struct virtnet_info *vi = netdev_priv(dev);
	int qnum = skb_get_queue_mapping(skb);
	struct send_queue *sq = &vi->sq[qnum];
	int err;
	struct netdev_queue *txq = netdev_get_tx_queue(dev, qnum);
	bool kick = !netdev_xmit_more();
	bool use_napi = sq->napi.weight;

	/* Free up any pending old buffers before queueing new ones. */
	do {
		if (use_napi)
			virtqueue_disable_cb(sq->vq);
		/* 释放backend处理过的desc */
		free_old_xmit_skbs(sq, false);

	} while (use_napi && kick &&
	       unlikely(!virtqueue_enable_cb_delayed(sq->vq)));

	/* timestamp packet in software */
	skb_tx_timestamp(skb);

    /* 发包 */
	/* Try to transmit */
	err = xmit_skb(sq, skb);

	/* This should not happen! */
	if (unlikely(err)) {
		dev->stats.tx_fifo_errors++;
		if (net_ratelimit())
			dev_warn(&dev->dev,
				 "Unexpected TXQ (%d) queue failure: %d\n",
				 qnum, err);
		dev->stats.tx_dropped++;
		dev_kfree_skb_any(skb);
		return NETDEV_TX_OK;
	}

	/* Don't wait up for transmitted skbs to be freed. */
	if (!use_napi) {
		skb_orphan(skb);
		nf_reset_ct(skb);
	}

	/* If running out of space, stop queue to avoid getting packets that we
	 * are then unable to transmit.
	 * An alternative would be to force queuing layer to requeue the skb by
	 * returning NETDEV_TX_BUSY. However, NETDEV_TX_BUSY should not be
	 * returned in a normal path of operation: it means that driver is not
	 * maintaining the TX queue stop/start state properly, and causes
	 * the stack to do a non-trivial amount of useless work.
	 * Since most packets only take 1 or 2 ring slots, stopping the queue
	 * early means 16 slots are typically wasted.
	 */
	if (sq->vq->num_free < 2+MAX_SKB_FRAGS) {
		netif_stop_subqueue(dev, qnum);
		if (!use_napi &&
		    unlikely(!virtqueue_enable_cb_delayed(sq->vq))) {
			/* More just got used, free them then recheck. */
			free_old_xmit_skbs(sq, false);
			if (sq->vq->num_free >= 2+MAX_SKB_FRAGS) {
				netif_start_subqueue(dev, qnum);
				virtqueue_disable_cb(sq->vq);
			}
		}
	}

	if (kick || netif_xmit_stopped(txq)) {
		if (virtqueue_kick_prepare(sq->vq) && virtqueue_notify(sq->vq)) {
			u64_stats_update_begin(&sq->stats.syncp);
			sq->stats.kicks++;
			u64_stats_update_end(&sq->stats.syncp);
		}
	}

	return NETDEV_TX_OK;
}
```

#### xmit_skb

具体的发包过程由 `xmit_skb` 函数完成，函数主要将数据包头部和数据包填入 `scatterlist`，然后调用 `virtqueue_add_outbuf`将 `sg table` 写入 `desc` 描述符表，并将 `head desc`信息写入 `vring.avail`。

`源码位置` [/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static int xmit_skb(struct send_queue *sq, struct sk_buff *skb)
{
	struct virtio_net_hdr_mrg_rxbuf *hdr;	/* 数据包头 */
	const unsigned char *dest = ((struct ethhdr *)skb->data)->h_dest;
	struct virtnet_info *vi = sq->vq->vdev->priv;
	int num_sg;
	unsigned hdr_len = vi->hdr_len;
	bool can_push;

	pr_debug("%s: xmit %p %pM\n", vi->dev->name, skb, dest);

	can_push = vi->any_header_sg &&
		!((unsigned long)skb->data & (__alignof__(*hdr) - 1)) &&
		!skb_header_cloned(skb) && skb_headroom(skb) >= hdr_len;
	/* Even if we can, don't push here yet as this would skew
	 * csum_start offset below. */
	if (can_push)
		hdr = (struct virtio_net_hdr_mrg_rxbuf *)(skb->data - hdr_len);
	else
		hdr = skb_vnet_hdr(skb);

    /* 配置GSO、校验和相关配置 */
	if (virtio_net_hdr_from_skb(skb, &hdr->hdr,
				    virtio_is_little_endian(vi->vdev), false,
				    0))
		return -EPROTO;
	
    /* 判断设备是否支持合并buffer */
	if (vi->mergeable_rx_bufs)
		hdr->num_buffers = 0;

	sg_init_table(sq->sg, skb_shinfo(skb)->nr_frags + (can_push ? 1 : 2));
	if (can_push) {
		__skb_push(skb, hdr_len);
		num_sg = skb_to_sgvec(skb, sq->sg, 0, skb->len);
		if (unlikely(num_sg < 0))
			return num_sg;
		/* Pull header back to avoid skew in tx bytes calculations. */
		__skb_pull(skb, hdr_len);
	} else {
        /* 数据包头部填入scatterlist */
		sg_set_buf(sq->sg, hdr, hdr_len);
        /* 数据包填入scatterlist */
		num_sg = skb_to_sgvec(skb, sq->sg + 1, 0, skb->len);
		if (unlikely(num_sg < 0))
			return num_sg;
		num_sg++;
	}
    /* sg table 写入desc描述符表，head desc信息写vring.avail */
	return virtqueue_add_outbuf(sq->vq, sq->sg, num_sg, skb, GFP_ATOMIC);
}
```

##### sg_set_buf

`源码位置` [/include/linux/scatterlist.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/scatterlist.h)

```c
static inline void sg_set_buf(struct scatterlist *sg, const void *buf,
			      unsigned int buflen)
{
#ifdef CONFIG_DEBUG_SG
	BUG_ON(!virt_addr_valid(buf));
#endif
	sg_set_page(sg, virt_to_page(buf), buflen, offset_in_page(buf));
}
```

##### skb_to_sgvec

`源码位置` [/net/core/skbuff.c](https://github1s.com/torvalds/linux/blob/HEAD/net/core/skbuff.c)

```c
int skb_to_sgvec(struct sk_buff *skb, struct scatterlist *sg, int offset, int len)
{
	int nsg = __skb_to_sgvec(skb, sg, offset, len, 0);

	if (nsg <= 0)
		return nsg;

	sg_mark_end(&sg[nsg - 1]);

	return nsg;
}
```

`源码位置` [/net/core/skbuff.c](https://github1s.com/torvalds/linux/blob/HEAD/net/core/skbuff.c)

```c
static int
__skb_to_sgvec(struct sk_buff *skb, struct scatterlist *sg, int offset, int len,
	       unsigned int recursion_level)
{
	int start = skb_headlen(skb);
	int i, copy = start - offset;
	struct sk_buff *frag_iter;
	int elt = 0;

	if (unlikely(recursion_level >= 24))
		return -EMSGSIZE;

	if (copy > 0) {
		if (copy > len)
			copy = len;
		sg_set_buf(sg, skb->data + offset, copy);
		elt++;
		if ((len -= copy) == 0)
			return elt;
		offset += copy;
	}

	for (i = 0; i < skb_shinfo(skb)->nr_frags; i++) {
		int end;

		WARN_ON(start > offset + len);

		end = start + skb_frag_size(&skb_shinfo(skb)->frags[i]);
		if ((copy = end - offset) > 0) {
			skb_frag_t *frag = &skb_shinfo(skb)->frags[i];
			if (unlikely(elt && sg_is_last(&sg[elt - 1])))
				return -EMSGSIZE;

			if (copy > len)
				copy = len;
			sg_set_page(&sg[elt], skb_frag_page(frag), copy,
				    skb_frag_off(frag) + offset - start);
			elt++;
			if (!(len -= copy))
				return elt;
			offset += copy;
		}
		start = end;
	}

	skb_walk_frags(skb, frag_iter) {
		int end, ret;

		WARN_ON(start > offset + len);

		end = start + frag_iter->len;
		if ((copy = end - offset) > 0) {
			if (unlikely(elt && sg_is_last(&sg[elt - 1])))
				return -EMSGSIZE;

			if (copy > len)
				copy = len;
			ret = __skb_to_sgvec(frag_iter, sg+elt, offset - start,
					      copy, recursion_level + 1);
			if (unlikely(ret < 0))
				return ret;
			elt += ret;
			if ((len -= copy) == 0)
				return elt;
			offset += copy;
		}
		start = end;
	}
	BUG_ON(len);
	return elt;
}
```

`sg_set_buf` 与 `skb_to_sgvec` 会调用 `sg_set_page` 将 `sk_buffer`（逻辑buffer）中的物理块的 `page` 信息、`offset`、`len` 信息放入 `sg` 中。

###### sg_set_page

`源码位置` [/include/linux/scatterlist.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/scatterlist.h)

```c
static inline void sg_set_page(struct scatterlist *sg, struct page *page,
			       unsigned int len, unsigned int offset)
{
	sg_assign_page(sg, page);
	sg->offset = offset;
	sg->length = len;
}
```

##### virtqueue_add_outbuf

写入desc描述符表

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
int virtqueue_add_outbuf(struct virtqueue *vq,
			 struct scatterlist *sg, unsigned int num,
			 void *data,
			 gfp_t gfp)
{
	return virtqueue_add(vq, &sg, num, 1, 0, data, NULL, gfp);
}
```

##### virtqueue_add

`virtqueue_add` 函数将要发送的 `skb` 转换的 `sg` 再转换为 `desc chain` 。

`virtqueue_add` 首先判断是否支持间接描述符表并且`total_sg > 1`(否则也不需要使用间接描述符表了)，如果支持 `indirect` 把 `descs_used` 设置为1，否则设为实际的 `sg entry` 总数 `total_sg`。如果主描述符表的空闲表项数小于 `total_sg` 就错误返回。然后在for循环中先调用 `vring_map_one_sg` 函数得到 `scatterlist` 中数据的 `GPA` ,然后将 `sg table` 中 `entry` 信息分别对应到描述符表中。标记 `last desc` 信息，更新空闲 `desc` 数量以及 `free_head` ，最后将 `head desc` 信息写入 `vring.avail->ring[]` 并更新 `idx` 。

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
static inline int virtqueue_add(struct virtqueue *_vq,
				struct scatterlist *sgs[],
				unsigned int total_sg,
				unsigned int out_sgs,
				unsigned int in_sgs,
				void *data,
				void *ctx,
				gfp_t gfp)
{
	struct vring_virtqueue *vq = to_vvq(_vq);

	return vq->packed_ring ? virtqueue_add_packed(_vq, sgs, total_sg,
					out_sgs, in_sgs, data, ctx, gfp) :
				 virtqueue_add_split(_vq, sgs, total_sg,
					out_sgs, in_sgs, data, ctx, gfp);
}
```

###### virtqueue_add_split

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
static inline int virtqueue_add_split(struct virtqueue *_vq,
				      struct scatterlist *sgs[],
				      unsigned int total_sg,
				      unsigned int out_sgs,
				      unsigned int in_sgs,
				      void *data,
				      void *ctx,
				      gfp_t gfp)
{
	struct vring_virtqueue *vq = to_vvq(_vq);
	struct scatterlist *sg;
	struct vring_desc *desc;
	unsigned int i, n, avail, descs_used, prev, err_idx;
	int head;
	bool indirect;

	START_USE(vq);

	BUG_ON(data == NULL);
	BUG_ON(ctx && vq->indirect);

	if (unlikely(vq->broken)) {
		END_USE(vq);
		return -EIO;
	}

	LAST_ADD_TIME_UPDATE(vq);

	BUG_ON(total_sg == 0);

	head = vq->free_head;

    /* 判断是否支持间接描述符 */
	if (virtqueue_use_indirect(vq, total_sg))
		desc = alloc_indirect_split(_vq, total_sg, gfp);
	else {
		desc = NULL;
		WARN_ON_ONCE(total_sg > vq->split.vring.num && !vq->indirect);
	}

	if (desc) {
		/* Use a single buffer which doesn't continue */
		indirect = true;
		/* Set up rest to use this indirect table. */
		i = 0;
		descs_used = 1;
	} else {
		indirect = false;
		desc = vq->split.vring.desc;
		i = head;
		descs_used = total_sg;
	}

	if (unlikely(vq->vq.num_free < descs_used)) {
		pr_debug("Can't add buf len %i - avail = %i\n",
			 descs_used, vq->vq.num_free);
		/* FIXME: for historical reasons, we force a notify here if
		 * there are outgoing parts to the buffer.  Presumably the
		 * host should service the ring ASAP. */
		if (out_sgs)
			vq->notify(&vq->vq);
		if (indirect)
			kfree(desc);
		END_USE(vq);
		return -ENOSPC;
	}

    /* sg table中entry信息记录到对应的desc表中 */
	for (n = 0; n < out_sgs; n++) {
		for (sg = sgs[n]; sg; sg = sg_next(sg)) {
            /* 得到总线地址addr，即GPA */
			dma_addr_t addr = vring_map_one_sg(vq, sg, DMA_TO_DEVICE);
			if (vring_mapping_error(vq, addr))
				goto unmap_release;

			prev = i;
			/* Note that we trust indirect descriptor
			 * table since it use stream DMA mapping.
			 */
            /* GPA等信息填入desc[]中 */
			i = virtqueue_add_desc_split(_vq, desc, i, addr, sg->length,
						     VRING_DESC_F_NEXT,
						     indirect);
		}
	}
	for (; n < (out_sgs + in_sgs); n++) {
		for (sg = sgs[n]; sg; sg = sg_next(sg)) {
			dma_addr_t addr = vring_map_one_sg(vq, sg, DMA_FROM_DEVICE);
			if (vring_mapping_error(vq, addr))
				goto unmap_release;

			prev = i;
			/* Note that we trust indirect descriptor
			 * table since it use stream DMA mapping.
			 */
			i = virtqueue_add_desc_split(_vq, desc, i, addr,
						     sg->length,
						     VRING_DESC_F_NEXT |
						     VRING_DESC_F_WRITE,
						     indirect);
		}
	}
	/* Last one doesn't continue. */
    /* 标记last desc信息 */
	desc[prev].flags &= cpu_to_virtio16(_vq->vdev, ~VRING_DESC_F_NEXT);
	if (!indirect && vq->use_dma_api)
		vq->split.desc_extra[prev & (vq->split.vring.num - 1)].flags &=
			~VRING_DESC_F_NEXT;

	if (indirect) {
		/* Now that the indirect table is filled in, map it. */
		dma_addr_t addr = vring_map_single(
			vq, desc, total_sg * sizeof(struct vring_desc),
			DMA_TO_DEVICE);
		if (vring_mapping_error(vq, addr))
			goto unmap_release;

		virtqueue_add_desc_split(_vq, vq->split.vring.desc,
					 head, addr,
					 total_sg * sizeof(struct vring_desc),
					 VRING_DESC_F_INDIRECT,
					 false);
	}
	/* 更新空闲desc数量 */
	/* We're using some buffers from the free list. */
	vq->vq.num_free -= descs_used;

    /* 更新free_head指针 */
	/* Update free pointer */
	if (indirect)
		vq->free_head = vq->split.desc_extra[head].next;
	else
		vq->free_head = i;

	/* Store token and indirect buffer state. */
	vq->split.desc_state[head].data = data;
	if (indirect)
		vq->split.desc_state[head].indir_desc = desc;
	else
		vq->split.desc_state[head].indir_desc = ctx;

    /* desc链中第一个desc放入vring_avail中的ring[] */
	/* Put entry in available array (but don't update avail->idx until they
	 * do sync). */
	avail = vq->split.avail_idx_shadow & (vq->split.vring.num - 1);
	vq->split.vring.avail->ring[avail] = cpu_to_virtio16(_vq->vdev, head);

	/* Descriptors and available array need to be set before we expose the
	 * new available array entries. */
	virtio_wmb(vq->weak_barriers);
	vq->split.avail_idx_shadow++;
    /* 更新vring_avail中idx */
	vq->split.vring.avail->idx = cpu_to_virtio16(_vq->vdev,
						vq->split.avail_idx_shadow);
	vq->num_added++;

	pr_debug("Added buffer head %i to %p\n", head, vq);
	END_USE(vq);

	/* This is very unlikely, but theoretically possible.  Kick
	 * just in case. */
	if (unlikely(vq->num_added == (1 << 16) - 1))
		virtqueue_kick(_vq);

	return 0;

unmap_release:
	err_idx = i;

	if (indirect)
		i = 0;
	else
		i = head;

	for (n = 0; n < total_sg; n++) {
		if (i == err_idx)
			break;
		if (indirect) {
			vring_unmap_one_split_indirect(vq, &desc[i]);
			i = virtio16_to_cpu(_vq->vdev, desc[i].next);
		} else
			i = vring_unmap_one_split(vq, i);
	}

	if (indirect)
		kfree(desc);

	END_USE(vq);
	return -ENOMEM;
}
```

#### virtqueue_kick

`virtqueue_kick` 通知后端驱动，host 主机队列里面有消息需要它处理。

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
bool virtqueue_kick(struct virtqueue *vq)
{
    /* 再次判断是否需要kick，需要的话调用virtqueue_notify */
	if (virtqueue_kick_prepare(vq))
		return virtqueue_notify(vq);
	return true;
}
```

##### virtqueue_notify

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
bool virtqueue_notify(struct virtqueue *_vq)
{
	struct vring_virtqueue *vq = to_vvq(_vq);

	if (unlikely(vq->broken))
		return false;

	/* Prod other side to tell it about changes. */
	if (!vq->notify(_vq)) {
		vq->broken = true;
		return false;
	}
	return true;
}
```

##### skb_xmit_done

若配置了设备发送中断，则 `skb_recv_done` 函数将被调用，主要完成释放分配给保存数据包的内存，以及更新由驱动程序维护的统计信息或计数器。

`源码位置`[/drivers/net/virtio_net.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/net/virtio_net.c)

```c
static void skb_xmit_done(struct virtqueue *vq)
{
	struct virtnet_info *vi = vq->vdev->priv;
	struct napi_struct *napi = &vi->sq[vq2txq(vq)].napi;

	/* Suppress further interrupts. */
	virtqueue_disable_cb(vq);

	if (napi->weight)
		virtqueue_napi_schedule(napi, vq);
	else
		/* We were probably waiting for more output buffers. */
		netif_wake_subqueue(vi->dev, vq2txq(vq));
}
```

## 后端驱动

![virtio-net-xmit2](image/virtio-net%E5%8F%91%E5%8C%85%E6%B5%81%E7%A8%8B/virtio-net-xmit2%20(2).png)

### 总流程

```c
virtio_queue_host_notifier_read	/* 接收guest的通知 */
    ---->virtio_queue_notify_vq
    	---->vq->handle_output(vdev, vq)
    		---->virtio_net_handle_tx_bh
    			---->qemu_bh_schedule
    			---->virtio_net_tx_bh
    				---->qemu_bh_schedule
    				---->virtio_net_flush_tx
    					---->virtqueue_pop	/* 从avail ring中取数据buffer的head desc */
    					---->qemu_sendv_packet_async	/* 调用qemu发包函数，通过qemu nic发送 */
    					---->virtqueue_push	/* 取消内存映射，更新used vring信息 */
    					---->virtio_notify	/*  */
```

### virtio_queue_host_notifier_read

`qemu` 中收到来自前端的通知后会执行 `virtio_queue_host_notifier_read`。

`源码位置`[/hw/virtio/virtio.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/virtio/virtio.c)

```c
void virtio_queue_host_notifier_read(EventNotifier *n)
{
    VirtQueue *vq = container_of(n, VirtQueue, host_notifier);
    if (event_notifier_test_and_clear(n)) {
        virtio_queue_notify_vq(vq);
    }
}
```

#### virtio_queue_notify_vq

`源码位置`[/hw/virtio/virtio.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/virtio/virtio.c)

```c
static void virtio_queue_notify_vq(VirtQueue *vq)
{
    if (vq->vring.desc && vq->handle_output) {
        VirtIODevice *vdev = vq->vdev;

        if (unlikely(vdev->broken)) {
            return;
        }

        trace_virtio_queue_notify(vdev, vq - vdev->vq, vq);
        vq->handle_output(vdev, vq);

        if (unlikely(vdev->start_on_kick)) {
            virtio_set_started(vdev, true);
        }
    }
}
```

##### vq->handle_output(vdev, vq)

`vq->handle_output(vdev, vq)` 函数在 `virtio_net_class_init` -> `virtio_net_device_realize` -> `virtio_net_add_queue` -> `virtio_add_queue` 中进行注册。对于接收数据包，注册为 `virtio_net_handle_rx`，对于发送数据包，注册为 `virtio_net_handle_tx_timer` 和`virtio_net_handle_tx_bh`。

`源码位置` [/hw/virtio/virtio.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/virtio/virtio.c)

```c
static void virtio_net_add_queue(VirtIONet *n, int index)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(n);

    n->vqs[index].rx_vq = virtio_add_queue(vdev, n->net_conf.rx_queue_size,
                                           virtio_net_handle_rx);

    if (n->net_conf.tx && !strcmp(n->net_conf.tx, "timer")) {
        n->vqs[index].tx_vq =
            virtio_add_queue(vdev, n->net_conf.tx_queue_size,
                             virtio_net_handle_tx_timer);
        n->vqs[index].tx_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL,
                                              virtio_net_tx_timer,
                                              &n->vqs[index]);
    } else {
        n->vqs[index].tx_vq =
            virtio_add_queue(vdev, n->net_conf.tx_queue_size,
                             virtio_net_handle_tx_bh);
        n->vqs[index].tx_bh = qemu_bh_new(virtio_net_tx_bh, &n->vqs[index]);
    }

    n->vqs[index].tx_waiting = 0;
    n->vqs[index].n = n;
}
```

virtio_add_queue

`源码位置`[/hw/virtio/virtio.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/virtio/virtio.c)

```c
static void virtio_net_add_queue(VirtIONet *n, int index)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(n);

    n->vqs[index].rx_vq = virtio_add_queue(vdev, n->net_conf.rx_queue_size,
                                           virtio_net_handle_rx);

    if (n->net_conf.tx && !strcmp(n->net_conf.tx, "timer")) {
        n->vqs[index].tx_vq =
            virtio_add_queue(vdev, n->net_conf.tx_queue_size,
                             virtio_net_handle_tx_timer);
        n->vqs[index].tx_timer = timer_new_ns(QEMU_CLOCK_VIRTUAL,
                                              virtio_net_tx_timer,
                                              &n->vqs[index]);
    } else {
        n->vqs[index].tx_vq =
            virtio_add_queue(vdev, n->net_conf.tx_queue_size,
                             virtio_net_handle_tx_bh);
        n->vqs[index].tx_bh = qemu_bh_new(virtio_net_tx_bh, &n->vqs[index]);
    }

    n->vqs[index].tx_waiting = 0;
    n->vqs[index].n = n;
}
```

virtio_net_handle_rx

接收数据包函数

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_handle_rx(VirtIODevice *vdev, VirtQueue *vq)
{
    VirtIONet *n = VIRTIO_NET(vdev);
    int queue_index = vq2q(virtio_get_queue_index(vq));

    qemu_flush_queued_packets(qemu_get_subqueue(n->nic, queue_index));
}
```

virtio_net_handle_tx_timer

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_handle_tx_timer(VirtIODevice *vdev, VirtQueue *vq)
{
    VirtIONet *n = VIRTIO_NET(vdev);
    VirtIONetQueue *q = &n->vqs[vq2q(virtio_get_queue_index(vq))];

    if (unlikely((n->status & VIRTIO_NET_S_LINK_UP) == 0)) {
        virtio_net_drop_tx_queue_data(vdev, vq);
        return;
    }

    /* This happens when device was stopped but VCPU wasn't. */
    if (!vdev->vm_running) {
        q->tx_waiting = 1;
        return;
    }

    if (q->tx_waiting) {
        /* We already have queued packets, immediately flush */
        timer_del(q->tx_timer);
        virtio_net_tx_timer(q);
    } else {
        /* re-arm timer to flush it (and more) on next tick */
        timer_mod(q->tx_timer,
                  qemu_clock_get_ns(QEMU_CLOCK_VIRTUAL) + n->tx_timeout);
        q->tx_waiting = 1;
        virtio_queue_set_notification(vq, 0);
    }
}
```

###### virtio_net_handle_tx_bh

`virtio_net_handle_tx_bh` 函数中调用了 `qemu_bh_schedule(q->tx_bh)` 其参数为需要运行的函数，参数 `tx_bh` 在 `virtio_net_add_queue` 中调用 `qemu_bh_new(virtio_net_tx_bh, &n->vqs[index])` 注册为 `virtio_net_tx_bh` 

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_handle_tx_bh(VirtIODevice *vdev, VirtQueue *vq)
{
    VirtIONet *n = VIRTIO_NET(vdev);
    VirtIONetQueue *q = &n->vqs[vq2q(virtio_get_queue_index(vq))];

    if (unlikely((n->status & VIRTIO_NET_S_LINK_UP) == 0)) {
        virtio_net_drop_tx_queue_data(vdev, vq);
        return;
    }

    if (unlikely(q->tx_waiting)) {
        return;
    }
    q->tx_waiting = 1;
    /* This happens when device was stopped but VCPU wasn't. */
    if (!vdev->vm_running) {
        return;
    }
    virtio_queue_set_notification(vq, 0);
    qemu_bh_schedule(q->tx_bh);
}
```

qemu_bh_schedule

`源码位置` [/util/async.c](https://github1s.com/qemu/qemu/blob/HEAD/util/async.c)

```c
void qemu_bh_schedule(QEMUBH *bh)
{
    aio_bh_enqueue(bh, BH_SCHEDULED);
}
```

virtio_net_tx_bh

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static void virtio_net_tx_bh(void *opaque)
{
    VirtIONetQueue *q = opaque;
    VirtIONet *n = q->n;
    VirtIODevice *vdev = VIRTIO_DEVICE(n);
    int32_t ret;

    /* This happens when device was stopped but BH wasn't. */
    if (!vdev->vm_running) {
        /* Make sure tx waiting is set, so we'll run when restarted. */
        assert(q->tx_waiting);
        return;
    }

    q->tx_waiting = 0;

    /* Just in case the driver is not ready on more */
    if (unlikely(!(vdev->status & VIRTIO_CONFIG_S_DRIVER_OK))) {
        return;
    }

    ret = virtio_net_flush_tx(q);
    if (ret == -EBUSY || ret == -EINVAL) {
        return; /* Notification re-enable handled by tx_complete or device
                 * broken */
    }

    /* If we flush a full burst of packets, assume there are
     * more coming and immediately reschedule */
    if (ret >= n->tx_burst) {
        qemu_bh_schedule(q->tx_bh);
        q->tx_waiting = 1;
        return;
    }

    /* If less than a full burst, re-enable notification and flush
     * anything that may have come in while we weren't looking.  If
     * we find something, assume the guest is still active and reschedule */
    virtio_queue_set_notification(q->tx_vq, 1);
    ret = virtio_net_flush_tx(q);
    if (ret == -EINVAL) {
        return;
    } else if (ret > 0) {
        virtio_queue_set_notification(q->tx_vq, 0);
        qemu_bh_schedule(q->tx_bh);
        q->tx_waiting = 1;
    }
}
```

`virtio_net_tx_bh` 函数调用发送函数 `virtio_net_flush_tx`

virtio_net_flush_tx

`virtio_net_flush_tx` 函数获取报文 `elem` ，之后调用 `virtqueue_pop` 从 `avail ring` 中取数据 `buffer` 的 `head desc` ，然后调用 `qemu_sendv_packet_async` 发送 `packet` 。

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

```c
static int32_t virtio_net_flush_tx(VirtIONetQueue *q)
{
    VirtIONet *n = q->n;
    VirtIODevice *vdev = VIRTIO_DEVICE(n);
    VirtQueueElement *elem;
    int32_t num_packets = 0;
    int queue_index = vq2q(virtio_get_queue_index(q->tx_vq));
    if (!(vdev->status & VIRTIO_CONFIG_S_DRIVER_OK)) {
        return num_packets;
    }

    if (q->async_tx.elem) {
        virtio_queue_set_notification(q->tx_vq, 0);
        return num_packets;
    }

    for (;;) {
        ssize_t ret;
        unsigned int out_num;
        struct iovec sg[VIRTQUEUE_MAX_SIZE], sg2[VIRTQUEUE_MAX_SIZE + 1], *out_sg;
        struct virtio_net_hdr_mrg_rxbuf mhdr;
		/* 从avail ring中取数据buffer的head desc */
        elem = virtqueue_pop(q->tx_vq, sizeof(VirtQueueElement));
        if (!elem) {
            break;
        }

        out_num = elem->out_num;
        out_sg = elem->out_sg;
        if (out_num < 1) {
            virtio_error(vdev, "virtio-net header not in first element");
            virtqueue_detach_element(q->tx_vq, elem, 0);
            g_free(elem);
            return -EINVAL;
        }

        if (n->has_vnet_hdr) {
            if (iov_to_buf(out_sg, out_num, 0, &mhdr, n->guest_hdr_len) <
                n->guest_hdr_len) {
                virtio_error(vdev, "virtio-net header incorrect");
                virtqueue_detach_element(q->tx_vq, elem, 0);
                g_free(elem);
                return -EINVAL;
            }
            if (n->needs_vnet_hdr_swap) {
                virtio_net_hdr_swap(vdev, (void *) &mhdr);
                sg2[0].iov_base = &mhdr;
                sg2[0].iov_len = n->guest_hdr_len;
                out_num = iov_copy(&sg2[1], ARRAY_SIZE(sg2) - 1,
                                   out_sg, out_num,
                                   n->guest_hdr_len, -1);
                if (out_num == VIRTQUEUE_MAX_SIZE) {
                    goto drop;
                }
                out_num += 1;
                out_sg = sg2;
            }
        }
        /*
         * If host wants to see the guest header as is, we can
         * pass it on unchanged. Otherwise, copy just the parts
         * that host is interested in.
         */
        assert(n->host_hdr_len <= n->guest_hdr_len);
        if (n->host_hdr_len != n->guest_hdr_len) {
            unsigned sg_num = iov_copy(sg, ARRAY_SIZE(sg),
                                       out_sg, out_num,
                                       0, n->host_hdr_len);
            sg_num += iov_copy(sg + sg_num, ARRAY_SIZE(sg) - sg_num,
                             out_sg, out_num,
                             n->guest_hdr_len, -1);
            out_num = sg_num;
            out_sg = sg;
        }
		/* 调用qemu发包函数，通过qemu nic发送 */
        ret = qemu_sendv_packet_async(qemu_get_subqueue(n->nic, queue_index),
                                      out_sg, out_num, virtio_net_tx_complete);
        if (ret == 0) {
            virtio_queue_set_notification(q->tx_vq, 0);
            q->async_tx.elem = elem;
            return -EBUSY;
        }

drop:
        /* 取消内存映射，更新used vring信息 */
        virtqueue_push(q->tx_vq, elem, 0);
        virtio_notify(vdev, q->tx_vq);
        g_free(elem);

        if (++num_packets >= n->tx_burst) {
            break;
        }
    }
    return num_packets;
}
```

## 流程总结

* 虚拟机用户态应用程序通过 `write` 系统调用 `socket`；
* 写入的内容经过 `VFS` 层，内核协议栈，到达虚拟机里面的内核的网络设备驱动，即 `virtio_net`;
* `virtio_net` 网络设备有一个操作结构 `struct net_device_ops`，里面定义了发送一个网络包调用的函数为 `start_xmit`;
* 在 `virtio_net` 的前端驱动和 `qemu` 中的后端驱动之间，有两个队列 `virtqueue`，一个用于发送，一个用于接收。然后，我们需要在 `start_xmit` 中调用 `virtqueue_add`，将网络包放入发送队列，然后调用 `virtqueue_notify` 通知 `qemu`，前端驱动处理完成;
* `qemu` 本来处于 `KVM_RUN` 的状态，收到通知后，发起一次 `vm-exit` 操作退出客户机模式，进入宿主机模式。发送网络包的时候，`virtio_net_handle_tx_bh` 函数会被调用;
* 循环调用 `virtqueue_pop` ，从传输队列中获取要发送的数据，然后调用 `qemu_sendv_packet_async `进行发送。

# 参考

[Virtio网络发包过程分析 | Lauren·weblog (lihanlu.cn)](http://lihanlu.cn/virtio-net-xmit/)

[Virtio-Net 技术分析 (openeuler.org)](https://www.openeuler.org/zh/blog/xinleguo/2020-11-23-Virtio_Net_Technology.html)

[Linux virtio-net driver - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/540370469)
