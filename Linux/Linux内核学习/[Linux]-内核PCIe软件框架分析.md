Title: PCIe总线-Linux内核PCIe软件框架分析（十一）

URL Source: https://blog.csdn.net/u011037593/article/details/140743419

Published Time: 2024-07-27T23:49:23+08:00

Markdown Content:

# PCIe总线-Linux内核PCIe软件框架分析（十一）转

### 1.简介

Linux 内核PCIe软件框架]如下图所示，按照PCIe的模式，可分为RC和EP软件框架。RC的软件框架分为五层，第一层为RC Controller Driver，和RC Controller硬件直接交互，不同的RC Controller，其驱动实现也不相同；第二层为Core层，该层将Controller进行了抽象，提供了统一的接口和数据结构，将所有的Controller管理起来，同时提供通用PCIe设备驱动注册和匹配接口，完成驱动和设备的绑定，管理所有PCIe设备；第三层为PCIe设备驱动层，包含了Storage、Ethernet、PCI桥等设备驱动；第四层为设备驱动层，根据设备类型，可分为字符设备驱动、网络设备驱动和块设备驱动。第五层为虚拟文件系统层，该层会在用户空间创建设备节点，提供了应用程序访问PCIe设备的路径。EP的软件框架分为六层，第一层为EP Controller Driver，和RC Controller Driver的功能相似；第二层为EP Controller Core层，该层向下将EP Controller进行了抽象，提供了统一的接口和数据结构，将所有的EP Controller管理起来；第三层为EP Function Core，该层统一管理EPF驱动和EPF设备，并提供两者相互匹配的方法；第四层为EP Configfs，在用户空间提供了配置和绑定EPF的接口，用户可以通过这些接口配置EPF，而无需修改驱动；第五层为EP Function Driver，和PCIe设备的具体功能相关；第六层为虚拟文件系统层，和RC的功能相同（EP也有设备驱动层，篇幅所限，图中未画出）。

![Image 1: PCIe软件框架](image/[Linux]-内核PCIe软件框架分析/fb52fb465aed42c7b75af8e9560ec6d5.png#pic_center)

### 2.RC软件框架

#### 2.1.RC Controller Driver

RK3588 PCIe RC Controller Driver驱动定义如下所示。

```c
MODULE_DEVICE_TABLE(of, rk_pcie_of_match);
static struct platform_driver rk_plat_pcie_driver = {
	.driver = {
		.name	= "rk-pcie",
		.of_match_table = rk_pcie_of_match,
		.suppress_bind_attrs = true,
		.pm = &rockchip_dw_pcie_pm_ops,
	},
	.probe = rk_pcie_probe,
};

module_platform_driver(rk_plat_pcie_driver);
```

#### 2.2.Core

##### 2.2.1.Host Bridge

RC Core层使用`struct pci_host_bridge`数据结构 描述Host Bridge。`bus`描述Root bus，其他bus都在该数据结构的链表中。`ops`和`child_ops`描述Root bus和其他bus上的设备的配置空间访问方法。`windows`链表保存bus-range和ranges的资源。`dma_ranges`链表保存dma-ranges的资源。使用`pci_alloc_host_bridge`和`devm_pci_alloc_host_bridge`函数分配`struct pci_host_bridge`数据结构，使用`pci_free_host_bridge`释放`struct pci_host_bridge`数据结构。`pci_host_probe`枚举Host Bridge下面所有PCIe设备。

```c
[include/linux/pci.h]
struct pci_host_bridge {
	struct device	dev;
	struct pci_bus	*bus;		/* Root bus */
	struct pci_ops	*ops;       /* Low-level architecture-dependent routines */
	struct pci_ops	*child_ops;
	void		*sysdata;
	int		busnr;
	struct list_head windows;	 /* resource_entry */
	struct list_head dma_ranges; /* dma ranges resource list */
	......
};
struct pci_host_bridge *pci_alloc_host_bridge(size_t priv);
struct pci_host_bridge *devm_pci_alloc_host_bridge(struct device *dev,
	size_t priv);
void pci_free_host_bridge(struct pci_host_bridge *bridge);
int pci_host_probe(struct pci_host_bridge *bridge);
```

`struct pci_ops`描述访问PCIe设备配置空间的方法，需要RC Controller Driver实现。常用的是`map_bus`、`read`和`write`，`map_bus`用于映射访问配置空间的region，`read`和`write`用于读写配置空间。

```c
[include/linux/pci.h]
struct pci_ops {
	int (*add_bus)(struct pci_bus *bus);
	void (*remove_bus)(struct pci_bus *bus);
	void __iomem *(*map_bus)(struct pci_bus *bus, unsigned int devfn, int where);
	int (*read)(struct pci_bus *bus, unsigned int devfn, int where, int size, u32 *val);
	int (*write)(struct pci_bus *bus, unsigned int devfn, int where, int size, u32 val);
};
```

##### 2.2.2.Bus

RC Core层使用`struct pci_bus`数据结构描述PCIe bus。所有PCIe bus组成一个PCIe树型结构。`parent`指向Parent buses，`children`指向Child buses。`devices`链表保存该bus上的所有设备。`number`为该bus的总线编号，`primary`表示上游总线编号，`busn_res`保存桥下游总线编号范围，`max_bus_speed`表示该bus支持的最大速度，`cur_bus_speed`表示该bus当前的速度。`pci_find_bus`根据PCIe域和总线编号查找`struct pci_bus`，`pci_add_new_bus`创建一个`struct pci_bus`并添加到父总线上，注册Host Bridge时会自动创建bus0的数据结构，`pci_bus_insert_busn_res`和`pci_bus_update_busn_res_end`更新PCIe bus编号资源。

```c
[include/linux/pci.h]
struct pci_bus {
	struct list_head node;		/* Node in list of buses */
	struct pci_bus	*parent;	/* Parent bus this bridge is on */
	struct list_head children;	/* List of child buses */
	struct list_head devices;	/* List of devices on this bus */
	struct pci_dev	*self;		/* Bridge device as seen by parent */
	struct list_head slots;		/* List of slots on this bus;
					   protected by pci_slot_mutex */
	struct resource *resource[PCI_BRIDGE_RESOURCE_NUM];
	struct list_head resources;	/* Address space routed to this bus */
	struct resource busn_res;	/* Bus numbers routed to this bus */

	struct pci_ops	*ops;		/* Configuration access functions */
	struct msi_controller *msi;	/* MSI controller */
	void		*sysdata;	/* Hook for sys-specific extension */
	struct proc_dir_entry *procdir;	/* Directory entry in /proc/bus/pci */

	unsigned char	number;		/* Bus number */
	unsigned char	primary;	/* Number of primary bridge */
	unsigned char	max_bus_speed;	/* enum pci_bus_speed */
	unsigned char	cur_bus_speed;	/* enum pci_bus_speed */
	......
};

struct pci_bus *pci_find_bus(int domain, int busnr);
struct pci_bus *pci_add_new_bus(struct pci_bus *parent,
	struct pci_dev *dev, int busnr);
void pci_remove_bus(struct pci_bus *bus);

int pci_bus_insert_busn_res(struct pci_bus *b, int bus, int busmax);
int pci_bus_update_busn_res_end(struct pci_bus *b, int busmax);
```

##### 2.2.3.Device

RC Core层使用`struct pci_dev`数据结构描述PCIe Devices。`devfn`表述device和function编号，`vendor`、`device`等保存PCIe设备配置空间头信息，`driver`指向该设备使用的驱动。`resource`保存设备的资源，如BAR、ROMs等。PCIe bus也是一个PCIe设备。`pci_alloc_dev`分配`struct pci_dev`数据结构，`pci_dev_put`释放`struct pci_dev`数据结构，`pci_device_add`向总线上添加PCIe设备。`pci_bus_add_devices`和`pci_bus_add_device`匹配PCIe设备和PCIe驱动。

```c
[include/linux/pci.h]
/* The pci_dev structure describes PCI devices */
struct pci_dev {
	struct list_head bus_list;	/* Node in per-bus list */
	struct pci_bus	*bus;		/* Bus this device is on */
	struct pci_bus	*subordinate;	/* Bus this device bridges to */

	void		*sysdata;	/* Hook for sys-specific extension */
	struct proc_dir_entry *procent;	/* Device entry in /proc/bus/pci */
	struct pci_slot	*slot;		/* Physical slot this device is in */

	unsigned int	devfn;		/* Encoded device & function index */
	unsigned short	vendor;
	unsigned short	device;
	unsigned short	subsystem_vendor;
	unsigned short	subsystem_device;
	unsigned int	class;		/* 3 bytes: (base,sub,prog-if) */
	......
	struct pci_driver *driver;	/* Driver bound to this device */
	......
	int		cfg_size;		/* Size of config space */
	/*
	 * Instead of touching interrupt line and base address registers
	 * directly, use the values stored here. They might be different!
	 */
	unsigned int	irq;
	struct resource resource[DEVICE_COUNT_RESOURCE]; /* I/O and memory regions + expansion ROMs */
	bool		match_driver;		/* Skip attaching driver */
	......
};

struct pci_dev *pci_alloc_dev(struct pci_bus *bus);
void pci_dev_put(struct pci_dev *dev);
void pci_device_add(struct pci_dev *dev, struct pci_bus *bus);
void pci_bus_add_device(struct pci_dev *dev);
void pci_bus_add_devices(const struct pci_bus *bus);
```

##### 2.2.4.Driver

RC Core层使用`struct pci_driver`数据结构描述PCIe设备驱动。PCIe设备和驱动匹配的信息保存到`id_table`中。`pci_register_driver`注册PCIe设备驱动，`pci_unregister_driver`注销PCIe设备驱动。

```c
[include/linux/pci.h]
struct pci_driver {
	struct list_head	node;
	const char		*name;
	/* Must be non-NULL for probe to be called */
	const struct pci_device_id *id_table;
	/* New device inserted */
	int  (*probe)(struct pci_dev *dev, const struct pci_device_id *id);
	/* Device removed (NULL if not a hot-plug capable driver) */
	void (*remove)(struct pci_dev *dev);
	/* Device suspended */
	int  (*suspend)(struct pci_dev *dev, pm_message_t state);
	/* Device woken up */
	int  (*resume)(struct pci_dev *dev);
	void (*shutdown)(struct pci_dev *dev);
	/* On PF */
	int  (*sriov_configure)(struct pci_dev *dev, int num_vfs);
	/*  */
	const struct pci_error_handlers *err_handler;
	......
};
/* pci_register_driver() must be a macro so KBUILD_MODNAME can be expanded */
#define pci_register_driver(driver)		\
	__pci_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)

void pci_unregister_driver(struct pci_driver *dev);
```

`pci_bus_type`用于匹配PCIe设备和驱动。

```c
struct bus_type pci_bus_type = {
	.name		= "pci",
	.match		= pci_bus_match,
	.uevent		= pci_uevent,
	.probe		= pci_device_probe,
	.remove		= pci_device_remove,
	.shutdown	= pci_device_shutdown,
	.dev_groups	= pci_dev_groups,
	.bus_groups	= pci_bus_groups,
	.drv_groups	= pci_drv_groups,
	.pm		= PCI_PM_OPS_PTR,
	.num_vf		= pci_bus_num_vf,
	.dma_configure	= pci_dma_configure,
};
```

##### 2.2.5.设备驱动

不同的PCIe设备，需要不同的PCIe设备驱动。下面列出PCIe桥和NVMe硬盘驱动。

###### 2.2.5.1.桥驱动

如下所示，PCIe桥使用`"pcieport"`驱动。

```c
[drivers/pci/pcie/portdrv_pci.c]
static const struct pci_device_id port_pci_ids[] = {
	/* handle any PCI-Express port */
	{ PCI_DEVICE_CLASS(((PCI_CLASS_BRIDGE_PCI << 8) | 0x00), ~0) },
	/* subtractive decode PCI-to-PCI bridge, class type is 060401h */
	{ PCI_DEVICE_CLASS(((PCI_CLASS_BRIDGE_PCI << 8) | 0x01), ~0) },
	/* handle any Root Complex Event Collector */
	{ PCI_DEVICE_CLASS(((PCI_CLASS_SYSTEM_RCEC << 8) | 0x00), ~0) },
	{ },
};
static struct pci_driver pcie_portdriver = {
	.name		= "pcieport",
	.id_table	= &port_pci_ids[0],

	.probe		= pcie_portdrv_probe,
	.remove		= pcie_portdrv_remove,
	.shutdown	= pcie_portdrv_remove,

	.err_handler	= &pcie_portdrv_err_handler,

	.driver.pm	= PCIE_PORTDRV_PM_OPS,
};
static int __init pcie_portdrv_init(void)
{
	if (pcie_ports_disabled)
		return -EACCES;

	pcie_init_services();
	dmi_check_system(pcie_portdrv_dmi_table);

	return pci_register_driver(&pcie_portdriver);
}
device_initcall(pcie_portdrv_init);
```

###### 2.2.5.2.NVMe驱动

M.2 NVMe硬盘使用下面的驱动。

```c
[drivers/nvme/host/pci.c]
static struct pci_driver nvme_driver = {
	.name		= "nvme",
	.id_table	= nvme_id_table,
	.probe		= nvme_probe,
	.remove		= nvme_remove,
	.shutdown	= nvme_shutdown,
#ifdef CONFIG_PM_SLEEP
	.driver		= {
		.pm	= &nvme_dev_pm_ops,
	},
#endif
	.sriov_configure = pci_sriov_configure_simple,
	.err_handler	= &nvme_err_handler,
};

static int __init nvme_init(void)
{
	BUILD_BUG_ON(sizeof(struct nvme_create_cq) != 64);
	BUILD_BUG_ON(sizeof(struct nvme_create_sq) != 64);
	BUILD_BUG_ON(sizeof(struct nvme_delete_queue) != 64);
	BUILD_BUG_ON(IRQ_AFFINITY_MAX_SETS < 2);

	return pci_register_driver(&nvme_driver);
}
module_init(nvme_init);
module_exit(nvme_exit);
```

### 3.EP软件框架

#### 3.1.EP Controller Driver

RK3399 PCIe EP Controller Driverr驱动定义如下所示。

```c
[drivers/pci/controller/pcie-rockchip-ep.c]
static const struct of_device_id rockchip_pcie_ep_of_match[] = {
	{ .compatible = "rockchip,rk3399-pcie-ep"},
	{},
};

static struct platform_driver rockchip_pcie_ep_driver = {
	.driver = {
		.name = "rockchip-pcie-ep",
		.of_match_table = rockchip_pcie_ep_of_match,
	},
	.probe = rockchip_pcie_ep_probe,
};

builtin_platform_driver(rockchip_pcie_ep_driver);
```

#### 3.2.EP Controller Core

##### 3.2.1.EPC Device

EP Controller Core层使用`struct pci_epc`描述PCIe Endpoint Controller Device。EPC的所有的functions都挂到`pci_epf`链表上，`ops`指向了EPC提供的回调函数集合，用于设置EPC的配置空间、设置region、设置和发送中断等，`windows`保存了EPC的Outbound的地址段，`num_windows`表示Outbound的地址段的数量，`max_functions`保存了functions的最大数量。使用`pci_epc_create`和`devm_pci_epc_create`函数创建`struct pci_epc`，`devm_pci_epc_destroy`和`pci_epc_destroy`销毁`struct pci_epc`。

```c
[include/linux/pci-epc.h]
/* struct pci_epc - represents the PCI EPC device */
struct pci_epc {
	struct device			dev;
	struct list_head		pci_epf;
	const struct pci_epc_ops	*ops;
	struct pci_epc_mem		**windows;
	struct pci_epc_mem		*mem;
	unsigned int			num_windows;
	u8				max_functions;
	struct config_group		*group;
	/* mutex to protect against concurrent access of EP controller */
	struct mutex			lock;
	unsigned long			function_num_map;
	struct atomic_notifier_head	notifier;
};

#define pci_epc_create(dev, ops)    \
		__pci_epc_create((dev), (ops), THIS_MODULE)
#define devm_pci_epc_create(dev, ops)    \
		__devm_pci_epc_create((dev), (ops), THIS_MODULE)
void devm_pci_epc_destroy(struct device *dev, struct pci_epc *epc);
void pci_epc_destroy(struct pci_epc *epc);
```

`struct pci_epc_ops`如下图所示，这些回调函数很重要，EP Controller Driver必须实现。EPF驱动会调用这些函数配置EPC。

```c
[include/linux/pci-epc.h]
struct pci_epc_ops {
	int	(*write_header)(struct pci_epc *epc, u8 func_no,
				struct pci_epf_header *hdr);
	int	(*set_bar)(struct pci_epc *epc, u8 func_no,
			   struct pci_epf_bar *epf_bar);
	void	(*clear_bar)(struct pci_epc *epc, u8 func_no,
			     struct pci_epf_bar *epf_bar);
	int	(*map_addr)(struct pci_epc *epc, u8 func_no,
			    phys_addr_t addr, u64 pci_addr, size_t size);
	void	(*unmap_addr)(struct pci_epc *epc, u8 func_no,
			      phys_addr_t addr);
	int	(*set_msi)(struct pci_epc *epc, u8 func_no, u8 interrupts);
	int	(*get_msi)(struct pci_epc *epc, u8 func_no);
	int	(*set_msix)(struct pci_epc *epc, u8 func_no, u16 interrupts,
			    enum pci_barno, u32 offset);
	int	(*get_msix)(struct pci_epc *epc, u8 func_no);
	int	(*raise_irq)(struct pci_epc *epc, u8 func_no,
			     enum pci_epc_irq_type type, u16 interrupt_num);
	int	(*start)(struct pci_epc *epc);
	void	(*stop)(struct pci_epc *epc);
	const struct pci_epc_features* (*get_features)(struct pci_epc *epc,
						       u8 func_no);
	struct module *owner;
};
```

##### 3.2.2.EPF绑定EPC

每个EP的Function都对应一个`struct pci_epf`设备，即EPF设备，EPF设备和EPC通过`pci_epc_add_epf`绑定，通过`pci_epc_remove_epf`解除绑定。

```c
[include/linux/pci-epc.h]
int pci_epc_add_epf(struct pci_epc *epc, struct pci_epf *epf);
void pci_epc_remove_epf(struct pci_epc *epc, struct pci_epf *epf);
```

##### 3.2.3.EPC API

下面的接口是对`struct pci_epc_ops`封装，供EPF驱动调用。

```c
[include/linux/pci-epc.h]
int pci_epc_write_header(struct pci_epc *epc, u8 func_no,
			 struct pci_epf_header *hdr);
int pci_epc_set_bar(struct pci_epc *epc, u8 func_no,
		    struct pci_epf_bar *epf_bar);
void pci_epc_clear_bar(struct pci_epc *epc, u8 func_no,
		       struct pci_epf_bar *epf_bar);
int pci_epc_map_addr(struct pci_epc *epc, u8 func_no,
		     phys_addr_t phys_addr,
		     u64 pci_addr, size_t size);
void pci_epc_unmap_addr(struct pci_epc *epc, u8 func_no,
			phys_addr_t phys_addr);
int pci_epc_set_msi(struct pci_epc *epc, u8 func_no, u8 interrupts);
int pci_epc_get_msi(struct pci_epc *epc, u8 func_no);
int pci_epc_set_msix(struct pci_epc *epc, u8 func_no, u16 interrupts,
		     enum pci_barno, u32 offset);
int pci_epc_get_msix(struct pci_epc *epc, u8 func_no);
int pci_epc_raise_irq(struct pci_epc *epc, u8 func_no,
		      enum pci_epc_irq_type type, u16 interrupt_num);
int pci_epc_start(struct pci_epc *epc);
void pci_epc_stop(struct pci_epc *epc);
const struct pci_epc_features *pci_epc_get_features(struct pci_epc *epc,
						    u8 func_no);
```

#### 3.3.EP Function Core

EP Function Core层定义了EPF Driver和EPF Device的数据结构，并提供注册、创建及绑定接口。

##### 3.3.1.EPF Driver

EPF Driver的数据结构为`struct pci_epf_driver`。当EPF Device和EPC Device绑定后，会回调`ops`函数以通知EPF Driver，`id_table`定义EPF Driver和EPF Device匹配的信息。`pci_epf_register_driver`注册EPF Driver，`pci_epf_unregister_driver`注销EPF Driver。

```c
[include/linux/pci-epf.h]
struct pci_epf_driver {
	int	(*probe)(struct pci_epf *epf);
	int	(*remove)(struct pci_epf *epf);

	struct device_driver	driver;
	struct pci_epf_ops	*ops;
	struct module		*owner;
	struct list_head	epf_group;
	const struct pci_epf_device_id	*id_table;
};
struct pci_epf_ops {
	int	(*bind)(struct pci_epf *epf);
	void	(*unbind)(struct pci_epf *epf);
};

#define pci_epf_register_driver(driver)    \
		__pci_epf_register_driver((driver), THIS_MODULE)
void pci_epf_unregister_driver(struct pci_epf_driver *driver);
```

##### 3.3.2.EPF Device

每个EP Function都对应一个EPF Device。EPF Device的数据结构为`struct pci_epf_driver`。`header`保存了该EP Function配置空间头信息，`bar[6]`保存了6个BAR映射的物理地址，`msi_interrupts`和`msix_interrupts`分别表示EP Function需要的中断数量，`func_no`表述EP Function的编号。`pci_epf_create`和`pci_epf_destroy`创建和销毁EPF Device。

```c
[include/linux/pci-epf.h]
struct pci_epf {
	struct device		dev;
	const char		*name;
	struct pci_epf_header	*header;
	struct pci_epf_bar	bar[6];
	u8			msi_interrupts;
	u16			msix_interrupts;
	u8			func_no;

	struct pci_epc		*epc;
	struct pci_epf_driver	*driver;
	struct list_head	list;
	struct notifier_block   nb;
	/* mutex to protect against concurrent access of pci_epf_ops */
	struct mutex		lock;
};
struct pci_epf *pci_epf_create(const char *name);
void pci_epf_destroy(struct pci_epf *epf);
```

##### 3.3.3.EPF Device匹配EPF Driver

`pci_epf_bus_type`用于匹配EPF Device和EPF Driver。

```c
[drivers/pci/endpoint/pci-epf-core.c]
static struct bus_type pci_epf_bus_type = {
	.name		= "pci-epf",
	.match		= pci_epf_device_match,
	.probe		= pci_epf_device_probe,
	.remove		= pci_epf_device_remove,
};
```

#### 3.4.EP Configfs

EP Configfs会在/sys目录下创建文件节点，使用者可以在用户空间通过这些文件节点，配置和创建EPP Device，绑定EPP Device、EPP Driver及EPC Device。

#### 3.5.EP Function Driver。

下面是pci_epf_test的EP Function Driver。

```c
[drivers/pci/endpoint/functions/pci-epf-test.c]
static struct pci_epf_ops ops = {
	.unbind	= pci_epf_test_unbind,
	.bind	= pci_epf_test_bind,
};

static struct pci_epf_driver test_driver = {
	.driver.name	= "pci_epf_test",
	.probe		= pci_epf_test_probe,
	.id_table	= pci_epf_test_ids,
	.ops		= &ops,
	.owner		= THIS_MODULE,
};
```

### 参考资料

1.   PCIEXPRESS体系结构导读
2.   PCI Express technology 3.0
3.   PCI Express® Base Specification Revision 5.0 Version 1.0
4.   Rockchip RK3588 TRM
5.   Linux kernel 5.10