---
layout: post
title:  "打开被提示为已损坏，需要移到废纸篓的软件"
date:   2019-06-21 10:36:00 +0800
author: Siglud
categories:
  - OSX
tags:
  - OSX
comment: true
share: true
---

因为用D版的原因，OSX有时候会提示“XX已损坏，需要移到废纸篓”，但是其实这货明显是能够打开的

之前的解决之道是
```
sudo spctl --master-disable
```
然后在安全与隐私中选择“任何来源”

但是这次连这个都失效了，只能使用绝招，绕开一开始的验证流程：

```
xattr -r -d com.apple.quarantine /Applications/<YourAPPName>
```
这样就直接跳过了一开始的自检流程，直接可以打开咯