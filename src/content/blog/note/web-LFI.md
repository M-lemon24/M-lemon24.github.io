---
title: 文件包含漏洞
link: web-LFI
catalog: true
date: 2026-9-29 16:00
description: 文件包含漏洞的理解
tags:
  - web
categories:
  - 笔记
draft: false
---


## 文件包含漏洞的原理
程序使用了文件包含函数（如include、require），这类函数本身是合法的功能（用于引用公共代码、配置文件等）。
包含函数的 “路径函数”可被用户控制（如通过URL的==?file===、==这是高亮文字==，==?action===参数传递），且未经过严格校验。
