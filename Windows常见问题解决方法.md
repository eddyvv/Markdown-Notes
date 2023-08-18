# VMWare安装出现`Failed to install the hcmon driver`

![image-20230710084706295](image/Windows%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20230710084706295.png)

解决方法：

新建文本文件输入以下内容：

```bash
Windows Registry Editor Version 5.00
 
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion]
"ProgramFilesDir"="C:\\Program Files"
"CommonFilesDir"="C:\\Program Files\\Common Files"
"ProgramFilesDir (x86)"="C:\\Program Files (x86)"
"CommonFilesDir (x86)"="C:\\Program Files (x86)\\Common Files"
"CommonW6432Dir"="C:\\Program Files\\Common Files"
"DevicePath"=hex(2):25,00,53,00,79,00,73,00,74,00,65,00,6d,00,52,00,6f,00,6f,\
  00,74,00,25,00,5c,00,69,00,6e,00,66,00,00,00
"MediaPathUnexpanded"=hex(2):25,00,53,00,79,00,73,00,74,00,65,00,6d,00,52,00,\
  6f,00,6f,00,74,00,25,00,5c,00,4d,00,65,00,64,00,69,00,61,00,00,00
"ProgramFilesPath"=hex(2):25,00,50,00,72,00,6f,00,67,00,72,00,61,00,6d,00,46,\
  00,69,00,6c,00,65,00,73,00,25,00,00,00
"ProgramW6432Dir"="C:\\Program Files"
```

将文本文件后缀改为`.reg`后执行该文件。

将vmware安装位置配置为默认的C盘。





# Windows Terminal

[消除WSL中ls Windows文件夹时背光配色的方法_wsl字体颜色_天色微凉的博客-CSDN博客](https://blog.csdn.net/qq_33882435/article/details/116264702)

[去掉ubuntu下windows文件夹绿色背景_有硬币就是土豪的博客-CSDN博客](https://blog.csdn.net/gengli2017/article/details/82917827)

# 禁用驱动程序强制签名

打开设置（win+i）->更新和安全->恢复->高级启动->立即重新启动->疑难解答->高级选项->启动设置->重启->根据提示输入`7`禁用驱动程序强制签名
