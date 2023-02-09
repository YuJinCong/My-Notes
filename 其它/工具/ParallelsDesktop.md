

可配置网段：10.211.55.1-254

子网掩码：255.255.255.0

子网：10.211.55.0

## 安装CentOS 7 虚拟机

### 防火墙

1. 关闭防火墙并设置禁用：`systemctl stop firewalld`；`systemctl disable firewalld`

### 永久关闭SELinux

1. 执行命令：`vim /etc/selinux/config` 将SELINUX属性由默认的enforcing改为disabled

### 克隆后需要做的事

1. 配置IP地址：`vim /etc/sysconfig/network-scripts/ifcfg-eth0`
   - 将BOOTPROTO由dhcp改为static
   - 添加 IPADDR=10.211.55.xx（模版机是10，建议新建的克隆节点IP依次顺延 好记）
   - 添加PREFIX=24
   - 添加GATEWAY=10.211.55.1
   - 添加DNS1=8.8.8.8  DNS2=192.168.1.1
2. 修改主机名：`hostnamectl set-hostname xx`
3. 重启机器：`reboot`