# docker

Docker 属于 Linux 容器的一种封装，提供简单易用的容器使用接口。

## docker的主要用途

- 提供一次性的环境。比如，本地测试他人的软件、持续集成的时候提供单元测试和构建的环境。
- 提供弹性的云服务。因为 Docker 容器可以随开随关，很适合动态扩容和缩容。
- 组建微服务架构。通过多个容器，一台机器可以跑多个服务，因此在本机就可以模拟出微服务架构。

## docker的安装

ArchLinux下的安装

```SH
sudo pacman -S docker
```

安装完成后，运行下面的命令，验证是否安装成功。

```SH
sudo docker version
# 或者
sudo docker info
```

Docker 需要用户具有 sudo 权限，为了避免每次命令都输入sudo，可以把用户加入 Docker 用户组

```SH
sudo usermod -aG docker $USER
# 或者
sudo gpasswd -a docker $USER
```

Docker 是服务器----客户端架构。命令行运行docker命令的时候，需要本机有 Docker 服务。如果这项服务没有启动，可以用下面的命令启动（官方文档）。

```SH
# service 命令的用法
sudo service docker start

# systemctl 命令的用法
sudo systemctl start docker
```

## 设置使用国内的镜像网站

```SH
sudo vim /etc/docker/daemon.json
# 没有该文件，就直接创建，然后添加下面几行
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```

## image 文件

__Docker 把应用程序及其依赖，打包在 image 文件里面。__ 只有通过这个文件，才能生成 Docker 容器。image 文件可以看作是容器的模板。Docker 根据 image 文件生成容器的实例。同一个 image 文件，可以生成多个同时运行的容器实例。

image 是二进制文件。实际开发中，一个 image 文件往往通过继承另一个 image 文件，加上一些个性化设置而生成。举例来说，你可以在 Ubuntu 的 image 基础上，往里面加入 Apache 服务器，形成你的 image。

```SH
# 列出本机的所有 image 文件。
$ docker image ls

# 删除 image 文件
$ docker image rm [imageName]
```

image 文件是通用的，一台机器的 image 文件拷贝到另一台机器，照样可以使用。

## HelloWorld实例

首先，运行下面的命令，将 image 文件从仓库抓取到本地。

```SH
docker image pull library/hello-world
```

其中`docker image pull`是抓取image的命令， `library/hello-world` 是image文件在仓库里面的位置，其中 `library` 是 image 文件所在的组，__hello-world__ 是 image 文件的名字。

由于 Docker 官方提供的 image 文件，都放在 `library` 组里面，所以它的是默认组，可以省略。因此，上面的命令可以写成下面这样:

```SH
docker image pull hello-world
```

抓取成功以后，就可以在本机看到这个 image 文件了

```SH
docker image ls
```

运行这个image文件

```SH
docker container run hello-world
```

`docker container run`命令会从 image 文件，生成一个正在运行的容器实例。

ps：`docker container run`命令 __具有自动抓取 image 文件的功能__。如果发现本地没有指定的 image 文件，就会从仓库自动抓取。因此，前面的docker image pull命令并不是必需的步骤。

如果运行成功，你会在屏幕上读到下面的输出

```SH
$ docker container run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.
... ...
```

输出这段提示以后，`hello world`就会停止运行，容器自动终止。

__有些容器不会自动终止，因为提供的是服务。__ 比如，安装运行 Ubuntu 的 image，就可以在命令行体验 Ubuntu 系统。

```SH
docker container run -it ubuntu bash
```

对于那些不会自动终止的容器，必须使用`docker container kill`命令手动终止。

```SH
docker container kill [containID]
```

另外，使用`docker search 镜像名`命令来查找仓库中的镜像

## 容器文件

__image 文件生成的容器实例，本身也是一个文件，称为容器文件。__ 也就是说，一旦容器生成，就会同时存在两个文件： image 文件和容器文件。而且关闭容器并不会删除容器文件，只是容器停止运行而已。

```SH
# 列出本机正在运行的容器
docker container ls

# 列出本机所有容器，包括终止运行的容器
docker container ls --all
```

上面命令的输出结果之中，包括容器的 ID。

终止运行的容器文件，依然会占据硬盘空间，可以使用`docker container rm`命令删除。

```SH
docker container rm [containerID]
```

运行上面的命令后，再使用`docker container ls --all`命令，就会发现被删除的容器文件已经消失了。

## Dockerfile 文件

Dockerfile 文件。它是一个文本文件，用来配置 image.Docker 根据 该文件生成二进制的 image 文件。

## 制作自己的 Docker 容器

以 koa-demos 项目为例，实现让用户在 Docker 容器里面运行 Koa 框架

首先下载源码

```SH
git clone https://github.com/ruanyf/koa-demos.git
cd koa-demos
```

### 1.编写Dockerfile文件

首先，在项目的根目录下，新建一个文本文件.dockerignore，写入下面的内容。

```TXT
.git
node_modules
npm-debug.log
```

上面代码表示，这三个路径要排除，不要打包进入 image 文件。如果没有路径要排除，可以不用创建这个文件

然后，在项目的根目录下，新建一个文本文件 Dockerfile，写入下面的内容。

```TXT
FROM node:8.4
COPY . /app
WORKDIR /app
RUN npm install --registry=https://registry.npm.taobao.org
EXPOSE 3000
```

上面代码一共五行，含义如下。

```MARKDOWN
- FROM node:8.4：该 image 文件继承官方的 node image，冒号表示标签，这里标签是8.4，即8.4版本的 node。
- COPY . /app：将当前目录下的所有文件（除了.dockerignore排除的路径），都拷贝进入 image 文件的/app目录。
- WORKDIR /app：指定接下来的工作路径为/app。
- RUN npm install：在/app目录下，运行npm install命令安装依赖。注意，安装后所有的依赖，都将打包进入 image 文件。
- EXPOSE 3000：将容器 3000 端口暴露出来， 允许外部连接这个端口。
```

### 2.创建 image 文件

有了 Dockerfile 文件以后，就可以使用`docker image build`命令创建 image 文件了。

```SH
docker image build -t koa-demo .
# 或者
docker image build -t koa-demo:0.0.1 .
```

面代码中，`-t`参数用来指定 image 文件的名字，后面还可以用冒号指定标签。如果不指定，默认的标签就是`latest`。最后的那个点表示 Dockerfile 文件所在的路径，上例是当前路径，所以是一个点。

如果运行成功，就可以通过下面的命令看到新生成的 image 文件koa-demo了。

```SH
docker image ls
```

### 3.生成容器

`docker container run`命令会从 image 文件生成容器。

```SH
docker container run -p 8000:3000 -it koa-demo /bin/bash
# 或者
docker container run -p 8000:3000 -it koa-demo:0.0.1 /bin/bash
```

上面命令的各个参数含义如下：

```MARKDOWN
- -p参数：容器的 3000 端口映射到本机的 8000 端口。
- -it参数：容器的 Shell 映射到当前的 Shell，然后你在本机窗口输入的命令，就会传入容器。
- koa-demo:0.0.1：image 文件的名字（如果有标签，还需要提供标签，默认是 latest 标签）。
- /bin/bash：容器启动以后，内部第一个执行的命令。这里是启动 Bash，保证用户可以使用 Shell。
```

如果一切正常，运行上面的命令以后，就会返回一个命令行提示符。

```SH
root@c9d0fa1988bc:/app#
```

这表示你已经在容器里面了，返回的提示符就是容器内部的 Shell 提示符。执行下面的命令。

```SH
root@66d80f4aaf1e:/app# node demos/01.js
```

这时，Koa 框架已经运行起来了。打开本机的浏览器，访问`http://127.0.0.1:8000`，网页显示"Not Found"，这是因为这个 demo 没有写路由。

这个例子中，Node 进程运行在 Docker 容器的虚拟环境里面，进程接触到的文件系统和网络接口都是虚拟的，与本机的文件系统和网络接口是隔离的，因此需要定义容器与物理机的端口映射（map）。

现在，在容器的命令行，按下`Ctrl + c`停止Node进程，然后按下 `Ctrl + d（或者输入 exit）`退出容器。此外，也可以用`docker container kill`终止容器运行。

```SH
# 在本机的另一个终端窗口，查出容器的 ID
docker container ls

# 停止指定的容器运行
docker container kill [containerID]
```

容器停止运行之后，并不会消失，用下面的命令删除容器文件。

```SH
# 查出容器的 ID
docker container ls --all

# 删除指定的容器文件
docker container rm [containerID]
```

也可以使用`docker container run`命令的`--rm`参数，在容器终止运行后自动删除容器文件。

```SH
docker container run --rm -p 8000:3000 -it koa-demo /bin/bash
```

### 4.CMD命令

上一节的例子里面，容器启动以后，需要手动输入命令node demos/01.js。我们可以把这个命令写在 Dockerfile 里面，这样容器启动以后，这个命令就已经执行了，不用再手动输入了

```TXT
FROM node:8.4
COPY . /app
WORKDIR /app
RUN npm install --registry=https://registry.npm.taobao.org
EXPOSE 3000
CMD node demos/01.js
```

上面的 Dockerfile 里面，多了最后一行`CMD node demos/01.js`，它表示容器启动后自动执行`node demos/01.js`。

### 5. MCD命令和RUN命令的区别

`RUN`命令在image文件的构建阶段执行，执行结果都会打包进入 image 文件；`CMD`命令则是在容器启动后执行。另外，一个 Dockerfile 可以包含多个`RUN`命令，但是只能有一个`CMD`命令。

注意，指定了`CMD`命令以后，`docker container run`命令就不能附加命令了（比如前面的`/bin/bash`），否则它会覆盖`CMD`命令。

现在，启动容器可以使用下面的命令：

```SH
docker container run --rm -p 8000:3000 -it koa-demo
```

### 6. 发布image文件

容器运行成功后，就确认了 image 文件的有效性。这时，我们就可以考虑把 image 文件分享到网上，让其他人使用。

首先，注册一个docker账户。然后使用`docker login`命令交互式的输入用户名及密码来完成在命令行界面登录。

ps：如果是注册的阿里云的镜像仓库，使用命令`sudo docker login --username=13963693378 registry.cn-qingdao.aliyuncs.com`登录，其中username是手机号。

接着，为本地的 image 标注用户名和版本:

```SH
docker image tag [imageName] [username]/[repository]:[tag]
# 实例
docker image tag koa-demos wfb/wfb:0.0.1
```

也可以不标注用户名，重新构建一下 image 文件:

```SH
docker image build -t [username]/[repository]:[tag] .
```

最后，发布 image 文件:

```SH
docker image push [username]/[repository]:[tag]
```

发布成功后，登录账户，就可以看到已经发布的image文件

## 其他有用的命令

- `docker container start [containerID]`命令，用来启动已经生成，但停止运行的容器。它不像`docker container run`命令，每次都会新建一个容器。
- `docker container stop [containerID]`命令也可以终止程序的运行。区别于`docker container kill`命令是对容器中的主进程发送SIGKILL信号；`docker container stop`命令会先发送SIGTERM信号，再过一段事假你发送SIGKILL信号，给予程序自行进行收尾操作的时间。
- `docker container logs [containerID]`命令用来查看docker容器的输出，即容器里面shell的标准输出。如果`docker run`命令没有使用`- it`参数，就要用这个命令查看输出
- `docker container exec`命令用于进入一个正在运行的docker容器。如果`docker run`命令运行容器的时候，没有使用`- it`参数，就要使用这个命令进入容器。一旦进入了容器，就可以在容器的shell执行命令了，而且退出时不会停止容器的执行。示例：`docker exec -it [containID] bash`。
- `docker container cp`命令用于从正在运行的 Docker 容器里面，将文件拷贝到本机。如：拷贝到当前目录的写法为：`docker container cp [containID]:[/path/to/file] .`
- 删除tag标记：`docker image rm wfb/start-spring:0.0.1`，其中`wfb/start-spring:0.0.1`是镜像的完整名

## 后台运行

更多的时候，需要让 Docker 在后台运行而不是直接把执行命令的结果输出在当前宿主机下。此时，可以通过添加 -d 参数来实现。

下面举两个例子来说明一下。

如果不使用 -d 参数运行容器。

```SH
$ docker run ubuntu:18.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
hello world
hello world
hello world
hello world
```

容器会把输出的结果 (STDOUT) 打印到宿主机上面

如果使用了 -d 参数运行容器。

```SH
$ docker run -d ubuntu:18.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
77b2dc01fe0f3f1265df143181e7b9af5e05279a884f4776ee75350ea9d8017a
```

此时容器会在后台运行并不会把输出的结果 (STDOUT) 打印到宿主机上面(输出结果可以用`docker logs`查看)。

ps：容器是否会长久运行，是和`docker run`指定的命令有关，和`-d`参数无关。

要获取容器的输出信息，可以通过`docker container logs`命令：

```SH
$ docker container logs [container ID or NAMES]
hello world
hello world
hello world
. . .
```

## 阿里云镜像仓库操作

### 登录阿里云Docker Registry

```SH
sudo docker login --username=13963693378 registry.cn-qingdao.aliyuncs.com
```

用于登录的用户名为阿里云账号全名，密码为开通服务时设置的密码。

### 从Registry中拉取镜像

```SH
sudo docker pull registry.cn-qingdao.aliyuncs.com/wfb/wfb:[镜像版本号]
```

### 将镜像推送到Registry

```SH
sudo docker login --username=13963693378 registry.cn-qingdao.aliyuncs.com
sudo docker tag [ImageId] registry.cn-qingdao.aliyuncs.com/wfb/wfb:[镜像版本号]
sudo docker push registry.cn-qingdao.aliyuncs.com/wfb/wfb:[镜像版本号]
```

### 选择合适的镜像仓库地址

从ECS推送镜像时，可以选择使用镜像仓库内网地址。推送速度将得到提升并且将不会损耗您的公网流量。

如果您使用的机器位于VPC网络，请使用 registry-vpc.cn-qingdao.aliyuncs.com 作为Registry的域名登录，并作为镜像命名空间前缀。

### 示例

使用`docker tag`命令重命名镜像，并将它通过专有网络地址推送至Registry：

```SH
$ sudo docker images
REPOSITORY                                                         TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
registry.aliyuncs.com/acs/agent                                    0.7-dfb6816         37bb9c63c8b2        7 days ago          37.89 MB
$ sudo docker tag 37bb9c63c8b2 registry-vpc.cn-qingdao.aliyuncs.com/acs/agent:0.7-dfb6816
```

使用`docker images`命令找到镜像，将该镜像名称中的域名部分变更为Registry专有网络地址：

```SH
sudo docker push registry-vpc.cn-qingdao.aliyuncs.com/acs/agent:0.7-dfb6816
```
