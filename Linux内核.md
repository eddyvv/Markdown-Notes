



![Linux_kernel_map](image/Linux%E5%86%85%E6%A0%B8/Linux_kernel_map.png)

<center>Linux Kernel Map</center>

![1200px-Simplified_Structure_of_the_Linux_Kernel.svg](image/Linux%E5%86%85%E6%A0%B8/1200px-Simplified_Structure_of_the_Linux_Kernel.svg.png)



# KConfig文件

主要用来完成config条目界面，一般用来控制一个条件/目录的编译动作。

## config条目

* prompt：提示信息，显示在配置界面中的信息。

* 变量类型：
  * bool - 二值型变量（y/n）
  * tristate - 三值型变量（y/m/n）
  * string - 字符串变量
  * hex - 十六进制变量
  * int - 整型变量
* default：config条目的默认值
* depends on：依赖项，表示当前条目必须在依赖的项选中后方能出现
* select：饭想依赖，当前项被选中，反向依赖的项目自动被选中
* help：帮助信息

## menu条目

表示一个菜单

以menu开头，endmenu结束，中间的内容即是菜单的内容。

![image-20230308161616689](image/Linux%E5%86%85%E6%A0%B8/image-20230308161616689.png)

## source条目

表示需要去调用其他位置的Kconfig文件

## choice条目

多选一选项，以choice开始，endchoice结束，中间的内容即是多选一选项中的内容。

![image-20230308163907590](image/Linux%E5%86%85%E6%A0%B8/image-20230308163907590.png)

## menuconfig条目

表示带选项的菜单，通常与if/endif联用，表示选中的条目才会出现。

![image-20230308163451056](image/Linux%E5%86%85%E6%A0%B8/image-20230308163451056.png)

## comment条目

文字信息，不可选



`make menuconfig` 配置完成之后，配置结果（变量的值）会保存在内核源码顶层目录的`.config`文件中，Makefile编译内核时回去读取`.config`文件中的变量，来控制编译行为。

![image-20230308164749688](image/Linux%E5%86%85%E6%A0%B8/image-20230308164749688.png)

```makefile
obj-y 编译进内核
obj-m 编译成模块
obj-n 不编译
```

# 修改内核配置菜单实现对新加入内核源码的控制

1. 将源码拷贝到内核对应的文件夹下
2. 为配置界面添加控制新加入源代码的内容

​	在加入源码的目录下创建`Kconfig`文件，便添加相应控制内容

3. 修改上层`Kconfig`文件，包含新加入源码的`Kconfig`文件。
4. 修改`Makefile`文件
5. 修改上一级目录的Makefile



```bash
make[1]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
make: *** [Makefile:1868: certs] Error 2
```

解决方法[(104) how to fix the error "No rule to make target 'debian canonical certs pem', needed by 'certs x509 cer - YouTube](https://www.youtube.com/watch?v=7G3UkugVhOw)：

```bash
nano .config
查找到debian/canonical-certs.pem和debian/canonical-revoked-certs.pem字符串删除
```

```bash
BTF: .tmp_vmlinux.btf: pahole (pahole) is not available
```

解决方法

```bash
sudo apt install dwarves
```



# qemu+gdb调试linux内核

## 下载linux源码

## 安装qemu

```bash
sudo apt install qemu
```



## 配置调试版内核

对内核进行调试需要解析符号信息，所以得编译一个调试版内核。

```bash
cd linux-5.15
make menuconfig
```

这里需要开启内核参数`CONFIG_DEBUG_INFO`和`CONFIG_GDB_SCRIPTS`。GDB提供了Python接口来扩展功能，内核基于Python接口实现了一系列辅助脚本，简化内核调试，开启`CONFIG_GDB_SCRIPTS`参数就可以使用了。

```bash
Kernel hacking  --->
    [*] Kernel debugging
    Compile-time checks and compiler options  --->
        [*] Compile the kernel with debug info
        [*]   Provide GDB scripts for kernel debugging
```

## 构建initramfs根文件系统

Linux系统启动阶段，boot loader加载完内核文件vmlinuz后，内核紧接着需要挂载磁盘根文件系统，但如果此时内核没有相应驱动，无法识别磁盘，就需要先加载驱动，而驱动又位于`/lib/modules`，得挂载根文件系统才能读取，这就陷入了一个两难境地，系统无法顺利启动。于是有了initramfs根文件系统，其中包含必要的设备驱动和工具，boot loader加载initramfs到内存中，内核会将其挂载到根目录`/`,然后运行`/init`脚本，挂载真正的磁盘根文件系统。

这里借助[BusyBox](https://www.busybox.net/)构建极简initramfs，提供基本的用户态可执行程序。

编译BusyBox，配置`CONFIG_STATIC`参数，编译静态版BusyBox，编译好的可执行文件`busybox`不依赖动态链接库，可以独立运行，方便构建initramfs。

```bash
cd busybox-1.28.0
make menuconfig
```

```bash
Settings  --->
    [*] Build static binary (no shared libs)
```

```bash
make -j 20
make install
```

会安装在`_install`目录:

```bash
ls _install
bin  linuxrc  sbin  usr
```

创建initramfs，其中包含BusyBox可执行程序、必要的设备文件、启动脚本`init`。这里没有内核模块，如果需要调试内核模块，可将需要的内核模块包含进来。`init`脚本只挂载了虚拟文件系统`procfs`和`sysfs`，没有挂载磁盘根文件系统，所有调试操作都在内存中进行，不会落磁盘。

```bash
mkdir initramfs
cd initramfs
cp ../_install/* -rf ./
mkdir dev proc sys
sudo cp -a /dev/{null, console, tty, tty1, tty2, tty3, tty4} ./dev/
rm linuxrc
vim init
chmod a+x init
ls
bin   dev  init  proc  sbin  sys   usr
```

init文件的内容

```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
exec /bin/sh
```

打包initramfs

```bash
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
```

## 调试

```bash
cd busybox-xxx
qemu-system-x86_64 -s -kernel /path/to/vmlinux -initrd initramfs.cpio.gz -nographic -append "console=ttyS0"
```

* `-s` 是`-gdb tcp::1234`的缩写，监听1234端口，在GDB中通过`target remote localhost:1234`连接；
* `-kernel` 指定编译好的调试内核`vmlinux`路径；
* `initrd` 指定制作好的`initramfs`；
* `-nographic`取消图形输出窗口，试qemu成简单的命令行程序。
* `-append "console=ttyS0"`将输出重定向到console，将会显示在标准输出stdio。

启动后的根目录

```bash
/ ls
bin   dev  init  proc  root  sbin  sys   usr
```















# 参考

[使用QEMU和GDB调试Linux内核 | Consen](https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/)

