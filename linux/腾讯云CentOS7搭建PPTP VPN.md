# 腾讯云CentOS7搭建PPTP VPN

## 搭建pptp

### 检查系统是否支持ppp

```sh
[root@centos ~]# cat /dev/ppp
cat: /dev/ppp: 没有那个设备或地址
```

如果出现以上提示则说明ppp是开启的，可以正常架设pptp服务，若出现Permission denied等其他提示，你需要先去VPS面板里看看有没有enable ppp的功能开关。

### 设置内核转发，开启路由转发

```sh
[root@centos ~]# vim /etc/sysctl.conf
# 配置    net.ipv4.ip_forward = 1
[root@centos ~]# sysctl -p
```

### 安装pptp

```sh
[root@centos ~]# yum -y install pptpd
```

### 配置主配置文件

```sh
[root@centos ~]# cp /etc/pptpd.conf{,.bak}
vim /etc/pptpd.conf
# 添加如下两行配置
# localip 192.168.100.1
# remoteip 192.168.100.100-200 #分配给VPN 客户端的地址，一般是内网网段地址
```

### 配置账号文件

 ```sh
vim /etc/ppp/chap-secrets
# 添加如下配置
# root pptpd password *
 ```

## 启动服务

```sh
[root@centos ~]# systemctl start pptpd
```

## 检查服务是否启动

```sh
[root@centos ~]#  ps -ef |grep pptpd
# root      7549     1  0 21:02 ?        00:00:00 /usr/sbin/pptpd -f
[root@centos ~]# ss -nutlp |grep pptpd
# tcp    LISTEN     0      3         *:1723                  *:*                   users:(("pptpd",pid=7549,fd=6))
```

## 客户端连接

新版的mac已经不支持这种vpn了
