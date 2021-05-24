---
title: git 基础命令
date: 2021-02-21 10:21:00 +0800
category: Tools
---

git是2005年linus花了2周时间，用c语言开发的一款分布式版本控制系统，推荐参考<https://www.liaoxuefeng.com/wiki/896043488029600>。

### 设置用户名邮箱
```bash
git config --global user.name "username"
git config --global user.email "email"
```
### 查看指定文件变更历史
```bash
git log -p abc.txt
```
### 查看删除文件
```bash
# see the changes of a file, works even 
# if the file was deleted
git log -- [file_path]

# limit the output of Git log to the 
# last commit, i.e. the commit which delete the file
# -1 to see only the last commit
# use 2 to see the last 2 commits etc
git log -1 -- [file_path]

# include stat parameter to see
# some statics, e.g., how many files were 
# deleted
git log -1 --stat -- [file_path] 
```
## 拉取远程分支
```bash
git branch -a                                //获取远程分支名称
git checkout -b dpdk19 remotes/origin/dpdk19 //拉取远程分支
git branch                                   //查看本地分支是否成功
```
## 新建分支
```bash
1.从已有分支(master)创建(-b)新分支(dev)并检出(checkout)
  git checkout -b dev
2.提交该分支到远程仓库
  git push origin dev
3.设置默认提交获取分支信息(设置后可直接git push/pull)
  git branch --set-upstream-to=origin/dev
```
## 合并分支
```bash
git checkout master && git pull         //更新待合并分支
git checkout dpdk19 && git merge master //合并分支
解决冲突
git add .                               //添加所有文件到暂存区
git commit -m "merge master"            //将所有文件从暂存区提交到本地仓库
git push                                //将所有文件从本地仓库推送到远程仓库
```
## git add
![git-add.jpg](/assets/images/git/git-add.jpg)
## git各空间交互
![git-all.png](/assets/images/git/git-all.png)