---
title: AWS多可用区的VPC网络环境
date: 2023-10-12T16:47:12+08:00
tags: ["AWS", "VPC", "Subnet", "Internet Gateway", "Route Table"]
series: ["AWS基础"]
featured: true
---

本文介绍了AWS Region、Available zone、VPC及其Subnet、Internet Gateway和Route Table等概念，创建了一个多可用区的VPC，并介绍多可用区如何保证服务高可用性。

<!--more-->

## Region
AWS Region即AWS地区，是AWS提供的基础设施服务的物理部署地点。每个Region都是独立于其他Region，这样能够最大限度保证容错率和可用性。
## Available zone
AWS Available zone即AWS可用区，每个Region都拥有多个Available zone，每个Available zone都是独立的物理数据中心，拥有独立的电力、网络、安全等设施。
全球现拥有25个亚马逊云科技地理区域和81个可用区，如下图所示：

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/aws-region.png">}}

## VPC
AWS VPC即AWS虚拟私有网络，可以理解为在AWS云中创建了一块自己的专有网路环境。创建VPC时，需要指定VPC的CIDR。CIDR是一个IP地址段，CIDR的格式为：`x.x.x.x/x`，其中`x.x.x.x`为IP地址，`/x`为子网掩码，表示该IP地址段中有多少个IP地址。可以使用[cidr.xyz](https://cidr.xyz/)网站来计算CIDR的IP地址段和可用IP数量。
在`Asia Pacific (Singapore) ap-southeast-1`区域中创建VPC，输入`10.0.0.0/16`表示IP地址段`10.0.0.0`到`10.0.255.255`，共有65,536个IP地址。如下图所示：

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/aws-vpc.png" height="500">}}

### Subnet
子网是 VPC 内的 IP 地址范围。创建子网时，必须提供子网名称、CIDR和可用区（如果不指定，AWS会自动指定）。在添加子网后，您可以在 VPC 中部署 AWS 资源。子网可以公开或私有。公开子网内的资源可以有公共 IPv4 地址，可访问 Internet。私有子网内资源无法访问 Internet。
根据我们上面创建的VPC，我们设计了3个私有子网和3个公开子网，如下表所示：

| 子网名称             | CIDR          | 可用IP范围                    | 可用IP数量 | BASE IP    | BROADCAST IP |
|------------------|---------------|---------------------------|--------|------------|--------------|
| subnet-public-a  | 10.0.0.0/20   | 10.0.0.1 - 10.0.15.254    | 4094   | 10.0.0.0   | 10.0.15.255  |
| subnet-public-b  | 10.0.16.0/20  | 10.0.16.1 - 10.0.31.254   | 4094   | 10.0.16.0  | 10.0.31.255  |
| subnet-public-c  | 10.0.32.0/20  | 10.0.32.1 - 10.0.47.254   | 4094   | 10.0.32.0  | 10.0.47.255  | 
| subnet-private-a | 10.0.128.0/20 | 10.0.128.1 - 10.0.143.254 | 4094   | 10.0.128.0 | 10.0.143.255 |  
| subnet-private-b | 10.0.144.0/20 | 10.0.144.1 - 10.0.159.254 | 4094   | 10.0.144.0 | 10.0.159.255 | 
| subnet-private-c | 10.0.160.0/20 | 10.0.160.1 - 10.0.175.254 | 4094   | 10.0.160.0 | 10.0.175.255 |

在AWS console中创建3个公开子网和3个私有子网，下图是创建一个私有子网的示例：

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/aws-vpc-subnet.png" height="500">}}

### Internet Gateway
Internet Gateway 是 VPC 组件，可使 VPC 实例与 Internet 通信。如下图创建一个Internet Gateway。

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/aws-vpc-internet-gateway.png" height="500">}}

创建Internet Gateway后，需要将Internet Gateway与VPC进行关联，如下图所示：

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/attach-internet-gateway-to-vpc.png">}}

### Route Table
路由表是一组规则，用于确定网络流量的目的地。目前我们创建的公开子网，还不拥有访问Internet的能力，因为还没有配置路由规则。创建路由表并指定VPC

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/route-table-creation.png">}}

然后再将路由表和子网进行关联，如下图所示：

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/associate-route-table-to-subnet.png">}}

对于public路由表，需要添加一条路由规则，将目的地为`0.0.0.0/0`的流量指向Internet Gateway，如下图所示：

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/add-route-to-internet-gateway.png">}}

## AWS Console快速创建VPC

AWS console中提供了快速的创建VPC网络环境的功能，如下图所示：

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/aws-vpc-quick-create.png">}}

当配置好需要创建的资源后，点击`Create VPC`即可创建所有资源。

## 多可用区的VPC网络环境
经过上面的操作，我们已经创建了基于三个可用区的VPC网络环境。多可用区的VPC网络环境，是AWS服务高可用的基础。

当创建了多可用区的VPC网络环境后，我们可以在多个可用区中创建多个相同的资源。当某个可用区出现故障时，其他可用区的资源可以继续提供服务，从而保证了服务的高可用性。

需要注意的是如何配置应用在多个可用区之间的切换。以EC2实例为例，我们在多个可用区中创建相同的EC2实例，通过Application Load Balancer（ALB）分发流量。当某个可用区出现故障时，ALB会将流量切换到其他可用区中的EC2实例上，从而保证了服务的高可用性。

{{< figure src="/images/blog/aws-multiple-available-zones-vpc/aws-vpc-multi-az.png">}}

