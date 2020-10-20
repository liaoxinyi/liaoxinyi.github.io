---
layout:     post
title:      "积跬步以至千里——Linux命令积累"
subtitle:   "日常用到的Linux命令记录"
date:       2020-10-20
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/linux.bmp"
tags:
    - Linux
---
> 来源于《鸟哥的私房菜》+平时使用过程中的积累+IT公众号的积累，不定期更新

### Linux 命令格式
一般执行 Linux 命令格式都是这样的：命令名称 [命令参数] [命令对象]  
注意：它们之间是有空格的
### 查看帮助
Linux系统中，有很多命令，我怎么知道某个命令是干嘛用的，这时可以执行帮助命令查看：  
man man  
### 常用系统命令
##### 系统运行
- ps 命令（只显示**瞬间进程的状态**，如果要实时监控应该用top命令）  
1. 显示所有进程信息，连同命令行：ps -ef  
2. **查找特定进程，经常与ef一起使用：ps -ef|grep 进程名称**  
- pidof 命令：（感觉不是很好用）
- kill 命令和killall：配合进程的pid一起使用
- reboot 命令，重启，关机对应poweroff
- 根据端口获取pid：lsof -i:XX
- 查看端口是否被占用：netstat  -anp  |grep   XXX

##### 服务启停
- 启动服务：systemctl start 服务名称
- 重启服务：systemctl restart 服务名称
- 停止服务：systemctl stop 服务名称
- 加入到开机启动项：systemctl enable 服务名称
- 取消加入到开机启动项：systemctl disable 服务名称
- 查看服务状态：systemctl status 服务名称 

##### 密码
- 修改centos root密码：sudo -i passwd
##### 防火墙
- 增加防火墙黑名单：iptables -I INPUT -s 10.41.60.26 -j DROP
- 删除防火墙黑名单：iptables -D INPUT -s 10.41.60.26 -j DROP
- 关闭防火墙：systemctl stop firewalld.service
- 开放端口：firewall-cmd --zone=public --add-port=1883/tcp --permanent

##### 时间相关
- date 命令，用于**显示**及**设置**系统的时间或日期，  
date [选项] [指定的格式]    
1. 设置系统时间  
date -s "20160429 15:30:00"  该命令需要root权限  
2. 普通查看系统时间  
date  
3. 按照 “年-月-日 小时：分钟：秒” 的格式查看当前系统时间  
date “+%y-%m-%d %H:%M:%S”  

##### 下载安装
- wget 命令  
emsp;emsp;用于在终端中下载网络文件，wget 是一种下载工具，相当于迅雷，并且可以在用户退出系统的之后在后台执行，如果是由于网络的原因下载失败，wget会不断的尝试，直到整个文件下载完毕  
wget [选项] [URL]  
1. wget -O 647.zip http://XXX 下载文件并重命名为647.zip  
2. wget -b http://XXX  启动后转入后台  
- yum 命令：yum install 软件名称

##### 其他
- echo 命令，用于在终端输出字符串或变量提取后的值  
echo [字符串] [$变量]  

### 工作目录命令
- 显示用户当前所处的工作目录：pwd 
- 返回到上一次所处的目录：cd -
- 进入上一级目录：cd ..
- 返回当前用户根目录：cd ~
- 查看全部文件（包括隐藏文件）：ls -a
- 查看文件属性、大小等详细信息：ls -l

### 文本文件编辑命令
- 查看纯文本文件（内容较少的）:cat 文件名
- 查看纯文本文件顺带显示行号（内容较少的）：cat -n 文件名
- 用于查看纯文本文件（内容较多的）：more
- 用于查看纯文本文档的前 N 行：head 8 文件名







