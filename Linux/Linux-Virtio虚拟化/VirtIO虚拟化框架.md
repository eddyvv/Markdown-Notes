# virtio虚拟化框架

系统虚拟化技术是云计算最重要的核心技术之一。云计算平台的资源池化，资源统一管理以及后续的动态分配都是基于系统虚拟化技术才得以实现的。在计算机系统中，主要有计算资源，存储资源和网络资源。所以，系统虚拟化技术又可以细分为计算虚拟化，存储虚拟化和网络虚拟化。其中存储设备和网络设备一般都是以外设的形式和CPU通过I/O总线连接起来的，所以存储虚拟化和网络虚拟化又可以统一归类为I/O虚拟化技术。

`virtio` 是半虚拟化 `hypervisor` 中位于设备之上的抽象层。`virtio` 由 `Rusty Russell` 开发，他当时的目的是支持自己的虚拟化解决方案 `lguest`。

`virtio` 是对半虚拟化 `hypervisor` 中的一组通用模拟设备的抽象。该设置还允许 `hypervisor` 导出一组通用的模拟设备，并通过一个通用的应用编程接口（API）让它们变得可用。下图展示了为什么这很重要。有了半虚拟化 `hypervisor` 之后，来宾操作系统能够实现一组通用的接口，在一组后端驱动程序之后采用特定的设备模拟。后端驱动程序不需要是通用的，因为它们只实现前端所需的行为。

![image-20221209141129715](image/VirtIO%E8%99%9A%E6%8B%9F%E5%8C%96%E6%A1%86%E6%9E%B6/image-20221209141129715.png)

除了前端驱动程序（在来宾操作系统中实现）和后端驱动程序（在 `hypervisor` 中实现）之外，`virtio` 还定义了两个层来支持来宾操作系统到 `hypervisor` 的通信。在顶级（称为 virtio）的是虚拟队列接口，它在概念上将前端驱动程序附加到后端驱动程序。驱动程序可以使用 0 个或多个队列，具体数量取决于需求。例如，`virtio` 网络驱动程序使用两个虚拟队列（一个用于接收，另一个用于发送），而 `virtio` 块驱动程序仅使用一个虚拟队列。虚拟队列实际上被实现为跨越来宾操作系统和 `hypervisor` 的衔接点。但这可以通过任意方式实现，前提是来宾操作系统和 `hypervisor` 以相同的方式实现它。

I/O虚拟化技术可以分为两种，一种是`全虚拟化`，另一种是`半虚拟化`。

- 全虚拟化：Guest OS不知道自己是虚拟机，所以也就不需要修改Guest OS的驱动程序。这样Guest OS移植性好，但是虚拟机的I/O性能不高。
- 半虚拟化：Guest OS知道自己是虚拟机，通过前端驱动和后端模拟设备互相配合实现IO虚拟化。和全虚拟化相比，半虚拟化技术可以帮助虚拟机大幅度提高I/O的性能。目前在云计算场景中广泛使用的就是virtio在Linux上的具体实现，而且virtio已经成为虚拟化I/O技术的抽象接口规范和事实标准，即使非Linux系统也可以适用。

## virtio基本原理

`virtio` 是一个通用的 `I/O` 虚拟化框架， `hypervisor` 通过他模拟出一系列的虚拟化设备，例如：`virtio-net`、`virtio-blk`等，并使得这些设备在虚拟机内部通过 `api` 调用的方式变得可用。它包含4个部分：前端驱动、后端驱动、 `vring` 及通信间统一的接口。与其他的模拟 `I/O` 方式对比， `virtio` 减少了虚拟机的退出和数据拷贝，能够极大地提高 `I/O` 性能。计算机中存在不同的总线标准，而 `virtio` 采用的是 `pci` 总线（当然也可以用其他总线来实现）。每一个 `virtio` 设备就是一个 `pci` 设备。

## virtio架构层次

如果每个 virtio 设备都完整的实现自己的功能，就会造成不必要的代码冗余。针对这个问题，virtio 又设计了 `virtio_pci` 模块，以处理所有 virtio 设备的共性部分。这样一来所有的 virtio 设备在系统看来都是一个 PCI 设备，其设备驱动都是 `virtio_pci`。

PCI 设备标识的 Vendor ID `0x1AF4` （`PCI_VENDOR_ID_REDHAT_QUMRANET`）和 Device ID `0x1000` through `0x107F` 都是 virtio device，其中 `0x1000` through `0x103F` 为 legacy device，而 `0x1040` through `0x107F` 为 modern device。

`源码位置` [/drivers/virtio/virtio_pci_common.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_pci_common.c)

```c
/* Qumranet donated their vendor ID for devices 0x1000 thru 0x10FF. */
static const struct pci_device_id virtio_pci_id_table[] = {
	{ PCI_DEVICE(PCI_VENDOR_ID_REDHAT_QUMRANET, PCI_ANY_ID) },
	{ 0 }
};

MODULE_DEVICE_TABLE(pci, virtio_pci_id_table);
```

`源码位置` [/include/linux/pci.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/pci.h)

```c
/**
 * PCI_DEVICE - macro used to describe a specific PCI device
 * @vend: the 16 bit PCI Vendor ID
 * @dev: the 16 bit PCI Device ID
 *
 * This macro is used to create a struct pci_device_id that matches a
 * specific device.  The subvendor and subdevice fields will be set to
 * PCI_ANY_ID.
 */
#define PCI_DEVICE(vend,dev) \
	.vendor = (vend), .device = (dev), \
	.subvendor = PCI_ANY_ID, .subdevice = PCI_ANY_ID
```

`源码位置`[/include/linux/pci_ids.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/pci_ids.h)

```c
#define PCI_VENDOR_ID_REDHAT_QUMRANET    0x1af4
```

![img](image/VirtIO%E8%99%9A%E6%8B%9F%E5%8C%96%E6%A1%86%E6%9E%B6/virtio-architecture.jpeg)



![img](image/VirtIO%E8%99%9A%E6%8B%9F%E5%8C%96%E6%A1%86%E6%9E%B6/virtio-layer.png)

> 图片引用自 [半虚拟化 I/O 框架 virtio | Keep Coding | 苏易北 (abelsu7.top)](https://abelsu7.top/2019/09/02/virtio-in-kvm/)

如上图所示，virtio大致分为三个层次：**前端驱动**（位于客户机）、**后端驱动**（位于 QEMU）以及中间的**传输层**。

综上，每个 virtio 设备在系统层面上看，都是一个 `PCI设备` 。这些设备之间既有相同点，也有差异。

**相同点：**

* 都需要**挂接相应的 buffer 队列操作 virtqueue_ops**；
* 都需要**申请若干个 buffer 队列**，当执行 I/O 输出时，需要**向队列写入数据**；
* 都需要**执行 pci_iomap 将设备配置寄存器区间映射到内存区间**；
* 都需要**设置中断处理**；
* 等中断来了，都需要**从队列读出数据**，并**通知客户机系统，数据已入队**，

差异点：

* 与设备相关联的系统、业务、队列中**写入的数据含义各不相同**

### 前端驱动

`virtio ` 前端驱动位于 `Linux` 内核中，运行在虚拟机 `VM`，针对不同类型的设备有不同类型的驱动程序，包括`virtio-net`、`virtio-blk`、`virtio-pci`等，这些驱动程序与后端驱动交互的接口都是统一的。

virtio 前端驱动相关源码：

```c
/
 ├─ drivers
 |   ├─ block
 |   |   └─ virtio_blk.c
 |   |
 |   ├─ char
 |   |   ├─ hw_random
 |   |   |   └─ virtio_rng.c
 |   |   |
 |   |   └─ virtio_console.c
 |   |
 |   ├─ net
 |   |   └─ virtio_net.c
 |   |
 |   ├─ scsi
 |   |   └─ virtio_scsi.c
 |   |
 |   └─ virtio
 |       ├─ virtio_balloon.c
 |       ├─ virtio_mmio.c
 |       ├─ virtio_pci.c
 |       ├─ virtio_ring.c
 |       └─ virtio.c
 |
 ├─ include
 |   ├─ linux
 |   |   ├─ virtio_caif.h
 |   |   ├─ virtio_config.h
 |   |   ├─ virtio_console.h
 |   |   ├─ virtio_mmio.h
 |   |   ├─ virtio_ring.h
 |   |   ├─ virtio_scsi.h
 |   |   ├─ virtio.h
 |   |   └─ vring.h
 |   |
 |   └─ uapi
 |       └─ linux
 |           ├─ virtio_9p.h
 |           ├─ virtio_balloon.h
 |           ├─ virtio_blk.h
 |           ├─ virtio_config.h
 |           ├─ virtio_console.h
 |           ├─ virtio_ids.h
 |           ├─ virtio_net.h
 |           ├─ virtio_pci.h
 |           ├─ virtio_ring.h
 |           └─ virtio_rng.h
 |
 └─ tools
     └─ virtio
         ├─ linux
         |   ├─ virtio_config.h
         |   ├─ virtio_ring.h
         |   ├─ virtio.h
         |   └─ vring.h
         |
         ├─ virtio_test.c
         └─ vringh_test.c
```

### 传输层

#### virtio层

`virtio`层实现虚拟队列接口，作为`前后端通信的桥梁`，不同类型的设备使用的虚拟队列数量不同，例如 `virtio-net` 使用两个虚拟队列，一个用于接收，一个用于发送；`virtio-blk` 驱动仅使用一个虚拟队列。虚拟队列实际上被实现为跨越客户机操作系统和 `hypervisor` 的衔接点，可以通过任意方式实现，前提是客户机操作系统和 `virtio` 后端程序都遵循一定的标准，以相互匹配的方式实现它。

#### virtio-ring层

`virtio-ring` 是虚拟队列的具体实现，其中实现了环形缓冲区（`ring buffer`），用于保存前端驱动和后端处理程序执行的信息，并且它可以一次性保存前端驱动的多次I/O请求，并且交由后端驱动去批量处理，最后实际调用宿主机中设备驱动实现物理上的I/O操作，这样做就可以根据约定实现批量处理而不是客户机中每次I/O请求都需要处理一次，从而提高客户机与 `hypervisor` 信息交换的效率。

### 后端驱动

`virtio` 后端驱动位于 [qemu](https://github.com/qemu/qemu) ，后端设备承担的主体功能分为两部分：

1. virtio后端设备的模拟；
2. 依据virtio协议处理来自虚拟机端发送的请求。

在 `QEMU` 的实现中， `virtio` 设备是 `QEMU` 为虚拟机模拟的 `PCI` 设备，遵循 `PCI-SIG` 定义的 `PCI` 规范，具有配置空间、中断配置等功能； `virtio` 后端驱动运行在宿主机中，用于实现 `virtio` 后端操作硬件设备，例如向内核协议栈发送一个数据包完成虚拟机对网络数据的操作。


## virtio数据流交互机制

`virtio` 主要分为两部分：QEMU和内核驱动程序。Virtio设备的模拟通过QEMU完成的，QEMU代码在虚拟机启动之前，创建虚拟设备。虚拟机启动后检测到设备，调用内核的virtio设备驱动程序来加载这个virtio设备。

对于KVM虚拟机，都是通过QEMU这个用户空间程序创建的，每个KVM虚拟机都是一个QEMU进程，虚拟机的virtio设备是QEMU进程模拟的，虚拟机的内存也是从QEMU进程的地址空间内分配的。

虚拟机IO请求的整个流程如下图所示：

![img](image/VirtIO%E8%99%9A%E6%8B%9F%E5%8C%96%E6%A1%86%E6%9E%B6/774036-20211126153504501-1486553797.png)

步骤：

1. 虚拟机产生的IO请求被前端的virtio设备接收，并存放在virtio设备散列表`scatterList` 中；
2. virtio设备的`virtio` 提供add_buf将散列表中的数据映射至前后端数据共享区域 `vring`;
3. virtqueue通过`kick`函数来通知后端qemu进程。kick通过写pci配置空间的寄存器产生`kvm_exit`;
4. Qemu端注册`ioport_write/read` 函数监听PCI配置空间的改变，获取前端的通知消息；
5. Qemu端维护的`virtqueue`队列从数据共享区`vring`中获取数据;
6. Qemu将数据封装成`virtioreq`;
7.  Qemu进程将请求发送至硬件层。

前后端主要通过PCI配置空间的寄存器完成前后端的通信，而IO请求的数据地址则存在 `vring` 中，并通过共享 `vring` 这个区域来实现 `I/O` 请求数据的共享。



## virtio对象层次

从虚拟机的角度看，virtio 的结构体层次结构如下图所示。

![virtio基本数据结构层次](image/VirtIO%E8%99%9A%E6%8B%9F%E5%8C%96%E6%A1%86%E6%9E%B6/structure.gif)

 `virtio_driver` 在Guest OS中表示前端驱动程序 `Front-End` 简称 FE。与该驱动程序匹配的设备由 `virtio_device`（设备在虚拟机操作系统中的表示）封装。该结构体引用 `virtio_config_ops` 结构（它定义配置 `virtio device `的操作）。 `virtio_device` 由 `virtqueue` 引用。最后，每个 `virtqueue` 对象引用 `virtqueue_ops` 对象，后者定义处理 `hypervisor` 的驱动程序的底层队列操作。Linux中并没有实现`struct virtqueue_ops`，但是实现了对于 `virtqueue` 操作的函数。

### virtio_driver

`源码位置`[/include/linux/virtio.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/virtio.h)

```c
/**
 * struct virtio_driver - operations for a virtio I/O driver
 * @driver: underlying device driver (populate name and owner).
 * @id_table: the ids serviced by this driver.
 * @feature_table: an array of feature numbers supported by this driver.
 * @feature_table_size: number of entries in the feature table array.
 * @feature_table_legacy: same as feature_table but when working in legacy mode.
 * @feature_table_size_legacy: number of entries in feature table legacy array.
 * @probe: the function to call when a device is found.  Returns 0 or -errno.
 * @scan: optional function to call after successful probe; intended
 *    for virtio-scsi to invoke a scan.
 * @remove: the function to call when a device is removed.
 * @config_changed: optional function to call when the device configuration
 *    changes; may be called in interrupt context.
 * @freeze: optional function to call during suspend/hibernation.
 * @restore: optional function to call on resume.
 */
struct virtio_driver {
	struct device_driver driver;
    /* 设备 ID 列表 */
	const struct virtio_device_id *id_table;
    /* 特性表单 */
	const unsigned int *feature_table;
	unsigned int feature_table_size;
	const unsigned int *feature_table_legacy;
	unsigned int feature_table_size_legacy;
    /* 回调函数 */
	int (*validate)(struct virtio_device *dev);
	int (*probe)(struct virtio_device *dev);
	void (*scan)(struct virtio_device *dev);
	void (*remove)(struct virtio_device *dev);
	void (*config_changed)(struct virtio_device *dev);
#ifdef CONFIG_PM
	int (*freeze)(struct virtio_device *dev);
	int (*restore)(struct virtio_device *dev);
#endif
};
```

`virtio_driver` 结构体通过 `register_virtio_driver` 进行注册。

当 `hypervisor ` 识别到与设备列表中的设备 ID 相匹配的新设备时，将调用该设备驱动中的 `probe` 函数（由
`virtio_driver` 对象提供）来传入 `virtio_device` 对象。

### virtio_device

`virtio_device` 不包含到 `virtqueue` 的引用（但 `virtqueue` 引用 `virtio_device`）。要识别与该 `virtio_device` 相关联的 `virtqueue`，需要结合使用 `virtio_config_ops` 对象和 `find_vq` 函数。该对象返回与这个 `virtio_device` 实例相关联的虚拟队列。`find_vq` 函数还允许为 `virtqueue` 指定一个回调函数。

`源码位置`[/include/linux/virtio.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/virtio.h)

```c
/**
 * struct virtio_device - representation of a device using virtio
 * @index: unique position on the virtio bus
 * @failed: saved value for VIRTIO_CONFIG_S_FAILED bit (for restore)
 * @config_enabled: configuration change reporting enabled
 * @config_change_pending: configuration change reported while disabled
 * @config_lock: protects configuration change reporting
 * @dev: underlying device.
 * @id: the device type identification (used to match it with a driver).
 * @config: the configuration ops for this device.
 * @vringh_config: configuration ops for host vrings.
 * @vqs: the list of virtqueues for this device.
 * @features: the features supported by both driver and device.
 * @priv: private pointer for the driver's use.
 */
struct virtio_device {
	int index;
	bool failed;
	bool config_enabled;
	bool config_change_pending;
	spinlock_t config_lock;
	spinlock_t vqs_list_lock; /* Protects VQs list access */
	struct device dev;
	struct virtio_device_id id;
	const struct virtio_config_ops *config;
	const struct vringh_config_ops *vringh_config;
	struct list_head vqs;
	u64 features;
	void *priv;
};
```

### virtio_config_ops

对于 `virtio_device` 需要指定相应的操作函数，包括 `PCI` 配置空间设置和读取，`status` 操作，`virtqueue` 操作，和总线 `bus` 操作。

`源码位置`[/include/linux/virtio_config.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/virtio_config.h)

```c
/**
 * virtio_config_ops - operations for configuring a virtio device
 * Note: Do not assume that a transport implements all of the operations
 *       getting/setting a value as a simple read/write! Generally speaking,
 *       any of @get/@set, @get_status/@set_status, or @get_features/
 *       @finalize_features are NOT safe to be called from an atomic
 *       context.
 * @get: read the value of a configuration field
 *	vdev: the virtio_device
 *	offset: the offset of the configuration field
 *	buf: the buffer to write the field value into.
 *	len: the length of the buffer
 * @set: write the value of a configuration field
 *	vdev: the virtio_device
 *	offset: the offset of the configuration field
 *	buf: the buffer to read the field value from.
 *	len: the length of the buffer
 * @generation: config generation counter (optional)
 *	vdev: the virtio_device
 *	Returns the config generation counter
 * @get_status: read the status byte
 *	vdev: the virtio_device
 *	Returns the status byte
 * @set_status: write the status byte
 *	vdev: the virtio_device
 *	status: the new status byte
 * @reset: reset the device
 *	vdev: the virtio device
 *	After this, status and feature negotiation must be done again
 *	Device must not be reset from its vq/config callbacks, or in
 *	parallel with being added/removed.
 * @find_vqs: find virtqueues and instantiate them.
 *	vdev: the virtio_device
 *	nvqs: the number of virtqueues to find
 *	vqs: on success, includes new virtqueues
 *	callbacks: array of callbacks, for each virtqueue
 *		include a NULL entry for vqs that do not need a callback
 *	names: array of virtqueue names (mainly for debugging)
 *		include a NULL entry for vqs unused by driver
 *	Returns 0 on success or error status
 * @del_vqs: free virtqueues found by find_vqs().
 * @synchronize_cbs: synchronize with the virtqueue callbacks (optional)
 *      The function guarantees that all memory operations on the
 *      queue before it are visible to the vring_interrupt() that is
 *      called after it.
 *      vdev: the virtio_device
 * @get_features: get the array of feature bits for this device.
 *	vdev: the virtio_device
 *	Returns the first 64 feature bits (all we currently need).
 * @finalize_features: confirm what device features we'll be using.
 *	vdev: the virtio_device
 *	This sends the driver feature bits to the device: it can change
 *	the dev->feature bits if it wants.
 * Note: despite the name this can be called any number of times.
 *	Returns 0 on success or error status
 * @bus_name: return the bus name associated with the device (optional)
 *	vdev: the virtio_device
 *      This returns a pointer to the bus name a la pci_name from which
 *      the caller can then copy.
 * @set_vq_affinity: set the affinity for a virtqueue (optional).
 * @get_vq_affinity: get the affinity for a virtqueue (optional).
 * @get_shm_region: get a shared memory region based on the index.
 * @disable_vq_and_reset: reset a queue individually (optional).
 *	vq: the virtqueue
 *	Returns 0 on success or error status
 *	disable_vq_and_reset will guarantee that the callbacks are disabled and
 *	synchronized.
 *	Except for the callback, the caller should guarantee that the vring is
 *	not accessed by any functions of virtqueue.
 * @enable_vq_after_reset: enable a reset queue
 *	vq: the virtqueue
 *	Returns 0 on success or error status
 *	If disable_vq_and_reset is set, then enable_vq_after_reset must also be
 *	set.
 */
typedef void vq_callback_t(struct virtqueue *);
struct virtio_config_ops {
	void (*get)(struct virtio_device *vdev, unsigned offset,
		    void *buf, unsigned len);
	void (*set)(struct virtio_device *vdev, unsigned offset,
		    const void *buf, unsigned len);
	u32 (*generation)(struct virtio_device *vdev);
	u8 (*get_status)(struct virtio_device *vdev);
	void (*set_status)(struct virtio_device *vdev, u8 status);
	void (*reset)(struct virtio_device *vdev);
	int (*find_vqs)(struct virtio_device *, unsigned nvqs,
			struct virtqueue *vqs[], vq_callback_t *callbacks[],
			const char * const names[], const bool *ctx,
			struct irq_affinity *desc);
	void (*del_vqs)(struct virtio_device *);
	void (*synchronize_cbs)(struct virtio_device *);
	u64 (*get_features)(struct virtio_device *vdev);
	int (*finalize_features)(struct virtio_device *vdev);
	const char *(*bus_name)(struct virtio_device *vdev);
	int (*set_vq_affinity)(struct virtqueue *vq,
			       const struct cpumask *cpu_mask);
	const struct cpumask *(*get_vq_affinity)(struct virtio_device *vdev,
			int index);
	bool (*get_shm_region)(struct virtio_device *vdev,
			       struct virtio_shm_region *region, u8 id);
	int (*disable_vq_and_reset)(struct virtqueue *vq);
	int (*enable_vq_after_reset)(struct virtqueue *vq);
};
```

### virtqueue

`virtqueue` 是Guest OS 内存的一部分，用作 Guest 和 Host 的数据传输缓存。每一个 `virtqueue` 包含了对应的 `virtio_device` 以及对应的队列操作回调函数（callback）。

`源码位置`[/include/linux/virtio.h](https://github1s.com/torvalds/linux/blob/HEAD/include/linux/virtio.h)

```c
/**
 * struct virtqueue - a queue to register buffers for sending or receiving.
 * @list: the chain of virtqueues for this device
 * @callback: the function to call when buffers are consumed (can be NULL).
 * @name: the name of this virtqueue (mainly for debugging)
 * @vdev: the virtio device this queue was created for.
 * @priv: a pointer for the virtqueue implementation to use.
 * @index: the zero-based ordinal number for this queue.
 * @num_free: number of elements we expect to be able to fit.
 * @num_max: the maximum number of elements supported by the device.
 * @reset: vq is in reset state or not.
 *
 * A note on @num_free: with indirect buffers, each buffer needs one
 * element in the queue, otherwise a buffer will need one element per
 * sg element.
 */
struct virtqueue {
	struct list_head list;
	void (*callback)(struct virtqueue *vq);
	const char *name;
	struct virtio_device *vdev;
    /* 队列的索引 */
	unsigned int index;
    /* desc table 中 free 的数量，初始值为 Queue Size */
	unsigned int num_free;
	unsigned int num_max;
	void *priv;
	bool reset;
};
```

### virtpqeue_ops

Linux中没有实现 `struct virtqueue_ops`，但是实现了对于 `virtqueue` 操作的函数，针对 `virtqueue` 的操作包括 `add_buf`、`kick`、`get_buf`、`disable_cb`、`enable_cb` 等，定义了在guest操作系统和 `hypervisor` 之间移动命令和数据的方式。

#### add_buf

`add_buf` 用于向 queue 中添加一个新的 buffer，参数 data 是一个非空的令牌，用于识别 buffer，当 buffer 内容被消耗后，data 会返回。

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
/*
 * Generic functions and exported symbols.
 */

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

#### kick & notify

当通过 `add_buf` 向 hypervisor 发出请求时，Guest OS 能够通过 `kick` 函数通知 hypervisor 新的请求。为了获得最佳的性能，Guest OS 应该在通过 kick 发出通知之前将尽可能多的缓冲池装载到 `virtqueue`。Guest 再调用 `virtqueue_notify`来通知 Host。

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
/**
 * virtqueue_kick_prepare - first half of split virtqueue_kick call.
 * @_vq: the struct virtqueue
 *
 * Instead of virtqueue_kick(), you can do:
 *	if (virtqueue_kick_prepare(vq))
 *		virtqueue_notify(vq);
 *
 * This is sometimes useful because the virtqueue_kick_prepare() needs
 * to be serialized, but the actual virtqueue_notify() call does not.
 */
bool virtqueue_kick_prepare(struct virtqueue *_vq)
{
	struct vring_virtqueue *vq = to_vvq(_vq);

	return vq->packed_ring ? virtqueue_kick_prepare_packed(_vq) :
				 virtqueue_kick_prepare_split(_vq);
}
EXPORT_SYMBOL_GPL(virtqueue_kick_prepare);

/**
 * virtqueue_notify - second half of split virtqueue_kick call.
 * @_vq: the struct virtqueue
 *
 * This does not need to be serialized.
 *
 * Returns false if host notify failed or queue is broken, otherwise true.
 */
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
EXPORT_SYMBOL_GPL(virtqueue_notify);

/**
 * virtqueue_kick - update after add_buf
 * @vq: the struct virtqueue
 *
 * After one or more virtqueue_add_* calls, invoke this to kick
 * the other side.
 *
 * Caller must ensure we don't call this with other virtqueue
 * operations at the same time (except where noted).
 *
 * Returns false if kick failed, otherwise true.
 */
bool virtqueue_kick(struct virtqueue *vq)
{
	if (virtqueue_kick_prepare(vq))
		return virtqueue_notify(vq);
	return true;
}
EXPORT_SYMBOL_GPL(virtqueue_kick);
```

#### get_buf

Guest OS 仅需调用该函数或通过提供的 `virtqueue callback` 函数等待通知就可以实现轮询。当 Guest OS 知道缓冲区可用时，调用 `get_buf` 返回完成的缓冲区。

该函数返回使用过的 buffer，len 为写入到 buffer 中数据的长度。获取数据，释放 buffer，更新 vring 描述符表格中的 index。

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
/**
 * virtqueue_get_buf_ctx - get the next used buffer
 * @_vq: the struct virtqueue we're talking about.
 * @len: the length written into the buffer
 * @ctx: extra context for the token
 *
 * If the device wrote data into the buffer, @len will be set to the
 * amount written.  This means you don't need to clear the buffer
 * beforehand to ensure there's no data leakage in the case of short
 * writes.
 *
 * Caller must ensure we don't call this with other virtqueue
 * operations at the same time (except where noted).
 *
 * Returns NULL if there are no used buffers, or the "data" token
 * handed to virtqueue_add_*().
 */
void *virtqueue_get_buf_ctx(struct virtqueue *_vq, unsigned int *len,
			    void **ctx)
{
	struct vring_virtqueue *vq = to_vvq(_vq);

	return vq->packed_ring ? virtqueue_get_buf_ctx_packed(_vq, len, ctx) :
				 virtqueue_get_buf_ctx_split(_vq, len, ctx);
}
EXPORT_SYMBOL_GPL(virtqueue_get_buf_ctx);

void *virtqueue_get_buf(struct virtqueue *_vq, unsigned int *len)
{
	return virtqueue_get_buf_ctx(_vq, len, NULL);
}
EXPORT_SYMBOL_GPL(virtqueue_get_buf);
```

#### virtqueue_disable_cb

示意 guest 不再需要再知道一个 buffer 已经使用了，也就是关闭 device 的中断。驱动会在初始化时注册一个回调函数（在 virtqueue 中由 virtqueue 初始化的 callback 函数），`disable_cb` 通常在这个 virtqueue 回调函数中使用，用于关闭再次的回调发生。

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
/**
 * virtqueue_disable_cb - disable callbacks
 * @_vq: the struct virtqueue we're talking about.
 *
 * Note that this is not necessarily synchronous, hence unreliable and only
 * useful as an optimization.
 *
 * Unlike other operations, this need not be serialized.
 */
void virtqueue_disable_cb(struct virtqueue *_vq)
{
	struct vring_virtqueue *vq = to_vvq(_vq);

	/* If device triggered an event already it won't trigger one again:
	 * no need to disable.
	 */
	if (vq->event_triggered)
		return;

	if (vq->packed_ring)
		virtqueue_disable_cb_packed(_vq);
	else
		virtqueue_disable_cb_split(_vq);
}
EXPORT_SYMBOL_GPL(virtqueue_disable_cb);
```

#### virtqueue_enable_cb

与 `disable_cb` 刚好相反，用于重新开启设备中断的上报。

`源码位置`[/drivers/virtio/virtio_ring.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_ring.c)

```c
/**
 * virtqueue_enable_cb - restart callbacks after disable_cb.
 * @_vq: the struct virtqueue we're talking about.
 *
 * This re-enables callbacks; it returns "false" if there are pending
 * buffers in the queue, to detect a possible race between the driver
 * checking for more work, and enabling callbacks.
 *
 * Caller must ensure we don't call this with other virtqueue
 * operations at the same time (except where noted).
 */
bool virtqueue_enable_cb(struct virtqueue *_vq)
{
	unsigned int last_used_idx = virtqueue_enable_cb_prepare(_vq);

	return !virtqueue_poll(_vq, last_used_idx);
}
EXPORT_SYMBOL_GPL(virtqueue_enable_cb);
```








# 参考

[virtio-net 实现机制_老王不让用的博客-CSDN博客_vring_interrupt](https://blog.csdn.net/wangquan1992/article/details/120649182)

[Virtual I/O Device (VIRTIO) Version 1.1 (oasis-open.org)](https://docs.oasis-open.org/virtio/virtio/v1.1/csprd01/virtio-v1.1-csprd01.html)virtio v1.1规范

[virtio详细介绍和1.1新功能 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/361918197)

[virtio学习 | TO DO (juniorprincewang.github.io)](http://juniorprincewang.github.io/2018/03/01/virtio-docs/)

[半虚拟化 I/O 框架 virtio | Keep Coding | 苏易北 (abelsu7.top)](https://abelsu7.top/2019/09/02/virtio-in-kvm/)
