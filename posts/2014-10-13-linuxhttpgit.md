---
layout: post
title: Linux下基于HTTP协议带用户认证的GIT开发环境设置
category: 工具
description:
---

Git 的访问可以采用HTTP或SSH协议安全的访问，通常我们使用 gitlib 进行 Web 管理，但是在 Linux 命令行开发环境下，基本都是使用 SSH 协议，只需要在gitlib里面配置好 SSH Key 就可以，由于现在开发环境的特殊情况，我们在Linux 命令行开发环境下，也只能采用 HTTP 协议进行访问，以下步骤介绍如何在 Linux 命令行环境下进行 GIT 开发环境配置。

- 创建 用户名/密码 文件(明文密码)  

在自己的 $HOME 目录下，编辑 .netrc 文件，内容如下：  

        machine git.xxxxx.net
        login xxx@xxx.com password xxxxxx

- 创建 GnuPG 密钥 

在自己的$HOME 目录下，执行命令：  

        gpg --gen-key  

注：默认回车即可，RSA密钥选择1024，2048太慢，但安全性好

可以使用以下命令查看已生成的密钥：  

        gpg --list-key  

- 加密 用户名/密码 文件  

在自己的 $HOME 目录下，执行命令：  

        gpg -o ~/.netrc.gpg -er chen ~/.netrc  

注：执行完成后，可以删除明文密码文件 .netrc

- 设置用户的 Git 配置  

在自己的 $HOME 目录下，执行命令：  

        git config --global credential.helper 'netrc -f ~/.netrc.gpg -d'

此时可以编辑 .gitconfig 文件，填写更多信息：  

        [user]
            name = XXX
            email = xxx@xxx.com
        [core]
            excludesfile = /home/xxx/.gitignoreglobal
        [credential]
            helper = netrc -f ~/.netrc.gpg -d

- 开始GIT 环境  

        git clone http://git.xxxxx.net:port/project/my_project.git  

