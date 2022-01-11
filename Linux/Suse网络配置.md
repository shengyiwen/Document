### 方法一

- 指定ip地址和子网掩码

```shell
ifconfig eth0 192.178.1.108 netmask 255.255.255.0 up
```

- 指定网关

```shell
route add default gw 192.168.1.1
```

这种做法的缺点就是不会永久生效，重启后就失效


### 方法二

- 配置ip地址

```shell
sudo vim /etc/sysconfig/network/ifcfg-eth0
```

```
# 开机启动方式
STARTMODE='auto'
# 静态方式
BOOTPROTO='static'
# 指定ip地址
IPADDR='192.168.1.108'
# 子网掩码
NETMASK='255.255.255.0'
```

- 设置网关

```shell
sudo vim /etc/sysconfig/network/routes
```

```
default 192.168.1.1
```

- 设置DNS

```shell
sudo vim /etc/resolv.conf
```

```
nameserver=114.114.114.114
```

### 方法三

```shell
sudo yast lan
```

### 重启网络

```shell
service network restart
```

### 总结

主要和centos不同之处在于，centos配置网络的时候，可以在/etc/sysconfig/network-script/ifcfg-eth0的地方就指定
