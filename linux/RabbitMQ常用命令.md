# RabbitMQ

## 安装

```SH
安装rabbitmq
sudo pacman -S community/rabbitmq
# 将当前用户加入rabbitmq组
sudo gpasswd -a $USER rabbitmq
```

## 常用命令

```SH
# 启动
systemctl start rabbitmq

# 重启
systemctl restart rabbitmq

# 关闭
systemctl stop rabbitmq

# 查看状态
rabbitmqctl status

# 开启web插件
rabbitmq-plugins enable rabbitmq_management

# 添加一个用户名为wfb，密码为123的用户
rabbitmqctl add_user wfb 123

# 设置wfb用户的角色为管理员
rabbitmqctl set_user_tags wfb administrator

# 配置wfb用户可以远程登录
rabbitmqctl set permissions -p / wfb ".*"".*"".*"
```

## 默认配置

- 默认的web端端口为`15672`
- 默认用户：`guest`，密码：`guest`。只能本地登录
