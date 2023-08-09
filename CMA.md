# cma_alloc: alloc failed

```bash
[   69.494324][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 8 pages, ret: -12
[   69.504897][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 16 pages, ret: -12
[   69.512576][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 8 pages, ret: -12
[   69.522832][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 16 pages, ret: -12
[   69.533160][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 8 pages, ret: -12
[   69.542897][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 16 pages, ret: -12
[   69.553301][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 8 pages, ret: -12
[   69.563268][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 16 pages, ret: -12
[   69.571260][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 8 pages, ret: -12
[   69.578911][ 5] [  T887] cma: cma_alloc: alloc failed, req-size: 16 pages, ret: -12

```

```bash
$ while true ; do grep CmaFree /proc/meminfo ; sleep 1 ; done
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
CmaFree:              16 kB
```

```bash
$ sudo cat /proc/meminfo
MemTotal:       32817604 kB
MemFree:        29805828 kB
MemAvailable:   30963704 kB
PageSize:              4 kB
Buffers:           18120 kB
Cached:          1415560 kB
SwapCached:            0 kB
Active:           440808 kB
Inactive:        1306448 kB
Active(anon):     314228 kB
Inactive(anon):      528 kB
Active(file):     126580 kB
Inactive(file):  1305920 kB
Unevictable:          16 kB
Mlocked:              16 kB
SwapTotal:      39378940 kB
SwapFree:       39378940 kB
Dirty:               512 kB
Writeback:             0 kB
AnonPages:        309800 kB
Mapped:           204604 kB
Shmem:              1148 kB
KReclaimable:      68640 kB
Slab:            1059060 kB
SReclaimable:      68640 kB
SUnreclaim:       990420 kB
KernelStack:        5824 kB
PageTables:         4956 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    55787740 kB
Committed_AS:    1682256 kB
VmallocTotal:   262930368 kB
VmallocUsed:       27056 kB
VmallocChunk:          0 kB
Percpu:            12352 kB
HardwareCorrupted:     0 kB
AnonHugePages:    124928 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
CmaTotal:          32768 kB
CmaFree:              16 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB

```

解决方法

```bash
sudo vim /etc/default/grub
```

在`GRUB_CMDLINE_LINUX_DEFAULT`后添加`cma=128M`，具体的分配大小根据自己的机器决定。

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash cma=128M"
```

