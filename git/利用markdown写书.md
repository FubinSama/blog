# 利用markdown写书

## 安装gitbook-cli

### 1. 首先安装node和npm

用以下命令在arch上安装node和npm

```SH
sudo pamcan -S node
sudo pacman -S npm
```

输入以下命令查看是否安装完成

```SH
# 查看 node 版本
node -v

# 查看 npm 版本
npm -v
```

### 2. 全局安装gitbook-cli

```SH
# 更改npm为淘宝源
npm config set registry https://registry.npm.taobao.org/
# 安装gitboot-cli
sudo npm i -g gitbook-cli #因为全局路径为`/usr/lib/node_modules/`所以需要sudo权限
```

## 初始化电子书

```SH
# 创建一个文件夹
mkdir gitbook-demo
cd gitbook-demo

# 初始化电子书，会生成README.md和SUMMARY.md两个文件，其中
# README.md是对电子书的介绍
# SUMMARY.md是电子书的目录结构
# ps：第一次使用需要安装GitBook，会很慢
gitbook init

# 编辑目录结构
vim SUMMARY.md

# 生成目录结构
gitbook init

# 编辑各章节的内容
vim chapter1/section1.1.md

# 编译电子书
gitbook serve
```

### `SUMMARY.md`文件

SUMMARY.md是电子书的目录结构，它的机构如下所示：

```MARKDOWN
* [电子书名称](README.md)
* [第一章](chapter1/README.md)
    * [xxxx](chapter1/section1.1.md)
    * [xxxx](chapter1/section1.2.md)
* [第二章](chapter2/README.md)
    * [xxxx](chapter2/section2.1.md)
    * [xxxx](chapter2/section2.2.md)
```

编写SUMMARY.md文件后，执行`gitbook init`命令会自动生成目录结构文件。

### `gitbook serve`命令

`gitbook serve`命令实际上会首先调用`gitbook build`编译书籍，完成以后会打开一个 web 服务器，监听在本地的`4000`端口。

如果当前书籍写完了，想要发布到自己的网站的话，也可以使用命令输出成html文件使用：

```SH
gitbook build [书籍路径] [输出路径]
```
