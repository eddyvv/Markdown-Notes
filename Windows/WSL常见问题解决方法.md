

## Sub-process /usr/bin/dpkg returned an error code (1)

```bash
dpkg: error processing package libc-bin (--configure):
 installed libc-bin package post-installation script subprocess returned error exit status 134
Errors were encountered while processing:
 libc-bin
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

解决方法

```bash
sudo mv /var/lib/dpkg/info/libc-bin.* /tmp/
sudo dpkg --remove --force-remove-reinstreq libc-bin
sudo dpkg --purge libc-bin
sudo apt install libc-bin
sudo mv /tmp/libc-bin.* /var/lib/dpkg/info/
```

参考[libc-bin update error · Issue #4760 · microsoft/WSL --- libc-bin 更新错误 · 问题 #4760 · microsoft/WSL (github.com)](https://github.com/microsoft/WSL/issues/4760)

## /bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)

```bash
sudo localedef -i en_US -f UTF-8 en_US.UTF-8
```



