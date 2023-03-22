# 安装petalinux2022.2

1. 安装依赖包

```bash
sudo apt-get install iproute2 gawk python3 python build-essential gcc git make net-tools libncurses5-dev tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget git-core diffstat chrpath socat xterm autoconf libtool tar unzip texinfo zlib1g-dev gcc-multilib automake zlib1g:i386 screen pax gzip cpio python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3 libtinfo5
```

2. 更改文件夹权限

```bash
sudo mkdir petalinux2022.2
sudo chmod 777 petalinux2022 ./
```

3. 下载petalinux2022.2安装包

地址：[Downloads (xilinx.com)](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/embedded-design-tools.html)

4. 更改默认shell

```bash
bash
```

5. 安装

```bash
./petalinux-v2022.2-10141622-installer.run -d ./petalinux2022.2
```

安装过程需要同意许可信息，输入三次y。





# qemu启动

## 1. 生成.BIN镜象

粘贴相应的`.bit`至`plnx-proj-root/images/linux/`

```bash
cd <plnx-proj-root>
petalinux-package --boot --format BIN --fsbl --u-boot --pmufw --fpga ./images/linux/design_1_wrapper.bit --force
```

## 2. 封装预构建镜像

```bash
petalinux-package --prebuilt --fpga ./images/linux/design_1_wrapper.bit --force
```

## 3. 启动qemu

```bash
petalinux-boot --qemu --prebuilt 3
```

## 打包BSP

```bash
petalinux-package --bsp -p 2021-25-mcdma/ --hwsource eth_25g_2021-3-8/ -o 2021-25-mcdma.bsp --force
```





psu_init 是FSBL中的一个函数，用于初始化PS端

# petalinux qemu 调试linux内核

1. 启动qemu

   ```bash
   petalinux-boot --qemu --prebuilt 3
   ```

2. 进入`<plnx-proj-root>image/linux`文件夹

3. 进行gdb调试

   ```bash'
   petalinux-util --gdb vmlinux
   ```
   

[gdb 调试 Linux 内核网络源码（附视频） (wenfh2020.com)](https://wenfh2020.com/2021/05/19/gdb-kernel-networking/)

## 调试步骤[^6]

可参考《UG1144》 Debugging节 

1. 启动qemu

1. 执行gdb调试命令

```bash
petalinux-util --gdb vmlinux
```







# 创建`zynqMP`自定义工程[^1]

## 创建`.xsa`文件

使用`Vivado`创建工程并生成`xsa`文件

## 创建自定义工程

```bash
petalinux-create -t project --template zynqMP --name <name>
```

## 工程配置

```bash
$ cd <plnx-proj-root>
petalinux-config --get-hw-description <name>.xsa

petalinux-config
#默认配置

petalinux-config -c kernel
#配置内核

petalinux-config -c rootfs
#配置文件系统

petalinux-config -c u-boot
#配置u-boot
```

对构建的工程进行实际的配置

## 工程编译

```bash
cd <plnx-proj-root>
petalinux-build
#整个工程编译，编译后生成image.ub

petalinux-build -c kernel
#编译内核

petalinux-build -c rootfs
#编译文件系统

petalinux-build -c u-boot
#编译u-boot

petalinux-build -x distclean
#清空编译后产生的临时文件和下载的sstate cache文件

petalinux-build -x mrproper
#清空所有的临时文件，<PROJECT>/images/,  <PROJECT>/build/  and <PROJECT>/components/plnx_workspace/ directories


"petalinux-build -x package" 是 PetaLinux 工具的命令，它用于构建 PetaLinux 项目。PetaLinux 是一个用于嵌入式设备的 Linux 发行版，由 Xilinx 公司开发。PetaLinux 提供了一些工具和命令，用于构建、调试和定制 Linux 系统。其中 "petalinux-build" 就是用于构建 PetaLinux 项目的命令，而 "-x package" 则表示只构建指定的软件包。

#打包到image.ub

petalinux-build -c mymodule -x do_cleansstate
#清除用户模块
```

## 创建自定义模块

```bash
$ cd <plnx-proj-root>
petalinux-create -t modules --name <user-module-name> --enable

petalinux-create -t apps --template c --name myapp --enable
```


# JTAG烧写

## `.tcl`文件



# 常见问题

## petalinux 2022.2编译工程，显示bitbake版本不兼容[^2]

```bash
$PETALINUX/components/yocto/buildtools/sysroots/x86_64-petalinux-linux/usr/bin/python3 ./components/yocto/layers/core/scripts/contrib/convert-overrides.py ./project-spec/meta-user/
```

错误[^5]

```bash
[   52.763091] alloc_netdev: Unable to allocate device with zero queues
[   52.793075] xilinx_axienet: probe of a0010000.ethernet failed with error -12
```



```bash
25g-3-6$ petalinux-config --get-hw-description eth25g_wrapper.xsa 
[INFO] Sourcing buildtools
INFO: Getting hardware description...
INFO: Renaming eth25g_wrapper.xsa to system.xsa
[INFO] Generating Kconfig for project
ERROR: Failed to generate /home/pdc/00-petalinux-workspace/25g-3-6/build/misc/config/Kconfig.syshw
ERROR: Failed to Kconfig project
ERROR: Failed to generate System hardware Kconfig file.
```

解决方法

```bash
sudo apt-get install libtinfo5
```

## ERROR /dev 

```bash
ERROR: There's no '/dev' on rootfs
```

[ERROR: There's no '/dev' on rootfs (xilinx.com)](https://support.xilinx.com/s/question/0D52E00006hprw1SAA/error-theres-no-dev-on-rootfs?language=en_US)

```bash
petalinux-config -> Image packaging configuration -> INITRAMFS/INITRD Image name -> petalinux-image-minimal
```



# 如何修改设备树[^3]

## 基于自动生成的设备树，添加或修改节点

通过修改`./project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi`，来实现对之前已经定义的设备树节点中的属性进行变更，也可以添加没有自动生成的设备树节点。

设备树语法可参考 《devicetree-specification.pdf》[^4]

设备树更改完成后需要执行以下命令

```bash
petalinux-build -c device-tree -x cleansstate
生成pl.dtsi
petalinux-config -c device-tree
petalinux-build -c device-tree
```

## 设备树反编译

```bash
../../build/tmp/sysroots-components/x86_64/dtc-native/usr/bin/dtc -I dtb -O dts -o system.dts system.dtb
```





# 手册

PG332 ERNIC 手册

PG203 CMAC 手册

PG210 25G AXI Ethernet 手册

PG288 MCDMA手册

UG1085-Zynq-Ultrascale-trm 手册 250 SoC芯片手册

[Zynq UltraScale+ Devices Register Reference (xilinx.com)](https://www.xilinx.com/htmldocs/registers/ug1087/ug1087-zynq-ultrascale-registers.html) ug1085-Zynq-Ultrascale寄存器手册

UG1144 Petalinux Tools手册



错误

```bash
RROR: perftest-1.0-r0 do_compile: oe_runmake failed
ERROR: perftest-1.0-r0 do_compile: Execution of '/home/dpu/zjzhe/250soc_ernic/build/tmp/work/cortexa72-cortexa53-xilinx-linux/perftest/1.0-r0/temp/run.do_compile.180753' failed with exit code 1:
make  all-am
make[1]: Entering directory '/home/dpu/zjzhe/250soc_ernic/build/tmp/work/cortexa72-cortexa53-xilinx-linux/perftest/1.0-r0/git'
make[1]: *** No rule to make target '../../../umm/1.0-r0/libumm.so', needed by 'libperftest.a'.  Stop.

```

解决方法

```bash
dpu@dpu-PC:~/zjzhe/250soc_ernic/build/tmp/work/cortexa72-cortexa53-xilinx-linux/perftest/1.0-r0/git$ find ..// -name "libumm*"
..//recipe-sysroot/usr/lib/libumm.so.1.0
..//recipe-sysroot/usr/lib/libumm.so.1
..//recipe-sysroot/usr/lib/libumm.so
dpu@dpu-PC:~/zjzhe/250soc_ernic/build/tmp/work/cortexa72-cortexa53-xilinx-linux/perftest/1.0-r0$ ln -s ./recipe-sysroot/usr/lib/libumm.so ../../umm/1.0-r0/libumm.so

```





# 参考

[^1]:[PetaLinux - Public Docs - Trenz Electronic Wiki (trenz-electronic.de)](https://wiki.trenz-electronic.de/display/PD/PetaLinux) petalinux工程创建等使用

[^2]:[PetaLinux Yocto Tips - Xilinx Wiki - Confluence (atlassian.net)](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842475/PetaLinux+Yocto+Tips) Yocto相关使用
[^3]:[Xilinx Petalinux工程设备树介绍](https://support.xilinx.com/s/article/1197597?language=zh_CN)
[^4]:[Releases · devicetree-org/devicetree-specification (github.com)](https://github.com/devicetree-org/devicetree-specification/releases?page=2)

[中断（一）：让 Linux 接收来自 PL 的自定义中断信号 (zynqnotes.com)](http://www.zynqnotes.com/pl-ps-interrupt) zynqMP PL2PS、PS2PL设备树中断配置

[^5]:[Linux AXI Ethernet driver - Xilinx Wiki - Confluence (atlassian.net)](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842485/Linux+AXI+Ethernet+driver) Xlinx-AXI-Ethernet-driver相关说明

[^6]:[petalinux调试内核_StarLish0715的博客-CSDN博客](https://blog.csdn.net/baidu_37503452/article/details/80106440) 

[主页 • 文档门户 (xilinx.com)](https://docs.xilinx.com/) Xlinx 文档下载

[Xilinx/u-boot-xlnx: The official Xilinx u-boot repository (github.com)](https://github.com/Xilinx/u-boot-xlnx) xlinx维护的uboot

[Xilinx/linux-xlnx: The official Linux kernel from Xilinx (github.com)](https://github.com/Xilinx/linux-xlnx) xlinx维护的linux

[Petalinux2020.2 开发ZYNQ的AXI DMA - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/478422063) petalinux创建modules

[PetaLinux常用命令汇总 - schips - 博客园 (cnblogs.com)](https://www.cnblogs.com/schips/p/xilinx-petalinux-common-commands.html)
