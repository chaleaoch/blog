---
title: "archlinux安装xrdp踩坑记"
publish_time: "2023-06-07"
hidden: false
---
先记录步骤, 最后说明哪里踩坑.

1. 安装

```txt
yay -S xrdp xorgxrdp
```

1. `cat /etc/X11/Xwrapper.config`

```txt
allowed_users=anybody
```

1. 新版0.9.21.1-1 需要 `cat /etc/xrdp/sesman.ini`

```txt
[Xorg]
# 其他不变
param=/usr/lib/Xorg
```

1. `sudo pacman -S xorg-xinit`
2. `cp /etc/X11/xinit/xinitrc ~/.xinitrc`
3. `cat ~/.xinitrc` 最后几行

```txt
#twm &
#xclock -geometry 50x50-1+1 &
#xterm -geometry 80x50+494+51 &
#xterm -geometry 80x20+494-0 &
#exec xterm -geometry 80x66+0+0 -name login
exec dbus-run-session -- startplasma-x11
```

1. 常规操作

```txt
systemctl enable xrdp
systemctl enable xrdp-sesman
systemctl restart xrdp
systemctl restart xrdp-sesman
```

问题出在第4和5步, 其实网上很多有类似提示,但是新安装的arch是没有这个文件的. 安装上即可.

以下是关键字. 如果你在`sudo systemctl status xrdp`中遇到类似关键字, 那差不多咱们遇到的是同一个问题.

```txt
[ERROR] xrdp_mm_chansrv_connect: error in trans_connect chan
Feb 18 00:28:13 pc xrdp[30146]: [ERROR] SSL_shutdown: Failure in SSL library (protocol error?)
Feb 18 00:28:13 pc xrdp[30146]: [ERROR] SSL: error:0A000123:SSL routines::application data after close notify
```
