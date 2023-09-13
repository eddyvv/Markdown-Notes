# Linux设备驱动依赖配置

若a驱动的部分函数依赖于b驱动，需要在a驱动的makefile中加入 `Module.symvers`符号依赖。

```bash
KBUILD_EXTRA_SYMBOLS = $(PWD)/B/Module.symvers
```



# Linux系统启动自动加载设备驱动

```bash
$ cp aaa.ko /lib/modules/$(uname -r)/kernel/drivers/xxx/xxx
$ depmod -a
```







# 参考

[Linux ubuntu开机自动加载设备驱动_ubuntu insmod_Gaosiy的博客-CSDN博客](https://blog.csdn.net/huiyuanliyan/article/details/120882037)
