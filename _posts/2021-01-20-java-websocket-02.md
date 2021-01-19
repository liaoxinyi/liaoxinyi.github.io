---
layout:     post
title:      "推呀推，推呀推，推到客户端"
subtitle:   "心跳问题"
date:       2021-01-20
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/websocket02.jpg"
tags:
    - WebSocket
    - Java
---
> 资料来源于网络上各位前辈（M1lo等）

### 心跳问题
##### 为什么要做心跳？
Websocket是基于TCP的应用层技术，虽然TCP本身是有keepalive机制的，但是仅仅靠它自己的心跳是完全不够的。如果不信，看看以下场景是否还能正常运行：  
- **client异常挂死**

此时keepalive机制无法反馈真实的client状态

- **client异常断电断网出现TCP假死**

keepalive并不能根本性解决问题，实际上互联网环境很不稳定

- ws在应用层，基于传输层，想要在ws中操作TCP来维持心跳是很不方便的

##### 不同场景对应的策略

- **客户端**

|client操作|server处理方法|client处理方法|处理思路|  
|----|----|----|----|  
|关闭浏览器|触发onClose回调|/|应用层ws主动关掉连接（优雅关闭）|  
|杀掉浏览器|先触发onError回调再触发onClose回调|/|在操作系统中，应用程序对应的进程被干掉的时候会关闭其端口，也就是触发了TCP四次挥手。对于ws来讲直接在外部断开TCP会触发ws异常,对于ws来讲这样的关闭方式为非优雅关闭会触发异常|  
|断电断网|检测client最后心跳上报时间|触发onClose（断网）|server角度：如果client最后上报时间已经超过正常周期\*3，server认为其离线；client角度：断电就不说了。断网的情况client之所以触发了onClose我认为可能是当断网时操作系统关闭了所有对外的网络端口或者操作系统通知了浏览器断网（由此看出操作系统的知识真的是太重要了）；所以此时三个心跳周期过后当我们认为此session已经断开时不要忘记通知ws close掉这个session,不然有可能出现大量服务端TCP假死.接下来说重连,大家要注意重连对于server是来讲是一个新的连接，大家可以通过断网重连后server产生的session判断出断网重连实际上是产生了一个新的连接。对于server的原session如何处理我做了这样一个测试，当客户端断网后server依然通过原session发送数据给client当发送的数据超过一定时间一定数量没有回复后server会触发onError和onClose方法，对于原session server在client断开后从来不给这个client发消息的情况也就是重连的情况,我们要在新的session产生时及时清掉旧的session.同TCP假死处理一致|  

- **服务端**

|server操作|server处理方法|client处理方法|处理思路|  
|----|----|----|----|  
|重启服务|/|触发onClose|应用层ws主动关掉连接（优雅关闭）|  
|杀掉服务进程(kill -9 pid)|/|先触发onError回调再触发onClose回调|(同client被杀死)|  
|断电断网|检测client最后心跳上报时间|心跳异常|（见下表：server断电断网时client如何感知），也就是说对于client来讲，只要正常发送心跳给server就可以了。如果server断开网络超过20分钟（心跳：次/10mins）所有client均会掉线|  

##### 归纳
**开启应用层心跳是非常有必要的，如果只靠Websocket自身的重连机制，那么连接超过一定时间后还是会出现掉线的情况。同时心跳推荐时间为4分半,用以适配所有浏览器**


