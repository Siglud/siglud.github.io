---
layout: post
title:  "TigerVNC升级到1.11.0-4无法启动问题"
date:   2020-09-14 11:08:00 +0800
author: Siglud
categories:
  - Linux
tags:
  - Linux
comment: true
share: true
---

一大早熟练的来一把sudo pacman -Syu, 看到升级了Kernel于是就重启，然后又把自己给启动挂了

症状是过了三分钟VNC连上去毫无反应，提示端口不可用，我这台机器是headless的，根本没有显示器可言，于是先检查一下启动了没，SSH轻松登陆——说明至少启动了，登上去查看~/.vnc/下的LOG,看到了以下的不详字段

```
Gdk-Message: 10:44:34.881: xfdesktop: Fatal IO error 11 (资源暂不可用) on X server :1.0.
```

估计是启动不了了，先看看社区的反应，CN社区很平静，看来大家都没有遇到我这个问题，上英文社区果然有人提问了。解决我的问题的主要是这位大佬的帖子

https://bbs.archlinux.org/viewtopic.php?pid=1925998#p1925998

解决步骤也很简单，主要原因应该是升级了之后TigerVNC的启动方式、依赖都发生了变化，可能对新手而言不需要自己建立systemd的启动文件了显得简单，但是老手而言就是不兼容问题了

解决方式是：

* 把 /etc/pam.d/tigervnc 文件备份一下，mv /etc/pam.d/tigervnc.pacnew /etc/pam.d/tigervnc，新版本的配置文件多了两行
* 把wiki里面以前指引的自己建立的systemd的配置文件干掉，把包里面自带的copy到systemd目录里面来 sudo cp /usr/lib/systemd/system/vncserver@.service /etc/systemd/system，启动的方式还是不变的——之前是 sudo systemctl status vncserver@:1.service 现在还是
* 如果以前没有安装dm的，一定要装一个，我就是那个dm都没安装的人——因为完全用不到嘛，但是现在不管你用不用你都得装一个，否则VNC会报告 No Xsession file available, vncsession-start just exits with status 0，解决方法是我安装了lightdm
* 因为以前是在systemd里面指定用户的，现在改为了通用的systemd文件，那么自然需要有地方指定你要用的用户，这个文件就在/etc/tigervnc/vncserver.users，编辑它，把id和名字指定一下，比如我改成 :1=siglud

全部搞定之后sudo systemctl restart vncserver@:1.service问题解决

另外，在这次升级之后 ~/.vnc/xstartup 这个文件已经过时了，如果需要运行自定义脚本，则必须要安装xorg-xinit

经过我当前的验证是 ~/.xinitrc 和 ~/. 都是读不到的，根据LOG显示，它只会去读这几个文件

```
Loading profile from /etc/profile
Loading xinit script /etc/X11/xinit/xinitrc.d/40-libcanberra-gtk-module.sh
Loading xinit script /etc/X11/xinit/xinitrc.d/50-systemd-user.sh
```
如果像我这样只是想要改个输入法的而已的花放那里都无所谓了