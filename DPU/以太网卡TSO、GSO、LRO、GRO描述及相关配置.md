# 以太网卡TSO、GSO、LRO、GRO描述及相关配置

## 硬件包拆分与合并
### TSO（TCP Segmentation Offload ）

TSO针对TCP在发送端网卡上的拆包

### UFO（UDP Fragmentation Offload ）

已弃用UFO [segmentation-offloads](https://www.kernel.org/doc/Documentation/networking/segmentation-offloads.txt)

![image-20221205151526276](image/Bluefiled2%E9%85%8D%E7%BD%AETSO%E3%80%81GSO/image-20221205151526276.png)

UFO针对UDP在发送端网卡上的拆包

### LRO（Large Receive Offload ）

LRO针对TCP在接收端网卡的组包

TSO、UFO、LRO基于网卡特性，可在网卡上进行包的合并与拆分，减轻CPU的负荷。

## 软件包拆分与合并

### GSO（Generic Segmentation Offload）

软件实现包拆分，若网卡不支持分片、重组offload能力（如TSO、UFO、LRO）的情况下，GSO推迟数据分片直至数据发送到网卡驱动之前进行分片后再发往网卡。

拆包过程：首先查询网卡是否支持 TSO 功能，如果硬件支持 TSO 则使用网卡的硬件分片能力执行分片；如果网卡不支持 TSO 功能，则将分片的执行，延缓到了将数据推送到网卡的前一刻执行。

| TSO  | GSO  | 分片阶段          |
| ---- | ---- | ----------------- |
| off  | on   | GSO、网卡驱动阶段 |
| on   | on   | TSO、网卡硬件阶段 |

### GRO（Generic Receive Offload）

软件实现包合并，GRO将大量的小报文合并为少量的大报文，再将合并后的大报文提交给OS协议栈处理。

同时GSO、GRO不仅支持TCP和UDP包，还可支持vxlan和gre。

| LRO  | GRO  | 阶段         |
| ---- | ---- | ------------ |
| off  | off  | TCP阶段      |
| on   | off  | 网卡硬件阶段 |
| off  | on   | 网卡驱动阶段 |

## Bluefiled2网卡配置TSO、GSO、LRO、GRO

安装`ethtool`工具

```bash
sudo apt install ethtool
```

查询TSO

```bash
sudo ethtool -k eth<x>
```

eg

```bash
sudo ethtool -k enps4f0np0
```

设置TSO

```bash
sudo ethtool -K eth<x> [rx on|off] [tx on|off] [sg on|off] [tso on|off] [lro on|off]
```

eg

```bash
sudo ethtool -K enps4f0np0 tso on
```

`注意`：ethtool命令的相关配置不可持续，在计算机下电后会被清除；

# 参考

[Ethernet Driver Usage and Configuration - MLNX_EN v5.8-1.0.1.1 LTS - NVIDIA Networking Docs](https://docs.nvidia.com/networking/display/MLNXENv581011/Ethernet+Driver+Usage+and+Configuration) 迈络思网卡配置手册。

[关于网卡特性TSO、UFO、GSO、LRO、GRO_51CTO博客_网卡的特性](https://blog.51cto.com/u_15333820/3459534)

[网卡TSO、UFO、GSO、LRO、GRO和RSS介绍_小雨的网络文档的技术博客_51CTO博客](https://blog.51cto.com/shutdownsky/1441761)

[SmartNIC — TSO、GSO、LRO、GRO 技术_范桂飓的博客-CSDN博客](https://blog.csdn.net/Jmilk/article/details/118858113)

[TSO和GSO_测试里的小虾米的技术博客_51CTO博客](https://blog.51cto.com/u_14009921/5757723)

[TSO、UFO、GSO、LRO、GRO和RSS介绍-飞鸿无痕-ChinaUnix博客](http://blog.chinaunix.net/uid-20639775-id-3529535.html)

[网络虚拟化中的 offload 技术：LSO/LRO、GSO/GRO、TSO/UFO、VXLAN_yeasy的博客-CSDN博客](https://blog.csdn.net/yeasy/article/details/19204639)

[https://www.kernel.org/doc/Documentation/networking/segmentation-offloads.txt](https://www.kernel.org/doc/Documentation/networking/segmentation-offloads.txt) Linux内核offload的描述

[Linux GRO流程分析_gro linux_zhenghuaduo的博客-CSDN博客](https://blog.csdn.net/zgy666/article/details/106989856) 内核GRO流程分析。
