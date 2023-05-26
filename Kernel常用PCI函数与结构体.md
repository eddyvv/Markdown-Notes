# 内核常用PCI相关配置结构体及函数

## 结构体

### struct pci_device_id

`include/linux/mod_devicetable.h`

```c
/**
 * struct pci_device_id - PCI device ID structure
 * @vendor:		Vendor ID to match (or PCI_ANY_ID)
 * @device:		Device ID to match (or PCI_ANY_ID)
 * @subvendor:		Subsystem vendor ID to match (or PCI_ANY_ID)
 * @subdevice:		Subsystem device ID to match (or PCI_ANY_ID)
 * @class:		Device class, subclass, and "interface" to match.
 *			See Appendix D of the PCI Local Bus Spec or
 *			include/linux/pci_ids.h for a full list of classes.
 *			Most drivers do not need to specify class/class_mask
 *			as vendor/device is normally sufficient.
 * @class_mask:		Limit which sub-fields of the class field are compared.
 *			See drivers/scsi/sym53c8xx_2/ for example of usage.
 * @driver_data:	Data private to the driver.
 *			Most drivers don't need to use driver_data field.
 *			Best practice is to use driver_data as an index
 *			into a static list of equivalent device types,
 *			instead of using it as a pointer.
 */
struct pci_device_id {
	__u32 vendor, device;		/* Vendor and device ID or PCI_ANY_ID*/
	__u32 subvendor, subdevice;	/* Subsystem ID's or PCI_ANY_ID */
	__u32 class, class_mask;	/* (class,subclass,prog-if) triplet */
	kernel_ulong_t driver_data;	/* Data private to the driver */
};
```

`struct pci_device_id`用于存放本驱动所支持的所有PCI设备。通常使用`PCI_DEVICE`宏来完成`struct pci_device_id`前两个参数 `vendor`、`device`的填充。当对结构体数组填充完毕后需要调用`MODULE_DEVICE_TABLE`宏导出`pci_device_id`到用户空间，使热插拔或模块装载系统知道什么模块针对什么硬件设备。

### struct pci_driver

`include/linux/pci.h`

```c
/**
 * struct pci_driver - PCI driver structure
 * @node:	List of driver structures.
 * @name:	Driver name.
 * @id_table:	Pointer to table of device IDs the driver is
 *		interested in.  Most drivers should export this
 *		table using MODULE_DEVICE_TABLE(pci,...).
 * @probe:	This probing function gets called (during execution
 *		of pci_register_driver() for already existing
 *		devices or later if a new device gets inserted) for
 *		all PCI devices which match the ID table and are not
 *		"owned" by the other drivers yet. This function gets
 *		passed a "struct pci_dev \*" for each device whose
 *		entry in the ID table matches the device. The probe
 *		function returns zero when the driver chooses to
 *		take "ownership" of the device or an error code
 *		(negative number) otherwise.
 *		The probe function always gets called from process
 *		context, so it can sleep.
 * @remove:	The remove() function gets called whenever a device
 *		being handled by this driver is removed (either during
 *		deregistration of the driver or when it's manually
 *		pulled out of a hot-pluggable slot).
 *		The remove function always gets called from process
 *		context, so it can sleep.
 * @suspend:	Put device into low power state.
 * @resume:	Wake device from low power state.
 *		(Please see Documentation/power/pci.rst for descriptions
 *		of PCI Power Management and the related functions.)
 * @shutdown:	Hook into reboot_notifier_list (kernel/sys.c).
 *		Intended to stop any idling DMA operations.
 *		Useful for enabling wake-on-lan (NIC) or changing
 *		the power state of a device before reboot.
 *		e.g. drivers/net/e100.c.
 * @sriov_configure: Optional driver callback to allow configuration of
 *		number of VFs to enable via sysfs "sriov_numvfs" file.
 * @err_handler: See Documentation/PCI/pci-error-recovery.rst
 * @groups:	Sysfs attribute groups.
 * @driver:	Driver model structure.
 * @dynids:	List of dynamically added device IDs.
 */
struct pci_driver {
    /* PCI驱动程序链表节点，用于将该驱动与其他驱动程序链接在一起 */
	struct list_head	node;
    /* 驱动程序的名称，必须唯一，内核启动之后显示在sysfs中（/sys/bus/pci/drivers） */
	const char		*name;
    /* 用于指示该驱动支持的PCI设备的ID列表，包括厂商ID、设备ID子系统ID等信息 */
	const struct pci_device_id *id_table;	/* Must be non-NULL for probe to be called */
    /* 在PCI设备被检测到时调用的函数指针，用于初始化和配置PCI设备 */
	int  (*probe)(struct pci_dev *dev, const struct pci_device_id *id);	/* New device inserted */
    /* PCI设备被卸载时调用的函数指针，用于清理设备资源 */
	void (*remove)(struct pci_dev *dev);	/* Device removed (NULL if not a hot-plug capable driver) */
    /* 用于设备被挂起时调用 */
	int  (*suspend)(struct pci_dev *dev, pm_message_t state);	/* Device suspended */
    /* 系统恢复时调用 */
	int  (*resume)(struct pci_dev *dev);	/* Device woken up */
    /* 系统关闭时调用 */
	void (*shutdown)(struct pci_dev *dev);
    /* 虚拟化环境中用于配置PCI设备的虚拟化功能 */
	int  (*sriov_configure)(struct pci_dev *dev, int num_vfs); /* On PF */
    /* 处理PCI设备错误的函数 */
	const struct pci_error_handlers *err_handler;
	const struct attribute_group **groups;
    /* 包含设备驱动程序的基本信息 */
	struct device_driver	driver;
    /* 动态ID列表，用于运行时动态的删除或添加设备ID */
	struct pci_dynids	dynids;
};
```

`struct pci_driver`结构体用于表示PCI设备驱动，，该结构体提供了许多用于管理PCI设备驱动程序的函数指针和成员变量，使得驱动程序能够与操作系统进行交互、配置与控制。

## 设备注册、注销

### pci_register_driver

`include/linux/pci.h`

```c
/* pci_register_driver() must be a macro so KBUILD_MODNAME can be expanded */
#define pci_register_driver(driver)		\
	__pci_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)

/* drivers/pci/pci-driver.c */
/**
 * __pci_register_driver - register a new pci driver
 * @drv: the driver structure to register
 * @owner: owner module of drv
 * @mod_name: module name string
 *
 * Adds the driver structure to the list of registered drivers.
 * Returns a negative value on error, otherwise 0.
 * If no error occurred, the driver remains registered even if
 * no device was claimed during registration.
 */
int __pci_register_driver(struct pci_driver *drv, struct module *owner,
			  const char *mod_name)
{
	/* initialize common driver fields */
	drv->driver.name = drv->name;
	drv->driver.bus = &pci_bus_type;
	drv->driver.owner = owner;
	drv->driver.mod_name = mod_name;
	drv->driver.groups = drv->groups;

	spin_lock_init(&drv->dynids.lock);
	INIT_LIST_HEAD(&drv->dynids.list);

	/* register with core */
	return driver_register(&drv->driver);
}
EXPORT_SYMBOL(__pci_register_driver);
```

用于在内核中注册PCI设备驱动的函数，当驱动注册成功后，当操作系统检测到新的PCI设备时，会自动调用该函数完成程序的`probe`函数，以对该设备进行初始化和配置。

### pci_unregister_driver

`divers/pci/pci-driver.c`

```c
/**
 * pci_unregister_driver - unregister a pci driver
 * @drv: the driver structure to unregister
 *
 * Deletes the driver structure from the list of registered PCI drivers,
 * gives it a chance to clean up by calling its remove() function for
 * each device it was responsible for, and marks those devices as
 * driverless.
 */

void pci_unregister_driver(struct pci_driver *drv)
{
	driver_unregister(&drv->driver);
	pci_free_dynids(drv);
}
EXPORT_SYMBOL(pci_unregister_driver);
```

`pci_unregister_driver`用于将程序注销的函数

## 设备使能、禁用

### pci_enable_device

`/drivers/pci/pci.c`

```c
static int pci_enable_device_flags(struct pci_dev *dev, unsigned long flags)
{
	struct pci_dev *bridge;
	int err;
	int i, bars = 0;

	/*
	 * Power state could be unknown at this point, either due to a fresh
	 * boot or a device removal call.  So get the current power state
	 * so that things like MSI message writing will behave as expected
	 * (e.g. if the device really is in D0 at enable time).
	 */
	if (dev->pm_cap) {
		u16 pmcsr;
		pci_read_config_word(dev, dev->pm_cap + PCI_PM_CTRL, &pmcsr);
		dev->current_state = (pmcsr & PCI_PM_CTRL_STATE_MASK);
	}

	if (atomic_inc_return(&dev->enable_cnt) > 1)
		return 0;		/* already enabled */

	bridge = pci_upstream_bridge(dev);
	if (bridge)
		pci_enable_bridge(bridge);

	/* only skip sriov related */
	for (i = 0; i <= PCI_ROM_RESOURCE; i++)
		if (dev->resource[i].flags & flags)
			bars |= (1 << i);
	for (i = PCI_BRIDGE_RESOURCES; i < DEVICE_COUNT_RESOURCE; i++)
		if (dev->resource[i].flags & flags)
			bars |= (1 << i);

	err = do_pci_enable_device(dev, bars);
	if (err < 0)
		atomic_dec(&dev->enable_cnt);
	return err;
}

/**
 * pci_enable_device_io - Initialize a device for use with IO space
 * @dev: PCI device to be initialized
 *
 * Initialize device before it's used by a driver. Ask low-level code
 * to enable I/O resources. Wake up the device if it was suspended.
 * Beware, this function can fail.
 */
int pci_enable_device_io(struct pci_dev *dev)
{
	return pci_enable_device_flags(dev, IORESOURCE_IO);
}
EXPORT_SYMBOL(pci_enable_device_io);

/**
 * pci_enable_device_mem - Initialize a device for use with Memory space
 * @dev: PCI device to be initialized
 *
 * Initialize device before it's used by a driver. Ask low-level code
 * to enable Memory resources. Wake up the device if it was suspended.
 * Beware, this function can fail.
 */
int pci_enable_device_mem(struct pci_dev *dev)
{
	return pci_enable_device_flags(dev, IORESOURCE_MEM);
}
EXPORT_SYMBOL(pci_enable_device_mem);

/**
 * pci_enable_device - Initialize device before it's used by a driver.
 * @dev: PCI device to be initialized
 *
 * Initialize device before it's used by a driver. Ask low-level code
 * to enable I/O and memory. Wake up the device if it was suspended.
 * Beware, this function can fail.
 *
 * Note we don't actually enable the device many times if we call
 * this function repeatedly (we just increment the count).
 */
int pci_enable_device(struct pci_dev *dev)
{
	return pci_enable_device_flags(dev, IORESOURCE_MEM | IORESOURCE_IO);
}
EXPORT_SYMBOL(pci_enable_device);

```

在对PCI设备进行配置空间的读写之前需要调用`pci_enable_device`函数来完成设备的初始化和检测，以确保设备的状态正常，内核会将该设备的状态设置为已启用，并分配相应的系统资源，如I/O空间、内存空间和中断等。

### pci_disable_device

`/drivers/pci/pci.c`

```c
/**
 * pci_disable_device - Disable PCI device after use
 * @dev: PCI device to be disabled
 *
 * Signal to the system that the PCI device is not in use by the system
 * anymore.  This only involves disabling PCI bus-mastering, if active.
 *
 * Note we don't actually disable the device until all callers of
 * pci_enable_device() have called pci_disable_device().
 */
void pci_disable_device(struct pci_dev *dev)
{
	struct pci_devres *dr;

	dr = find_pci_dr(dev);
	if (dr)
		dr->enabled = 0;

	dev_WARN_ONCE(&dev->dev, atomic_read(&dev->enable_cnt) <= 0,
		      "disabling already-disabled device");

	if (atomic_dec_return(&dev->enable_cnt) != 0)
		return;

	do_pci_disable_device(dev);

	dev->is_busmaster = 0;
}
EXPORT_SYMBOL(pci_disable_device);
```

`pci_disable_device`用于禁用一个已经启用的PCI设备，以便释放相应的系统资源，避免资源浪费和冲突。在禁用PCI设备之前，内核会关闭该设备的所有功能，释放该设备占用的所有资源，以便于其他设备使用。

### pci_set_master

```c
/**
 * pci_set_master - enables bus-mastering for device dev
 * @dev: the PCI device to enable
 *
 * Enables bus-mastering on the device and calls pcibios_set_master()
 * to do the needed arch specific settings.
 */
void pci_set_master(struct pci_dev *dev)
{
	__pci_set_master(dev, true);
	pcibios_set_master(dev);
}
```

通过写命令到PCI设备的配置空间，给予设备作为master主动访问PCI总线的权限，这种权限对于设备发起DMA读写操作是必须的。

## PCI配置空间的读取写入

### pci_read_config_xxx

`/drivers/pci/access.c`

```c
/* 8位 */
int pci_read_config_byte(const struct pci_dev *dev, int where, u8 *val)
{
	if (pci_dev_is_disconnected(dev)) {
		*val = ~0;
		return PCIBIOS_DEVICE_NOT_FOUND;
	}
	return pci_bus_read_config_byte(dev->bus, dev->devfn, where, val);
}
EXPORT_SYMBOL(pci_read_config_byte);

/* 16位 */
int pci_read_config_word(const struct pci_dev *dev, int where, u16 *val)
{
	if (pci_dev_is_disconnected(dev)) {
		*val = ~0;
		return PCIBIOS_DEVICE_NOT_FOUND;
	}
	return pci_bus_read_config_word(dev->bus, dev->devfn, where, val);
}
EXPORT_SYMBOL(pci_read_config_word);

/* 32位 */
int pci_read_config_dword(const struct pci_dev *dev, int where,
					u32 *val)
{
	if (pci_dev_is_disconnected(dev)) {
		*val = ~0;
		return PCIBIOS_DEVICE_NOT_FOUND;
	}
	return pci_bus_read_config_dword(dev->bus, dev->devfn, where, val);
}
EXPORT_SYMBOL(pci_read_config_dword);
```

`pci_read_config_xxx`函数用于从PCI设备的配置空间中读取数据，以便获取设备配置信息。

### pci_write_config_xxx

`/drivers/pci/access.c`

```c
/* 8位 */
int pci_write_config_byte(const struct pci_dev *dev, int where, u8 val)
{
	if (pci_dev_is_disconnected(dev))
		return PCIBIOS_DEVICE_NOT_FOUND;
	return pci_bus_write_config_byte(dev->bus, dev->devfn, where, val);
}
EXPORT_SYMBOL(pci_write_config_byte);

/* 16位 */
int pci_write_config_word(const struct pci_dev *dev, int where, u16 val)
{
	if (pci_dev_is_disconnected(dev))
		return PCIBIOS_DEVICE_NOT_FOUND;
	return pci_bus_write_config_word(dev->bus, dev->devfn, where, val);
}
EXPORT_SYMBOL(pci_write_config_word);

/* 32位 */
int pci_write_config_dword(const struct pci_dev *dev, int where,
					 u32 val)
{
	if (pci_dev_is_disconnected(dev))
		return PCIBIOS_DEVICE_NOT_FOUND;
	return pci_bus_write_config_dword(dev->bus, dev->devfn, where, val);
}
EXPORT_SYMBOL(pci_write_config_dword);
```

`pci_write_config_xxx`函数用于向PCI设备的配置空间中写入数据。

## 获取区域信息（bar、info）

### pci_resource_xxx

`/include/linux/pci.h`

```c
/*
 * These helpers provide future and backwards compatibility
 * for accessing popular PCI BAR info
 */
#define pci_resource_start(dev, bar)	((dev)->resource[(bar)].start)
#define pci_resource_end(dev, bar)	((dev)->resource[(bar)].end)
#define pci_resource_flags(dev, bar)	((dev)->resource[(bar)].flags)
#define pci_resource_len(dev,bar) \
	((pci_resource_start((dev), (bar)) == 0 &&	\
	  pci_resource_end((dev), (bar)) ==		\
	  pci_resource_start((dev), (bar))) ? 0 :	\
							\
	 (pci_resource_end((dev), (bar)) -		\
	  pci_resource_start((dev), (bar)) + 1))
/*Bar值的范围为0-5*/
```

`pci_resource_start`用于返回PCI设备的资源起始地址，`pci_resource_end`用于获取PCI设备的资源结束地址，`pci_resource_flags`用于获取PCI设备的资源标志，包括`IORESOURCE_IO`、`IORESOURCE_MEM`、`IORESOURCE_IRQ`等，`pci_resource_len`用于获取PCI设备的资源长度。

## 区域申请与释放

### pci_request_regions

`/drivers/pci/pci.c`

```c
/**
 * pci_request_regions - Reserve PCI I/O and memory resources
 * @pdev: PCI device whose resources are to be reserved
 * @res_name: Name to be associated with resource.
 *
 * Mark all PCI regions associated with PCI device @pdev as
 * being reserved by owner @res_name.  Do not access any
 * address inside the PCI regions unless this call returns
 * successfully.
 *
 * Returns 0 on success, or %EBUSY on error.  A warning
 * message is also printed on failure.
 */
int pci_request_regions(struct pci_dev *pdev, const char *res_name)
{
	return pci_request_selected_regions(pdev,
			((1 << PCI_STD_NUM_BARS) - 1), res_name);
}
EXPORT_SYMBOL(pci_request_regions);

```

内核要操作PCI的资源区域需要先进行申请，之后才能进行操作,`pci_request_regions`函数用于申请PCI设备的资源区域。

###　pci_release_region

`/drivers/pci/pci.c`

```c
/**
 * pci_release_region - Release a PCI bar
 * @pdev: PCI device whose resources were previously reserved by
 *	  pci_request_region()
 * @bar: BAR to release
 *
 * Releases the PCI I/O and memory resources previously reserved by a
 * successful call to pci_request_region().  Call this function only
 * after all use of the PCI regions has ceased.
 */
void pci_release_region(struct pci_dev *pdev, int bar)
{
	struct pci_devres *dr;

	if (pci_resource_len(pdev, bar) == 0)
		return;
	if (pci_resource_flags(pdev, bar) & IORESOURCE_IO)
		release_region(pci_resource_start(pdev, bar),
				pci_resource_len(pdev, bar));
	else if (pci_resource_flags(pdev, bar) & IORESOURCE_MEM)
		release_mem_region(pci_resource_start(pdev, bar),
				pci_resource_len(pdev, bar));

	dr = find_pci_dr(pdev);
	if (dr)
		dr->region_mask &= ~(1 << bar);
}
EXPORT_SYMBOL(pci_release_region);
```

`pci_release_region`函数用于释放PCI设备的资源区域。

## 空间映射

### pci_ioremap_bar

`/drivers/pci/pci.c`

```c
void __iomem *pci_ioremap_bar(struct pci_dev *pdev, int bar)
{
	struct resource *res = &pdev->resource[bar];

	/*
	 * Make sure the BAR is actually a memory resource, not an IO resource
	 */
	if (res->flags & IORESOURCE_UNSET || !(res->flags & IORESOURCE_MEM)) {
		pci_warn(pdev, "can't ioremap BAR %d: %pR\n", bar, res);
		return NULL;
	}
	return ioremap(res->start, resource_size(res));
}
EXPORT_SYMBOL_GPL(pci_ioremap_bar);
```

内核通过`pci_ioremap_bar`函数完成物理地址到虚拟地址的映射，以便内核访问设备的I/O空间，作用与`ioremap`相同，只不过`pci_ioremap_bar`做了一些必要的检查。

## PCI驱动私有数据设置与获取

### pci_xxx_drvdata

`/include/linux/pci.h`

```c
/*
 * Similar to the helpers above, these manipulate per-pci_dev
 * driver-specific data.  They are really just a wrapper around
 * the generic device structure functions of these calls.
 */
static inline void *pci_get_drvdata(struct pci_dev *pdev)
{
	return dev_get_drvdata(&pdev->dev);
}

static inline void pci_set_drvdata(struct pci_dev *pdev, void *data)
{
	dev_set_drvdata(&pdev->dev, data);
}
```

`pci_get_drvdata`用于获取PCI设备驱动私有数据，`pci_set_drvdata`用于设置PCI设备驱动私有数据。

## PCI中断相关函数

### pci_irq_vector

`/drivers/pci/msi.c`

```c
/**
 * pci_irq_vector - return Linux IRQ number of a device vector
 * @dev: PCI device to operate on
 * @nr: device-relative interrupt vector index (0-based).
 */
int pci_irq_vector(struct pci_dev *dev, unsigned int nr)
{
	if (dev->msix_enabled) {
		struct msi_desc *entry;
		int i = 0;

		for_each_pci_msi_entry(entry, dev) {
			if (i == nr)
				return entry->irq;
			i++;
		}
		WARN_ON_ONCE(1);
		return -EINVAL;
	}

	if (dev->msi_enabled) {
		struct msi_desc *entry = first_pci_msi_entry(dev);

		if (WARN_ON_ONCE(nr >= entry->nvec_used))
			return -EINVAL;
	} else {
		if (WARN_ON_ONCE(nr > 0))
			return -EINVAL;
	}

	return dev->irq + nr;
}
EXPORT_SYMBOL(pci_irq_vector);

```

`pci_irq_vector`用于获取指定PCI设备中指定中断的中断向量的函数。

### pci_free_irq_vectors

`/drivers/pci/msi.c`

```c
/**
 * pci_free_irq_vectors - free previously allocated IRQs for a device
 * @dev:		PCI device to operate on
 *
 * Undoes the allocations and enabling in pci_alloc_irq_vectors().
 */
void pci_free_irq_vectors(struct pci_dev *dev)
{
	pci_disable_msix(dev);
	pci_disable_msi(dev);
}
EXPORT_SYMBOL(pci_free_irq_vectors);

```

### pci_alloc_irq_vectors

`/include/linux/pci.h`

```c
static inline int
pci_alloc_irq_vectors(struct pci_dev *dev, unsigned int min_vecs,
		      unsigned int max_vecs, unsigned int flags)
{
	return pci_alloc_irq_vectors_affinity(dev, min_vecs, max_vecs, flags,
					      NULL);
}

/* /drivers/pci/msi.c */
/**
 * pci_alloc_irq_vectors_affinity - allocate multiple IRQs for a device
 * @dev:		PCI device to operate on
 * @min_vecs:		minimum number of vectors required (must be >= 1)
 * @max_vecs:		maximum (desired) number of vectors
 * @flags:		flags or quirks for the allocation
 * @affd:		optional description of the affinity requirements
 *
 * Allocate up to @max_vecs interrupt vectors for @dev, using MSI-X or MSI
 * vectors if available, and fall back to a single legacy vector
 * if neither is available.  Return the number of vectors allocated,
 * (which might be smaller than @max_vecs) if successful, or a negative
 * error code on error. If less than @min_vecs interrupt vectors are
 * available for @dev the function will fail with -ENOSPC.
 *
 * To get the Linux IRQ number used for a vector that can be passed to
 * request_irq() use the pci_irq_vector() helper.
 */
int pci_alloc_irq_vectors_affinity(struct pci_dev *dev, unsigned int min_vecs,
				   unsigned int max_vecs, unsigned int flags,
				   const struct irq_affinity *affd)
{
	static const struct irq_affinity msi_default_affd;
	int vecs = -ENOSPC;

	if (flags & PCI_IRQ_AFFINITY) {
		if (!affd)
			affd = &msi_default_affd;
	} else {
		if (WARN_ON(affd))
			affd = NULL;
	}

	if (flags & PCI_IRQ_MSIX) {
		vecs = __pci_enable_msix_range(dev, NULL, min_vecs, max_vecs,
				affd);
		if (vecs > 0)
			return vecs;
	}

	if (flags & PCI_IRQ_MSI) {
		vecs = __pci_enable_msi_range(dev, min_vecs, max_vecs, affd);
		if (vecs > 0)
			return vecs;
	}

	/* use legacy irq if allowed */
	if (flags & PCI_IRQ_LEGACY) {
		if (min_vecs == 1 && dev->irq) {
			pci_intx(dev, 1);
			return 1;
		}
	}

	return vecs;
}
EXPORT_SYMBOL(pci_alloc_irq_vectors_affinity);
```

`pci_alloc_irq_vectors`用于Linux内核分配指定的PCI设备的中断向量的函数。

### pci_request_irq

```c
/**
 * pci_request_irq - allocate an interrupt line for a PCI device
 * @dev:	PCI device to operate on
 * @nr:		device-relative interrupt vector index (0-based).
 * @handler:	Function to be called when the IRQ occurs.
 *		Primary handler for threaded interrupts.
 *		If NULL and thread_fn != NULL the default primary handler is
 *		installed.
 * @thread_fn:	Function called from the IRQ handler thread
 *		If NULL, no IRQ thread is created
 * @dev_id:	Cookie passed back to the handler function
 * @fmt:	Printf-like format string naming the handler
 *
 * This call allocates interrupt resources and enables the interrupt line and
 * IRQ handling. From the point this call is made @handler and @thread_fn may
 * be invoked.  All interrupts requested using this function might be shared.
 *
 * @dev_id must not be NULL and must be globally unique.
 */
int pci_request_irq(struct pci_dev *dev, unsigned int nr, irq_handler_t handler,
		irq_handler_t thread_fn, void *dev_id, const char *fmt, ...)
{
	va_list ap;
	int ret;
	char *devname;

	va_start(ap, fmt);
	devname = kvasprintf(GFP_KERNEL, fmt, ap);
	va_end(ap);

	ret = request_threaded_irq(pci_irq_vector(dev, nr), handler, thread_fn,
			IRQF_SHARED, devname, dev_id);
	if (ret)
		kfree(devname);
	return ret;
}
EXPORT_SYMBOL(pci_request_irq);

```

向内核中注册相应中断的中断处理函数调用`pci_request_irq`。，在驱动加载完成后会在可执行`cat /proc/interrupts`查看注册的PCI中断。



# 简单的PCI设备驱动

```c
#include <linux/module.h>
#include <linux/pci.h>

/* 私有成员 */
struct pci_card
{
   //端口读写变量
   resource_size_t io;
   long range,flags;
   void __iomem *ioaddr;
   int irq;
};

static struct pci_device_id ids[] = {
	{ PCI_DEVICE(0x8086, 0x100e) },
	{ PCI_DEVICE(0x10ec,0x0001) },
	{ 0, }  //最后一组是0，表示结束
};
MODULE_DEVICE_TABLE(pci, ids);

void pci_info(struct pci_dev *dev)
{
	uint8_t val1;
	uint16_t val2;
	uint32_t val4;

	pci_read_config_word(dev,PCI_VENDOR_ID, &val2);
	printk("vendorID:%x",val2);
	pci_read_config_word(dev,PCI_DEVICE_ID, &val2);
	printk("deviceID:%x",val2);
	pci_read_config_byte(dev, PCI_REVISION_ID, &val1);
	printk("revisionID:%x",val1);
	pci_read_config_dword(dev,PCI_CLASS_REVISION, &val4);
	printk("class:%x",val4);
}
/* 设备中断服务*/
static irqreturn_t mypci_interrupt(int irq, void *dev_id)
{
   struct pci_card *mypci = (struct pci_card *)dev_id;
   printk("irq = %d,mypci_irq = %d\n",irq,mypci->irq);
    
   return IRQ_HANDLED;
}

static int mypci_probe(struct pci_dev *dev, const struct pci_device_id *id)
{
	int retval = 0;
	struct pci_card *mypci;
	printk("mypci_probe func\n"); 
	if(pci_enable_device(dev))
	{
		printk (KERN_ERR "IO Error.\n");
		return -EIO;
	}
	mypci = kmalloc(sizeof(struct pci_card),GFP_KERNEL);
   if(!mypci)
   {
      printk("In %s,kmalloc err!",__func__);
      return -ENOMEM;
   }

   mypci->irq = dev->irq;
   if(mypci->irq < 0)
   {
      printk("IRQ is %d, it's invalid!\n",mypci->irq);
      goto out_mypci;
   }

   mypci->io = pci_resource_start(dev, 0);
   mypci->range = pci_resource_end(dev, 0) - mypci->io + 1;
   mypci->flags = pci_resource_flags(dev,0);
   printk("start %llx %lx %lx\n",mypci->io,mypci->range,mypci->flags);
   printk("PCI base addr 0 is io%s.\n",(mypci->flags & IORESOURCE_MEM)? "mem":"port");

  //retval=request_mem_region(mypci->io,mypci->range, "pci");
   retval = pci_request_regions(dev,"pci");
   if(retval)
   {
      printk("PCI request regions err!\n");
      goto out_mypci;
   }
   mypci->ioaddr = pci_ioremap_bar(dev,0);
   //mypci->ioaddr = ioremap(mypci->io,mypci->range);  这里变量的类型与函数参数的类型必须一致，否则会出错
   if(!mypci->ioaddr)
   {
      printk("ioremap err!\n");
      retval = -ENOMEM;
      goto out_regions;
   }
   //申请中断IRQ并设定中断服务子函数
   retval = request_irq(mypci->irq, mypci_interrupt, IRQF_SHARED, "pci", mypci);
   if(retval)
   {
      printk (KERN_ERR "Can't get assigned IRQ %d.\n",mypci->irq);
      goto out_iounmap;
   }
   pci_set_drvdata(dev,mypci);
   printk("mypci_probe succeeds.PCIE ioport addr start at %llX, mypci->ioaddr is 0x%p,interrupt No. %d.\n",mypci->io,mypci->ioaddr,mypci->irq);
 	pci_info(dev);
   return 0;
  
out_iounmap:
	iounmap(mypci->ioaddr);
out_regions:
	pci_release_regions(dev);
out_mypci:
	kfree(mypci);
    
	return retval;
}

/* 移除PCI设备 */
static void mypci_remove(struct pci_dev *dev)
{
   struct pci_card *mypci = pci_get_drvdata(dev);
   free_irq (mypci->irq, mypci);
   iounmap(mypci->ioaddr);
   //release_mem_region(mypci->io,mypci->range);
   pci_release_regions(dev);
   kfree(mypci);
   pci_disable_device(dev);
   printk("Device is removed successfully.\n");
}

static struct pci_driver pci_driver = {
	.name = "pci",
	.id_table = ids,
	.probe = mypci_probe,
	.remove = mypci_remove,
};

static int __init pci_init(void)
{

	printk("%s start\n",__func__);
	return pci_register_driver(&pci_driver);
}

static void __exit pci_exit(void)
{

	printk("%s start\n",__func__);
	pci_unregister_driver(&pci_driver);
}

MODULE_LICENSE("GPL");

module_init(pci_init);
module_exit(pci_exit);
```

在内核启动的过程中，内核的PCI子系统会进行初始化工作，子系统负责PCI设备的扫描，识别所有PCI设备，并将设备信息保存在内核中。PCI子系统初始化完毕后，会根据当前的PCI设备的厂商ID（Vendor ID）、设备ID（Device ID）对内核中的PCI设备驱动进行加载，并进行初始化，此时，相应PCI设备驱动的`probe`函数便会执行，完成设备驱动的初始化工作，通常在`probe`函数中会完成PCI设备资源空间的初始化、PCI中断的注册，I/O空间的映射等操作，一边内核可以正常的访问PCI设备的I/O及MEM空间。

通常PCI驱动不会单独存在，他只是一种总线，具体的PCI设备可以说字符设备、块设备、网络设备、USB设备等，所以一般PCI驱动均包含两部分：PCI驱动、PCI设备驱动。

---

# 参考

[PCIE 之linux驱动分析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/399102423)

[第 12 章 PCI 驱动-Linux设备驱动第三版（中文版）- - (deansys.com)](http://www.deansys.com/doc/ldd3/ch12.html)

[Linux PCI总线子系统 — The Linux Kernel documentation](https://www.kernel.org/doc/html/next/translations/zh_CN/PCI/)
