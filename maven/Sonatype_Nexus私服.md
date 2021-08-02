# Sonatype Nexus

`Sonatype Nexus`存储库管理器是一种流行的开源软件，可以让您维护内部存储库并访问外部存储库。它允许存储库通过单个URL进行分组和访问。这使存储库管理员可以在后台添加和删除新的存储库，而无需开发人员更改其计算机上的配置。此外，它为使用Maven site生成的站点提供托管功能并提供`artifact`搜索功能。

## docker运行步骤

```SHELL
# 进入用来保存容器数据的目录
mkdir ./nexus-data && chown -R 200 ./nexus-data
docker run -d -p 8081:8081 --name nexus -v $(pwd)/nexus-data:/nexus-data sonatype/nexus3
```

```SHELL
cat nexus-data/admin.password
# 这就是默认的admin用户的密码
```

浏览器输入`http://0.0.0.0:8081/`进入管理员页面，使用`admin`和上一步获取的密码进行登陆，并更改密码。

ps：可使用`docker exec -it nexus bash`命令进入容器

## docker重启步骤

```SHELL
docker stop --time=120 nexus
docker container rm nexus
```

然后通过重启[docker服务](#重启docker服务)，来清理掉`docker-proxy`等不受管理的资源，最后执行[docker运行步骤](#docker运行步骤)

## 重启docker服务

```SHELL
su - root
systemctl stop docker
rm -f /var/lib/docker/network/files/local-kv.db
systemctl start docker
exit
```
