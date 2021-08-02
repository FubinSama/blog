# centOS命令笔记

## lrzsz

1. 安装lrzsz：`yum -y install lrzsz`
2. 上传文件`rz`
3. 下载文件`sz`

## ssh

添加SSH公钥：`cat id_rsa.pub >> ./.ssh/authorized_keys`

## yum

yum的基本使用：`yum [options] [command] [package ...]`

1. 安装yum包：`yum install _包名_`
2. 搜索yum包：`yum search _包名_`
3. 重新安装yum包：`yum reinstall _包名_`
4. 显示yum包的信息：`yum info _包名_`
5. 列出所有可更新的软件清单命令：`yum check-update`
6. 更新所有软件命令：`yum update`
7. 仅更新指定的软件命令：`yum update _包名_`
8. 列出所有可安裝的软件清单命令：`yum list`
9. 删除软件包命令：`yum remove _包名_`
10. 清除缓存命令:
    1. 清除缓存目录下的软件包：`yum clean packages`
    2. 清除缓存目录下的 headers：`yum clean headers`
    3. 清除缓存目录下旧的 headers：`yum clean oldheaders`
    4. 清除缓存目录下的软件包及旧的headers：`yum clean, yum clean all   (= yum clean packages; yum clean oldheaders)`

## RabbitMQ的安装

### 安装Erlang环境

1. 下载安装包：`wget http://erlang.org/download/otp_src_21.0.tar.gz`
2. 解压文件：`tar -zxvf otp_src_21.0.tar.gz` `cd  otp_src_21.0`
3. 编译：`./otp_build autoconf` `./configure` `make`
4. 安装：`make install`
5. 校验：`erl`

### 安装RabbitMQ

## 创建软连接

ln -s 目标 生成的连接名或存放的目录
如：`ln -s /usr/local/nginx/sbin/nginx /usr/local/bin/`

## 安装docker

Docker 要求 CentOS 系统的内核版本高于 3.10 ，查看本页面的前提条件来验证你的CentOS 版本是否支持 Docker。

1. 通过`uname -r`命令查看你当前的内核版本
2. 将`yum`包更新到最新：`sudo yum update`
3. 卸载旧版本(如果安装过旧版本的话)：`sudo yum remove docker  docker-common docker-selinux docker-engine`
4. 安装需要的软件包， yum-util 提供yum-config-manager功能，另外两个是devicemapper驱动依赖的：`sudo yum install -y yum-utils device-mapper-persistent-data lvm2`
5. 设置yum源：`sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`
6. 可以查看所有仓库中所有docker版本，并选择特定版本安装：`yum list docker-ce --showduplicates | sort -r`
7. 安装docker：

    ```SH
    sudo yum install docker-ce  #由于repo中默认只开启stable仓库，故这里安装的是最新稳定版17.12.0
    sudo yum install <FQPN>  # 例如：sudo yum install docker-ce-17.12.0.ce
    ```

8. 启动并加入开机启动

    ```SH
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

9. 验证安装是否成功(有client和service两部分表示docker安装启动都成功了)：`docker version`

## 安装nodejs

1. 更改yum源：`curl --silent --location https://rpm.nodesource.com/setup_10.x | sudo bash`
2. 清楚yum缓存：`sudo yum clean all`
3. 安装：`sudo yum -y install nodejs`
