# docker安装mongodb

ps：mongodb的镜像无法使用windows的文件来挂载

## 拉取最新版镜像

```shell
docker pull mongo:latest
```

## 运行容器

创建数据卷

```SHELL
docker volume create mongo-data
```

指定该文件夹作为卷，来启动容器

```shell
docker run --name mongo \
-p 27017:27017 \
-v mongo-data:/data/db \
 -d mongo
```

可加`--auth`表示需要密码才能访问容器服务。

## 创建登录身份

```shell
docker exec -it mongo mongo admin
# 使用admin数据库
use admin
# 创建一个名为 admin，密码为 123456 的用户。
db.createUser({ user:'admin',pwd:'123456',roles:[ { role:'userAdminAnyDatabase', db: 'admin'},"readWriteAnyDatabase"]});
尝试使用上面创建的用户信息进行连接。
db.auth('admin', '123456')
# 创建mall-port数据库
use mall-port
```
