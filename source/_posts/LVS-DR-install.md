---
title: LVS-DR模式安装与配置
date: 2016-05-09 15:40:00
categories: linux
tags: python
---


### 文章首发站点: [OpensGalaxy.com](http://opensgalaxy.com)

## 一、介绍
## 什么是LVS(Linux Virtual Server)？
Linux虚拟服务器是一个高度可扩展的、高可用的服务器建立在真实服务器的集群和负载均衡，在Linux操作系统上运行。服务器集群体系结构对用户是完全透明的，使用户像访问一台服务器一样。
## LVS的应用
Linux虚拟服务器作为一种先进的负载均衡解决方案可用于构建高度可扩展的、高可用的网络服务，如Web、FTP、邮件、缓存、媒体和VoIP服务。
## LVS负载均衡模式
VS/NAT、VS/TUN和VS/DR
本文将介绍LVS-DR（Direct Routing）模式的安装与配置，VS/DR模式的拓扑图: 
![image](http://opensgalaxy.oss-cn-beijing.aliyuncs.com/images/lvs/01142525-56dfb6e2f4794ba78b075f527b835994.jpg)

该图是官方提供的VS/DR体系结构图，在VS/DR模式下，Load Balancer和所有的Real Server在物理上有一个网卡通过不分断的局域网相连，调度器和真实服务器必须绑定同一VIP，该VIP在调度器上对外可见，而真实服务器上只需将VIP配置在Non-ARP网络设备上，它对外不可见，只是用于欺骗真实服务器用于处理目标地址为VIP的网络请求。真实服务器将请求处理后，直接返回给用户，不需要在通过调度器返回，所以在VS/DR模式下，真实服务器的网关地址不需要指向调度器。
## 二、安装ipvs
1、检查Load Balancer服务器是否已支持ipvs。

```[root@YumServer ~]# modprobe -l |grep ipvs```

若有类似以下输出，则表示服务器已支持ipvs: 

2、检查是否有必须的依赖包：Kernel-devel、gcc、openssl、openssl-devel、popt 

![image](http://opensgalaxy.oss-cn-beijing.aliyuncs.com/images/lvs/modprobe.tiff)

如果没有包，可以使用yum install 安装。

![image](http://opensgalaxy.oss-cn-beijing.aliyuncs.com/images/lvs/rpm.tiff)

3、在Load Balancer服务器上安装ipvsadm
直接使用YUM源安装

```[root@YumServer ~]# yum install ipvsadm```

或是到官网下载:http://www.linuxvirtualserver.org 注意要根据自己的内核版本选择对应的ipvsadm版本。本文使用YUM安装。

查看ipvsadm版本

```[root@YumServer ~]# ipvsadm -v
ipvsadm v1.26 2008/5/15 (compiled with popt and IPVS v1.2.1)```



## 三、配置
----

环境: 

* Load Balancer(调度器)

IP(eth0):  10.10.10.215/24

VIP(eth0:0): 10.10.10.213/32

GATEWAY: 10.10.10.254

* Real Server1

IP(eth0): 10.10.10.216/24

VIP(lo:0): 10.10.10.213/32

* Real Server2

IP(eth0): 10.10.10.217/24

VIP(lo:0):  10.10.10.213/32

拓扑图如下: 

![image](http://opensgalaxy.oss-cn-beijing.aliyuncs.com/images/lvs/lvslan.png)

### (一) Load Balancer(调度器)

1、eth0 本机ip

10.10.10.215

2、添加VIP 10.10.10.213

```

cp ifcfg-eth0 ifcfg-eth0:0

#编辑结果如下: 

[root@YumServer network-scripts]# cat ifcfg-eth0:0

DEVICE="eth0:0"

BOOTPROTO="static"

NM_CONTROLLED="yes"

ONBOOT="yes"

IPADDR=10.10.10.213

NETMASK=255.255.255.255

TYPE="Ethernet"

DNS1=10.10.10.20

DNS2=10.10.10.21

```

3、编辑配置内核参数

添加如下配置: 

```
# 打开ip转发
net.ipv4.ip_forward = 1
# 关闭路由重定向功能
net.ipv4.conf.all.send_redirects = 0  
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.eth0.send_redirects = 0

```
配置生效

```
sysctl -p

```

4、添加负载均衡配置(轮询算法，默认是权重相同)

```
ipvsadm -A -t 10.10.10.213:80 -s rr
ipvsadm -a -t 10.10.10.213:80 -r 10.10.10.216:80 -g -w 1
ipvsadm -a -t 10.10.10.213:80 -r 10.10.10.217:80 -g -w 1

```
5、保存配置

```

[root@YumServer ~]# service ipvsadm save

或者是直接编辑文件添加
[root@YumServer ~]# cat /etc/sysconfig/ipvsadm
-A -t 10.10.10.213:80 -s rr
-a -t 10.10.10.213:80 -r 10.10.10.216:80 -g -w 1
-a -t 10.10.10.213:80 -r 10.10.10.217:80 -g -w 1

```
###（二）RealServer配置

1、eth0 本机ip

10.10.10.216

2、添加VIP 10.10.10.213

```

cp ifcfg-eth0 ifcfg-lo:0

#编辑结果如下：

[root@RealServer01 html]# cat /etc/sysconfig/network-scripts/ifcfg-lo:0
DEVICE=lo:0
IPADDR=10.10.10.213
NETMASK=255.255.255.255
ONBOOT=yes

```

3、编辑配置内核参数

添加如下配置：

```
net.ipv4.ip_forward = 1
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.lo.arp_announce = 2
```
---
> ### 有关arp_ignore的相关介绍：
> arp_ignore: 定义对目标地址为本地IP的ARP询问不同的应答模式0 

> 0 - (默认值): 回应任何网络接口上对任何本地IP地址的arp查询请求 

> 1 - 只回答目标IP地址是来访网络接口本地地址的ARP查询请求 

> 2 -只回答目标IP地址是来访网络接口本地地址的ARP查询请求,且来访IP必须在该网络接口的子网段内 

> 3 - 不回应该网络界面的arp请求，而只对设置的唯一和连接地址做出回应 

> 4-7 - 保留未使用 

> 8 -不回应所有（本地地址）的arp查询

> ### 有关arp_announce的相关介绍：

> arp_announce:对网络接口上，本地IP地址的发出的，ARP回应，作出相应级别的限制: 确定不同程度的限制,宣布对来自本地源IP地址发出Arp请求的接口 

> 0 - (默认) 在任意网络接口（eth0,eth1，lo）上的任何本地地址 

> 1 -尽量避免不在该网络接口子网段的本地地址做出arp回应. 当发起ARP请求的源IP地址是被设置应该经由路由达到此网络接口的时候很有用.此时会检查来访IP是否为所有接口上的子网段内ip之一.如果改来访IP不属于各个网络接口上的子网段内,那么将采用级别2的方式来进行处理. 

> 2 - 对查询目标使用最适当的本地地址.在此模式下将忽略这个IP数据包的源地址并尝试选择与能与该地址通信的本地地址.首要是选择所有的网络接口的子网中外出访问子网中包含该目标IP地址的本地地址. 如果没有合适的地址被发现,将选择当前的发送网络接口或其他的有可能接受到该ARP回应的网络接口来进行发送.



----
配置生效

```
sysctl -p

```

4、添加路由

```
route add -host 10.10.10.213 dev lo:0

```

### RealServer01 配置完毕,RealServer02除本机ip不同外，其他配置一致。

##(三)验证


* 两台RealServer服务器都安装http服务用于验证负载均衡设置是否生效（分别放置可以显示RealServer1和RealServer2的html页面，用于验证负载均衡效果）如下图: 

![image](http://opensgalaxy.oss-cn-beijing.aliyuncs.com/images/lvs/rs1.tiff)

刷新后出现: 

![image](http://opensgalaxy.oss-cn-beijing.aliyuncs.com/images/lvs/rs2.tiff)

查看请求分发情况

![image](http://opensgalaxy.oss-cn-beijing.aliyuncs.com/images/lvs/ipvsadm.tiff)


## 附录（关于出现SYN_RECV状态）

刚搭建完成lvs时，发现刷新好半天，只请求到了其中一台服务器，而我明明使用的是轮询，通过查看连接情况发现：

```
[root@YumServer network-scripts]# ipvsadm -lnc 
IPVS connection entries
pro expire state       source             virtual            destination
TCP 01:48  FIN_WAIT    10.10.10.22:3065   10.10.10.213:80    10.10.10.217:80
TCP 01:51  FIN_WAIT    10.10.10.22:3095   10.10.10.213:80    10.10.10.217:80
TCP 01:50  FIN_WAIT    10.10.10.22:3082   10.10.10.213:80    10.10.10.217:80
TCP 00:49  SYN_RECV    10.10.10.22:3068   10.10.10.213:80    10.10.10.216:80
TCP 01:55  FIN_WAIT    10.10.10.22:3129   10.10.10.213:80    10.10.10.217:80
TCP 01:56  FIN_WAIT    10.10.10.22:3149   10.10.10.213:80    10.10.10.217:80
TCP 01:45  FIN_WAIT    10.10.10.22:3048   10.10.10.213:80    10.10.10.217:80
TCP 00:58  SYN_RECV    10.10.10.22:3164   10.10.10.213:80    10.10.10.216:80
TCP 01:47  FIN_WAIT    10.10.10.22:3057   10.10.10.213:80    10.10.10.217:80
TCP 00:56  SYN_RECV    10.10.10.22:3137   10.10.10.213:80    10.10.10.216:80
TCP 00:57  SYN_RECV    10.10.10.22:3152   10.10.10.213:80    10.10.10.216:80
TCP 01:57  FIN_WAIT    10.10.10.22:3153   10.10.10.213:80    10.10.10.217:80
TCP 00:53  SYN_RECV    10.10.10.22:3105   10.10.10.213:80    10.10.10.216:80
TCP 01:54  FIN_WAIT    10.10.10.22:3114   10.10.10.213:80    10.10.10.217:80
TCP 01:56  FIN_WAIT    10.10.10.22:3140   10.10.10.213:80    10.10.10.217:80
TCP 00:51  SYN_RECV    10.10.10.22:3079   10.10.10.213:80    10.10.10.216:80
TCP 01:55  FIN_WAIT    10.10.10.22:3127   10.10.10.213:80    10.10.10.217:80
TCP 00:50  SYN_RECV    10.10.10.22:3074   10.10.10.213:80    10.10.10.216:80
TCP 00:54  SYN_RECV    10.10.10.22:3120   10.10.10.213:80    10.10.10.216:80

```
会出现大量的SYN_RECV状态，SYN-RECV 代表LB已经发起SYN请求，realserver 无响应或有异常。
请求已经发出，但是realserver无响应？所以查看realserver的apache配置发现，我选择监听的是realserver的本机ip

```
[root@RealServer01 conf]# cat httpd.conf  |grep Lis
Listen 10.10.10.216:80

```
是这个配置导致了错误，更改成如下配置后，状态正常。

```
[root@RealServer01 conf]# cat httpd.conf  |grep Lis
Listen 10.10.10.213:80

```
或者

```
[root@RealServer01 conf]# cat httpd.conf  |grep Lis
Listen 80

```
