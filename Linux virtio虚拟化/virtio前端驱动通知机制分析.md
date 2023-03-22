# virtio前端驱动通知机制分析

`virtio` 前后端主要通过PCI配置空间的寄存器来完成通信，`I/O` 请求的数据地址存放于 `vring` 中，并通过共享vring这个区域来实现 `I/O` 请求数据的共享。

![img](image/virtio%E5%89%8D%E7%AB%AF%E9%A9%B1%E5%8A%A8%E9%80%9A%E7%9F%A5%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90/774036-20211126153504501-1486553797.png)

由上图可知，虚拟机与主机之间交互用到了两个结构体：`pci_config` 和 `vring` ，`virtio` 设备的前后端通知机制由PCI配置空间完成。

## PCI配置空间

虚拟机中创建的 `virtio` 设备都是 `PCI` 设备，他们被挂载在 `PCI` 总线上，遵循通用的 `PCI` 设备的发现、挂在等机制。

当虚拟机启动时发现 `virtio PCI` 设备时，只有 `PCI` 配置空间可以被访问，配置空间内保存着该设备工作所需的厂家、功能、资源要求等信息，通过对这个空间信息的读取，完成对 `PCI` 设备的配置。同时配置空间上有一块存储器空间，里面包含了一些寄存器和 `I/O` 空间。

前后端的通知消息就是写在这些存储空间的寄存器，`virtio` 会为它的 `PCI`设备注册一个 `PCI BAR` 来访问这块寄存器空间。配置空间如下图所示：

![img](image/virtio%E5%89%8D%E7%AB%AF%E9%A9%B1%E5%8A%A8%E9%80%9A%E7%9F%A5%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90/774036-20220125112821030-231804051.png)

### PCI配置空间的获取

虚拟机系统在启动过程中在 `PCI` 总线上发现 `virtio-pci` 设备，就会调用 `virtio-pci` 的 `probe` 函数。该函数会将 `PCI` 配置空间上的寄存器映射到内存空间，并将这个地址赋值给 `virtio_pci_device` 的 `ioaddr` 变量。之后要对 `PCI` 配置空间上的寄存器操作时，只需要 `ioaddr+偏移量` 。

```c
virtio_pci_legacy_probe
    ---->vp_legacy_probe
    	---->pci_iomap(pci_dev, 0, 0);
```

### virtio_pci_legacy_probe

`源码位置`[/drivers/virtio/virtio_pci_legacy.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_pci_legacy.c)

```c
int virtio_pci_legacy_probe(struct virtio_pci_device *vp_dev)
{
	struct virtio_pci_legacy_device *ldev = &vp_dev->ldev;
	struct pci_dev *pci_dev = vp_dev->pci_dev;
	int rc;

	ldev->pci_dev = pci_dev;

	rc = vp_legacy_probe(ldev);
	if (rc)
		return rc;

	vp_dev->isr = ldev->isr;
	vp_dev->vdev.id = ldev->id;

	vp_dev->vdev.config = &virtio_pci_config_ops;

	vp_dev->config_vector = vp_config_vector;
	vp_dev->setup_vq = setup_vq;
	vp_dev->del_vq = del_vq;

	return 0;
}
```

#### vp_legacy_probe

`源码位置`[/drivers/virtio/virtio_pci_legacy_dev.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_pci_legacy_dev.c)

```c
int vp_legacy_probe(struct virtio_pci_legacy_device *ldev)
{
	struct pci_dev *pci_dev = ldev->pci_dev;
	int rc;

	/* We only own devices >= 0x1000 and <= 0x103f: leave the rest. */
	if (pci_dev->device < 0x1000 || pci_dev->device > 0x103f)
		return -ENODEV;

	if (pci_dev->revision != VIRTIO_PCI_ABI_VERSION)
		return -ENODEV;

	rc = dma_set_mask(&pci_dev->dev, DMA_BIT_MASK(64));
	if (rc) {
		rc = dma_set_mask_and_coherent(&pci_dev->dev, DMA_BIT_MASK(32));
	} else {
		/*
		 * The virtio ring base address is expressed as a 32-bit PFN,
		 * with a page size of 1 << VIRTIO_PCI_QUEUE_ADDR_SHIFT.
		 */
		dma_set_coherent_mask(&pci_dev->dev,
				DMA_BIT_MASK(32 + VIRTIO_PCI_QUEUE_ADDR_SHIFT));
	}

	if (rc)
		dev_warn(&pci_dev->dev, "Failed to enable 64-bit or 32-bit DMA.  Trying to continue, but this might not work.\n");

	rc = pci_request_region(pci_dev, 0, "virtio-pci-legacy");
	if (rc)
		return rc;
	/* 将PCI配置空间上的寄存器映射到内存空间 */
	ldev->ioaddr = pci_iomap(pci_dev, 0, 0);
	if (!ldev->ioaddr) {
		rc = -EIO;
		goto err_iomap;
	}

	ldev->isr = ldev->ioaddr + VIRTIO_PCI_ISR;

	ldev->id.vendor = pci_dev->subsystem_vendor;
	ldev->id.device = pci_dev->subsystem_device;

	return 0;
err_iomap:
	pci_release_region(pci_dev, 0);
	return rc;
}
```

`pci_iomap` 函数完成 `PCI BAR` 的映射，第一个参数是 `PCI` 设备的指针，第二个参数指定我们要映射的是0号 `BAR` ，第三个参数确定要映射的 `BAR` 空间多大，当第三个参数为0时，就将整个 `BAR` 空间都映射到内存空间上。 `VirtioPCI` 设备的0号BAR指向的就是配置空间的寄存器空间，也就是配置空间上用于消息通知的寄存器。

通过 `pci_iomap` 之后，我们就可以像操作普通内存一样（调用 `ioread` 和 `iowrite` ）来读写 `PCI` 硬件设备上的寄存器。

### 前端驱动

#### 总流程

```c
virtqueue_kick
    ---->virtqueue_kick_prepare	/* 再次判断是否需要kick，需要的话调用virtqueue_notify */
    	---->virtqueue_kick_prepare_split
    		---->vring_need_event
    		---->vring_avail_event
    ---->virtqueue_notify
    	---->vq->notify(_vq)
    		---->vp_notify	/* iowrite VIRTIO_PCI_QUEUE_NOTIFY通知KVM */
```

在`virtio` 发包流程中，`start_xmit` 的最后调用了 `virtqueue_kick` 函数来通知后端驱动。

`virtqueue_kick` 通知后端驱动，host主机队列里面有消息需要它处理。

#### virtqueue_kick

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

##### virtqueue_kick_prepare

`源码位置` [/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
bool virtqueue_kick_prepare(struct virtqueue *_vq)
{
	struct vring_virtqueue *vq = to_vvq(_vq);

	return vq->packed_ring ? virtqueue_kick_prepare_packed(_vq) :
				 virtqueue_kick_prepare_split(_vq);
}
```

virtqueue_kick_prepare_split

`源码位置` [/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
static bool virtqueue_kick_prepare_split(struct virtqueue *_vq)
{
	struct vring_virtqueue *vq = to_vvq(_vq);
    /* old表示上次kick后的vring.avail->idx;new表示当前idx */
	u16 new, old;
	bool needs_kick;

	START_USE(vq);
	/* We need to expose available array entries before checking avail
	 * event. */
	virtio_mb(vq->weak_barriers);

	old = vq->split.avail_idx_shadow - vq->num_added;
	new = vq->split.avail_idx_shadow;
	vq->num_added = 0;

	LAST_ADD_TIME_CHECK(vq);
	LAST_ADD_TIME_INVALID(vq);

    /* 当VIRTIO_F_EVENT_IDX被置位vq->event为1 */
	if (vq->event) {
		needs_kick = vring_need_event(virtio16_to_cpu(_vq->vdev,
					vring_avail_event(&vq->split.vring)),
					      new, old);
	} else {
		needs_kick = !(vq->split.vring.used->flags &
					cpu_to_virtio16(_vq->vdev,
						VRING_USED_F_NO_NOTIFY));
	}
	END_USE(vq);
	return needs_kick;
}
```

vring_need_event

在 `vring_need_event` 函数调用中传入的 `event_idx` 经过计算时 `used ring` 的最后一个元素的值。

`源码位置`[/include/uapi/linux/virtio_ring.h](https://github1s.com/torvalds/linux/blob/HEAD/include/uapi/linux/virtio_ring.h)

```c
static inline int vring_need_event(__u16 event_idx, __u16 new_idx, __u16 old)
{
	/* Note: Xen has similar logic for notification hold-off
	 * in include/xen/interface/io/ring.h with req_event and req_prod
	 * corresponding to event_idx + 1 and new_idx respectively.
	 * Note also that req_event and req_prod in Xen start at 1,
	 * event indexes in virtio start at 0. */
	return (__u16)(new_idx - event_idx - 1) < (__u16)(new_idx - old);
}
```

在上述代码中，如果 `(u16)(new_idx - event_idx - 1) < (u16)(new_idx - old)` 成立说明 `backend` 的处理速度够快，那么返回 `true` 表示可以 `kick backend` ，否则说明 `backend` 当前处理的位置 `event_idx` 落后于 `old` ，此时 `backend` 处理速度较慢，返回 `false` 等待下次一起 `kick backend` 。

vring_avail_event

`源码位置`[/include/uapi/linux/virtio_ring.h](https://github1s.com/torvalds/linux/blob/HEAD/include/uapi/linux/virtio_ring.h)

```c
#define vring_avail_event(vr) (*(__virtio16 *)&(vr)->used->ring[(vr)->num])
```

> 对于VirtIO的机制来说，backend一直消耗avail ring，frontend一直消耗used ring，因此backend用used ring的last entry告诉frontend自己当前处理到哪了。

##### virtqueue_notify

`virtqueue_notify` 函数调用了 `vq->notify(_vq)` ，`notify` 定义在 `struct ving_virtqueue` 中， `notify` 具体是哪个函数是在 `setup_vq` 中创建 `virtqueue` 时绑定的。

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

###### vring_virtqueue

```c
struct vring_virtqueue {
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

	/* Head of free buffer list. */
	unsigned int free_head;
	/* Number we've added since last sync. */
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

###### setup_vq

```c
static struct virtqueue *setup_vq(struct virtio_pci_device *vp_dev,
				  struct virtio_pci_vq_info *info,
				  unsigned int index,
				  void (*callback)(struct virtqueue *vq),
				  const char *name,
				  bool ctx,
				  u16 msix_vec)
{
	struct virtqueue *vq;
	u16 num;
	int err;
	u64 q_pfn;

	/* Check if queue is either not available or already active. */
	num = vp_legacy_get_queue_size(&vp_dev->ldev, index);
	if (!num || vp_legacy_get_queue_enable(&vp_dev->ldev, index))
		return ERR_PTR(-ENOENT);

	info->msix_vector = msix_vec;

	/* create the vring */
	vq = vring_create_virtqueue(index, num,
				    VIRTIO_PCI_VRING_ALIGN, &vp_dev->vdev,
				    true, false, ctx,
				    vp_notify, callback, name);
	if (!vq)
		return ERR_PTR(-ENOMEM);

	vq->num_max = num;

	q_pfn = virtqueue_get_desc_addr(vq) >> VIRTIO_PCI_QUEUE_ADDR_SHIFT;
	if (q_pfn >> 32) {
		dev_err(&vp_dev->pci_dev->dev,
			"platform bug: legacy virtio-pci must not be used with RAM above 0x%llxGB\n",
			0x1ULL << (32 + PAGE_SHIFT - 30));
		err = -E2BIG;
		goto out_del_vq;
	}

	/* activate the queue */
	vp_legacy_set_queue_address(&vp_dev->ldev, index, q_pfn);

    /* 需要写VIRTIO_PCI_QUEUE_NOTIFY，在vp_notify中用到 */
	vq->priv = (void __force *)vp_dev->ldev.ioaddr + VIRTIO_PCI_QUEUE_NOTIFY;

	if (msix_vec != VIRTIO_MSI_NO_VECTOR) {
		msix_vec = vp_legacy_queue_vector(&vp_dev->ldev, index, msix_vec);
		if (msix_vec == VIRTIO_MSI_NO_VECTOR) {
			err = -EBUSY;
			goto out_deactivate;
		}
	}

	return vq;

out_deactivate:
	vp_legacy_set_queue_address(&vp_dev->ldev, index, 0);
out_del_vq:
	vring_del_virtqueue(vq);
	return ERR_PTR(err);
}
```

在 `setup_vq` 函数中调用了 `vring_create_virtqueue` 将 `notify` 绑定为了 `vp_notify` 

###### vp_notify

```c
bool vp_notify(struct virtqueue *vq)
{
	/* we write the queue's selector into the notification register to
	 * signal the other end */
	iowrite16(vq->index, (void __iomem *)vq->priv);
	return true;
}
```

经过一系列的配置，可总结如下：

首先 `virtqueue_kick_prepare` 根据 `feature bit` 以及后端的处理速度来判断时候需要通知，如果需要则调用 `vp_notify` ,在其中 `iowrite VIRTIO_PCI_QUEUE_NOTIFY` 通知后端KVM主机

![img](image/virtio%E5%89%8D%E7%AB%AF%E9%A9%B1%E5%8A%A8%E9%80%9A%E7%9F%A5%E6%9C%BA%E5%88%B6%E5%88%86%E6%9E%90/virtio-frontend-kick1.jpeg)

### 后端驱动

`iowrite VIRTIO_PCI_QUEUE_NOTIFY` 后会产生一个 `vm-exit`，`KVM` 会判断 `exit_reason`， `I/O` 操作对应的执行函数是 `virtio_ioport_write()`  。

`源码位置`[/arch/x86/kvm/vmx/vmx.c](https://github1s.com/torvalds/linux/blob/HEAD/arch/x86/kvm/vmx/vmx.c)

```c
static int (*kvm_vmx_exit_handlers[])(struct kvm_vcpu *vcpu) = {
	[EXIT_REASON_EXCEPTION_NMI]           = handle_exception_nmi,
	[EXIT_REASON_EXTERNAL_INTERRUPT]      = handle_external_interrupt,
	[EXIT_REASON_TRIPLE_FAULT]            = handle_triple_fault,
	[EXIT_REASON_NMI_WINDOW]	      = handle_nmi_window,
	[EXIT_REASON_IO_INSTRUCTION]          = handle_io,
	[EXIT_REASON_CR_ACCESS]               = handle_cr,
	[EXIT_REASON_DR_ACCESS]               = handle_dr,
	[EXIT_REASON_CPUID]                   = kvm_emulate_cpuid,
	[EXIT_REASON_MSR_READ]                = kvm_emulate_rdmsr,
	[EXIT_REASON_MSR_WRITE]               = kvm_emulate_wrmsr,
	[EXIT_REASON_INTERRUPT_WINDOW]        = handle_interrupt_window,
	[EXIT_REASON_HLT]                     = kvm_emulate_halt,
	[EXIT_REASON_INVD]		      = kvm_emulate_invd,
	[EXIT_REASON_INVLPG]		      = handle_invlpg,
	[EXIT_REASON_RDPMC]                   = kvm_emulate_rdpmc,
	[EXIT_REASON_VMCALL]                  = kvm_emulate_hypercall,
	[EXIT_REASON_VMCLEAR]		      = handle_vmx_instruction,
	[EXIT_REASON_VMLAUNCH]		      = handle_vmx_instruction,
	[EXIT_REASON_VMPTRLD]		      = handle_vmx_instruction,
	[EXIT_REASON_VMPTRST]		      = handle_vmx_instruction,
	[EXIT_REASON_VMREAD]		      = handle_vmx_instruction,
	[EXIT_REASON_VMRESUME]		      = handle_vmx_instruction,
	[EXIT_REASON_VMWRITE]		      = handle_vmx_instruction,
	[EXIT_REASON_VMOFF]		      = handle_vmx_instruction,
	[EXIT_REASON_VMON]		      = handle_vmx_instruction,
	[EXIT_REASON_TPR_BELOW_THRESHOLD]     = handle_tpr_below_threshold,
	[EXIT_REASON_APIC_ACCESS]             = handle_apic_access,
	[EXIT_REASON_APIC_WRITE]              = handle_apic_write,
	[EXIT_REASON_EOI_INDUCED]             = handle_apic_eoi_induced,
	[EXIT_REASON_WBINVD]                  = kvm_emulate_wbinvd,
	[EXIT_REASON_XSETBV]                  = kvm_emulate_xsetbv,
	[EXIT_REASON_TASK_SWITCH]             = handle_task_switch,
	[EXIT_REASON_MCE_DURING_VMENTRY]      = handle_machine_check,
	[EXIT_REASON_GDTR_IDTR]		      = handle_desc,
	[EXIT_REASON_LDTR_TR]		      = handle_desc,
	[EXIT_REASON_EPT_VIOLATION]	      = handle_ept_violation,
	[EXIT_REASON_EPT_MISCONFIG]           = handle_ept_misconfig,
	[EXIT_REASON_PAUSE_INSTRUCTION]       = handle_pause,
	[EXIT_REASON_MWAIT_INSTRUCTION]	      = kvm_emulate_mwait,
	[EXIT_REASON_MONITOR_TRAP_FLAG]       = handle_monitor_trap,
	[EXIT_REASON_MONITOR_INSTRUCTION]     = kvm_emulate_monitor,
	[EXIT_REASON_INVEPT]                  = handle_vmx_instruction,
	[EXIT_REASON_INVVPID]                 = handle_vmx_instruction,
	[EXIT_REASON_RDRAND]                  = kvm_handle_invalid_op,
	[EXIT_REASON_RDSEED]                  = kvm_handle_invalid_op,
	[EXIT_REASON_PML_FULL]		      = handle_pml_full,
	[EXIT_REASON_INVPCID]                 = handle_invpcid,
	[EXIT_REASON_VMFUNC]		      = handle_vmx_instruction,
	[EXIT_REASON_PREEMPTION_TIMER]	      = handle_preemption_timer,
	[EXIT_REASON_ENCLS]		      = handle_encls,
	[EXIT_REASON_BUS_LOCK]                = handle_bus_lock_vmexit,
	[EXIT_REASON_NOTIFY]		      = handle_notify,
};

```

#### virtio_ioport_write

`virtio_ioport_write` 函数中对于 `VIRTIO_PCI_QUEUE_NOTIFY` 会调用 `virtio_queue_notify` ，进而调用模块 `VirtQueue` 对应的 `handle_output` 函数，该函数根据不同的设备有不同的实现。

`源码位置`[/hw/virtio/virtio-pci.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/virtio/virtio-pci.c)

```c
static void virtio_ioport_write(void *opaque, uint32_t addr, uint32_t val)
{
    VirtIOPCIProxy *proxy = opaque;
    VirtIODevice *vdev = virtio_bus_get_device(&proxy->bus);
    uint16_t vector;
    hwaddr pa;

    switch (addr) {
    case VIRTIO_PCI_GUEST_FEATURES:
        /* Guest does not negotiate properly?  We have to assume nothing. */
        if (val & (1 << VIRTIO_F_BAD_FEATURE)) {
            val = virtio_bus_get_vdev_bad_features(&proxy->bus);
        }
        virtio_set_features(vdev, val);
        break;
    case VIRTIO_PCI_QUEUE_PFN:
        pa = (hwaddr)val << VIRTIO_PCI_QUEUE_ADDR_SHIFT;
        if (pa == 0) {
            virtio_pci_reset(DEVICE(proxy));
        }
        else
            virtio_queue_set_addr(vdev, vdev->queue_sel, pa);
        break;
    case VIRTIO_PCI_QUEUE_SEL:
        if (val < VIRTIO_QUEUE_MAX)
            vdev->queue_sel = val;
        break;
    case VIRTIO_PCI_QUEUE_NOTIFY:
        if (val < VIRTIO_QUEUE_MAX) {
            virtio_queue_notify(vdev, val);
        }
        break;
    case VIRTIO_PCI_STATUS:
        if (!(val & VIRTIO_CONFIG_S_DRIVER_OK)) {
            virtio_pci_stop_ioeventfd(proxy);
        }

        virtio_set_status(vdev, val & 0xFF);

        if (val & VIRTIO_CONFIG_S_DRIVER_OK) {
            virtio_pci_start_ioeventfd(proxy);
        }

        if (vdev->status == 0) {
            virtio_pci_reset(DEVICE(proxy));
        }

        /* Linux before 2.6.34 drives the device without enabling
           the PCI device bus master bit. Enable it automatically
           for the guest. This is a PCI spec violation but so is
           initiating DMA with bus master bit clear. */
        if (val == (VIRTIO_CONFIG_S_ACKNOWLEDGE | VIRTIO_CONFIG_S_DRIVER)) {
            pci_default_write_config(&proxy->pci_dev, PCI_COMMAND,
                                     proxy->pci_dev.config[PCI_COMMAND] |
                                     PCI_COMMAND_MASTER, 1);
        }
        break;
    case VIRTIO_MSI_CONFIG_VECTOR:
        if (vdev->config_vector != VIRTIO_NO_VECTOR) {
            msix_vector_unuse(&proxy->pci_dev, vdev->config_vector);
        }
        /* Make it possible for guest to discover an error took place. */
        if (val < proxy->nvectors) {
            msix_vector_use(&proxy->pci_dev, val);
        } else {
            val = VIRTIO_NO_VECTOR;
        }
        vdev->config_vector = val;
        break;
    case VIRTIO_MSI_QUEUE_VECTOR:
        vector = virtio_queue_vector(vdev, vdev->queue_sel);
        if (vector != VIRTIO_NO_VECTOR) {
            msix_vector_unuse(&proxy->pci_dev, vector);
        }
        /* Make it possible for guest to discover an error took place. */
        if (val < proxy->nvectors) {
            msix_vector_use(&proxy->pci_dev, val);
        } else {
            val = VIRTIO_NO_VECTOR;
        }
        virtio_queue_set_vector(vdev, vdev->queue_sel, val);
        break;
    default:
        qemu_log_mask(LOG_GUEST_ERROR,
                      "%s: unexpected address 0x%x value 0x%x\n",
                      __func__, addr, val);
        break;
    }
}
```

##### virtio_queue_notify

`源码位置`[/hw/virtio/virtio.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/virtio/virtio.c)

```c
void virtio_queue_notify(VirtIODevice *vdev, int n)
{
    VirtQueue *vq = &vdev->vq[n];

    if (unlikely(!vq->vring.desc || vdev->broken)) {
        return;
    }

    trace_virtio_queue_notify(vdev, vq - vdev->vq, vq);
    if (vq->host_notifier_enabled) {
        event_notifier_set(&vq->host_notifier);
    } else if (vq->handle_output) {
        vq->handle_output(vdev, vq);	/*  */

        if (unlikely(vdev->start_on_kick)) {
            virtio_set_started(vdev, true);
        }
    }
}
```

###### vq->handle_output

`handle_output` 函数在驱动注册时赋值，例如 `virtio-net` ，在 `virtio_net_add_queue` 函数中将接收队列绑定为 `virtio_net_handle_rx` ，给发送队列绑定为 `virtio_net_handle_tx_bh` 或 `virtio_net_handle_tx_timer` 。

```c
virtio_net_class_init
    ->virtio_net_set_features
    	->virtio_net_set_multiqueue
    		->virtio_net_change_num_queue_pairs
    			->virtio_net_add_queue
    				->virtio_add_queue
    					->vdev->vq[i].handle_output = handle_output;
```

###### virtio_net_add_queue

`源码位置`[/hw/net/virtio-net.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/net/virtio-net.c)

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
对于`virtio-blk` ，`virtio_blk_device_realize` 函数中赋值为 `virtio_blk_handle_output`  函数。

```c
virtio_blk_class_init
    ->virtio_blk_device_realize
    	->virtio_add_queue(vdev, conf->queue_size, virtio_blk_handle_output)
    		->vdev->vq[i].handle_output = handle_output
```

###### virtio_blk_device_realize

`源码位置`[/hw/block/virtio-blk.c](https://github1s.com/qemu/qemu/blob/HEAD/hw/block/virtio-blk.c)

```c
static void virtio_blk_device_realize(DeviceState *dev, Error **errp)
{
    VirtIODevice *vdev = VIRTIO_DEVICE(dev);
    VirtIOBlock *s = VIRTIO_BLK(dev);
    VirtIOBlkConf *conf = &s->conf;
    Error *err = NULL;
    unsigned i;

    if (!conf->conf.blk) {
        error_setg(errp, "drive property not set");
        return;
    }
    if (!blk_is_inserted(conf->conf.blk)) {
        error_setg(errp, "Device needs media, but drive is empty");
        return;
    }
    if (conf->num_queues == VIRTIO_BLK_AUTO_NUM_QUEUES) {
        conf->num_queues = 1;
    }
    if (!conf->num_queues) {
        error_setg(errp, "num-queues property must be larger than 0");
        return;
    }
    if (conf->queue_size <= 2) {
        error_setg(errp, "invalid queue-size property (%" PRIu16 "), "
                   "must be > 2", conf->queue_size);
        return;
    }
    if (!is_power_of_2(conf->queue_size) ||
        conf->queue_size > VIRTQUEUE_MAX_SIZE) {
        error_setg(errp, "invalid queue-size property (%" PRIu16 "), "
                   "must be a power of 2 (max %d)",
                   conf->queue_size, VIRTQUEUE_MAX_SIZE);
        return;
    }

    if (!blkconf_apply_backend_options(&conf->conf,
                                       !blk_supports_write_perm(conf->conf.blk),
                                       true, errp)) {
        return;
    }
    s->original_wce = blk_enable_write_cache(conf->conf.blk);
    if (!blkconf_geometry(&conf->conf, NULL, 65535, 255, 255, errp)) {
        return;
    }

    if (!blkconf_blocksizes(&conf->conf, errp)) {
        return;
    }

    if (virtio_has_feature(s->host_features, VIRTIO_BLK_F_DISCARD) &&
        (!conf->max_discard_sectors ||
         conf->max_discard_sectors > BDRV_REQUEST_MAX_SECTORS)) {
        error_setg(errp, "invalid max-discard-sectors property (%" PRIu32 ")"
                   ", must be between 1 and %d",
                   conf->max_discard_sectors, (int)BDRV_REQUEST_MAX_SECTORS);
        return;
    }

    if (virtio_has_feature(s->host_features, VIRTIO_BLK_F_WRITE_ZEROES) &&
        (!conf->max_write_zeroes_sectors ||
         conf->max_write_zeroes_sectors > BDRV_REQUEST_MAX_SECTORS)) {
        error_setg(errp, "invalid max-write-zeroes-sectors property (%" PRIu32
                   "), must be between 1 and %d",
                   conf->max_write_zeroes_sectors,
                   (int)BDRV_REQUEST_MAX_SECTORS);
        return;
    }

    s->config_size = virtio_get_config_size(&virtio_blk_cfg_size_params,
                                            s->host_features);
    virtio_init(vdev, VIRTIO_ID_BLOCK, s->config_size);

    s->blk = conf->conf.blk;
    s->rq = NULL;
    s->sector_mask = (s->conf.conf.logical_block_size / BDRV_SECTOR_SIZE) - 1;

    for (i = 0; i < conf->num_queues; i++) {
        virtio_add_queue(vdev, conf->queue_size, virtio_blk_handle_output);
    }
    qemu_coroutine_inc_pool_size(conf->num_queues * conf->queue_size / 2);
    virtio_blk_data_plane_create(vdev, conf, &s->dataplane, &err);
    if (err != NULL) {
        error_propagate(errp, err);
        for (i = 0; i < conf->num_queues; i++) {
            virtio_del_queue(vdev, i);
        }
        virtio_cleanup(vdev);
        return;
    }

    s->change = qemu_add_vm_change_state_handler(virtio_blk_dma_restart_cb, s);
    blk_ram_registrar_init(&s->blk_ram_registrar, s->blk);
    blk_set_dev_ops(s->blk, &virtio_block_ops, s);

    blk_iostatus_enable(s->blk);

    add_boot_device_lchs(dev, "/disk@0,0",
                         conf->conf.lcyls,
                         conf->conf.lheads,
                         conf->conf.lsecs);
}
```

# 参考

[virtio前端通知机制分析 | Lauren·weblog (lihanlu.cn)](http://lihanlu.cn/virtio-frontend-kick/)

[virtIO前后端notify机制详解 - jack.chen - 博客园 (cnblogs.com)](https://www.cnblogs.com/ck1020/p/6066007.html)

[virtio简介（一）—— 框架分析 - Edver - 博客园 (cnblogs.com)](https://www.cnblogs.com/edver/p/14684104.html)
