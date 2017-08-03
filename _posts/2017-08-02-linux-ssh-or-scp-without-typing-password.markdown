---
layout:     post
title:      "转载 - SSH or SCP to Linux without typing password"
subtitle:   ""
date:       2017-08-02 20:10:40
author:     "Echo Yuan"
#header-img: "img/in-post/hello-world/hello-world.gif"
tags:
    - Linux
---
本文转载自`来自linux大神博客`的博客，原文链接[https://www.linuxdashen.com/ssh-key：两个简单步骤实现ssh无密码登录](https://www.linuxdashen.com/ssh-key%EF%BC%9A%E4%B8%A4%E4%B8%AA%E7%AE%80%E5%8D%95%E6%AD%A5%E9%AA%A4%E5%AE%9E%E7%8E%B0ssh%E6%97%A0%E5%AF%86%E7%A0%81%E7%99%BB%E5%BD%95)。

如果你管理一台Linux服务器，那么你就会知道每次SSH登录时或者使用scp复制文件时都要输入密码是一个多么繁琐的过程。这篇教程介绍使用SSH Key来实现SSH无密码登录，而且使用scp复制文件时也不需要再输入密码。除了方便SSH登录，scp复制文件外，SSH无密码登录也[为Linux服务器增加了又一道安全防线](http://www.linuxdashen.com/vps%e5%ae%89%e8%a3%85debian-8%e5%90%8e%e5%a6%82%e4%bd%95%e7%a6%81%e7%94%a8root-ssh%e7%99%bb%e5%bd%95)。
SSH无密码登录的设置步骤

首先我们在自己的Linux系统上生成一对SSH Key：SSH密钥和SSH公钥。密钥保存在自己的Linux系统上，
然后公钥上传到Linux服务器，之后我们就能无密码SSH登录了。SSH密钥就好比是你的身份证明。

1. 在自己的Linux系统上生成SSH密钥和公钥

    打开终端，使用下面的ssh-keygen来生成RSA密钥和公钥。`-t`表示type，就是说要生成RSA加密的钥匙。

    ```
    ssh-keygen -t rsa
    ```

    RSA也是默认的加密类型，所以你也可以只输入ssh-keygen，默认的RSA长度是2048位。如果你非常注重安全，那么可以指定4096位的长度。

    ```
    ssh-keygen -b 4096 -t rsa
    ```

    生成SSH Key的过程中会要求你指定一个文件来保存密钥，按Enter键使用默认的文件就行了，然后需要输入一个密码来加密你的SSH Key，密码至少要20位长度。SSH密钥会保存在home目录下的`.ssh/id_rsa`文件中，SSH公钥保存在`.ssh/id_rsa.pub`文件中。

    ```
    Generating public/private rsa key pair.
    Enter file in which to save the key (/home/matrix/.ssh/id_rsa): 　按Enter键
    Enter passphrase (empty for no passphrase): 　　输入一个密码
    Enter same passphrase again: 　　再次输入密码
    Your identification has been saved in /home/matrix/.ssh/id_rsa.
    Your public key has been saved in /home/matrix/.ssh/id_rsa.pub.
    The key fingerprint is:
    e1:dc:ab:ae:b6:19:b0:19:74:d5:fe:57:3f:32:b4:d0 matrix@vivid
    The key's randomart image is:
    +---[RSA 4096]----+
    | .. |
    | . . |
    | . . .. . |
    | . . o o.. E .|
    | o S ..o ...|
    | = ..+...|
    | o . . .o .|
    | .o . |
    | .++o |
    +-----------------+
    ```

    查看`.ssh/id_rsa文件`就会看到，这个文件是经过加密的（encrypted），也就是用你输入的密码来加密。

    `less .ssh/id_rsa`

2. 将SSH公钥上传到Linux服务器

    可以使用`ssh-copy-id`命令来完成

    ```
    ssh-copy-id username@remote-server
    ```

    输入远程用户的密码后，SSH公钥就会自动上传了，SSH公钥保存在远程Linux服务器的.ssh/authorized_keys文件中。

    上传完成后，SSH登录就不需要再次输入密码了，但是首次使用SSH Key登录时需要输入一次SSH密钥的加密密码。（只需要输入一次，将来会自动登录，不再需要输入密钥的密码。

    使用scp命令来传送文件时也不需要输入密码。

#### SSH Key的知识

Linux系统有一个钥匙环(keyring)的管理程序，钥匙环受到用户登录密码的保护。当你登录Linux系统时，会自动解开钥匙环的密码，从而可访问钥匙环。SSH的密钥和公钥也存储在钥匙环，所以初次使用SSH密钥登录远程Linux服务器时需要输入一次SSH密钥的密码，而将来使用SSH密钥登录时不再输入密码。Ubuntu的钥匙环程序是seahorse。

SSH密钥就好比是你的身份证明，远程Linux服务器用你生成的SSH公钥来加密一条消息，而只有你的SSH密钥可以解开这条消息，所以其他人如果没有你的SSH密钥，是无法解开加密消息的，从而也就无法登录你的Linux服务器。

SSH无密码登录的设置就是这么简单。

