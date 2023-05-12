# 安装RDMA IB网卡驱动指令

```bash
sudo apt-get install build-essential libelf-dev cmake
sudo apt-get install libibverbs1 libibverbs-dev librdmacm1 librdmacm-dev rdmacm-utils ibverbs-utils
sudo modprobe ib_core
sudo modprobe rdma_ucm
```

