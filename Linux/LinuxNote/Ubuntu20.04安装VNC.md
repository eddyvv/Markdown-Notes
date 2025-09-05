# Ubuntu20.04安装VNC(旧版)

安装ssh

```bash
sudo apt install openssh-server
```

配置ssh_config

```bash
sudo vi /etc/ssh/ssh_config
#将PasswordAuthentication设置为yes,之后重启ssh
sudo /etc/init.d/ssh restart
```

安装tightvncserver

```bash
sudo apt-get install tightvncserver
```

启动vncserver并配置密码

```bash
vncserver
```

删除端口

```bash
vncserver -kill :1
```

安装gnome-panel

```bash
sudo apt-get install gnome-panel
```

启动vncserver

```bash
vncserver
```

配置`xstartup`

```bash
cd ~/.vnc
mv xstartup xstartup.bak
vim xstartup

#键入
#!/bin/sh                                                                       
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
export XKL_XMODMAP_DISABLE=1
export XDG_CURRENT_DESKTOP="GNOME-Flashback:GNOME"
export XDG_MENU_PREFIX="gnome-flashback-"
[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources
xsetroot -solid grey
vncconfig -iconic &
#gnome-terminal &    
#nautilus &   
gnome-session --session=gnome-flashback-metacity --disable-acceleration-check &
```

添加可执行权限

```bash
chmod +x xstartup
```

启动vnc并配置分辨率

```bash
vncserver -geometry 1920x1080
```

注意：如果打开vncview显示灰色，使用`vncserver -kill :x`杀掉vncserver，重新启动即可，x为vncserver的端口号

# Ubuntu20.04安装VNC(基于x11vnc 可实现使用端与被使用端界面协同)

## 安装图形界面

```bash
sudo apt install ubuntu-desktop 

sudo apt-get install gnome-panel gnome-settings-daemon metacity nautilus gnome-terminal 

sudo reboot #重启即可看到图形界面

```

## 安装X11VNC

```bash
sudo apt-get install x11vnc
```

## 安装LightDM【LightDM从设计上就是支持本地图形界面以获得最好的兼容性】

```bash
sudo apt-get install lightdm
```

安装过程中会出现以下选项,选择`lightdm`然后回车即可

![image-20230803094446632](image/Ubuntu20.04安装VNC/lightdm.png#pic_center)

设置密码,设置密码后,会问你是否需要将密码保存在:/home/root1/.vnc/passwd,输入`y`确认即可

```bash
x11vnc -storepasswd
```

![image-20230803094446632](image/Ubuntu20.04安装VNC/storepasswd.png#pic_center)

## 设置vnc开机启动

```bash
sudo vim /lib/systemd/system/x11vnc.service
```

按`i`键进入编辑模式,添加如下信息

> **!!注意: `<USERNAME>`替换为您ubuntu用户名***,添加完成后按Esc键退出编辑,然后输入冒号:wq保存*

```vim
[Unit]
Description=Start x11vnc at startup.
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -auth guess -forever -loop -noxdamage -repeat -rfbauth /home/<USERNAME>/.vnc/passwd -rfbport 5900 -shared

[Install]
WantedBy=multi-user.target

```

## 设置开机启动

```bash
sudo systemctl enable x11vnc.service
```

## 启动服务

```bash
sudo systemctl start x11vnc.service
```

# 参考连接

https://blog.csdn.net/m0_48958478/article/details/121439484

https://www.cpolar.com/blog/remote-desktop-ubuntu