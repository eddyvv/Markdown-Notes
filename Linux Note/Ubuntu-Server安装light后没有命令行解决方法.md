# Ubuntu-Server安装light后进入桌面没有命令行，且不能切换至命令行界面解决方案。

重启后按住`shift`进入以下界面。

![image-20221117113459885](image/Ubuntu-Server%E5%AE%89%E8%A3%85light%E5%90%8E%E6%B2%A1%E6%9C%89%E5%91%BD%E4%BB%A4%E8%A1%8C%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20221117113459885.png)

选择`Advanced options for Ubuntu`

![image-20221117113548248](image/Ubuntu-Server%E5%AE%89%E8%A3%85light%E5%90%8E%E6%B2%A1%E6%9C%89%E5%91%BD%E4%BB%A4%E8%A1%8C%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20221117113548248.png)

选择`recovery mode`

![image-20221117113734953](image/Ubuntu-Server%E5%AE%89%E8%A3%85light%E5%90%8E%E6%B2%A1%E6%9C%89%E5%91%BD%E4%BB%A4%E8%A1%8C%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20221117113734953.png)

选择`root`回车后即可进入命令行

![image-20221117113836521](image/Ubuntu-Server%E5%AE%89%E8%A3%85light%E5%90%8E%E6%B2%A1%E6%9C%89%E5%91%BD%E4%BB%A4%E8%A1%8C%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/image-20221117113836521.png)

执行

```bash
root@eddy:~# apt remove lightdm
```



---

重启即可。



🎃

🎁

🎄

