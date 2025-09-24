# router-switch

实战架构

![拓扑图](https://github.com/shunetwork/router-switch/raw/main/多出口NAT 策略路由案例/拓扑图/拓扑.jpg)


多出口NAT 策略路由

出口路由关键配置步骤如下:


1、定义网段的access-list

```shell
access-list 10 permit 192.168.10.0 0.0.0.255
```

```shell
access-list 20 permit 192.168.20.0 0.0.0.255
```

2、定义分流的route-map

```shell
route-map fenliu permit 10
match ip address 10
set ip next-hop 50.1.1.1 60.1.1.1
```
!         
```shell
route-map fenliu permit 20
match ip address 20
set ip next-hop 60.1.1.1 50.1.1.1
```
!  

PBR (fenliu) 根据 ACL 对内部网段分流，不同网段走不同的 ISP。

3、Route-map for NAT 匹配接口

```shell
route-map dianxin permit 10
match interface Serial1/0
```

```shell
route-map liantong permit 20
match interface Serial1/1
```


4、 NAT 规则

```shell
ip nat inside source route-map dianxin interface Serial1/0 overload
ip nat inside source route-map liantong interface Serial1/1 overload
```

含义：

使用 route-map 来决定 NAT 转换时匹配哪些流量，并分别走不同的出口接口。

overload 表示 端口复用（PAT），多个内部主机可以共用一个公网 IP 地址。

第一条：符合 dianxin 的流量做 NAT，公网 IP 使用 Serial1/0 接口地址。

第二条：符合 liantong 的流量做 NAT，公网 IP 使用 Serial1/1 接口地址。

这里的设计目的是 双 ISP 出口 NAT：电信和联通分别作为上网出口。

