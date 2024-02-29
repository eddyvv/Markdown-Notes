# grub修复

![img](image/grub%E4%BF%AE%E5%A4%8D/boot.jpg)

进入系统出现如上界面，解决方法

`grub rescue>`表示当前处于救援模式。

## 常用命令

```bash
set	不带参数则查看所有变量，或者带参数设置这个变量
ls	不带参数则查看所有分区，或者带参数列出分区或者路径下文件
insmod	加载模块(*.mod)
```

## 救援步骤

### 查找文件系统分区

逐一尝试`gpt0`和`gpt6`，直到找到正确boot分区的设备，那么此时正确的root变量设置好了，就得设置正确的`prefix`变量

```bash
set prefix=(hd0,msdos6)/boot/grub
```

### 加载`normal.mod`模块

```bash
insmod /boot/grub/arm64-efi/normal.mod
```

### 进入正常的GRUB2界面

```bash
normal
```

# 参考

[GRUB2的rescue模式 | Lixingcong](http://lixingcong.github.io/2019/07/14/grub-rescue/)

[「Grub」- 手动引导启动 @20210224_11091184的技术博客_51CTO博客](https://blog.51cto.com/u_11101184/3136426)