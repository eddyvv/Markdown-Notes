Title: PCIe总线-Linux内核PCIe设备枚举流程分析（十三）

URL Source: https://blog.csdn.net/u011037593/article/details/142031554

Published Time: 2024-09-08T19:39:21+08:00

Markdown Content:
PCIe总线-Linux内核PCIe设备枚举流程分析（十三）_pci 总线secondary subordinate-CSDN博客

===============
PCIe总线-Linux内核PCIe设备枚举流程分析（十三）转
==============================

## 1.简介

当系统启动时或者有新的PCIe设备接入时，PCIe主机会扫描[PCIe总线](https://so.csdn.net/so/search?q=PCIe%E6%80%BB%E7%BA%BF&spm=1001.2101.3001.7020)上的PCIe设备，读取设备配置空间信息，建立设备的拓扑关系，然后为设备分配资源（如内存空间、I/O空间、中断、总线编号等），最后根据设备的类型匹配驱动。下面以 Linux 内核为例，介绍PCIe主机枚举PCIe设备的流程。

## 2.pci_host_probe

Linux内核中，PCIe RC驱动调用`pci_host_probe`函数枚举PCIe总线上的所有设备。`pci_host_probe`的执行流程如下图所示，总共有5部分，具体如下：

1.   调用`pci_scan_root_bus_bridge`函数扫描总线上的桥和设备。 
     1.   首先注册Host主桥。主桥上有root bus，root bus的编号通常是0。然后设置msi_controller和msi_domain，用于MSI/MSI-X中断。最后将root bus的数据结构挂到`pci_root_buses`上，便于统一管理。
     2.   调用`pci_scan_child_bus`函数扫描设备，后面详细介绍。

2.   确定连接到PCI总线上的PCI桥（包括标准PCI桥和PCI到PCIe桥）所需的地址空间大小，并分配相应的内存和I/O资源。最后，调整桥设备的子总线号，以确保正确的总线层次结构和设备访问路径。
3.   给总线上的设备和桥分配资源，即设置配置空间的BAR、BASE、Limit等寄存器，后面详细介绍。
4.   遍历PCIe总线上设备和桥，设置MPS、MRRS，可以参考[PCIe Linux MRRS和MPS参数设置策略](https://blog.csdn.net/u011037593/article/details/141305819?spm=1001.2014.3001.5502)。
5.   遍历PCIe总线上设备和桥，添加设备，调用PCIe总线提供的匹配方法匹配PCIe设备驱动。

![Image 18: pci_host_probe](image/[Linux]-内核PCIe设备枚举流程分析/e679ed45b5aa41078f0c7253eaf186e3.png#pic_center)

## 3.pci_scan_child_bus

Linux内核调用`pci_scan_child_bus`函数，从root bus开始枚举PCIe总线上的所有设备。整个枚举流程分为三步，具体如下：

1.   第一步：首先从root bus开始扫描。PCIe设备和功能号组合总共有256种，因此需要全部遍历一遍，devfn高5位是设备号，低3位是功能号，每次循坏扫描一个设备，默认扫描Fun0，如果设备是多功能设备，则会扫描其他Function。扫描设备的单个Function使用`pci_scan_single_device`函数，后面详细介绍。
2.   第二步：若bus下面挂的桥被BIOS配置过，则扫描桥下面的设备，如此递归，直到扫面完整个总线，若桥没有被配置过或需要重新配置，则先跳过，在第三步重新配置。对于被BIOS配置过的桥，其Primary、Secondary、Subordinate已经设置，内核会创建`pci_bus`，为其预留bus编号，同时将bus资源管理起来。
3.   第三步：遍历总线上的所有桥，设置所有没有配置的桥，即设置桥的Primary、Secondary、Subordinate，同时创建`pci_bus`，为其预留bus编号，同时将bus资源管理起来。

![Image 19: pci_scan_child_bus](image/[Linux]-内核PCIe设备枚举流程分析/b213e79a88f64d2292aeb4de8662663d.png#pic_center)

## 4.pci_scan_single_device

内核调用`pci_scan_single_device`函数扫描设备的单个Function，先扫描Function，再添加Function。执行的流程如下：

1.   内核通过读取Function的VID来判断设备是否存在，超时时间为60000毫秒。若读取到的VID是0xffffffff、0x00000000、0x0000ffff、0xffff0000，则表示没有设备，若读取到的是0x1，说明返回了CRS的completion包，设备还没准备好，延时循环读取PCI_VENDOR_ID(等待设备准备好)，若读到CRS的completion包，则延时翻倍。
2.   扫描到Function，则分配`pci_dev`数据结构，每个Function对对应一个`pci_dev`数据结构。
3.   初始化Function（设备）。 
    1.   设置Function的`bus_type`为`pci_bus_type`，用于匹配和初始化PCIe设备驱动。
    2.   读取Function的port类型，通过PCI Express Capability registers获取，主要是确定Function是upstream还是downstream ports。
    3.   获取Function的配置空间大小。通过读偏移地址为256处的值，若读到有效的扩展空间头则为4096字节，否则为256字节。
    4.   解析非桥设备的PCIe配置空间头。具体有`PCI_INTERRUPT_PIN`、`PCI_BASE_ADDRESS`、`PCI_SUBSYSTEM_VENDOR_ID`、`PCI_SUBSYSTEM_ID`。`PCI_BASE_ADDRESS`会读取Function的BAR，详细的后面说明。
    5.   解析桥设备的PCIe配置空间头。具体有`PCI_INTERRUPT_PIN`、`PCI_BASE_ADDRESS`、`PCI_EXP_SLTCAP`、`PCI_CAP_ID_SSVID`、`PCI_SSVID_VENDOR_ID`和桥的地址空间。

4.   配置、初始化和管理Function。主要是配置MPS、Extended Tags、ordering、LTR等，初始化MSI、PM、VPD、IOV、ATS、AER等，将`pci_dev`挂到`pci_bus`的`devices`链表，最后建立MSI IRQ domain。

![Image 20: pci_scan_single_device](image/[Linux]-内核PCIe设备枚举流程分析/d5ca809077c042ab98eae212b6136328.png#pic_center)

### 4.1.解析BAR

内核使用`pci_read_bases`函数解析Function的BAR和ROM，包括桥的BAR0和BAR1，非桥设备的BAR0-BAR5。BAR的使用方法参考3.3节[PCIe总线-配置空间介绍（三）](https://blog.csdn.net/u011037593/article/details/138016148?csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22138016148%22%2C%22source%22%3A%22u011037593%22%7D)。`pci_read_bases`函数的执行流程如下所示，主要有：

1.   先读BAR，读出的全为F，表示BAR不正常，意味着设备可能已经被移除，不再存在于总线上，或者设备已经进入了一种不能响应读操作的错误或低功耗状态。
2.   向BAR写入0xFFFFFFFF，向ROM写入0xFFFF8000，然后再度BAR和ROM，以确定BAR和ROM的属性和大小。
3.   将第一次读的BAR和ROM的默认值写回，恢复默认值。
4.   根据第一次读取BAR的值，解析BAR的类型和属性。具体有IO空间`IORESOURCE_IO`，32位非预取存储器空间`IORESOURCE_MEM`、64位非预取存储器空间`IORESOURCE_MEM_64`（虽然BAR是64位，但实际上RC只会给BAR配置32位PCIe总线地址），预取存储器空间`IORESOURCE_PREFETCH`。内核会根据解析的BAR信息分配PCIe总线地址。
5.   如果是64位BAR，则还需要解析高32位的BAR，步骤和第4步一样。
6.   计算BAR所需地址空间大小。
7.   将PCIe总线地址region转换成存储域地址resource，将存储域地址resource转换成PCIe总线地址Region，两者的起始地址必须相等。

![Image 21: pci_read_bases](image/[Linux]-内核PCIe设备枚举流程分析/bb466f63a6164789aff04abb97426f6f.png#pic_center)

### 4.2.解析桥 windows

内核使用`pci_read_bridge_windows`函数解析桥设备的`PCI_IO_BASE`、`PCI_PREF_MEMORY_BASE`、`PCI_PREF_BASE_UPPER32`。若`PCI_IO_BASE`寄存器 存在，则设置`bridge->io_window = 1`。若`PCI_PREF_MEMORY_BASE`寄存器存在，则设置`bridge->pref_window = 1`。若支持64位预取地址，且`PCI_PREF_BASE_UPPER32`寄存器存在，则设置`bridge->pref_64_window = 1`。`PCI_MEMORY_BASE`寄存器必须实现，这里可不用解析，后面直接分配地址。

```c
[drivers/pci/probe.c]
static void pci_read_bridge_windows(struct pci_dev *bridge)
{
	u16 io;
	u32 pmem, tmp;

	pci_read_config_word(bridge, PCI_IO_BASE, &io);
	if (!io) {
		pci_write_config_word(bridge, PCI_IO_BASE, 0xe0f0);
		pci_read_config_word(bridge, PCI_IO_BASE, &io);
		pci_write_config_word(bridge, PCI_IO_BASE, 0x0);
	}
	if (io)
		bridge->io_window = 1;

	/*
	 * DECchip 21050 pass 2 errata: the bridge may miss an address
	 * disconnect boundary by one PCI data phase.  Workaround: do not
	 * use prefetching on this device.
	 */
	if (bridge->vendor == PCI_VENDOR_ID_DEC && bridge->device == 0x0001)
		return;

	pci_read_config_dword(bridge, PCI_PREF_MEMORY_BASE, &pmem);
	if (!pmem) {
		pci_write_config_dword(bridge, PCI_PREF_MEMORY_BASE,
					       0xffe0fff0);
		pci_read_config_dword(bridge, PCI_PREF_MEMORY_BASE, &pmem);
		pci_write_config_dword(bridge, PCI_PREF_MEMORY_BASE, 0x0);
	}
	if (!pmem)
		return;

	bridge->pref_window = 1;

	if ((pmem & PCI_PREF_RANGE_TYPE_MASK) == PCI_PREF_RANGE_TYPE_64) {

		/*
		 * Bridge claims to have a 64-bit prefetchable memory
		 * window; verify that the upper bits are actually
		 * writable.
		 */
		pci_read_config_dword(bridge, PCI_PREF_BASE_UPPER32, &pmem);
		pci_write_config_dword(bridge, PCI_PREF_BASE_UPPER32,
				       0xffffffff);
		pci_read_config_dword(bridge, PCI_PREF_BASE_UPPER32, &tmp);
		pci_write_config_dword(bridge, PCI_PREF_BASE_UPPER32, pmem);
		if (tmp)
			bridge->pref_64_window = 1;
	}
}
```

## 5.分配资源（地址）

内核使用`pci_bus_assign_resources`函数给PCIe桥和非桥设备分配资源。详细的流程如下所示。针对每一条子总线，递归调用`__pci_bus_assign_resources`分配资源。一条子总线上的设备和桥分配资源的步骤如下：

1.   遍历子总线上的所有设备，给设备需要的资源排序。
2.   遍历子总线上设备需要的资源，调用`pci_assign_resource`函数分配资源，最终会调用到`allocate_resource`函数，先查找资源，再请求资源。这里的资源是PCIe总线地址。
3.   分配好资源后，调用`pci_update_resource`函数将资源配置到设备的BAR或ROM中，同时回读BAR和ROM进行校验。将PCIe总线地址设置到BAR或者ROM寄存器当中。
4.   如果是桥设备，则调用`pci_setup_bridge`初始化桥。桥的`IORESOURCE_IO`、`IORESOURCE_MEM`、`IORESOURCE_PREFETCH`资源范围，是桥下面所有设备相同资源类型之和。

![Image 23: pci_bus_assign_resources](image/[Linux]-内核PCIe设备枚举流程分析/ca47623d4d3f4dcfbd67994852a56844.png#pic_center)

## 参考资料

1.   PCIEXPRESS体系结构导读
2.   PCI Express technology 3.0
3.   PCI Express® Base Specification Revision 5.0 Version 1.0
4.   Rockchip RK3588 TRM
5.   Linux kernel 5.10