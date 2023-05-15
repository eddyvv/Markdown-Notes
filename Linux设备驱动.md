# Linux设备驱动依赖配置

若a驱动的部分函数依赖于b驱动，需要在a驱动的makefile中加入 `Module.symvers`符号依赖。

```bash
KBUILD_EXTRA_SYMBOLS = $(PWD)/B/Module.symvers
```

