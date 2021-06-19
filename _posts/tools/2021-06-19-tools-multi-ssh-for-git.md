---
title: 一个linux配置多个ssh公私钥
date: 2021-06-19 15:34:00 +0800
category: Tools
---

在公司的开发机上一般有好几个人公用，但每个人都有自己的公私钥，怎么配置才能不冲突呢？

#### 第一步
生成自己的公私钥
```bash
ssh-keygen -t rsa -C "zhaotao19860@qq.com" -f ~/.ssh/id_rsa.zhaotao19860
```
#### 第二步
配置config文件，指定用户密钥。
```bash
touch ~/.ssh/config
chmod 644 ~/.ssh/config
vi ~/.ssh/config
Host github.com
User zhaotao19860
IdentityFile ~/.ssh/id_rsa.zhaotao19860
(git clone ssh://zhaotao19860@github.com就会用~/.ssh/id_rsa.zhaotao19860来认证)
```
#### 第三步
将生成的SSH公钥复制到github上
```bash
cat ~/.ssh/id_rsa.zhaotao19860.pub
```