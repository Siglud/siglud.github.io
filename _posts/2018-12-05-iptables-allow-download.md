---
layout: post
title:  "Iptables禁止所有端口连接但是允许下载"
date:   2018-12-05 20:54:00 +0800
author: Siglud
categories:
  - Linux
tags:
  - Iptables
comment: true
share: true
---

本来打算禁用所有端口，结果一句
```bash
iptables -A INPUT -j DROP
```

之后发现端口确实是禁用掉了，但是是连着下载一起禁用掉了，查了一下发现，原来还是要增加一个条件

```bash
iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

其实重要的是RELATED这个，也就是OUTPUT之后相关的入栈流量需要允许，否则还是可能会导致无法正常下载