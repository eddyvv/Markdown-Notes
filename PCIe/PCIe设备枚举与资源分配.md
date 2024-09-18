# PCIe设备枚举与资源分配

`Linux kernel`支持两种PCI扫描方式：`ACPI`的方式和`非ACPI`的方式(Legacy mode)。系统默认使用ACPI的方式枚举PCIE/PCI设备。如果用户想使用非ACPI的方式枚举PCIE/PCI设备，可以通过在kernel 启动参数中加入`pci=noacpi`。

## 枚举步骤

PCI枚举是个不断递归调用发现新设备的过程，PCI枚举简单来说主要包括下面几个步骤：

* 利用深度优先算法遍历整个PCI设备树。从Root Complex出发，寻找设备和桥。发现桥后设置Bus,会发现一个PCI设备子树，递归回到A)；
* 递归的过程中通过读取BARs，记录所有MMIO和IO的需求情况并予以满足。；
* 设置必要的Capabilities；

上述过程结束后，一颗完整的资源分配树建立完成。

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

[PCIE枚举和资源分配 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/621287008)
