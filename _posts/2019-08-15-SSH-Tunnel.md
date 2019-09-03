---
layout: post
title:  "SSH的端口转发"
date:   2019-08-15 14:36:00 +0800
author: Siglud
categories:
  - Linux
tags:
  - Linux
comment: true
share: true
---

因为想要把家里的NAS的管理界面映射出来方便到外网管理，但是问题是家里的移动宽带完全没有外网IP，虽然有IPv6，但是似乎也被防火墙包裹得死死的，没办法，只能考虑把端口转发出来以保证外面正常访问。

策略就是SSH把本地端口8088转发到阿里云的公网服务器上，然后访问公网服务器搞定。

实现起来也就一句话：

```bash
ssh -NR 8088:127.0.0.1:8088 target
```
-N = 不要console界面
-R = 转发端口

为了保证持续通畅，考虑用autoSSH

然后就变成这样：

```bash
autossh -M 1001 -f -NR 8088:127.0.0.1:8088 target
```
但是autoSSH只能支持Key登陆，没办法输密码的。

然后就是如果需要映射到监听0.0.0.0的话，需要修改sshd_config，把默认的GatewayPorts选项从no改为yes，否则默认监听127.0.0.1。

其实监听本地也没啥太大问题就是了，大不了连接端再发起一个SSHTunnel，然后用-L参数把远端的端口再拖回到本地来，这个一来一回还用了加密，反而更加安全，也不失为一个好的办法。