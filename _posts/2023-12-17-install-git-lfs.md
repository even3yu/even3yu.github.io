---
layout: post
title: ubuntu 下安装 git lfs
date: 2023-12-16 23:50:00 +0800
author: Fisher
pin: True
meta: Post
categories: git
---


* content
{:toc}

---


## 0. unbuntu 安装git lfs



## 1. sudo apt-get update



## 2. 命令

```bash
curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash
sudo apt-get install git-lfs
git lfs install
```



## 3. 验证

```
sudo git lfs install --system
```



## 4. 问题

遇到 git submodule  ， .git 目录下 no permission 问题
删除

全部，重新拉去一遍，解决



## 参考

[Git LFS管理大文件 - 简书 (jianshu.com)](https://www.jianshu.com/p/e79d1de098b6)
