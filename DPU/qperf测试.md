# qperf测试


qperf 可以用来测试两个节点之间的带宽（bandwidth）和延迟（latency），不仅仅可以用来测试 TCP/IP 协议的性能指标，还可以用来测试 RDMA 传输的指标。使用方法是：一个节点运行 qperf 作为服务端，另一个节点则运行 qperf 作为客户端，与服务端建立连接之后打流，获取带宽和延迟等数据。

为了使用 RoCE 运行 qperf，应该在客户端添加 -cm1 标志。(mellonx)

GitHub源码

[https://github.com/linux-rdma/qperf.git](https://github.com/linux-rdma/qperf.git)



## 安装

```bash
sudo apt install qperf
```

## 测试

服务端

```bash
qperf
```

客户端

send/receive

```bash
qperf -cm1 192.168.159.131 rc_lat
qperf -cm1 192.168.159.131 rc_bw
```

write/read

```bash
qperf -cm1 192.168.159.131 rc_rdma_write_lat
qperf -cm1 192.168.159.131 rc_rdma_write_bw
```

数据包size： -m

```bash
qperf  -cm1 172.17.31.51 -m 1M rc_bw
```

数据包数量： -n

```bash
qperf  -cm1 172.17.31.51 -m 1M -n 1000 rc_bw
```

# 参考

[qperf - 手册页部分 1： 用户命令 (oracle.com)](https://docs.oracle.com/cd/E56344_01/html/E54075/qperf-1.html)

[qperf 网络性能测试工具_爱吃李子的十豆三的博客-CSDN博客_qperf测试](https://blog.csdn.net/u014516643/article/details/124803638)

[网络测试qperf_code_____monkey的博客-CSDN博客_qperf测试](https://blog.csdn.net/yonghutwo/article/details/123662592?ops_request_misc=%7B%22request%5Fid%22%3A%22166808120516782388022633%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=166808120516782388022633&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123662592-null-null.142^v63^control,201^v3^add_ask,213^v2^t3_esquery_v3&utm_term=网络测试qperf&spm=1018.2226.3001.4187)

[Measuring Network Performance in Linux with qperf - OpsDash](https://www.opsdash.com/blog/network-performance-linux.html)

[Measuring Network Performance in Linux with qperf - OpsDash](https://www.opsdash.com/blog/network-performance-linux.html)

[qperf(1): Measure RDMA/IP performance - Linux man page (die.net)](https://linux.die.net/man/1/qperf)

[网络性能测试工具qperf使用_51CTO博客_网络测试工具](https://blog.51cto.com/u_15333820/3464257)

[7.3. 配置 IPoIB 后使用 qperf 测试 RDMA 网络 Red Hat Enterprise Linux 8 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/configuring_infiniband_and_rdma_networks/testing-an-rdma-network-using-qperf-after-ipoib-is-configured_testing-infiniband-networks)

[linux-rdma/qperf (github.com)](https://github.com/linux-rdma/qperf)



