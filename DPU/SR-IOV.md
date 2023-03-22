# SR-IOV

&emsp;&emsp;SR-IOV 是技术是一种基于硬件的虚拟化解决方案，可提高性能和可伸缩性。SR-IOV标准允许在虚拟机之间高效共享PCIe（Peripheral Component Interconnect Express，快速外设组件互连）设备，由于采用硬件实现，所以可以获得能够与本机性能相媲美的I/O性能。SR-IOV规范定义了新的标准，根据该标准，创建的新设备可允许将虚拟机直接连接到I/O设备。

SR-IOV是通过改变物理设备硬件设计的方式来实现硬件设备的透传以及虚拟机共享的目的，同时也建立了一套基于PF->VF的标准，SR-IOV能够为虚拟机提供独立的内存空间、中断、DMA流来绕过[VMM](#VMM)实现数据访问。

# SR-IOV原理

## SR-IOV硬件实现

### SR-IOV基本结构

&emsp;&emsp;SR-IOV是在PCIe规范的基础上实现的，SR-IOV协议引入了两种类型功能的概念叫做PCIe Function

* PF（Physical Function）

&emsp;&emsp;包括SR-IOV的扩张能力，支持SR-IOV功能的PCI功能，如SR-IOV规范中定义，PF包含SR-IOV功能配置结构体，用于管理SR-IOV功能。PF是全功能的PCIe功能，可以像其他任何PCIe设备一样进行发现、管理和处理。PF拥有完全配置资源，可以用于配置或控制PCIe设备。

* VF（Virtual Function）

&emsp;&emsp;与物理功能关联的一种功能，VF包含轻量级的PCIe功能，相当于一个非常轻量级的虚拟网络设备，每一个VF又它自己独享的PCI配置区域，并且可能与其他VF共享同一个物理资源，VF仅允许拥有用于其自身行为的配置资源；

![1667024176064](image/SR-IOV/1667024176064.png)

<center>SR-IOV基本结构图</center>

《[Single Root I/O Virtualization and Sharing Specification Revision 1.1.pdf](https://composter.com.ua/documents/sr-iov1_1_20Jan10_cb.pdf)》

&emsp;&emsp;每个SR-IOV设备都可有一个物理功能PF，并且每个PF最多可有64000个与其相关联的虚拟功能VF，PF可以通过寄存器创建VF，这些寄存器设计有专用于此的属性。一旦在PF中启用了SR-IOV，就可以通过PF的总线、设备和功能编号（路由ID）访问各个VF的PCI配置空间，每个VF都具有一个PCI内存空间，用于映射其寄存器集。VF设备驱动程序对寄存器集进行操作以启用其功能，并且显示为实际存在的PCI设备。创建VF后，可以直接将其指定给IO来宾域或各个应用；

![1666945740139](image/SR-IOV/1666945740139.png)

<center>SR-IOV</center>

&emsp;&emsp;传统虚拟化系统使用 [Hypervisor](https://zh.wikipedia.org/zh-hans/Hypervisor)（或者KVM）软件对虚拟机进行管理，软件层既消耗CPU资源，又有较深的调用栈，使得PCIe设备的性能优势无法彻底发挥。而SR-IOV可以实现多个虚拟机共享物理资源，且Bypass Hypervisor（或者VMM）软件层，使得虚拟机可以使用到NVMe SSD的高性能。

&emsp;&emsp;对比传统虚拟化系统，再有SR-IOV技术的云业务虚拟机应用场景下，VM可直接与VF通信，不需要Hypervisor接入IO处理，节约vCPU资源的同时，又可以实现不同VF之间的

![1667007054460](image/SR-IOV/1667007054460.png)

**SR-IOV**优势：

* 实现SR-IOV之后，VMM把中断交给虚拟机处理，提高I/O性能；
* 虚拟机直接和PCIe设备交互，Bypass掉中间的Hypervisor/VMM层，减轻物理主机CPU负担，使之有能力承载更多虚拟机；
* SR-IOV虚拟化技术可以减少客户所需PCIe设备数量，进而节省PCIe插槽；
* SR-IOV可以与其他的I/O虚拟化技术进行结合，提供一个更加完整的兼具高性能和安全性的解决方案；

&emsp;&emsp;虚拟化分区方式场景下，通过SR-IOV技术挂载SSD方式如下图所示：

![1667021855626](image/SR-IOV/1667021855626.png)





# 相关词汇

HPA（Host Physical Address）宿主机物理内存地址；

GPA（Guest Physical Address）虚拟机访问的物理地址；

[IOMMU](https://zhuanlan.zhihu.com/p/348826888)（Input/Output Memory Mangement Uint）输入输出内存管理单元；

`<a id="VMM"/>`VMM（Virtual Machine Manage）虚拟机管理器
