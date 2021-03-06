---
title: ssh 公钥认证免密码登录
date: 2016-12-29 11:13:28
tags:
- Linux
categories:
- 技术
---
* **ssh 公钥认证**是 ssh 的认证方式之一，通过公钥认证可以实现 **ssh 免密码登录**。

* 在用户的主目录下有一个`.ssh` 目录，用来存放当前用户的 ssh 配置相关的文件。如不存在可以使用 `mkdir .ssh` 命令创建目录。

* 进入到.ssh 目录，通过 `ssh-keygen` 命令来生成 ssh 公钥认证所需要的公钥和私钥文件。如不指定文件名和密钥类型，默认生成 `id_rsa` 和 `id_rsa.pub` 两个文件，分别为私钥和公钥。

* 可以通过 `-f` 来指定 `ssh-keygen` 命令所生成密钥的文件名，通过 `-C "xxx"` 添加备注。

* 如使用 `ssh-keygen` 时未指定文件名，会有提示要求输入文件名，直接按回车使用默认文件名(`id_rsa`)。

* 之后会提示要求输入密码，如不需要直接回车。至此本机的公钥和私钥创建完毕。

* 最后需要将本机的公钥添加到服务器中的 `~/.ssh/authorized_keys` 文件中(如不存在需手动创建)。
