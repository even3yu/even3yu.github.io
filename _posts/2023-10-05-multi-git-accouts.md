---
layout: post
title: 一台电脑添加多个git账号
date: 2023-10-05 08:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: git
---


* content
{:toc}

---


## 1.  概述

电脑上已经配置了github的ssh连接。现在又有一个不同的git账户，也就是要在一台电脑上配置两个git账号。 



## 2. 取消git全局配置

之前配置github的时候，用命令

```bash
git config --golbal --list
git config --golbal user.name "XXX"
git config --golbal user.email "xxx@aa.com"
```

因为需要用到两个git账户，所以针对之前配置的全局配置就得取消。   
命令如下：

```bash
#全局配置账户已经移除
git config --global --unset user.name
#查看全局用户名
git config --global user.name


#全局配置邮箱已经移除
git config --global --unset user.email
#查看全局邮箱
git config --global user.email
```

 

## 3. 生成新的SSH KEYS

先用cd命令将当前目录切换到~/.ssh目录下
用ssh-keygen命令生成一组新的id_rsa_new和id_rsa_new.pub 
生成方法用命令

```bash
ssh-keygen -t rsa -C "xxx@aa.com"
```

这里确认之后和第一配置就有不同了。 
第一次给github配置sshkey时，直接按回车，其余什么都不管。最后看生成的id_rsa文件和id_rsa.pub文件。 
这次需要给这个生成的文件起一个名，例如id_rsa_new.步骤如图中所示。 

<img title="" src="{{ site.url }}{{ site.baseurl }}/images/multi-git-accunts.assets/1.png" alt="这里写图片描述" data-align="inline">

需要修改步骤1和步骤2

执行ssh-agent让ssh识别新的私钥 
命令为下面两步:

```bash
#Start the 'ssh-agent.exe' process
eval $(ssh-agent -s)
#install the SSH keys
ssh-add ~/.ssh/id_rsa_new
```



## 4. 配置多个账户的~/.ssh/config文件文件

```bash
# 该文件用于配置私钥对应的服务器
# first user
Host git@github.com
HostName https://github.com
User git
IdentityFile ~/.ssh/id_rsa

# second user
Host git@code.aliyun.com
HostName https://code.aliyun.com
User git
IdentityFile ~/.ssh/id_rsa_new
```



## 5. 把公钥添加到SSH KEYS

方法为： 
在github找到Settings->SSH and GPG keys。然后添加

![git_multiaccount_pub_github]({{ site.url }}{{ site.baseurl }}/images/multi-git-accunts.assets/2.png)



## 6. 测试

用命令

```bash
ssh -T git@github.com
```

成功的话，会返回包含

```bash
Hi XXXXX! You've successfully authenticated
```

的字符串。

 

## 7. 特别注意：github提交之后，contribution没有提交记录的小绿点问题

### 7.1 原因

这里，因为取消了全局的用户名和密码，在本地进行提交时，github不能将本地仓库对应的提交者和远程github账号对应的用户对应起来，所以就不记录了。 
可以通过在仓库根目录下git log查看提交记录，会发现有一些提交用户名和邮箱和GitHub的账号不对应。



### 7.2 解决方法

为每个仓库设置单独的用户名和密码。方法如下：



#### 7.2.1 进入到需要修改的仓库中

```bash
# 1.进入到需要修改的仓库中
git config user.name GitHub的用户名
git config user.email GitHub的登录邮箱
```



#### 7.2.2 查看是否修改成功的方法：  在代码仓库的.git目录中的config文件

```bash
[core]

[remote "origin"]

[branch "master"]

[user]
    name = 你的GitHub用户名
    email = 你的GitHub邮箱
```



#### 7.2.3 如果你已经提交了代码才发现这个问题也是有补救办法的。    在仓库中根目录新建一个shell脚本，命名为1.sh，内容如下

```bash
#!/bin/sh
git filter-branch --commit-filter '
        if [ "$GIT_AUTHOR_EMAIL" = "之前不对应的邮箱" ];
        then
                GIT_AUTHOR_NAME="对应的用户名";
                GIT_AUTHOR_EMAIL="对应的邮箱";
                git commit-tree "$@";
        else
                git commit-tree "$@";
        fi' HEAD
```
