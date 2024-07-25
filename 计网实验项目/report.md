# Report

## 基础架构
### 简介 

![项目整体架构设计](/Final-Source/整体架构设计/架构设计.png)

上图显示了项目整体网络拓扑结构的整体架构，主要分为五个部分，中间部分为核心路由器，左上角区域为服务器部分，右上角区域显示了外网连接，左侧为无线网络的相应连接，下侧部分显示了VLAN1-VLAN5，分别对应了我们设计的软件学院、电信学院、后勤部、教务处和科研部门，其中前四个部分相互连通，科研部分独立于这四个部分，但有权限访问其所有内容。

### 基础连接


## 功能设计

### DHCP 
在我们的项目中，VLAN 1-VLAN 5部分除了服务器外的所有部分均采用DHCP来动态分配IP地址，服务器部分中的学校DHCP服务器是实现这一功能的核心组件。


下图为我们配置的DHCP POOL 
![DHCP池](/Final-Source/功能设计/DHCP池.png)
其中显示了各个部门的动态地址分配的相应范围

为了实现DHCP功能，需要在相应的路由器和交换机上进行如下配置：

在router 1路由器上进行如下配置
![router1](/Final-Source/功能设计/router1.png)

在router 0路由器上进行如下配置
![router0](/Final-Source/功能设计/router0.png)

在router 2路由器上进行如下配置
![router2](/Final-Source/功能设计/router2.png)

在核心交换机上进行如下配置
![核心交换机1](/Final-Source/功能设计/核心交换机1.png)
![核心交换机2](/Final-Source/功能设计/核心交换机2.png)
配置后即可实现对相应部件IP地址的动态分配
### 内部通过DNS访问公共服务器
在项目中DNS的实现主要在于服务器部分的配置

首先我们在学校DHCP服务器配置了DNS Server 
![DHCP池](/Final-Source/功能设计/DHCP池.png)
之后在学校DNS服务器中，我们进行了如下配置
![DNS](/Final-Source/功能设计/DNS.png)
将www.baidu.com网址映射为172.16.1.10地址

经过了上述配置后，各个部门的主机和其他设备在访问学校web服务器时，可以通过域名访问。
### 部门间的隔离(科研部门)与互访(其它部门)

使用了三层交换机的技术，在二层交换机将多个物理网络分割成多个 vlan，并且在三层交换机上面配置了 RIP，实现了网络的动态路由，并且可以在此交换机上使用 ACL 相关的配置。

它现在可以像路由器一样，在不同的子网 vlan 之间进行数据包转发。通过这种方式，可以实现不同 vlan 之间的连通。

![ACL-related](/Final-Source/科研部门隔离/acl.png)

不允许任何源地址向 192.168.5.0/24 发送任何协议。允许任何源地址向 192.168.5.0/24 发送 icmp 协议的回复。

![ACL-related](/Final-Source/科研部门隔离/接口acl.png)

在三层交换机上配置如图的命令，将 ACL 应用到 VLAN 50 接口的出站流量上。

通过以上两个命令，可以实现科研部门可 ping 通其它所有机器，但是其它部门无法 ping 通科研部门。

![result](/Final-Source/科研部门隔离/可以连通外部.png)

![result](/Final-Source/科研部门隔离/外部无法连通.png)

但是，由于限制了 TCP、UDP 等除 ICMP 以外的其它协议，因此可能目前不能使用这个访问互联网。

### 无线WIFI接入功能
在实现过程中，我们对Wireless Router0 进行了如下配置
![wifi](/Final-Source/功能设计/wifi.png)
设置IP Address为192.198.0.1/24
之后在config wireless中设置了wifi名为TJ-WiFi，密码为1234567890
![wifi2](/Final-Source/功能设计/wifi2.png)
进行了上述配置后，打开smartphone，输入WiFi名和密码后即可连接到无线网
![wifi3](/Final-Source/功能设计/wifi3.png)

### 外网接口以及NAT配置

![NAT-related](/Final-Source/外网nat接口/list1.jpg)

这个访问列表允许 IP 范围 192.168.0.0/16 的地址通过 NAT 规则进行转换。

![NAT-related](/Final-Source/外网nat接口/nat设置.png)

如上图所示：

* 外部接口: Serial0/3/0 用来连接互联网
* 内部接口: FastEthernet0/0, FastEthernet0/1

以上所有命令指定了使用过载（PAT - 端口地址转换）的动态 NAT。它将访问列表1中的地址转换为 Serial0/3/0 接口的 IP 地址。

这样，内部访问外部的 IP 均被视为为一个 IP 地址发出的请求。如下图所示：

![result](/Final-Source/外网nat接口/result.png)

### 外网可访问Web服务器

![ACL-related](/Final-Source/外网可访问的web/list101.jpg)

这些 ACL 的设置可以实现拒绝外网访问内网，只能访问 Web 服务器。

允许所有到 100.64.1.0/24 的 IP 流量，拒绝所有到 192.168.0.0/16 的 IP 流量，拒绝所有到 172.16.1.0/24 的 IP 流量。

![ACL-related](/Final-Source/外网可访问的web/接口ACL.png)

将 ACL 应用到 s0/3/0 接口的入站流量上。

![NAT-related](/Final-Source/外网nat接口/nat设置.png)

图中的最后一行表示一个静态 NAT 规则，将内部 IP 地址 172.16.1.10 映射到外部 IP 地址 100.64.1.15。ACL 中不允许访问 192.168.0.0/16 以及 172.16.1.0/24 的所有 IP，可以实现对内部网络访问的拒绝，同时只允许访问 100.64.1.0/24，这个地址为 NAT 设置的静态的映射，可以访问内部的 Web 服务器。

最终结果如下：

![result](/Final-Source/外网可访问的web/result-trans.png)

上图可以看出，NAT 实现了预期的功能。

![result](/Final-Source/外网可访问的web/result-web.png)

如图，外网可以访问公共的 Web 服务器。
