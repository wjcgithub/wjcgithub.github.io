---
layout: post
title: lvs
categories: Blog
description: lvs
keywords: 
---

# lvs nat, ip-tun, dr
@(linux)[负载均衡, gateway, 网关]

`vip`: lvs 服务器所在ip  （`就是外界需要访问的一个ip地址`）
`rip`:  rip  real ip 真实ip （）

### 一块网卡上配置多块IP
例如`eth0`网卡上配置多个ip
```
eth0:1
eth0:1
eth0:2
```

### lvs NAT 模式
- ![Alt text](./QQ截图20160506142405.png)

- ![Alt text](./QQ截图20160505194603.png)
> 需要开启`linux`上的路由管道
> `优点`：只要一个vip为外网地址就够了
> `缺点`：请求和响应都要过lvs ，这样lvs 也会成为瓶颈
> 

### lvs IP-TUN ip隧道模式
- ![Alt text](./QQ截图20160506142324.png)

- ![Alt text](./QQ截图20160505194410.png)
- ![Alt text](./QQ截图20160506130654.png)
> `优点`：数据回来不用过lvs 
> `优点`：lvs 通过ip隧道把`client`请求发送给后面的`real server`,然后由`real server`直接响应给client
> `缺点`：需要每台real server上都必须是公网ip，只有公网ip才能和`client`通信
> `缺点`：不是所有的服务器都具备tun0这种协议的网卡

### 通过子网掩码查看一个ip地址所在网段
- ![Alt text](./QQ截图20160505195735.png)

#### ip-tun 实验规划
```
directorserver 192.158.10.1
realserver 192.168.10.2
realserver 192.168.10.3
vip 192.168.10.10

ifconfig tunl0 192.168.10.10 netmask 155.255.255.255 up
route add-host 192.168.10.10 dev tunl0
echo "1" > /proc/sys/net/ipv4/conf/tunl0/arp_ignore       忽略arp请求
echo "2" > /proc/sys/net/ipv4/conf/tunl0/arp_announce     
echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore
echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce 
```

#### 关闭 iptable  selinux
```
sudo /etc/init.d/iptables stop

1 查看selinux 是否开启
sestatus
getenforce

2、关闭selinux
2.1:永久性关闭（这样需要重启服务器后生效）
# sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

2.2:临时性关闭（立即生效，但是重启服务器后失效）
# setenforce 0 #设置selinux为permissive模式（即关闭）
# setenforce 1 #设置selinux为enforcing模式（即开启）


```

### lvs DR模式（直接路由模式）
- ![Alt text](./QQ截图20160506163145.png)
- ![Alt text](./QQ截图20160506142859.png)
> `优点`：数据回来不用过lvs 
> `优点`：lvs 通过改写Mac地址把`client`请求发送给后面的`real server`,然后由`real server`直接响应给client
> `缺点`：需要每台real server上都必须是公网ip，只有公网ip才能和`client`通信
> `缺点`：要求调度器也真是服务器都有一块网卡连在同一个物理网段上

### 模拟实验
##### 1. 准备三台机器
> > `lvs`  `eth0`:192.168.1.241   (`这个是client 需要访问的地址`)    `eth1`:192.168.2.251   (`这个是real server 的 gateway`)
> > `web1`  192.168.2.250
> > `web2`  192.168.2.249
> > `client` 192.168.1.110    `client 和 lvs 的 eth0 网卡 ip 在一个网段`
##### 2. 给里面的两个real server 添加网关
`route add default gw 192.168.2.251`
`route -n  查看网关`

##### 2. vmnet0   vmnet1  vmnet8 的说明
[网络原理，以及对VMware Workstation虚拟网络VMnet0、VMnet1、VMnet8的图解](http://blog.csdn.net/adultf/article/details/7290999)

1. `vmnet0`  bridge
> 在桥接模式下，VMware虚拟出来的操作系统就像是局域网中的一独立的主机，它可以访问网内任何一台机器。不过你需要多于一个的IP地址，并且需要手工为虚拟系统配置IP地址、子网掩码，而且还要和宿主机器处于同一网段，这样虚拟系统才能和宿主机器进行通信。如果你想利用VMware在局域网内新建一个虚拟服务器，为局域网用户提供网络服务，就应该选择桥接模式。

2. `vmnet1` host only
> 在某些特殊的网络调试环境中，要求将真实环境和虚拟环境隔离开，这时你就可采用Host-only模式。在Host-only模式中，所有的虚拟系统是可以相互通信的，但虚拟系统和真实的网络是被隔离开的。

3. `vmnet8` Nat
>在NAT网络中，会使用到VMnet8虚拟交换机，Host上的VMware Network Adapter VMnet8虚拟网卡被连接到VMnet8交换机上，来与VPC进行通信，但是VMware Network Adapter VMnet8虚拟网卡仅仅是用于和VMnet8虚拟交换机网段通信用的，它并不为VMnet8网段提供路由功能，处于虚拟NAT网络下的VPC是使用虚拟的NAT服务器连接的Internet的。

#### 配置ip
`sudo ifconfig eth0 192.168.2.251 netmask 255.255.0.0`

#### 开始配置LVS SERVER
```
sudo yum install -y ipvsadm*
echo 1 > /proc/sys/net/ipv4/ip_forward         ## 开启路由管道功能
ipvsadm -C     ## 将以前调度器里面的所有转换表清空
ipvsadm -At 192.168.1.241 -s rr    开启rr         ## -A 增加一个具有调度算法的转换表， -s  算法   
	## 当访问 192.168.1.241 ip 地址  执行 ss  调度算法
ipvsadm -at 192.168.1.241 -r 192.168.2.249 -m    ## 添加被轮训的服务器  -r real  -m 轮叫模式(伪造Masquerade) nat  -g  gate(dr)  -i ip tun
ipvsadm -at 192.168.1.241 -r 192.168.2.250 -m    ## 添加被轮训的服务器
```

#### 测试实验环境是否都能通过
`client`-->`lvs`  ======  `ping 192.168.1.241`

### lvs 调度算法 
> 1. 轮叫  `RR`
> 2. 加权轮叫  `WRR`
> 3. 最少链接 Least Connections   `LC`  加权最少链接 针对服务器比较强的服务器，多连点  `WLC`


### 常见问题
同主机之间不通
1. iptables 关闭
2. selinux  关闭
3. 是不是用的同一个虚拟网络
4. netmask  是否对，这个是当访问一个ip的时候，确定这个ip是否和当前主机ip在一个网段内， 不在的话就要将请求发往`gateway`
5. ip 地址是否在一个网段
6. 不同网段的时候是不是 `gateway` 没有，或者不对


## Dr实验配置

client_ip  : 随意
lvs: 172.16.146.110
lvs_dr service  172.16.146.130
real service1  172.16.146.131
real service2 172.16.146.132


### lvs脚本 
```powershell
directorserver 192.168.10.1  netmask: 255.255.255.0
vip 192.168.10.10　　　netmask: 255.255.255.255
realserver 192.168.10.2   netmask: 255.255.255.0
realserver 192.168.10.3   netmask: 255.255.255.0
```

### lvs　server（一个网卡配置多个ip）
```powershell
ifconfig eth0:0 200.168.10.10 netmask 255.255.255.255
route add -host 200.168.10.10 dev eth0:0
ipvs:
ipvsadm -At 200.168.10.10:80 -s rr
ipvsadm -at 200.168.10.10:80 -r 200.168.10.2 -g
ipvsadm -at 200.168.10.10:80 -r 200.168.10.3 -g
ipvsadm 
```

### Real　Server配置
```powershell
ifconfig lo:0 200.168.10.10 netmask 255.255.255.255 up
route add -host  200.168.10.10 dev lo:0
echo "1" > /proc/sys/net/ipv4/conf/lo/arg_ignore
echo "2" > /proc/sys/net/ipv4/conf/lo/arg_announce
echo "1" > /proc/sys/net/ipv4/conf/all/arg_ignore
echo "2" > /proc/sys/net/ipv4/conf/all/arg_announce
```