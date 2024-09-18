# ACoreLinux安装

1. 进入麒麟系统；
2. 挂载一个不使用的盘到`/mnt`下；
3. 传输`ACoreLinux`系统至`/mnt`下并解压；
4. 更改`grub`

`/boot/efi/BOOT/`

```bash
serial --unit=0 --speed=115200 --word=8 --parity=no --stop=1 
search --no-floppy --set=root -l 'boot' 
default=boot 
timeout=10

menuentry 'AcoreLinux Embedded version V1.0.0.F.2' {
        insmod part_gpt
        insmod ext2
        ### 1
        set root='hd0,gpt4'
        Echo  ‘Loading Linux 4.19 ACoreLinux-01 ...’
        ### 2 3
        linux  /boot/Image root=/dev/sda4
        console=ttyAMA1,115200 splash loglevel=7 rootdelay=5 KEYBOARDTYPE=pc KEYTABLE=us security=
}
## 1是内核文件Image-xxx.bin 所在的分区；
## 2是内核文件的文件名字；
## 2是根文件系统所在分区。
```

5. 重启