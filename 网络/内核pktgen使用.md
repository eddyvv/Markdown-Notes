# 内核pktgen使用

## 什么是pktgen

pktgen 是一款 Linux 发包工具，可在内核高速发包。由于`pktgen`是内核内置的，使用内核空间，所以它可以达到高带宽、高发包速率，以此更好地测试路由器、交换机和其他网络设备。

一旦加载`pktgen`模块后，会自动为每一个CPU创建一个内核线程[^1]，并将该线程绑定在对应的CPU上。与此同时，在`/proc/net/pktgen/`目录下，为每一个线程创建一个`kpktgend_X`文件（其中的X为CPU编号），用来控制和监控这些线程，以及一个`pgctrl`文件，用来控制`pktgen`程序。当卸载`pktgen`模块时，`/proc/net/pktgen`文件夹会被自动删除。

同时，当将某个网卡绑定在某个线程时，也会在`/proc/net/pktgen/`目录下自动创建以该网卡名称为名的文件，用于记录此网卡设备的配置信息和运行信息。

测试程序可以配置成一直运行，或者在发送固定数量的包之后结束。也可以使用快捷键Ctrl+C终止运行。

[^1]:<font color=red>注意</font>：一个设备只能属于一个线程。

## 使能 pktgen

使用`pktgen`需要在内核配置文件开启`CONFIG_NET_PKTGEN`，

内核在执行一下命令后：

```bash
$ sudo modprobe pktgen
```

会在文件夹`/proc/net/`下创建一个`pktgen`文件夹

## 使用pktgen发包

在加载pktgen之后会在`proc/net/pktgen`下创建出如下几个文件：

![image-20230915083323445](image/%E5%86%85%E6%A0%B8pktgen%E4%BD%BF%E7%94%A8/image-20230915083323445.png)

### 添加设备

```bash
$ echo "add_device ens33" > /proc/net/pktgen/kpktgend_0
```

![image-20230915091151661](image/%E5%86%85%E6%A0%B8pktgen%E4%BD%BF%E7%94%A8/image-20230915091151661.png)

### 设置报文

* 部分配置

```bash
$ echo "pkt_size 64" > /proc/net/pktgen/etn
$ echo "count 1000000" > /proc/net/pktgen/eth3
$ echo "dst_mac aa:bb:cc:dd:ee:ff" > /proc/net/pktgen/eth3
```

![image-20230915091238685](image/%E5%86%85%E6%A0%B8pktgen%E4%BD%BF%E7%94%A8/image-20230915091238685.png)

* 查看配置结果

```bash
cat /proc/net/pktgen/ens33
```

![image-20230915091552982](image/%E5%86%85%E6%A0%B8pktgen%E4%BD%BF%E7%94%A8/image-20230915091552982.png)

### 发送报文

```bash
$ echo "start" > /proc/net/pktgen/pgctrl
```

## 查看统计信息

```bash
$ cat /proc/net/pktgen/ens33
```

![image-20230915091751564](image/%E5%86%85%E6%A0%B8pktgen%E4%BD%BF%E7%94%A8/image-20230915091751564.png)

## 配置命令

### 控制命令

| 名称  | 含义               |
| ----- | ------------------ |
| start | 所有的线程开始发送 |
| stop  | 停止               |

### 线程命令

| 名称               | 含义                                     |
| ------------------ | ---------------------------------------- |
| add_device         | 添加某个端口到某个线程                   |
| rem_device_all     | 删除绑定在某个线程的所有端口             |
| max_before_softirq | 在最多发送多少个数据包后执行do_softirq() |

### 端口命令

| 名称  | 含义               |
| ----- | ------------------ |
| count | 发送数据包的个数，0 表示一直发送 |
| clone_skb | 对每个skb进行多少个复制，0表示不复制。对于Dos等测试必须至零 |
| debug | 调试 |
| frags | 数据包的片段数量 |
| delay | 发送两个数据包之间的延时 |
| src_mac_count | 源mac的数量，从src mac设置的mac开始轮询 |
| dst_mac_count | 目的mac的数量，从dst mac设置的mac开始轮询 |
| pkt_size | 链路包的大小（前去CRC的值） |
| min_pkt_size | 数据包最小值 |
| max_pkt_size | 数据包最大值 |
| udp_src_min | 最小源udp端口号 |
| udp_src_max | 最大源udp端口号 |
| udp_dst_min | 最小目的udp端口号 |
| udp_dst_max | 最大目的udp端口号 |
| flag<br />        IPSRC_RND<br />        TXSIZE_RND<br />        IPDST_RND<br />        UDPSRC_RND<br />        UDPDST_RND<br />        MACSRC_RND<br />        MACDST_RND | PSRC_RND 源IP随机发送 |
| dst_min | 目的IP的最小值 |
| dst_max | 目的IP的最大值 |
| src_min | 源IP最小值 |
| src_max | 源IP最大值 |
| dst_mac | 目的mac |
| src_mac | 源mac |
| clear_counters | 清空计数器，一般程序自动清空 |
| dst6 | 目的IPv6地址 |
| src6 | 源IPv6地址 |
| flows | 并发流的个数 |
| flowlen| 流的长度 |




# 参考

[pktgen | 孙勇峰的部落格 (sunyongfeng.com)](http://sunyongfeng.com/201603/networks/pktgen)

[pktgen——the packet generator within linux kernel-CSDN博客](https://blog.csdn.net/Hey___Man/article/details/98215294)

[kernel.org/doc/Documentation/networking/pktgen.txt](https://www.kernel.org/doc/Documentation/networking/pktgen.txt)

[networking:pktgen Wiki (linuxfoundation.org)](https://wiki.linuxfoundation.org/networking/pktgen)

[pktgen: A packet traffic generator in kernel space for testing network throughput. – Julio Faracco (wordpress.com)](https://juliofaracco.wordpress.com/2015/06/14/pktgen-a-kernel-space-traffic-generator-for-testing-network-throughput/)