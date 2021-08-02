# github笔记

***

## [github公私钥配置](https://help.github.com/en/github/authenticating-to-github/adding-a-new-ssh-key-to-your-github-account)

1. 生成公私钥：`ssh-keygen -t rsa -b 4096 -C "fubinsama@qq.com"`
2. 点击github的头像，选择settings->SSH and GPC keys->New SSH Key。将生成的公钥id_rsa.pub中的内容黏贴到key中，然后点击保存。

## github搜索仓库

1. 在特定文件中搜索关键字：`_关键字_in:_文件名_`
2. 搜索starts大于1000的：`stars:>1000`
3. 要求必须同时包含这几个关键字：`_关键字1_ + _关键字2_`
4. 要求关键字必须是一个整体：`'_关键字_'`
5. 根据特定文件中的代码内容搜索：`_关键字_ filename:_文件名_`
6. 直接输入关键字，则会在description中搜索
