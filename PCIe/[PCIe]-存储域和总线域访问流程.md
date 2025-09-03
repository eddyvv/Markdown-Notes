# 存储域和总线域访问流程



## 跨域访问

通常情况下，存储器域指的是CPU能直接访问的地址空间，包括了CPU寄存器、DRAM、PCIe控制器和PCIe窗口地址空间。PCIe总线域指由PCIe设备所能直接访问的地址空间组成，包括PCIe桥设备和PCIe Swtich，若由多条PCIe总线，则对应多个PCIe总线域。

* **Outbound**: 存储器域访问PCIe总线域称为Outbound传输，从RC或EP的角度看，也可以称为发送。

* **Inbound**: PCIe总线域访问存储器域称为Inbound传输，从RC或EP的角度看，也可以称为接收。
* **Outbound ATU（address translation unit）**: Outbound地址转换单元，负责将存储器域地址转换成PCIe总线域地址。
* **Inbound ATU（address translation unit）**: Inbound地址转换单元，负责将PCIe总线域地址转换成存储器域地址。



# 参考



[PCIe总线-存储器域和PCIe总线域访问流程分析（二）](https://blog.csdn.net/u011037593/article/details/137697527)

[pcie inbound、outbound及EP、RC间的互相訪问](https://blog.csdn.net/vc66vcc/article/details/103525878)

[从CPU角度理解PCIe续集](http://www.ssdfans.com/?p=106756)