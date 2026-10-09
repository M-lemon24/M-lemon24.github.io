---
title: ssti模板注入
link: your-custom-slug
catalog: true
date: "2026-10-09 14:00:00"
description: 一些对ssti的详细分析
tags:
  - Web安全
  - 学习笔记
categories:
  - 笔记
draft: false
---

## SSTI漏洞成因
SSTI就是服务器端模板注入，针对的是当前使用的一些比较成熟的框架，比如说python的flask，php的tp，Java的spring之类的较为成熟的MVC模式。
成因就是服务区接受攻击者的数据输入之后，未经任何处理就将其作为web应用模板的一部分，模板引擎在进行目标编译渲染的过程中，执行了攻击者插入的可以破坏模板的语句，从而达到攻击者的目的。
凡是使用模板的地方就可能会出现SSTI的问题，SSTI不属于任何一种语言 ，沙盒绕过也不是，沙盒绕过只是由于模板引擎发现了很大的安全漏洞，然后模板引擎设计出来的一种防护机制，不允许使用没有定义或者声明的模块，这适用于所有的模板引擎。

## SSTI的分类
### PHP里面的SSTI
常见的模板有twig，smarty，blade
#### Twig
Twig是一款php模板引擎，主要是将后端业务逻辑与前端展示页面（HTML）彻底解耦。
##### 
