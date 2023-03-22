# qemu-kvm VirtIO设备模拟及初始化流程

```c
main
    ->qemu_init
    	->qemu_create_late_backends
    		->net_init_clients	/* 初始化网络设备 */
    			->qemu_opts_foreach(qemu_find_opts("netdev"), net_init_netdev, NULL,&error_fatal)
    				->net_init_netdev
    					->net_client_init
    						->net_client_init1
    							->net_client_init_fun[netdev->type](netdev, netdev->id, peer, errp)	/* 根据传入的参数类型再 net_client_init_fun 选择相应函数处理 */
    	->qmp_x_exit_preconfig
			->qemu_create_cli_devices
    			->qemu_opts_foreach(qemu_find_opts("device"),	/* 根据需要虚拟化的设备的数量for循环执行 */
                      device_init_func, NULL, &error_fatal); ->device_init_func
device_init_func
    ->qdev_device_add
    	->qdev_device_add
    		->qdev_device_add_from_qdict
    			->qdev_get_device_class
    				->module_object_class_by_name
    					->type_initialize
    						->type_initialize_interface	/* class_init 赋值 */
    							->ti->class_init(ti->class, ti->class_data)    /* 根据对应的驱动类型调用相应的 class_init 函数初始化类 */
    			->qdev_new
    				->DEVICE(object_new(name))
    					->object_new_with_type
    						->object_initialize_with_type
    							->object_init_with_type
    								->ti->instance_init(obj)	/* 创建 virtio-xx 设备实例初始化对象 */
    			->qdev_realize
    				->object_property_set_bool
    					->object_property_set_qobject
    						->object_property_set
    							->prop->set(obj, v, name, prop->opaque, errp)	/* realize 调用 */
```

## Virtio设备的创建

使用 `qemu-kvm` 创建虚拟机的过程中，需要指定 `-device` 参数，之后再 `qemu` 的 `main` 函数中解析 `-device` 调用 `device_init_func` 来对相应的设备进行初始化。

首先从QEMU的命令行入手，创建一个使用virtio设备的虚拟机，可使用如下命令行：

```c
gdb --args ./x86_64-softmmu/qemu-system-x86_64 \
    -machine accel=kvm -cpu host -smp sockets=2,cores=2,threads=1 -m 3072M \
    -object memory-backend-file,id=mem,size=3072M,mem-path=/dev/hugepages,share=on \
    -hda /home/kvm/disk/vm0.img -mem-prealloc -numa node,memdev=mem \
    -vnc 0.0.0.0:00 -monitor stdio --enable-kvm \
    -netdev type=tap,id=eth0,ifname=tap30,script=no,downscript=no 
    -device e1000,netdev=eth0,mac=12:03:04:05:06:08 \
    -chardev socket,id=char1,path=/tmp/vhostsock0,server \
    -netdev type=vhost-user,id=mynet3,chardev=char1,vhostforce,queues=$QNUM 
    -device virtio-net-pci,netdev=mynet3,id=net1,mac=00:00:00:00:00:03,disable-legacy=on
```

其中，通过 `-device` 来实现虚拟设备的创建，上面的命令行中创建了一个 `virtio-net-pci` 设备：

```c
-device virtio-net-pci,netdev=mynet3,id=net1,mac=00:00:00:00:00:03,disable-legacy=on
```

`virtio-net-pci` 函数的调用过程如下图

![img](image/VirtIO%E8%AE%BE%E5%A4%87%E6%A8%A1%E6%8B%9F%E5%8F%8A%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B5%81%E7%A8%8B/ce3527a00a45c70a21ae57110bfffae7.png)

> 图片引用自[Linux虚拟化KVM-Qemu分析之Virtio设备-51CTO.COM](https://www.51cto.com/article/645696.html)

## 参数解析

QEMU的命令行解析在main函数进行，解析后按照qemu标准格式存储到本地。然后通过 `qemu_find_opts` 接口可以获取本地结构体中具有相应关键字的所有命令列表，对解析后的命令列表使用 `qemu_opts_foreach` 依次执行处理函数。

---

# virtio 设备初始化流程

kernel 在启动初始化阶段，pci 子系统调用 `pci_scan_device` 发现 pci 网卡设备，并初始化对应 `pci_dev` 结构，然后注册到 pci 总线上，设置 device 的 `vendor_id` 为 `0x1AF4` （virtio 的 pcivendor_id）

加载 virtio-pci 驱动时,调用 `module_pci_driver(virtio_pci_driver)` 将 virtio-pci 驱动注册在 pci 总线上时，在 linux 设备驱动模型中，这会导致对 pci 总线设备链表上未被驱动绑定的每个设备调用 pci 总线的 match 回调函数，即 `pci_bus_match` 函数。原型如下：

`源码位置`[/drivers/pci/pci-driver.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/pci/pci-driver.c)

```c
static int pci_bus_match(struct device *dev, struct device_driver *drv)
{
    /* 转换为pci_dev */
	struct pci_dev *pci_dev = to_pci_dev(dev);
	struct pci_driver *pci_drv;
	const struct pci_device_id *found_id;

	if (!pci_dev->match_driver)
		return 0;

    /* 转换为pci_driver */
	pci_drv = to_pci_driver(drv);
	found_id = pci_match_device(pci_drv, pci_dev);
	if (found_id)
		return 1;

	return 0;
}
```

`pci_bus_match` 函数将 linux 设备驱动模型核心的 `device` 结构转换为 `pci_dev` 结构，将 `device_driver` 结构转换为 `pci_driver` 结构，之后调用 `pci_match_device` 函数判断 pci 设备结构是否有匹配的 pci 设备 ID 结构。

`源码位置`[/drivers/pci/pci-driver.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/pci/pci-driver.c)

```c
static const struct pci_device_id *pci_match_device(struct pci_driver *drv,
						    struct pci_dev *dev)
{
	struct pci_dynid *dynid;
	const struct pci_device_id *found_id = NULL, *ids;

	/* When driver_override is set, only bind to the matching driver */
	if (dev->driver_override && strcmp(dev->driver_override, drv->name))
		return NULL;

	/* Look at the dynamic ids first, before the static ones */
	spin_lock(&drv->dynids.lock);
	list_for_each_entry(dynid, &drv->dynids.list, node) {
		if (pci_match_one_device(&dynid->id, dev)) {
			found_id = &dynid->id;
			break;
		}
	}
	spin_unlock(&drv->dynids.lock);

	if (found_id)
		return found_id;

	for (ids = drv->id_table; (found_id = pci_match_id(ids, dev));
	     ids = found_id + 1) {
		/*
		 * The match table is split based on driver_override.
		 * In case override_only was set, enforce driver_override
		 * matching.
		 */
		if (found_id->override_only) {
			if (dev->driver_override)
				return found_id;
		} else {
			return found_id;
		}
	}

	/* driver_override will always match, send a dummy id */
	if (dev->driver_override)
		return &pci_device_id_any;
	return NULL;
}
```

如果有则判断设备的 PCI ID 和驱动设置的 id_table 中是否一样，如果一样说明设备和驱动匹配（这里设备的vendor_id 和 virtio-pci 的 virtio_pci_id_table 匹配），将 struct device 的 driver 指针指向驱动，然后调用pci 总线的 probe 函数，即 `pci_deivce_probe` 函数。

`源码位置`[/drivers/pci/pci-driver.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/pci/pci-driver.c)

```c
static int pci_device_probe(struct device *dev)
{
	int error;
	struct pci_dev *pci_dev = to_pci_dev(dev);
	struct pci_driver *drv = to_pci_driver(dev->driver);

	if (!pci_device_can_probe(pci_dev))
		return -ENODEV;

	pci_assign_irq(pci_dev);

	error = pcibios_alloc_irq(pci_dev);
	if (error < 0)
		return error;

	pci_dev_get(pci_dev);
	error = __pci_device_probe(drv, pci_dev);
	if (error) {
		pcibios_free_irq(pci_dev);
		pci_dev_put(pci_dev);
	}

	return error;
}

```

函数再次将 `struct device` 强制转换成 `struct pci_dev` ，将设置在设备中的 `driver` 结构强制转换为 `struct pci_derver` 。它再次校验这个驱动能否支持这个设备，递增设备的引用计数，然后调用 pci 驱动 probe 函数（即 virtio-pci 的 probe 函数 virtio_pci_probe），传入它应该绑定到的 `struct pci_dev` 结构体指针。

`源码位置`[/drivers/virtio/virtio_pci_common.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio_pci_common.c)

```c
static int virtio_pci_probe(struct pci_dev *pci_dev,
			    const struct pci_device_id *id)
{
	struct virtio_pci_device *vp_dev, *reg_dev = NULL;
	int rc;

	/* allocate our structure and fill it out */
	vp_dev = kzalloc(sizeof(struct virtio_pci_device), GFP_KERNEL);
	if (!vp_dev)
		return -ENOMEM;

	pci_set_drvdata(pci_dev, vp_dev);
	vp_dev->vdev.dev.parent = &pci_dev->dev;
	vp_dev->vdev.dev.release = virtio_pci_release_dev;
	vp_dev->pci_dev = pci_dev;
	INIT_LIST_HEAD(&vp_dev->virtqueues);
	spin_lock_init(&vp_dev->lock);

	/* enable the device */
	rc = pci_enable_device(pci_dev);
	if (rc)
		goto err_enable_device;

	if (force_legacy) {
		rc = virtio_pci_legacy_probe(vp_dev);
		/* Also try modern mode if we can't map BAR0 (no IO space). */
		if (rc == -ENODEV || rc == -ENOMEM)
			rc = virtio_pci_modern_probe(vp_dev);
		if (rc)
			goto err_probe;
	} else {
		rc = virtio_pci_modern_probe(vp_dev);
		if (rc == -ENODEV)
			rc = virtio_pci_legacy_probe(vp_dev);
		if (rc)
			goto err_probe;
	}

	pci_set_master(pci_dev);

	vp_dev->is_legacy = vp_dev->ldev.ioaddr ? true : false;

    /* 将virtio_device的设备总线设置为virtio总线 */
	rc = register_virtio_device(&vp_dev->vdev);
	reg_dev = vp_dev;
	if (rc)
		goto err_register;

	return 0;

err_register:
	if (vp_dev->is_legacy)
		virtio_pci_legacy_remove(vp_dev);
	else
		virtio_pci_modern_remove(vp_dev);
err_probe:
	pci_disable_device(pci_dev);
err_enable_device:
	if (reg_dev)
		put_device(&vp_dev->vdev.dev);
	else
		kfree(vp_dev);
	return rc;
}
```

`virtio_pci_probe` 函数完成 `pci_dev` 部分的初始化，已经 `virtio_device` 部分初始化，然后调用 `register_virtio_device` 函数。

`函数位置`[/drivers/virtio/virtio.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio.c)

```c
int register_virtio_device(struct virtio_device *dev)
{
	int err;

	dev->dev.bus = &virtio_bus;
	device_initialize(&dev->dev);

	/* Assign a unique device index and hence name. */
	err = ida_alloc(&virtio_index_ida, GFP_KERNEL);
	if (err < 0)
		goto out;

	dev->index = err;
	err = dev_set_name(&dev->dev, "virtio%u", dev->index);
	if (err)
		goto out_ida_remove;

	err = virtio_device_of_init(dev);
	if (err)
		goto out_ida_remove;

	spin_lock_init(&dev->config_lock);
	dev->config_enabled = false;
	dev->config_change_pending = false;

	INIT_LIST_HEAD(&dev->vqs);
	spin_lock_init(&dev->vqs_list_lock);

	/* We always start by resetting the device, in case a previous
	 * driver messed it up.  This also tests that code path a little. */
	virtio_reset_device(dev);

	/* Acknowledge that we've seen the device. */
	virtio_add_status(dev, VIRTIO_CONFIG_S_ACKNOWLEDGE);

	/*
	 * device_add() causes the bus infrastructure to look for a matching
	 * driver.
	 */
	err = device_add(&dev->dev);
	if (err)
		goto out_of_node_put;

	return 0;

out_of_node_put:
	of_node_put(dev->dev.of_node);
out_ida_remove:
	ida_free(&virtio_index_ida, dev->index);
out:
	virtio_add_status(dev, VIRTIO_CONFIG_S_FAILED);
	return err;
}
EXPORT_SYMBOL_GPL(register_virtio_device);
```

`register_virtio_device` 函数将 `virtio_device` 的设备总线设置为 virtio 总线，然后调用 `device_register` 将 `virtio_device` 对应的设备添加到 virtio 总线上。这个添加总线的动作，会触发 virtio 总线的 match 函数即 `virtio_dev_match` 调用，同样该函数会比较设备 dev 的 pci id 和驱动 id（virtionet 的devid为1），如果匹配则 virtio bus 的 probe 函数 `virtio_dev_probe` 将被调用。其中又会调用对应驱动的 probe 函数，例如 `virtnet_probe`。而 `virtnet_probe` 将会完成 virtio net 设备 struct virtio_device 剩余部分的初始化。

`函数位置`[/drivers/virtio/virtio.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio.c)

```c
static int virtio_dev_match(struct device *_dv, struct device_driver *_dr)
{
	unsigned int i;
	struct virtio_device *dev = dev_to_virtio(_dv);
	const struct virtio_device_id *ids;

	ids = drv_to_virtio(_dr)->id_table;
	for (i = 0; ids[i].device; i++)
		if (virtio_id_match(dev, &ids[i]))
			return 1;
	return 0;
}
```

`函数位置`[/drivers/virtio/virtio.c](https://github1s.com/torvalds/linux/blob/HEAD/drivers/virtio/virtio.c)

```c
static int virtio_dev_probe(struct device *_d)
{
	int err, i;
	struct virtio_device *dev = dev_to_virtio(_d);
	struct virtio_driver *drv = drv_to_virtio(dev->dev.driver);
	u64 device_features;
	u64 driver_features;
	u64 driver_features_legacy;

	/* We have a driver! */
	virtio_add_status(dev, VIRTIO_CONFIG_S_DRIVER);

	/* Figure out what features the device supports. */
	device_features = dev->config->get_features(dev);

	/* Figure out what features the driver supports. */
	driver_features = 0;
	for (i = 0; i < drv->feature_table_size; i++) {
		unsigned int f = drv->feature_table[i];
		BUG_ON(f >= 64);
		driver_features |= (1ULL << f);
	}

	/* Some drivers have a separate feature table for virtio v1.0 */
	if (drv->feature_table_legacy) {
		driver_features_legacy = 0;
		for (i = 0; i < drv->feature_table_size_legacy; i++) {
			unsigned int f = drv->feature_table_legacy[i];
			BUG_ON(f >= 64);
			driver_features_legacy |= (1ULL << f);
		}
	} else {
		driver_features_legacy = driver_features;
	}

	if (device_features & (1ULL << VIRTIO_F_VERSION_1))
		dev->features = driver_features & device_features;
	else
		dev->features = driver_features_legacy & device_features;

	/* Transport features always preserved to pass to finalize_features. */
	for (i = VIRTIO_TRANSPORT_F_START; i < VIRTIO_TRANSPORT_F_END; i++)
		if (device_features & (1ULL << i))
			__virtio_set_bit(dev, i);

	err = dev->config->finalize_features(dev);
	if (err)
		goto err;

	if (drv->validate) {
		u64 features = dev->features;

		err = drv->validate(dev);
		if (err)
			goto err;

		/* Did validation change any features? Then write them again. */
		if (features != dev->features) {
			err = dev->config->finalize_features(dev);
			if (err)
				goto err;
		}
	}

	err = virtio_features_ok(dev);
	if (err)
		goto err;

    /* 调用对应驱动的probe函数 */
	err = drv->probe(dev);
	if (err)
		goto err;

	/* If probe didn't do it, mark device DRIVER_OK ourselves. */
	if (!(dev->config->get_status(dev) & VIRTIO_CONFIG_S_DRIVER_OK))
		virtio_device_ready(dev);

	if (drv->scan)
		drv->scan(dev);

	virtio_config_enable(dev);

	return 0;
err:
	virtio_add_status(dev, VIRTIO_CONFIG_S_FAILED);
	return err;

}
```

![image-20221228164902903](image/VirtIO%E8%AE%BE%E5%A4%87%E6%A8%A1%E6%8B%9F%E5%8F%8A%E5%88%9D%E5%A7%8B%E5%8C%96%E6%B5%81%E7%A8%8B/image-20221228164902903.png)

> 图片引用自[linux设备中virtio组织关系及设备初始化调用流程 - 嵌入式技术 - 今日大瓜 - infinigo.com!](https://dagua.infinigo.com/portal.php?mod=view&aid=2193)

# 参考

[qemu-kvm virtio 虚拟化-----Linux客户机 virtio设备初始化_zou128865的博客-CSDN博客](https://blog.csdn.net/zou128865/article/details/42486145)

[VIRTIO后端框架QEMU与VHOST分析_light_forest的博客-CSDN博客_virtio-net-pci](https://blog.csdn.net/leiyanjie8995/article/details/122503619#:~:text=1、VIRTIO设备创建流程 1 1、virtio-net-pci设备命令行 首先从QEMU的命令行入手，创建一个使用virtio设备的虚拟机，可使用如下命令行： gdb --args .%2Fx86_64-softmmu%2Fqemu-system-x86_64 \,... 3 3、设备实例初始化 在qdev_device_add函数中，首先会调用object_new，创建object（object是所有instance实例的根结构），最终是通过调用每个virtio-pci-net相应DeviceClass里的instance_init创建实例。 ... 4 4、virtio-net-pci设备realize流程 )

[virtio netdev的创建（基于kernel 3.10.0; qemu 2.0.0）_leoufung的博客-CSDN博客](https://blog.csdn.net/leoufung/article/details/48781137)

[virtio简介（四）—— 从零实现一个virtio设备 - Edver - 博客园 (cnblogs.com)](https://www.cnblogs.com/edver/p/15874178.html)

[linux设备中virtio组织关系及设备初始化调用流程 - 嵌入式技术 - 今日大瓜 - infinigo.com!](https://dagua.infinigo.com/portal.php?mod=view&aid=2193)

[转载\]qemu-kvm virtio 虚拟化-----Linux客户机 virtio设备初始化 - 圣哥 - 博客园 (cnblogs.com)](https://www.cnblogs.com/gesaint/p/4118525.html)

[virtio简介（三） —— virtio-balloon qemu设备创建 - Edver - 博客园 (cnblogs.com)](https://www.cnblogs.com/edver/p/14684117.html)

[virtio的工作流程——qemu中virtio-backend初始化(1) | 随便写写 (hanbaoying.com)](https://www.hanbaoying.com/2017/03/28/virtio-work-flow-1.html)

[孙雷: 虚拟化之——virtio-net基础篇 (qq.com)](https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247502660&idx=1&sn=69e50d7b05ba167e5dfd275f33dfae9c&source=41#wechat_redirect)

[虚拟化 qemu-kvm中的virtio浅析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/543205638)

[KVM 介绍（3）：I/O 全虚拟化和准虚拟化 - 墨天轮 (modb.pro)](https://www.modb.pro/db/382083)

[qemu-kvm网络前后端feature协商 (cdmana.com)](https://cdmana.com/2022/04/202204071015230170.html)



---



😥

🎭

🥟
