# Ubuntu20.04安装VNC

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

# 参考连接

https://blog.csdn.net/m0_48958478/article/details/121439484