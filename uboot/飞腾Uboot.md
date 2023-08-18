# uboot编译

```bash
make D2000_config
make ARCH=arm CROSS_COMPILE=/opt/gcc-linaro-6.1.1-2016.08-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
```

交叉编译工具链下载 [Linaro Releases](http://releases.linaro.org/components/toolchain/binaries/6.1-2016.08/aarch64-linux-gnu/)

# uboot打包

```bash
cd image_fix
./my_scripts/fix_parameter.sh
./my_scripts/image-fix.sh
```

