---
layout:     post
title:      "skynet 学习笔记 - 消息队列的实现机制"
subtitle:   "skynet服务器架构学习"
description: "一些外部的第三方系统可能需要访问系统内部的微服务。例如在网上商店的例子中，外部的推荐服务可能需要接入系统，以获取商店的商品目录信息。相对于内部服务之间的访问而言，外部系统的访问需要进行严格的安全控制。"
date:     2019-01-11
author:   ledis
image: "https://img.zhaohuabing.com/in-post/istio-traffic-shifting/crossroads.png"
published: true
tags:
     - Skynet
URL: "/2019/01/11/istio-traffic-shifting/"
categories: ["blog"]
---

## 介绍

skynet是一个基于C跟lua的开源服务端并发框架，实现了Actor模型，
gate是模板lualib/snax/gateserver.lua使用范例，系统自带的，位于service/gate.lua，是一个实现完整的网关服务器。

watchdog和agent都是在应用层写的脚本，可以参考源码中example目录下的watchdog.lua和agent.lua

在启动脚本main.lua（一般情况下）启动了watchdog服务（而watchdog的启动函数内又启动了gate服务），并向watchdog发送了start命令，而watchdog对start命令的处理就是向gate发送open命令，其实是让gate服务开启监听端口，此时gate记录了watchdog的的地址。

当一个客户端连接过来的时候，首先是经过gate处理（handler.connect），先建立fd -> connection的映射，然后转发给watchdog（SOCKET.open）处理，在其中新建一个agent服务，然后建立fd->agent服务地址的映射，然后向agent服务发送start命令，agent在start命令的处理中又向gate发送forward命令，gate在forward的处理中是通过建立agent -> connection的映射来建立转发，此时的 connection包含agent服务地址，然后调用gateserver.openclient放行消息。以后在客户端发来消息时，如果已经建立的转发，则直接转发到对应的agent服务上，否则，则发送给watchdog处理。
