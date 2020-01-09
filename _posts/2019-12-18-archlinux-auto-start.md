---
layout: post
title:  "Archlinux桌面的自动启动"
date:   2019-12-18 14:43:00 +0800
author: Siglud
categories:
  - Linux
tags:
  - Linux
comment: true
share: true
---

Archlinux的桌面是有很多方法实现自启动的，除了自带的Systemd方法之外，默认也类似Windows的开始-启动文件夹一样，配置了启动文件夹一样的存在。

这东西就在

/etc/xdg/autostart

这个是全局的x启动时候附带的启动。

另一个在各自用户下

~/.config/autostart

安装Telegram的时候，发现它也会自动启动，然后它的自动启动其实是位于

/usr/share/kservices5/tg.protocol

