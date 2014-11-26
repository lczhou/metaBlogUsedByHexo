title: openstack集群部署
date: 2014-11-25 16:38:15
category: openstack集群部署
tags: [cobbler, packstack, svmmaster]
toc: true
---

## 硬件准备

### [openstack](http://www.openstack.org/)非HA集群物理机包含的角色
>|角色|硬盘|网卡|
|-|-|-|
|控制节点|一块系统盘|eth0,eth1|
|计算节点|一块系统盘，一块SSD|eth0,eth1|
|存储节点|一块系统盘，五块数据盘|eth0,eth1|

### 对所有节点进行RAID配置

>|角色|RAID配置|
|-|-|-|
|控制节点|一块硬盘做RAID0，安装OS|
|计算节点|一块硬盘做RAID0，安装OS<br>一块SSD做RAID0，用作缓存|
|存储节点|一块硬盘做RAID0，安装OS<br>四块硬盘做RAID10，存放数据<br>一块硬盘做全局热备|

## 软件准备

### CentOS 7.0.1406 安装光盘

 - 下载[CentOS DVD ISO](http://mirrors.163.com/centos/7.0.1406/isos/x86_64/CentOS-7.0-1406-x86_64-DVD.iso)。
 - 用windows自带的刻录软件刻录光盘。

### [Cobbler](http://www.cobblerd.org/) 数据压缩包及配置压缩包

> |文件名|备注|
|-|-|
|etc.cobbler.tar.gz|解压生成目录`/etc/cobbler`，为cobbler的配置文件|
|var.www.cobbler.tar.gz|解压生成目录`/var/www/cobbler`，为cobbler的repo所在目录|
|var.lib.cobbler.tar.gz| 解压生成目录`/var/lib/cobbler`，为cobbler的kickstart所在目录|
|etc.yum.repos.d.cobbler-config.repo.tar.gz|解压生成文件`/etc/yum.repos.d/cobbler-config.repo`，为yum源文件|

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

 - 光驱中放入CentOS 7.0.1406 安装光盘。

 - 通过CentOS 7.0.1406 安装光盘安装CentOS 7 GNOME Desktop。

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
 - 关闭selinux(立即生效)。
```
setenforce Permissive
```

 - 修改配置/etc/sysconfig/selinux。
```
SELINUX=permissive
```

### 配置防火墙服务
 - 关闭并禁用firewalld服务。
```
systemctl stop firewalld
systemctl disable firewalld
```

 - 安装并启用iptables服务。
```
yum install iptables -y
systemctl start iptables
systemctl enable iptables
```

 - 初始化iptables规则。
```
iptables -F
iptables -X
iptables -Z
```

 - 保存iptables规则。
```
iptables-save > /etc/sysconfig/iptables
```

### 安装配置Cobbler

**配置临时yum源**

- 挂载光盘数据。
```
mkdir /mnt/centos7.0.1406
mount /dev/sr0 /mnt/centos7.0.1406
```

- 删除默认的yum源。
```
rm /etc/yum.repos.d/CentOS-*
```

- 创建yum repo文件`/etc/yum.repos.d/local.repo`。
```
[base]
name=base
baseurl=file:///mnt/centos7.0.1406
enable=1
gpgcheck=0
```

- 重新加载yum缓存。
```
yum clean all
yum makecache
```

**安装cobbler**

 - yum安装cobbler。
```
yum install cobbler cobbler-web -y
```

 - 解压cobbler相关数据和配置压缩包。
```
rm -rf /etc/cobbler
tar zxvf etc.cobbler.tar.gz -C /etc

rm -rf /var/lib/cobbler
tar zxvf var.lib.cobbler.tar.gz -C /var/lib

rm -rf /var/www/cobbler
tar zxvf var.www.cobbler.tar.gz -C /var/www
```

**修改yum源**

 - 删除临时yum源。
```
rm /etc/yum.repos.d/local.repo
```

- 解压yum repo文件`/etc/yum.repos.d/cobbler-config.repo`。
```
tar zxvf etc.yum.repos.d.cobbler-config.repo.tar.gz -C /etc/yum.repos.d
```

 - 重新加载yum缓存。
```
yum clean all
yum makecache
```

## 其他节点操作系统安装

> **NOTE:** 其他物理节点使用控制节点配置好的cobbler进行网络安装。

### 添加 cobbler system
 - 获取每个节点两块网卡的MAC地址。
 - 在控制节点上访问https://localhost/cobbler_web。
 - 通过Cobbler WebUI添加cobbler system(注：eth0所在网段为10.20.1.0/24，eth1所在网段为192.168.252.0/22)。

### 启动其他节点并从PXE引导
所有节点会通过PXE自动安装操作系统，等待所有节点安装完毕并重启。

## 存储节点配置

### 关闭并禁用NetworkManager
```
systemctl stop NetworkManager
systemctl disable NetworkManager
```

### 配置Linux bonding

**配置network scripts**

 - 创建eth0配置文件`/etc/sysconfig/network-scripts/ifcfg-eth0`。
```
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
# Settings for Bond
MASTER=bond0
SLAVE=yes
```

 - 创建eth1配置文件`/etc/sysconfig/network-scripts/ifcfg-eth1`。
```
DEVICE=eth1
BOOTPROTO=none
ONBOOT=yes
# Settings for Bond
MASTER=bond0
SLAVE=yes
```

- 创建bond0配置文件`/etc/sysconfig/network-scripts/ifcfg-bond0`。
```
DEVICE=bond0
BOOTPROTO=none
ONBOOT=yes
USERCTL=no
IPADDR=10.20.1.11
NETMASK=255.255.255.0
```

**配置bonding modprobe**

 - 创建modprobe配置文件`/etc/modprobe.d/bond.conf`。
```
/etc/modprobe.d/bond.conf
alias bond0 bonding
options bond0 mode=6 miimon=100
```

**重启使生效**


## 安装Openstack平台

### 控制节点上安装Packstack
```
yum install openstack-packstack
```

### 更新Packstack
```
rm -rf /usr/lib/python2.7/site-packages/packstack
tar zxvf packstack-icehouse.tar.gz -C /usr/lib/python2.7/site-packages
mv packstack-icehouse packstack
```

### 执行Packstack进行安装
```
packstack
```

> **NOTE:**
>
> - packstack命令会逐行提示用户输入配置参数，如：安装数据库服务的主机IP，安装计算服务的主机IP等等。
> - 如果执行失败，可以找到生成的answer file文件，重新指定该文件路径重新执行`packstack --answer-file /root/packstack-answers-XXX`即可。

### Openstack后续配置修改
 - 修改配额。
```
. /root/keystonerc_admin
nova quota-update --instances 1000 --cores 2000 \
--ram 5120000 --floating-ips 1000 --key-pairs 1000 \
`keystone tenant-list | grep admin | awk -F' ' '{print $2}'`
```

 - 修改安全组。
```
nova secgroup-add-rule default udp 1 65535 0.0.0.0/0
nova secgroup-add-rule default tcp 1 65535 0.0.0.0/0
nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
```


## Vegafs缓存相关配置

### Vegafs服务端(存储节点)配置

 - 修改文件`/var/lib/vegad/vols/image-volume/image-volume-fuse.vol`。
```
volume image-volume
    type performance/disk-cache
    option map-file-path /mnt/ssd/vegafs-disk-cache
    option cache-arena-size 220GB
    subvolumes image-volume-dht
end-volume
```

> **NOTE:** 选项cache-arena-size的大小与计算节点上/mnt/ssd/vegafs-disk-cache文件大小一致。

### Vegafs客户端(计算节点)配置
 - 初始化SSD盘。
```
yum -y install xfsprogs
parted /dev/sdb -s "mkpart primary 1 -1"
mkfs.xfs -f /dev/sdb1
uuid=`uuidgen`
xfs_admin -U $uuid /dev/sdb1
mkdir -p /mnt/ssd
echo "UUID=$uuid /mnt/ssd xfs defaults 0 0" >> /etc/fstab
mount -a
```

> **NOTE:** 确保设备/dev/sdb为SSD盘。

 - 初始化vegafs-disk-cache文件。
```
dd if=/dev/zero of=/mnt/ssd/vegafs-disk-cache bs=1M count=220K
```

 - 重新挂载vegafs image-volume。
```
umount /mnt/image-volume
mount -a
```

### 控制节点取消vegafs image-volume的挂载
 - 修改`/etc/fstab`。

## 安装后台管理系统Svmmaster

> **NOTE:** 以下操作也都在控制节点上执行。

### 安装JDK
```
yum install java-1.6.0-openjdk
```

### 安装Tomcat和Svmmaster
```
tar usr.local.tomcat6.0.tar.gz -C /usr/local
```

### 初始化数据库及数据库用户
```
tar zxvf root.svmmaster.sql -C /root
mysql -e "source /root/svmmaster.sql"
mysql -e "GRANT ALL PRIVILEGES ON svmmaster.* TO \
svmmaster@localhost IDENTIFIED BY 'svmmaster' \
WITH GRANT OPTION;"
mysql -e "GRANT ALL PRIVILEGES ON svmmaster.* TO \
svmmaster@10.20.1.2 IDENTIFIED BY 'svmmaster' \
WITH GRANT OPTION;"
mysql -e "GRANT ALL PRIVILEGES ON svmmaster.* TO \
svmmaster@"%" IDENTIFIED BY 'svmmaster' \
WITH GRANT OPTION;"
```

### 配置Svmmaster项目

 - 修改`/usr/local/tomcat6.0/webapps/svmmaster/WEB-INF/classes/config.properties`。
```
vmhttpIp=10.20.1.2
# 通过nova network-list查看net uuid
net_uuid=xxxx
```

 - 修改`/usr/local/tomcat6.0/webapps/svmmaster/WEB-INF/classes/monitor.properties`。
```
# node1,node2为计算节点的主机名
monitorItem=
node1,node2:cpuStat.cpu memStat.mem sda.disk sdb.disk /.par /home.par
```

### 配置Tomcat服务

**启动tomcat服务**
```
cd /usr/local/tomcat6.0
bin/startup.sh
```

**开机启动tomcat服务**

 - 配置rc.local可执行。
```
chmod +x /etc/rc.d/rc.local
```

 - 修改`/etc/rc.d/rc.local`。
```
cd /usr/local/tomcat6.0
bin/startup.sh
```

## 计算节点配置

> **NOTE:** 所有计算节点均需要执行以下操作

### 监控依赖的软件包
```
yum install sysstat smartmontools -y
```

### 修改libvirtd配置
 - 修改文件`/etc/libvirt/libvirtd.conf`。
```
listen_tls = 0
listen_tcp = 1
auth_tcp = "none"
```

 - 修改文件`/etc/sysconfig/libvirtd`。
```
LIBVIRTD_ARGS="--listen"
```

 - 重启libvirtd服务。
```
systemctl restart libvirtd
```

### 安装采集物理机信息所需的python脚本
```
tar zxvf home.pm.py.tar.gz -C /home
```

<br>

---
至此，可以通过地址`http://10.20.1.2:8080/svmmaster`对openstack平台进行管理了。
