# PCIe设备枚举与资源分配

`Linux kernel`支持两种PCI扫描方式：`ACPI`的方式和`非ACPI`的方式(Legacy mode)。系统默认使用ACPI的方式枚举PCIE/PCI设备。如果用户想使用非ACPI的方式枚举PCIE/PCI设备，可以通过在kernel 启动参数中加入`pci=noacpi`。

## 枚举步骤

PCI枚举是个不断递归调用发现新设备的过程，PCI枚举简单来说主要包括下面几个步骤：

* 利用深度优先算法遍历整个PCI设备树。从Root Complex出发，寻找设备和桥。发现桥后设置Bus,会发现一个PCI设备子树，递归回到A)；
* 递归的过程中通过读取BARs，记录所有MMIO和IO的需求情况并予以满足。；
* 设置必要的Capabilities；

上述过程结束后，一颗完整的资源分配树建立完成。

详细设备枚举过程一般分为以下几步

1. **设备扫描**：当Host主机启动时，PCIe主机控制器开始对PCIe总线进行扫描，以识别连接到PCIe总线上的所有设备；
2. **设备检测**：如果检测到一个设备，主机控制器会为该设备分配一个唯一的设备号并初始化该设备；
3. **设备初始化**：如果监测到一个设备，主机控制器会为该设备分配唯一的设备号，并初始化该设备，其中可能包括对某些寄存器的初始化以及为设备分配资源，如内存地址以及中断。
4. **功能级枚举**：如果PCIe设备支持多个功能，系统将为每个功能分配唯一的功能号，每个功能号都有自己的一组相关寄存器。
5. **设备空间访问**：系统通过配置事务（Configuration Transactions）访问PCIe设备的配置空间。配置空间包含了设备的重要信息，如设备ID、厂商ID、BAR（Base Address Registers）、中断信息等。
6. **BAR分配**：基地址寄存器（Base Address Registers，BARs）用于告诉系统如何寻址设备的内存和I/O资源。系统将分配适当的地址范围给BAR，确保设备能够在地址空间中正确寻址其资源。
7. **中断分配**： 系统可能会为设备分配中断向量，以便设备能够通过中断通知主机系统关于特定事件的发生。
8. **驱动加载**： 一旦PCIe设备被枚举和初始化，系统将加载适当的驱动程序，以便与设备进行通信和控制。

## UEFI对PCI/PCIe的支持

UEFI对于PCI总线的支持包括：

1. 提供分配PCI资源的协议；
2. 提供访问PCI设备的协议；
3. 提供PCI枚举器，枚举PCI总线上的设备以及分配设备所需的资源；
4. 提供各种Lib，方便驱动程序访问PCI/PCIe配置空间或者MMIO/IO空间。

[uboot PCI相关命令](../uboot/Uboot命令.md#pci命令)

## ACPI方式枚举PCIe/PCI设备



```c
acpi_init
	-->acpi_scan_init
	    -->acpi_pci_root_init
    		-->acpi_scan_add_handler_with_hotplug(&pci_root_handler, "pci_root");
				-->struct acpi_scan_handler pci_root_handler
                    -->acpi_pci_root_add
                    	-->pci_acpi_scan_root
                    		-->acpi_pci_root_create
                    			-->pci_create_root_bus
                    			-->pci_scan_child_bus
    	-->acpi_pci_link_init
```



# 参考

[PCIe学习笔记——4.设备的枚举过程](https://zhuanlan.zhihu.com/p/4993687465)
