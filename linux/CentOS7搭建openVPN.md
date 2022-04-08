# CentOS7搭建openVPN

## 使用easy-rsa制作openVPN证书

### 下载并解压easy-rsa软件包

```sh
[root@centos ~]# mkdir /data/tools -p
[root@centos ~]# wget -P /data/tools http://down.i4t.com/easy-rsa.zip
--2022-01-03 21:54:35--  http://down.i4t.com/easy-rsa.zip
正在解析主机 down.i4t.com (down.i4t.com)... 111.72.100.250, 111.72.100.251
正在连接 down.i4t.com (down.i4t.com)|111.72.100.250|:80... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：59661 (58K) [application/x-zip-compressed]
正在保存至: “/data/tools/easy-rsa.zip”

100%[======================================>] 59,661      --.-K/s 用时 0.04s

2022-01-03 21:54:37 (1.27 MB/s) - 已保存 “/data/tools/easy-rsa.zip” [59661/59661])

[root@centos ~]# unzip -d /usr/local /data/tools/easy-rsa.zip
```

### 编辑vars文件

```sh
[root@centos ~]# cd /usr/local/easy-rsa-old-master/easy-rsa/2.0/
[root@centos 2.0]# vim vars
```

```txt
export KEY_COUNTRY="CN"
export KEY_PROVINCE="GD"
export KEY_CITY="GZ"
export KEY_ORG="wfb"
export KEY_EMAIL="fubinsama@qq.com"
export KEY_CN=wfb
export KEY_NAME=wfb
export KEY_OU=wfb
export PKCS11_MODULE_PATH=changeme
export PKCS11_PIN=1234
```

```sh
[root@centos 2.0]# source vars
NOTE: If you run ./clean-all, I will be doing a rm -rf on /usr/local/easy-rsa-old-master/easy-rsa/2.0/keys
[root@centos 2.0]# ./clean-all
```

### 制作CA证书

```sh
[root@centos 2.0]# ./build-ca
# 生成根证书ca.crt和根密钥ca.key
# 因为在vars中填写了证书的基本信息，所以这里一路回车即可
[root@centos 2.0]# ls keys
ca.crt  ca.key  index.txt  serial
```

### 制作Server端证书

```sh
[root@centos 2.0]# ./build-key-server server
# 一直回车，2个Y
[root@centos 2.0]# ls keys
01.pem  ca.key     index.txt.attr  serial      server.crt  server.key
ca.crt  index.txt  index.txt.old   serial.old  server.csr
```

### 制作Client端证书

```sh
[root@centos 2.0]# ./build-key wfb
# 每一个登陆的VPN客户端需要有一个证书，每个证书在同一时刻只能供一个客户端连接
# 一路按回车，直到提示需要输入y/n时，输入y再按回车，一共两次
[root@centos 2.0]# ./build-dh
# 创建迪菲·赫尔曼密钥，会生成dh2048.pem文件（生成过程比较慢，在此期间不要去中断它）
```

```sh
[root@centos 2.0]# ll keys
总用量 84
-rw-r--r-- 1 root root 7997 1月   3 21:58 01.pem
-rw-r--r-- 1 root root 7868 1月   3 22:07 02.pem
-rw-r--r-- 1 root root 2289 1月   3 21:56 ca.crt
-rw------- 1 root root 3272 1月   3 21:56 ca.key
-rw-r--r-- 1 root root  424 1月   3 22:08 dh2048.pem
-rw-r--r-- 1 root root  211 1月   3 22:07 index.txt
-rw-r--r-- 1 root root   21 1月   3 22:07 index.txt.attr
-rw-r--r-- 1 root root   21 1月   3 21:58 index.txt.attr.old
-rw-r--r-- 1 root root  107 1月   3 21:58 index.txt.old
-rw-r--r-- 1 root root    3 1月   3 22:07 serial
-rw-r--r-- 1 root root    3 1月   3 21:58 serial.old
-rw-r--r-- 1 root root 7997 1月   3 21:58 server.crt
-rw-r--r-- 1 root root 1736 1月   3 21:57 server.csr
-rw------- 1 root root 3272 1月   3 21:57 server.key
-rw-r--r-- 1 root root 7868 1月   3 22:07 wfb.crt
-rw-r--r-- 1 root root 1732 1月   3 22:07 wfb.csr
-rw------- 1 root root 3272 1月   3 22:07 wfb.key
```

## 安装OpenVPN

```sh
[root@centos 2.0]# yum install -y openvpn
```

## 配置OpenVPN服务端

### 创建openVPN文件目录和证书目录

```sh
[root@centos 2.0]# mkdir /etc/openvpn
mkdir: 无法创建目录"/etc/openvpn": 文件已存在
# openVPN配置文件目录，yum安装默认存在
[root@centos 2.0]# mkdir /etc/openvpn/keys
#openvpn证书目录
```

### 生成tls-auth key并将其拷贝到证书目录中（防DDos攻击、UDP淹没等恶意攻击）

```sh
[root@centos 2.0]# openvpn --genkey --secret ta.key
[root@centos 2.0]# mv ./ta.key /etc/openvpn/keys/
```

### 将我们上面生成的CA证书和服务端证书拷贝到证书目录中

```sh
[root@centos 2.0]# cp /usr/local/easy-rsa-old-master/easy-rsa/2.0/keys/{server.crt,server.key,ca.crt,dh2048.pem} /etc/openvpn/keys/
[root@centos 2.0]# ll /etc/openvpn/keys/
总用量 24
-rw-r--r-- 1 root root 2289 1月   3 22:15 ca.crt
-rw-r--r-- 1 root root  424 1月   3 22:15 dh2048.pem
-rw-r--r-- 1 root root 7997 1月   3 22:15 server.crt
-rw------- 1 root root 3272 1月   3 22:15 server.key
-rw------- 1 root root  636 1月   3 22:14 ta.key
```

### 拷贝并配置OpenVPN配置文件

```sh
[root@centos 2.0]# cp /usr/share/doc/openvpn-2.4.10/sample/sample-config-files/server.conf /etc/openvpn/
[root@centos 2.0]# vim /etc/openvpn/server.conf
```

```txt
#################################################
# Sample OpenVPN 2.0 config file for            #
# multi-client server.                          #
#                                               #
# This file is for the server side              #
# of a many-clients <-> one-server              #
# OpenVPN configuration.                        #
#                                               #
# OpenVPN also supports                         #
# single-machine <-> single-machine             #
# configurations (See the Examples page         #
# on the web site for more info).               #
#                                               #
# This config should work on Windows            #
# or Linux/BSD systems.  Remember on            #
# Windows to quote pathnames and use            #
# double backslashes, e.g.:                     #
# "C:\\Program Files\\OpenVPN\\config\\foo.key" #
#                                               #
# Comments are preceded with '#' or ';'         #
#################################################

# Which local IP address should OpenVPN
# listen on? (optional)
;local a.b.c.d

# Which TCP/UDP port should OpenVPN listen on?
# If you want to run multiple OpenVPN instances
# on the same machine, use a different port
# number for each one.  You will need to
# open up this port on your firewall.
port 1194

# TCP or UDP server?
proto tcp
;proto udp

# "dev tun" will create a routed IP tunnel,
# "dev tap" will create an ethernet tunnel.
# Use "dev tap0" if you are ethernet bridging
# and have precreated a tap0 virtual interface
# and bridged it with your ethernet interface.
# If you want to control access policies
# over the VPN, you must create firewall
# rules for the the TUN/TAP interface.
# On non-Windows systems, you can give
# an explicit unit number, such as tun0.
# On Windows, use "dev-node" for this.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel if you
# have more than one.  On XP SP2 or higher,
# you may need to selectively disable the
# Windows firewall for the TAP adapter.
# Non-Windows systems usually don't need this.
;dev-node MyTap

# SSL/TLS root certificate (ca), certificate
# (cert), and private key (key).  Each client
# and the server must have their own cert and
# key file.  The server and all clients will
# use the same ca file.
#
# See the "easy-rsa" directory for a series
# of scripts for generating RSA certificates
# and private keys.  Remember to use
# a unique Common Name for the server
# and each of the client certificates.
#
# Any X509 key management system can be used.
# OpenVPN can also use a PKCS #12 formatted key file
# (see "pkcs12" directive in man page).
ca keys/ca.crt
cert keys/server.crt
key keys/server.key  # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh2048.pem 2048
dh keys/dh2048.pem

# Network topology
# Should be subnet (addressing via IP)
# unless Windows clients v2.0.9 and lower have to
# be supported (then net30, i.e. a /30 per client)
# Defaults to net30 (not recommended)
;topology subnet

# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 10.8.0.0 255.255.255.0

# Maintain a record of client <-> virtual IP address
# associations in this file.  If OpenVPN goes down or
# is restarted, reconnecting clients can be assigned
# the same virtual IP address from the pool that was
# previously assigned.
ifconfig-pool-persist ipp.txt

# Configure server mode for ethernet bridging.
# You must first use your OS's bridging capability
# to bridge the TAP interface with the ethernet
# NIC interface.  Then you must manually set the
# IP/netmask on the bridge interface, here we
# assume 10.8.0.4/255.255.255.0.  Finally we
# must set aside an IP range in this subnet
# (start=10.8.0.50 end=10.8.0.100) to allocate
# to connecting clients.  Leave this line commented
# out unless you are ethernet bridging.
;server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100

# Configure server mode for ethernet bridging
# using a DHCP-proxy, where clients talk
# to the OpenVPN server-side DHCP server
# to receive their IP address allocation
# and DNS server addresses.  You must first use
# your OS's bridging capability to bridge the TAP
# interface with the ethernet NIC interface.
# Note: this mode only works on clients (such as
# Windows), where the client-side TAP adapter is
# bound to a DHCP client.
;server-bridge

# Push routes to the client to allow it
# to reach other private subnets behind
# the server.  Remember that these
# private subnets will also need
# to know to route the OpenVPN client
# address pool (10.8.0.0/255.255.255.0)
# back to the OpenVPN server.
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"

# To assign specific IP addresses to specific
# clients or if a connecting client has a private
# subnet behind it that should also have VPN access,
# use the subdirectory "ccd" for client-specific
# configuration files (see man page for more info).

# EXAMPLE: Suppose the client
# having the certificate common name "Thelonious"
# also has a small subnet behind his connecting
# machine, such as 192.168.40.128/255.255.255.248.
# First, uncomment out these lines:
;client-config-dir ccd
;route 192.168.40.128 255.255.255.248
# Then create a file ccd/Thelonious with this line:
#   iroute 192.168.40.128 255.255.255.248
# This will allow Thelonious' private subnet to
# access the VPN.  This example will only work
# if you are routing, not bridging, i.e. you are
# using "dev tun" and "server" directives.

# EXAMPLE: Suppose you want to give
# Thelonious a fixed VPN IP address of 10.9.0.1.
# First uncomment out these lines:
;client-config-dir ccd
;route 10.9.0.0 255.255.255.252
# Then add this line to ccd/Thelonious:
#   ifconfig-push 10.9.0.1 10.9.0.2

# Suppose that you want to enable different
# firewall access policies for different groups
# of clients.  There are two methods:
# (1) Run multiple OpenVPN daemons, one for each
#     group, and firewall the TUN/TAP interface
#     for each group/daemon appropriately.
# (2) (Advanced) Create a script to dynamically
#     modify the firewall in response to access
#     from different clients.  See man
#     page for more info on learn-address script.
;learn-address ./script

# If enabled, this directive will configure
# all clients to redirect their default
# network gateway through the VPN, causing
# all IP traffic such as web browsing and
# and DNS lookups to go through the VPN
# (The OpenVPN server machine may need to NAT
# or bridge the TUN/TAP interface to the internet
# in order for this to work properly).
;push "redirect-gateway def1 bypass-dhcp"

# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
;push "dhcp-option DNS 208.67.222.222"
;push "dhcp-option DNS 208.67.220.220"

# Uncomment this directive to allow different
# clients to be able to "see" each other.
# By default, clients will only see the server.
# To force clients to only see the server, you
# will also need to appropriately firewall the
# server's TUN/TAP interface.
client-to-client

# Uncomment this directive if multiple clients
# might connect with the same certificate/key
# files or common names.  This is recommended
# only for testing purposes.  For production use,
# each client should have its own certificate/key
# pair.
#
# IF YOU HAVE NOT GENERATED INDIVIDUAL
# CERTIFICATE/KEY PAIRS FOR EACH CLIENT,
# EACH HAVING ITS OWN UNIQUE "COMMON NAME",
# UNCOMMENT THIS LINE OUT.
duplicate-cn

# The keepalive directive causes ping-like
# messages to be sent back and forth over
# the link so that each side knows when
# the other side has gone down.
# Ping every 10 seconds, assume that remote
# peer is down if no ping received during
# a 120 second time period.
keepalive 10 120

# For extra security beyond that provided
# by SSL/TLS, create an "HMAC firewall"
# to help block DoS attacks and UDP port flooding.
#
# Generate with:
#   openvpn --genkey --secret ta.key
#
# The server and each client must have
# a copy of this key.
# The second parameter should be '0'
# on the server and '1' on the clients.
tls-auth keys/ta.key 0 # This file is secret

# Select a cryptographic cipher.
# This config item must be copied to
# the client config file as well.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-CBC

# Enable compression on the VPN link and push the
# option to the client (v2.4+ only, for earlier
# versions see below)
;compress lz4-v2
;push "compress lz4-v2"

# For compression compatible with older clients use comp-lzo
# If you enable it here, you must also
# enable it in the client config file.
comp-lzo

# The maximum number of concurrently connected
# clients we want to allow.
;max-clients 100

# It's a good idea to reduce the OpenVPN
# daemon's privileges after initialization.
#
# You can uncomment this out on
# non-Windows systems.
;user nobody
;group nobody

# The persist options will try to avoid
# accessing certain resources on restart
# that may no longer be accessible because
# of the privilege downgrade.
persist-key
persist-tun

# Output a short status file showing
# current connections, truncated
# and rewritten every minute.
status openvpn-status.log

# By default, log messages will go to the syslog (or
# on Windows, if running as a service, they will go to
# the "\Program Files\OpenVPN\log" directory).
# Use log or log-append to override this default.
# "log" will truncate the log file on OpenVPN startup,
# while "log-append" will append to it.  Use one
# or the other (but not both).
;log         openvpn.log
log-append  openvpn.log

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 3

# Silence repeating messages.  At most 20
# sequential messages of the same message
# category will be output to the log.
;mute 20

# Notify the client that when the server restarts so it
# can automatically reconnect.
;explicit-exit-notify 1
```

### 开启内核路由转发

```sh
[root@centos ~]# vim /etc/sysctl.conf
# 配置    net.ipv4.ip_forward = 1
[root@centos ~]# sysctl -p
```

### 启动openvpn服务

```sh
[root@centos ~]# cd /etc/openvpn/
[root@centos openvpn]# openvpn --daemon --config /etc/openvpn/server.conf
[root@centos openvpn]# netstat -lntup|grep 1194
udp        0      0 0.0.0.0:1194            0.0.0.0:*                           19195/openvpn
```

### 配置安全组

在安全组中开放1194端口。如果服务器有防火墙，也要开启该端口

## 客户端连接测试

### 将Client证书、CA证书以及Client配置文件下载下来

```sh
[root@centos openvpn]# cp /usr/share/doc/openvpn-2.4.10/sample/sample-config-files/client.conf /root/
[root@centos openvpn]# vim /root/client.conf
```

```txt
##############################################
# Sample client-side OpenVPN 2.0 config file #
# for connecting to multi-client server.     #
#                                            #
# This configuration can be used by multiple #
# clients, however each client should have   #
# its own cert and key files.                #
#                                            #
# On Windows, you might want to rename this  #
# file so it has a .ovpn extension           #
##############################################

# Specify that we are a client and that we
# will be pulling certain config file directives
# from the server.
client

# Use the same setting as you are using on
# the server.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel
# if you have more than one.  On XP SP2,
# you may need to disable the firewall
# for the TAP adapter.
;dev-node MyTap

# Are we connecting to a TCP or
# UDP server?  Use the same setting as
# on the server.
proto tcp
;proto udp

# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
# 这里要注意改为服务器的地址
remote tecent.wfb.com 1194
;remote my-server-2 1194

# Choose a random host from the remote
# list for load-balancing.  Otherwise
# try hosts in the order specified.
;remote-random

# Keep trying indefinitely to resolve the
# host name of the OpenVPN server.  Very useful
# on machines which are not permanently connected
# to the internet such as laptops.
resolv-retry infinite

# Most clients don't need to bind to
# a specific local port number.
nobind

# Downgrade privileges after initialization (non-Windows only)
;user nobody
;group nobody

# Try to preserve some state across restarts.
persist-key
persist-tun

# If you are connecting through an
# HTTP proxy to reach the actual OpenVPN
# server, put the proxy server/IP and
# port number here.  See the man page
# if your proxy server requires
# authentication.
;http-proxy-retry # retry on connection failures
;http-proxy [proxy server] [proxy port #]

# Wireless networks often produce a lot
# of duplicate packets.  Set this flag
# to silence duplicate packet warnings.
;mute-replay-warnings

# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
ca ca.crt
cert wfb.crt
key wfb.key

# Verify server certificate by checking that the
# certicate has the correct key usage set.
# This is an important precaution to protect against
# a potential attack discussed here:
#  http://openvpn.net/howto.html#mitm
#
# To use this feature, you will need to generate
# your server certificates with the keyUsage set to
#   digitalSignature, keyEncipherment
# and the extendedKeyUsage to
#   serverAuth
# EasyRSA can do this for you.
remote-cert-tls server

# If a tls-auth key is used on the server
# then every client must also have the key.
tls-auth ta.key 1

# Select a cryptographic cipher.
# If the cipher option is used on the server
# then you must also specify it here.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-CBC

# Enable compression on the VPN link.
# Don't enable this unless it is also
# enabled in the server config file.
comp-lzo

# Set log file verbosity.
verb 3

# Silence repeating messages
;mute 20
```

```sh
# 修改后缀
[root@centos openvpn]# mv /root/client.conf /root/wfb.ovpn
# 将这几个文件下载下来
[root@centos openvpn]# sz /root/wfb.ovpn
[root@centos openvpn]# sz /etc/openvpn/keys/ca.crt
[root@centos openvpn]# sz /etc/openvpn/keys/ta.key
[root@centos ~]# sz /usr/local/easy-rsa-old-master/easy-rsa/2.0/keys/wfb.crt
[root@centos ~]# sz /usr/local/easy-rsa-old-master/easy-rsa/2.0/keys/wfb.key
```

### 尝试链接

将证书文件拷贝到一个目录下。在该目录下使用`sudo openvpn --config wfb.ovpn`连接vpn。
