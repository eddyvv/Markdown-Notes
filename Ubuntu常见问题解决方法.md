# Ubuntu常见问题解决方法



## Ubuntu只显示本地环回无法连接宿主机



```bash
# 先停止服务
sudo service network-manager stop  
# 删除文件
sudo rm /var/lib/NetworkManager/NetworkManager.state
# 修改文件
sudo vim /etc/NetworkManager/NetworkManager.conf 把false改成true　
#重启服务器
sudo service network-manager start
```

## Ubuntu不显示网络连接

```bash
sudo nmcli networking off

sudo nmcli networking on

sudo service network-manager restart
```



## Ubuntu安装VM-Tools

```bash
sudo apt-get install open-vm-tools
sudo apt-get install open-vm-tools-desktop
```

## Ubuntu一键切换到简体中文

[TheWhiteDog9487/ubuntu-language-onekey-zh_cn: 一键切换Ubuntu系统的终端语言到简体中文 (github.com)](https://github.com/TheWhiteDog9487/ubuntu-language-onekey-zh_cn)

# 查看Ubuntu版本

```bash
lsb_release
```





## Ubuntu切CPU性能

```bash
sudo apt install cpufrequtils
cpufreq-info
sudo cpufreq-set -g performance
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

## Ubuntu-Server安装图形界面

```bash
sudo apt install lightdm
sudo service lightdm start
```

## SFTP传输文件

常用命令

```bash
 SFTP命令汇总
                  cd 路径                        更改到远程目录的路径
                  lcd 路径                       更改到本地目录的路径
                  chgrp group path               将文件path的组更改为group
                  chmod mode path                将文件path的权限更改为mode
                  chown owner path               将文件path的属主更改为owner
                  exit                           退出 sftp
                  help                           显示这个帮助文本
                  get 远程路径                   下载文件
                  ln existingpath linkpath       符号链接远程文件
                  ls [选项] [路径]               显示远程目录列表
                  lls [选项] [路径]              显示本地目录列表
                  mkdir 路径                     创建远程目录
                  lmkdir 路径                    创建本地目录
                  mv oldpath newpath             移动远程文件
                  open [用户@]主机[:端口]        连接到远程主机
                  put 本地路径                   上传文件
                  pwd                            显示远程工作目录
                  lpwd                           打印本地工作目录
                  quit                           退出 sftp
                  rmdir 路径                     移除远程目录
                  lrmdir 路径                    移除本地目录
                  rm 路径                        删除远程文件
                  lrm 路径                       删除本地文件
                  symlink existingpath linkpath  符号链接远程文件
                  version                        显示协议版本
```



### Windows SFTP连接Linux

```bash
C:\User\eddy> sftp root@192.168.159.131
root@192.168.159.131's password:
Connected to 192.168.159.131.
sftp> 
```

### 下载Linux端文件夹test至本地F盘test目录下

```bash
sftp> get -r /home/test F:/test
```



### 上载本都端文件至Linux home目录下

```bash
sftp> put -r F:/msdia80.dll /home/
```

## 更改IP地址

```bash
$ sudo vi /etc/netplan/00-installer-config.yaml
```

键入

```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
     dhcp4: no
     addresses: [192.168.1.2/24]#ip地址
     gateway4: 192.168.1.1#网关
     nameservers:
       addresses: [114.114.114.114]
```

启动服务

```bash
sudo netplan apply
```

## Ubuntu安装中文`man`手册

```bash
1.sudo apt-get update # 更新你的下载源目录，此步骤可省略。
2.sudo apt-get install manpages-zh # 系统会自动下载并安装
3.vi ~/.bashrc # 编辑家目录下的bash配置文件
4.在最后一行输入：alias cman='man -M /usr/share/man/zh_CN' # 将中文的man命令重命名为cman命令，之后保存并退出编辑
5.source ~/.bashrc # 重新运行.bashrc文件
```

# Ubuntu20.04安装Perf工具

```bash
sudo apt install linux-tools-common
sudo apt install linux-tools-5.13.0-40-generic
sudo apt install linux-cloud-tools-5.13.0-40-generic
```

# Unknown symbol ib_register_device

安装 `ib_core.ko`

```bash
sudo apt-get install build-essential libelf-dev cmake
sudo apt-get install libibverbs1 libibverbs-dev librdmacm1 librdmacm-dev rdmacm-utils ibverbs-utils
sudo modprobe ib_core
sudo modprobe rdma_ucm
```

# 命令行打印export指定的变量值

```bash
printenv
```

# Ubuntu设置默认启动内核

以启动第四内核为例：

```bash
sudo vim /etc/default/grub
```

修改`/etc/default/grub`文件的`GRUB_DEFAULT`的值为`"1>4"`，注意此处需要双引号，且等号前后不能有空格。

![image-20230628142218366](image/Ubuntu%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20230628142218366.png)

其中的`1`表示菜单的选项1（菜单从0开始）

![image-20230515104210488](image/Ubuntu%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20230515104210488.png)

`4`表示二级菜单的选中的待默认启动的内核版本。

![image-20230517111217516](image/Ubuntu%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20230517111217516.png)

更改完成之后执行

```bash
sudo update-grub
```

