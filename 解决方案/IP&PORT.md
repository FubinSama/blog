# IP&PORT

## 获取外网IP

通过`curl ip.sb`命令，就会获得该服务器对应的外网ip

## 内核分配的ip范围

### 查看内核分配的ip范围

```shell
cat /proc/sys/net/ipv4/ip_local_port_range
```

在我`CentOS Linux release 7.6.1810 (Core)`（通过`cat /etc/centos-release`得到）上默认值为：`32768 ~ 60999`。

### 修改内核分配的PORT范围

之前有一次在测试环境上启动服务，提示端口被占用，发现服务器内核分配的port范围是`1024 ~ 65535`

发现服务器上的配置都不是走的默认的，都是有指定的。推测应该是Ansible安装时指定的。

```shell
 vim /etc/sysctl.conf
```

```TXT
#  省略前面
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_max_tw_buckets=10000
net.ipv4.ip_conntrack_max=2310720
net.ipv4.ip_forward = 1
#意思是如果某个TCP连接在idle 20分钟后,内核发起probe.如果probe 2次(每次2秒)不成功,内核彻底放弃,认为该连接已失效
net.ipv4.tcp_keepalive_time=1200
net.ipv4.tcp_keepalive_probes=2
net.ipv4.tcp_keepalive_intvl=1
net.ipv4.tcp_syncookies = 1
net.ipv4.ip_local_reserved_ports=8060,8080,8081,8082,9000,9080,9090,60020,12000-12100  # 这行配置给出了内核分配port的预留值，内核会保证不会分配预留的port
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_fin_timeout = 20
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.ip_local_port_range = 1024 65535  # 这行配置就是配置内核分配的ip范围的
#  省略后面
```

只需要在`ip_local_reserved_ports`中加入服务用的端口号，或者修改`ip_local_port_range`的范围，使其不会包含服务使用的端口号即可。

修改完成后，需要使用以下命令重新加载配置：

```shell
sysctl -p /etc/sysctl.conf 
```
