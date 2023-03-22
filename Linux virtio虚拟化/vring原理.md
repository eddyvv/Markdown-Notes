# vring原理

---

在  `virtio` 设备上进行批量数据传输的机制被称为 `virtqueue` 。每个设备可以拥有零个或多个 `virtqueue` ，当 `Driver` 想要向设备发送数据时，它会填充 `Descriptor Table` 中的一项（或将几项链接在一起），并将描述符索引写入 `Available Ring` 中，然后它通知 `Device` ，当 `Device` 完成后，它将描述符索引写入 `Used Ring` 中并发送中断。

`Virtio` 在 `Guest` 中实现了前端驱动，在 `Host` （`qemu`）中实现了后端驱动，前后端之间通过 `Virtqueue` (虚拟队列)交换数据， `Host` 中会使用后端驱动程序模拟一个 `PCI` 设备，因此也称前端驱动为 `Driver` ，后端驱动为 `Device` 。 `Guest` 在 `Host OS` 上表示为一个 `Qemu` 的进程， `Guest OS` 的 `PA` 实际上也属于 `Host OS` 的地址空间，因此 `Virtio` 采用的 `Virtqueue` 的方式来避免了 `Guest` 和 `Host` 主机间数据的复制。

在 `virtio` 规范中定义了每个 `virtio` 设备包含的结构（[https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf)第2节 `Basic Facilities of a Virtio Device`）：

```c
* Device status field；设备状态字段
* Feature bits；特征位
* Notifications；通知
* Device Configuration space；设备配置空间
* One or more virtqueues；一个或多个virtqueues
```

## Virtqueues

`virtqueue` 用作在 `Guest` 与 `Host` 之间传递数据， `Host` 可以在用户态（ `qemu` ）实现，也可以在内核态（ `vhost` ）实现。在 `virtio` 规范中定义了每个 `Virtqueues` 包含的结构（第2.6节 `Virtqueues`）：

```c
* Descriptor Table；描述符
* Available Ring；driver提供给device的额外数据
* Used Ring；device提供device的额外数据
```

### Descriptor Table

描述符表，每一项描述符指向一片内存，内存类型可以分为 `out` 类型和 `in` 类型，分别代表输出和输入，而内存的管理都由 `driver`来负责。其中 `Descriptor Table` 指的是驱动和设备的缓冲区，由  `Queue Size` 个 `Descriptor` 组成。 `Descriptor` 中存有 `GPA` 的字段 `addr` ，长度字段 `len` ，可以链接 `next Descriptor` 的 `next` 指针等(形成描述符链)。如果协商了 `VIRTIO_F_INDIRECT_DESC` `feature` 则可以使用 `Indirect Descriptors` 来增加 `ring` 的容量，详见 [desc](#desc)。

### Available Ring

可用描述符区域，用于记录设备可用的描述符 `ID` ，它的主体是数组 `ring` ，实际就是一个环形缓冲区；`Guest` 通过 `Avail Ring` 向 `Host` 提供 `buffer` ，指示 `Guest` 增加的 `buffer` 位置和当前工作的位置，

`Available Ring` 中的每个条目是一个是描述符链的头部，它仅由 `Driver` 写入并由 `Device` 读取， `Device` 获取 `Descriptor` 后， `Descriptor` 对应的缓冲区可能是可读的也可能是可写的，可读的用于 `Driver` 发送数据，可写的用于接收数据，详见 [avail](#avail)。

### Used Ring

已用描述符区域，用于记录设备已经处理完的描述符 `ID` ，同样，它的 `ring` 数组也是环形缓冲区，与`avail_vring`不同的是，它还记录了设备写回的数据长度；`Host` 通过 `Used Ring` 向 `Host` 提供信息，指示 `Host` 处理 `buffer` 的位置。`device` 通过 `used ring` 来将消费过的 `buffer` 返回给 `driver`，详见 [used](#used)。

---

# vring数据结构



![image-20221214093921283](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/image-20221214093921283.png)

>  图片引用自[virtio数据结构总览 | 图_rtoax的博客-CSDN博客_virtio数据结构详解](https://blog.csdn.net/Rong_Toa/article/details/115488292)

![image-20221214095034338](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/image-20221214095034338.png)

>  图片引用自：[VirtIO实现原理——vring数据结构_燕无鸻的博客-CSDN博客_vring](https://blog.csdn.net/weixin_43836778/article/details/103777170?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-103777170-blog-115488292.pc_relevant_multi_platform_whitelistv4&spm=1001.2101.3001.4242.1&utm_relevant_index=3)

![在这里插入图片描述](image/vring%E5%8E%9F%E7%90%86/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE2MDU0NjM5,size_16,color_FFFFFF,t_70.png)

>  图片引用自：[qemu-virtio基本原理 - CodeAntenna](https://codeantenna.com/a/JgpulLqLIu)

![image-20221214111958050](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/image-20221214111958050.png)

<center>Linux内核VRing数据结构</center>

> 图片引自：[Introduction to VirtIO (oracle.com)](https://blogs.oracle.com/linux/post/introduction-to-virtio)

![VirtQueue+VRing-struct-diagram](image/vring%E5%8E%9F%E7%90%86/Medium.jpeg)

<center>QEMU中VRing数据结构</center>

> 图片引自：[Introduction to VirtIO (oracle.com)](https://blogs.oracle.com/linux/post/introduction-to-virtio)

`vring_virtqueue` 是一个 `virtqueue`，它将 `VRing` 的实现隐藏在 `virtqueue` 下面，当一个 `virtio-blk` 设备真正要发送数据时，只要传入 `virtqueue` 就能找到 `VRing` 并实现数据收发

## vring_virtqueue

`源码位置` [/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
struct vring_virtqueue {
    /* 设备看到的VRing */
	struct virtqueue vq;

	/* Is this a packed ring? */
	bool packed_ring;

	/* Is DMA API used? */
	bool use_dma_api;

	/* Can we use weak barriers? */
	bool weak_barriers;

	/* Other side has made a mess, don't try any more. */
	bool broken;

	/* Host supports indirect buffers */
	bool indirect;

	/* Host publishes avail event idx */
	bool event;

	/* Head of free buffer list.
     * 当前Descriptor Table中空闲buffer的起始位置
     */
	unsigned int free_head;
	/* Number we've added since last sync. */
    /* 
     * 上一次通知Host后，Guest往VRing上添加了多少次buffer
     * 每添加一次buffer，num_added加1，每kick一次Host清空
     */
	unsigned int num_added;

	/* Last used index  we've seen.
	 * for split ring, it just contains last used index
	 * for packed ring:
	 * bits up to VRING_PACKED_EVENT_F_WRAP_CTR include the last used index.
	 * bits from VRING_PACKED_EVENT_F_WRAP_CTR include the used wrap counter.
	 */
	u16 last_used_idx;

	/* Hint for event idx: already triggered no need to disable. */
	bool event_triggered;

	union {
		/* Available for split ring */
		struct vring_virtqueue_split split;

		/* Available for packed ring */
		struct vring_virtqueue_packed packed;
	};

	/* How to notify other side. FIXME: commonalize hcalls! */
	bool (*notify)(struct virtqueue *vq);

	/* DMA, allocation, and size information */
	bool we_own_ring;

#ifdef DEBUG
	/* They're supposed to lock for us. */
	unsigned int in_use;

	/* Figure out if their kicks are too delayed. */
	bool last_add_time_valid;
	ktime_t last_add_time;
#endif
};
```

### vring_virtqueue_split

```c
struct vring_virtqueue_split {
	/* Actual memory layout for this queue.
     * 实现数据传输的VRing结构
     */
	struct vring vring;

	/* Last written value to avail->flags */
	u16 avail_flags_shadow;

	/*
	 * Last written value to avail->idx in
	 * guest byte order.
	 */
    /* Guest每添加一次buffer，avail_idx_shadow加1
     * 每删除一次buffer，avail_idx_shadow减1
     */
	u16 avail_idx_shadow;

	/* Per-descriptor state. */
	struct vring_desc_state_split *desc_state;
	struct vring_desc_extra *desc_extra;

	/* DMA address and size information */
	dma_addr_t queue_dma_addr;
	size_t queue_size_in_bytes;

	/*
	 * The parameters for creating vrings are reserved for creating new
	 * vring.
	 */
	u32 vring_align;
	bool may_reduce_num;
};
```

---

#### vring

`Virtqueue` 有三个核心的数据结构，由 `struct vring` 负责组织。`VRings` 是 `guest ` 与其 `host` 之间交换数据的一种方式。

```c
struct vring {
	unsigned int num;

	vring_desc_t *desc;

	vring_avail_t *avail;

	vring_used_t *used;
};
```

##### <a id=desc>desc</a>

`desc` 在前端驱动里结构是 `vring_desc`，他是一个环形的 `buffer`，又被称为 `descriptor ring`，，其中包括一个数组，数组中每一项的元素包括指向 `guest buffer` 的地址和长度。另外每一个 `desc` 还包含一些其他信息。例如这个 `desc` 指向的不是真实的 `buffer` 而是一组 `desc` 时需要标为 `INDIRECT` 。如果这组 `buffer` 标记位 `device` 只写，设置 `WRITE` ，反之如果只读则清除 `WRITE` 。

> 注意：只有`driver` 才可以向 `desc ring` 添加（写入）描述符，后端设备只有在描述符标志表明缓冲区是可写的情况下才能写入设备可写缓冲区。缓冲区可以是只写, 也可以是只读. 但不能同时是可读可写.

`flags` 用于通知设备或驱动程序一些信息：

* 下一个描述符是否有相关数据；
* 此缓冲区是否只写；
* 缓冲区是否包含简介描述符表；

`源码位置` [/include/uapi/linux/virtio_ring.h](https://github1s.com/torvalds/linux/blob/HEAD/include/uapi/linux/virtio_ring.h)

```c
struct vring_desc {
	__virtio64 addr;	/* guest物理地址 GPA */
	__virtio32 len;		/* buffer长度 */
	__virtio16 flags;	/* 标志位, 比如NEXT, WRITE, INDIRECT */
	__virtio16 next;	/* 下一个链接的描述符索引 */
};
```

![描述符表](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/Medium.jpeg)

<center>Descriptor Ring/Table</center>

在上图中存在一个 `descriptor ring`，包含四个描述符，其中两个连接在一起，其中：

* `descriptor[0]` 表明数据缓冲区的 `GPA` 为 `0x600`，数据长度为 `0x100`，标志是 `device writable`， 没有 `next` 标志，描述符链无下一个。
* `descriptor[1]` 表明数据缓冲区的 `GPA` 为 `0x810`，数据长度为 `0x200`，标志是 `device writable-next`，即 `device` 可写，由于存在 `next` 则此描述符是描述符链的 `head` ，指向描述符表的 `ring[2]`。
* `descriptor[2]` 表明数据缓冲区的 `GPA` 为 `0xA10`，数据长度为 `0x200`，标志是 `device writable`， 没有 `next` 标志，描述符链无下一个。
* `descriptor[3]` 表明数据缓冲区的 `GPA` 为 `0x525`，数据长度为 `0x050`，标志是 `device read-only`， 没有 `next` 标志，描述符链无下一个。

##### <a id=desc>avail</a>

`avail` 在前端驱动里结构是 `vring_avail`，也是一个环形 `buffer`，又被称为 `avail vring` 。`driver` 需要给 `device` 提供数据，而数据的元数据存放在 `descriptor ring` 中，所以此时填入其中的是 `descriptor ring` 中的 `index` ， `device` 根据 `index` 去 `desc ring` 中找到对应的 `desc` ，然后获得地址信息 `GPA` 和 `len` ，最后再转换成 `HVA` 来消费这些数据。`driver` 在放置这些 `buffer`时并不意味着 `device`需要立即取出处理。

在 `vring_avail` 中 `flags` `idx` 比较重要：

*  `flags` 的最低位表明 `driver` 是否需要中断通知 `VIRTQ_AVAIL_F_NO_INTERRUPT`。

* ` idx` 指向下一个 `driver` 可用的 `desc ring index` ，`idx` 存放的 `ring[]` 数组索引， `ring[idx]` 存放才是下一次添加的 `buffer` 头在 `Descriptor Table` 的位置；

在这 `idx` 和 `flags`之后，是和 `desc ring` 相同长度的数组，即实际可用的 `ring` 数组，其中存放的是 `descriptor ring` 中的 `index` 。

`源码位置` [/include/uapi/linux/virtio_ring.h](https://github1s.com/torvalds/linux/blob/HEAD/include/uapi/linux/virtio_ring.h)

```c
struct vring_avail {
	__virtio16 flags;	/* 配置标志位 */
	__virtio16 idx;		/* 下一个可用 ring 索引 */
	__virtio16 ring[];	/* 实际的 avail ring */
};
```

> 注意：只有 `driver` 可以配置并向 `avail ring` 添加条目，而相应的 `device` 只能从中读取。

`avail vring` 初始结构如下图：

![avail-ring-init](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/Medium-1670998108741-19.jpeg)

在上图中，是一个没有条目和标志设置的可用 `vring`。`index` 是表明下一个可用 `avail ring` 数组。此处`avail ring` 的索引为`idx[0]ring[0]`。

对 `avail vring` 添加第一个条目如下：

![image-20221214140151347](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/image-20221214140151347.png)

在上述 `avail ring` 中由于没有 `next` ，所以 `device` 只能读取 `desc ring` 中第一个 `ring`。此处索引为 `avail_ring[0]idx[1]desc_ring[0]`。

向 `avail ring` 添加下一个描述符条目：

![Descriptor Table + Avail Ring 2](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/Medium-1670998948883-22.jpeg)

此处的描述符第二个和第三个连接在一起，`avail ring[1]` 指向描述符链的 `head`，`head` 链接到下一个描述符。此索引头为 `avail_ring[1]idx[2]desc_ring[1]` 。

添加第三个条目：

![desc-avail-ring3](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/Medium-1670999744491-25.jpeg)

注意这里的描述符索引为3，此处索引为 `avail_ring[2]idx[3]desc_ring[3]` 。

上述描述符填充的步骤可总结如下：

1. `driver` 分配内存，并添加一个 `buffer` 至 `virtqueue` 中；
2. 更新并填充 `desc` 指向这块 `buffer` 。
3. 填充完成后，`driver` 需要发布 `desc` 至 `avail ring` 并更新描述符索引值。
4. 发布完成过后，`buffer` 归属 `device` 管理， `driver` 需要通知 `device` ，如果需要将发送 `notifications`。

![Diagram: Process to make a buffer available](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/2020-07-08-virtio-fig4.png)

> 图片引用自 [Virtqueues and virtio ring: How the data travels (redhat.com)](https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels)

##### <a id=desc>used</a>

`used` 在前端驱动里结构是 `vring_used`，`device` 通过 `used ring` 将消费过的 `buffer` 返回给 `driver` 。

`源码位置` [/include/uapi/linux/virtio_ring.h](https://github1s.com/torvalds/linux/blob/HEAD/include/uapi/linux/virtio_ring.h)

```c
/* u32 is used here for ids for padding reasons. */
struct vring_used_elem {
	/* Index of start of used descriptor chain. */
	__virtio32 id;	/* 索引指向desc ring上的元素 */
	/* Total length of the descriptor chain which was used (written to) */
	__virtio32 len;	/* 写入descriptor buffer的数据长度 */
};

typedef struct vring_used_elem __attribute__((aligned(VRING_USED_ALIGN_SIZE)))
	vring_used_elem_t;

struct vring_used {
	__virtio16 flags;	/* 配置标志位 */
	__virtio16 idx;		/* 下一个可用的 avail ring 索引 */
	vring_used_elem_t ring[];	/* 实际使用的ring数组 */
};
```

`used ring` 结构与`avail ring` 结构基本一致：

![used-ring-init](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/Medium-1671001688948-30.jpeg)

与 `avail ring` 类似， `used ring` 也使用 `index` 字段， `index` 字段与 `avail ring` 的 `index` 字段相同，不同之处在于对于 `used ring` ，它表示 `used ring` 数组中的下一个可用条目。

>  注意：与 `avail ring` 相反，只有 `device` 以配置和添加条目到 `used ring` ，而相应的 `driver` 程序只能从它读取。

首先处理第一个 `used ring` 条目，第一个描述符的数据缓冲区被标记为设备可写，假设设备将字节写入描述符的设备可写缓冲区，写入的字节数为0x50，写入后的结果如下图所示：

![image-20221214163956379](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/image-20221214163956379.png)

在上图中，能看到使用过的 `ring` 条目，`idx` 为1表示下一个可用的条目索引为1。

处理第二个 `used ring` 条目，第二个描述符的数据缓冲区被标记为设备可写，且存在两个节点，假设设备将字节写入描述符的设备可写缓冲区，写入的字节数为0x350，写入后的结果如下图所示：

![Descriptor Table + Avail Ring + Used Ring 2](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/Medium-1671002423877-36.jpeg)

设备写入的字节为0x350，长度超出了 `desc[1]` 的可写长度，但由于`used ring` 写入的描述符数据长度表示每个链式描述符数据缓冲区的字节总数， 所以超过的部分将被写入至链接的下一个数据缓冲区。此时，`idx` 指向下一个可用的条目索引2。

最后，处理第三个条目，此处由于 `flags` 为只读，所以此处写入后的结果如下：

![Descriptor Table + Avail Ring + Used Ring 3](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/Medium-1671002435801-39.jpeg)

在 `device` 收到通知后，将会通知 `driver` 程序，并使用 `used queue` 的 `flags` 通知。

![Diagram: Process to mark a buffer as used](image/vring%E5%8E%9F%E7%90%86/2020-07-08-virtio-fig7.png)

> 图片引用自 [Virtqueues and virtio ring: How the data travels (redhat.com)](https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels)

---

# vring使用实例

例如： `Virtio-net` 设备发包过程为例讲解上述机制， `Driver` 将 `sk_buffer` 填充进 `scatterlist table` 中(只是设置地址没有数据搬移)，然后通过计算得到 `GPA` 并将 `GPA` 写入 `Descriptor Table` 中，同时将 `Desc chain` 的 `head` 记录到 `Available Ring` 中，然后通过 `PIO` 的方式通知 `Device` ， `Device` 发包并更新 `Used Ring` 。

![img](image/vring%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/virtiovring.jpeg)

> 图片引用自[Virtio原理简介 | Lauren·weblog (lihanlu.cn)](http://lihanlu.cn/virtio-introduction/)



---



# 参考

[VirtIO实现原理——vring数据结构_燕无鸻的博客-CSDN博客_vring](https://blog.csdn.net/weixin_43836778/article/details/103777170?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-103777170-blog-115488292.pc_relevant_multi_platform_whitelistv4&spm=1001.2101.3001.4242.1&utm_relevant_index=3)

[virtio数据结构总览 | 图_rtoax的博客-CSDN博客_virtio数据结构详解](https://blog.csdn.net/Rong_Toa/article/details/115488292)

[VirtIO实现原理——vring数据结构_燕无鸻的博客-CSDN博客_vring](https://blog.csdn.net/weixin_43836778/article/details/103777170?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-103777170-blog-115488292.pc_relevant_multi_platform_whitelistv4&spm=1001.2101.3001.4242.1&utm_relevant_index=3)

[https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.pdf) virtio space 1.1

[【原创】Linux虚拟化KVM-Qemu分析（十一）之virtqueue - LoyenWang - 博客园 (cnblogs.com)](https://www.cnblogs.com/LoyenWang/p/14589296.html)

[Virtqueues and virtio ring: How the data travels (redhat.com)](https://www.redhat.com/en/blog/virtqueues-and-virtio-ring-how-data-travels)

[Introduction to VirtIO (oracle.com)](https://blogs.oracle.com/linux/post/introduction-to-virtio)

[virtio系列-split virtqueue数据流_wjx5210的博客-CSDN博客_virtio split ring](https://blog.csdn.net/faxiang1230/article/details/120209543)

[Virtio原理简介 | Lauren·weblog (lihanlu.cn)](http://lihanlu.cn/virtio-introduction/)

[virtio-net 实现机制 - 豆奶特 (dounaite.com)](https://www.dounaite.com/article/627a49cfac359fc913293bc4.html)

[qemu-virtio基本原理 - CodeAntenna](https://codeantenna.com/a/JgpulLqLIu)



😲

📸

⏲

