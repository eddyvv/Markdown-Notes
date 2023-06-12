# PCIe设备枚举与资源分配

`Linux kernel`支持两种PCI扫描方式：`ACPI`的方式和`非ACPI`的方式(Legacy mode)。系统默认使用ACPI的方式枚举PCIE/PCI设备。如果用户想使用非ACPI的方式枚举PCIE/PCI设备，可以通过在kernel 启动参数中加入`pci=noacpi`。

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

