# preftest测试

perftest是一组基于uverbs编写的测试程序，是RDMA性能相关的micro-benchmark。可用于软硬件调优以及功能测试。

![image-20221206111828373](image/perftest%E6%B5%8B%E8%AF%95/image-20221206111828373.png)

## 源码位置

[linux-rdma/perftest: Infiniband Verbs Performance Tests (github.com)](https://github.com/linux-rdma/perftest)

## 安装

直接安装preftest

```bash
sudo apt install perftest
```

## Send带宽测试

服务器端

```bash
ib_send_bw -d rxe_0
```

![image-20221104095535331](image/perftest%E6%B5%8B%E8%AF%95/image-20221104095535331.png)

客户端

```bash
ib_send_bw -d rxe_0 <server_ip>
```

![image-20221104095554364](image/perftest%E6%B5%8B%E8%AF%95/image-20221104095554364.png)

ib_send_bw是用来测试RDMA Send操作的带宽的程序（infiniband_send*bandwidth），其中*<server_ip>表示服务器端的IP，根据自己虚拟机对应关系填写。

可以看到BW peak为测试的峰值带宽信息。

```bash
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 MSG大小     迭代次数         峰值速率             平均速率               包转发率
```

## Send延迟测试

服务器端

```bash
ib_send_lat -d rxe_0
```

客户端

```bash
ib_send_lat -d rxe_0 <server_ip>
```

![image-20221111090818758](image/perftest%E6%B5%8B%E8%AF%95/image-20221111090818758.png)

ib_send_lat是用来测试RDMA Send操作的延迟的程序（infiniband_send*bandwidth），其中*<server_ip>表示服务器端的IP，根据自己虚拟机对应关系填写。

```bash
 #bytes #iterations    t_min[usec]    t_max[usec]  t_typical[usec]    t_avg[usec]    t_stdev[usec]   99% percentile[usec]   99.9% percentile[usec]
  MSG大小  迭代次数        延迟最小值      延迟最大值     典型值               
```

更改MTU大小

```bash
sudo ifconfig ens33 mtu 4200
```

将网卡端口MTU设置为1500，active_mtu将更改为1024

将网卡端口MTU设置为2200，active_mtu将更改为2048

将网卡端口MTU设置为4200，active_mtu将更改为4096

或指定配置项

```bash
ib_send_bw -R -d rxe0 -i 1 -a -s 1M -m 512
```

```bash
ib_send_bw -R -d rxe0 -i 1 192.168.159.131 -a -s 1M -m 512
```

设置QP数量

```bash
ib_send_bw -R -d rxe0 -i 1 -a -s 1M -m 512 -q 2
```

```bash
ib_send_bw -R -d rxe0 -i 1 192.168.159.131 -a -s 1M -m 512 -q 2
```



# 参考

[linux-rdma/perftest: Infiniband Verbs Performance Tests (github.com)](https://github.com/linux-rdma/perftest)

[在25G以太网环境下使用Perftest对RoCEv2性能进行测试 | 生命不息 折腾不止 (sysu.tech)](http://blog.sysu.tech/Benchmark/在25G以太网环境下使用Perftest对RoCEv2性能进行测试/)

[RDMA测试工具perftest简介 - 墨天轮 (modb.pro)](https://www.modb.pro/db/485278)

[监视并排除 IB 设备故障 - Oracle Solaris 管理：设备和文件系统](https://docs.oracle.com/cd/E26926_01/html/E25884/gjwwf.html)Oracle IB相关命令

