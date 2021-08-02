# opensuse命令笔记

## zypper的基本使用

1. 搜索软件包：**zypper search _包名_**
2. 安装软件：**zypper install _包名_**
3. 安装某个版本的软件包：**zypper install _包名_=_版本号_**
4. 安装以某个单词名字开头的所有软件包：**zypper install _包名前缀_\***
5. 更新软件：**zypper update _包名_**
6. 获取所有可用新包的列表：**zypper list-updates**
7. 检验软件包的依赖关系的完整性：**zypper verify _软件包名_**
8. 执行源代码软件安装和其依赖：**zypper source-install _包名.tgz_**
9. 卸载软件：**zypper remove _包名_**
10. 执行源刷新：**zypper refresh**