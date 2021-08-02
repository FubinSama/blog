# mariadb数据库的配置

## 安装

```sh
sudo pacman -S mariadb
mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
systemctl start mysqld
mysql_secure_installation #在里面配置root的密码
```

## 配置root的密码

```sh
mysqladmin -u root password '1070952257wfb.'
```

## 开机启动

```sh
sudo systemctl enable mysqld
```

## 停止mysql守护进程

```sh
mysqld_safe --skip-grant-tables &
```

## 配置一个远程访问用户

1. 创建用户`create user 'wfb'@'%' identified by '1070952257';`
2. 用户授权`GRANT ALL PRIVILEGES ON \*.\* TO 'wfb'@'%' IDENTIFIED BY '1070952257' WITH GRANT OPTION;`
3. 刷新权限`flush privileges;`
