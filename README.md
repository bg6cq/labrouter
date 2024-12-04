# 实验室网络NAT出口的配置

## 1. 简介

实验室内部网络NAT出口设备，需要如下功能：
```
IPv4 NAT, IPv6桥接
网络NAT日志
DHCP
```
本文给出用Rocky 8 Linux用过实验室网络NAT出口设备的配置方法。

硬件设备，建议购置N100 CPU的软路由，taobao上可以在1000元以内。

## 2. 基础系统安装

### 2.1 U盘准备

下载balenaEtcher，下载 https://mirrors.ustc.edu.cn/rocky/8/isos/x86_64/ 中的 Rocky-8.10-x86_64-minimal.iso(2.7G，直接安装) 或 Rocky-x86_64-boot.iso(1G，安装时必须联网下载软件包)，写入U盘。

### 2.2 启动安装

启动后，按DELETE键进入BIOS，选择U盘启动，进入安装界面

### 2.3 启动网络

如果使用的Rocky-x86_64-boot.iso，首先需要启动网络，正常上网。

### 2.4 选择 安装的磁盘，最小安装，设置密码，开始安装


## 4. 操作系统修改

### 4.1 修改网卡命名方式为eth0、eth1

```
cp -f /etc/default/grub /etc/default/grub.bak
sed -i -e 's/quiet"/quiet net.ifnames=0"/' /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
```

### 4.2 修改时区
```
rm -f /etc/localtime
ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 4.3 启用Devel 软件包
```
#enable Devel rep
sed -i -e "s/enabled=0/enabled=1" /etc/yum.repos.d/Rocky-Devel.repo
```

### 4.4 修改日志保存时间、禁用SElinux
```
sed -i -e "s/rotate 4/rotate 40/" /etc/logrotate.conf
sed -i -e "s/=enforcing/=disabled/" /etc/selinux/config
```

### 4.5 安装软件包

在网络畅通的情况下
```
yum update -y
yum install -y epel-release gcc git lz4-devel openssl-devel dialog
yum install -y tcpdump telnet traceroute gd-devel httpd make m4
yum install -y libpcap-devel lm_sensors bind-utils net-tools dhcp-server
yum install -y bridge-utils conntrack-tools mtr tar autoconf 
yum install -y automake libtool indent
```

### 4.6 安装简单软件
```
# 网卡流量显示
cd /usr/src
git clone https://git.ustc.edu.cn/james/traffic.git
cd /usr/src/traffic
make
cp traffic.html /var/www/html/index.html


# 日志记录
mkdir /natlog

cd /usr/src
git clone https://git.ustc.edu.cn/james/gzlog.git
cd /usr/src/gzlog
make


# ebtables-legacy 提供经典的桥接功能
cd /usr/src
git clone https://git.netfilter.org/ebtables
cd ebtables
./autogen.sh
./configure
make
make install
```

### 4.7 启用、停止一些服务

禁止系统的NetworkManger、firewalld，改用手工管理
```
systemctl enable httpd.service
systemctl disable NetworkManager.service
systemctl disable firewalld.service
```

### 4.8 修改 /etc/rc.d/rc.local

改为可执行
```
chmod u+x /etc/rc.d/rc.local
```

这个文件是启动时一次执行的，修改为如下内容
```
touch /var/lock/subsys/local

hostname NAT-LAB

ip link set eth0 up
ip link set eth1 up
ip link set eth2 up

#出口的IP、掩码和网关
ip add add x.x.x.x/x dev eth0
ip route add 0/0 via x.x.x.y

#内网IP
ip add add 192.168.1.1/24 dev eth1

ip add add 192.168.2.1/24 dev eth2

echo 1 > /proc/sys/net/ipv4/ip_forward

modprobe nf_conntrack hashsize=400000
cd /proc/sys/net/netfilter
echo 1 > nf_conntrack_acct
echo 60 > nf_conntrack_generic_timeout
echo 10 > nf_conntrack_icmp_timeout
echo 10 > nf_conntrack_tcp_timeout_syn_sent
echo 10 > nf_conntrack_tcp_timeout_syn_recv
echo 1800 > nf_conntrack_tcp_timeout_established

/etc/rc.d/rc.firewall
/etc/rc.d/rc.ipv6

/usr/src/traffic/iftrafficd &

screen -S natlog -d -m /usr/src/gzlog/natlog.sh -e NEW,DESTROY
screen -S dnslog -d -m /usr/src/gzlog/dnslog.sh

```

### 4.9 修改 /etc/rc.d/rc.firewall

```
touch /etc/rc.d/rc.firewall
chmod u+x /etc/rc.d/rc.firewall
```

这个文件每次修改规则后可以执行以便生效
```
#!/bin/bash

iptables -F
iptables -t nat -F

iptables -t nat -A POSTROUTING -j MASQUERADE -o eth0 

iptables -A INPUT -j ACCEPT -m state --state ESTABLISHED,RELATED
iptables -A INPUT -j ACCEPT -i lo
iptables -A INPUT -j ACCEPT -i eth1
iptables -A INPUT -j ACCEPT -p icmp

#仅仅允许eth1内网连接22和80端口
iptables -A INPUT -j ACCEPT  -p tcp --dport 22 -i eth1
iptables -A INPUT -j ACCEPT  -p tcp --dport 80 -i eth1
iptables -A INPUT -j DROP -i eth0

iptables -A FORWARD -j ACCEPT -m state --state ESTABLISHED
iptables -A FORWARD -j DROP -p tcp --dport 445
iptables -A FORWARD -j DROP -p tcp --dport 135
iptables -A FORWARD -j DROP -p tcp --dport 139
```

### 4.10 修改 /etc/rc.d/rc.ipv6

```
touch /etc/rc.d/rc.ipv6
chmod u+x /etc/rc.d/rc.ipv6
```

这个文件用来实现IPv6数据包桥接功能，每次修改规则后可以执行以便生效
```
#!/bin/bash

brctl addbr br6
ifconfig br6 up
brctl addif br6 eth0
brctl addif br6 eth1
brctl addif br6 eth2
ebtables-legacy -t broute -A BROUTING -p ! ipv6 -j DROP
```

### 4.11 修改 crontab，自动删除日志

crontab -e
```
0 5 * * * find /natlog -maxdepth 1 -mtime +200 -name "*gz" -exec rm -rf {} \;
```

### 4.12 修改 /etc/dhcp/dhcpd.conf

```
option domain-name "ustc.edu.cn";
option domain-name-servers 202.38.64.56, 202.38.64.17;

default-lease-time 6000;
max-lease-time 7200;

ddns-update-style none;

# eth0的地址段，仅仅声明
subnet 202.38.64.128 netmask 255.255.255.128 {
}

subnet 192.168.8.0 netmask 255.255.255.0 {
  range 192.168.8.2 192.168.8.200;
  option routers 192.168.8.1;
}
```
启动dhcpd服务
```
systemctl start dhcpd
systemctl enable dhcpd
```

