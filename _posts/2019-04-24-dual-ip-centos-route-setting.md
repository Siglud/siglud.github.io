---
layout: post
title:  "在CentOS上设置双IP"
date:   2019-04-24 12:38:00 +0800
author: Siglud
categories:
  - 运维
tags:
  - Linux
comment: true
share: true
---

公司一台服务器接入了三个IP，其中两个外网一个内网IP，当前的目标是两个外网IP都能畅通，并且能对外通讯。

最后写出来就是这样的了：

```bash
ip route flush table net2
ip route add default via YOUR_GATEWAY dev eth0 src YOUR_IP_1 table net2
ip rule add from YOUR_IP_1 table net2
ip route add default via YOUR_GATEWAY dev eth0 src YOUR_IP_1 table net2
ip rule add from YOUR_IP_1 table net2
ip route add YOUR_GATEWAY/YOUR_GATEWAY_MASK dev eth0
ip route add default via YOUR_IP_1 dev eth0
ip route flush table net3
ip route add default via YOUR_GATEWAY_2 dev eth0 src YOUR_IP_2 table net3
ip rule add from YOUR_IP_2 table net3
```

为了保证自启动，放到/etc/rc.local里面完成。