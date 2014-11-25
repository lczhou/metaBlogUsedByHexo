title: openstack集群部署
date: 2014-11-25 16:38:15
category: openstack集群部署
tags: [cobbler, packstack, svmmaster]
---

## 硬件准备

[openstack](http://www.openstack.org/)非HA集群物理机包含以下角色：

>|角色|硬盘|网卡|
|-|-|-|
|控制节点|一块系统盘|eth0,eth1|
|存储节点|一块系统盘，一块SSD|eth0,eth1|
|存储节点|一块系统盘，若干数据盘|eth0,eth1|

## 软件准备

### CentOS 7.0.1406 安装光盘

 - 下载[CentOS DVD ISO](http://mirrors.163.com/centos/7.0.1406/isos/x86_64/CentOS-7.0-1406-x86_64-DVD.iso)
 - 用windows自带的刻录软件刻录光盘

### [Cobbler](http://www.cobblerd.org/) 数据压缩包及配置压缩包

> |文件名|备注|
|-|-|
|etc.cobbler.tar.gz|解压生成目录`/etc/cobbler`，为cobbler的配置文件|
|var.www.cobbler.tar.gz|解压生成目录`/var/www/cobbler`，为cobbler的repo所在目录|
|var.lib.cobbler.tar.gz| 解压生成目录`/var/lib/cobbler`，为cobbler的kickstart所在目录|
|etc.yum.repos.d.cobbler-config.repo.tar.gz|解压生成文件/etc/yum.repos.d/cobbler-config.repo，为yum源文件|

### Packstack 软件压缩包

>|文件名|备注|
|-|-|-|
|packstack-icehouse.tar.gz|解压生成目录`/usr/lib/python2.7/site-packages/packstack-icehouse`，为packstack命令所在目录|

### Svmmaster 软件包及相关文件

>|文件名|备注|
|-|-|
|usr.local.tomcat6.0.tar.gz|解压生成目录`/usr/local/tomcat6.0`，为tomcat web server和项目svmmaster所在目录|
|svmmaster.sql|通过该文件可生成svmmaster依赖的数据库表|
|pm.py|svmmaster采集物理机信息所需的python脚本|


## 控制节点安装

### 安装操作系统

 - 光驱中放入CentOS 7.0.1406 安装光盘

 - 通过CentOS 7.0.1406 安装光盘安装CentOS 7 GNOME Desktop

### 配置网络
 - 管理网络
接口：eth0
地址： 10.20.1.2/24
网关：0.0.0.0

 - Public网络
接口：eth1
地址：192.168.252.2/22
网关：192.168.2.1

> **NOTE:**
> 
>  - 如果只有一块网卡，管理网络和Public网络可以配置在同一个接口上。
>  -  可以使用`nmtui`进行网络的配置。
>  

### 关闭Selinux
 - 立即关闭
```
setenforce Permissive
```

 - 修改配置/etc/sysconfig/selinux
```
SELINUX=permissive
```

### 配置防火墙服务
 - 关闭并禁用firewalld服务
```
systemctl stop firewalld
systemctl disable firewalld
```

 - 安装并启用iptables服务
```
yum install iptables -y
systemctl start iptables
systemctl enable iptables
```

 - 初始化iptables规则 
```
iptables -F
iptables -X
iptables -Z
```

 - 保存iptables规则
```
iptables-save > /etc/sysconfig/iptables
```

### 安装配置Cobbler

**配置临时yum源**

- 挂载光盘数据
```
mkdir /mnt/centos7.0.1406
mount /dev/sr0 /mnt/centos7.0.1406
```

- 删除默认的yum源
```
rm /etc/yum.repos.d/CentOS-*
```

- 创建yum repo文件`/etc/yum.repos.d/local.repo`
```
[base]
name=base
baseurl=file:///mnt/centos7.0.1406
enable=1
gpgcheck=0
```

- 重新加载yum缓存
```
yum clean all
yum makecache
```

**安装cobbler**

 - yum安装cobbler
```
yum install cobbler cobbler-web -y
```

 - 解压cobbler相关数据和配置压缩包
```
rm -rf /etc/cobbler
tar zxvf etc.cobbler.tar.gz -C /etc

rm -rf /var/lib/cobbler
tar zxvf var.lib.cobbler.tar.gz -C /var/lib

rm -rf /var/www/cobbler
tar zxvf var.www.cobbler.tar.gz -C /var/www
```

**修改yum源**

 - 删除临时yum源
```
rm /etc/yum.repos.d/local.repo
```

- 解压yum repo文件`/etc/yum.repos.d/cobbler-config.repo`
```
tar zxvf etc.yum.repos.d.cobbler-config.repo.tar.gz -C /etc/yum.repos.d
```

 - 重新加载yum缓存
```
yum clean all
yum makecache
```

## 其他节点操作系统安装

> **NOTE:** 其他物理节点使用控制节点配置好的cobbler进行网络安装

### 添加 cobbler system
 - 获取每个节点两块网卡的MAC地址
 - 在控制节点上访问https://localhost/cobbler_web
 - 通过Cobbler WebUI添加cobbler system(注：eth0所在网段为10.20.1.0/24，eth1所在网段为192.168.252.0/22) 

### 启动其他节点并从PXE启动

### 等待所有节点网络安装完毕

## 存储节点配置

### 关闭并禁用NetworkManager
```
systemctl stop NetworkManager
systemctl disable NetworkManager
```

### 配置Linux bonding

**配置network scripts**

 - 创建eth0配置文件`/etc/sysconfig/network-scripts/ifcfg-eth0`
```
DEVICE=eth0
BOOTPROTO=none 
ONBOOT=yes 
# Settings for Bond 
MASTER=bond0 
SLAVE=yes
```

 - 创建eth1配置文件`/etc/sysconfig/network-scripts/ifcfg-eth1`
```
DEVICE=eth1
BOOTPROTO=none 
ONBOOT=yes 
# Settings for Bond 
MASTER=bond0 
SLAVE=yes
```

- 创建bond0配置文件`/etc/sysconfig/network-scripts/ifcfg-bond0`
```
DEVICE=bond0 
BOOTPROTO=none 
ONBOOT=yes
USERCTL=no
IPADDR=10.20.1.11
NETMASK=255.255.255.0 
```

**配置bonding modprobe**

 - 创建modprobe配置文件`/etc/modprobe.d/bond.conf`
```
/etc/modprobe.d/bond.conf
alias bond0 bonding
options bond0 mode=6 miimon=100
```

**重启使生效**