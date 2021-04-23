---
title: git lfs
date: 2021-02-20 14:23:00 +0800
category: Tools
---
[git-lfs](https://git-lfs.github.com/)(Large File Storage)，简化了git对于大文件的管理。

| <font color="#dd0000">windows</font>                                                  |
| :------------------------------------------------------------ |
| 1.下载并安装: https://github.com/git-lfs/git-lfs/releases/download/v2.4.2/git-lfs-windows-2.4.2.exe<br />2. 设置钩子：git lfs install(每台机子需要执行一次)<br />3. 关联大文件：git lfs track bigFile<br />4. 将管理文件.gitattributes添加入git仓库git add .gitattributes<br />5. git add bigFile<br />6. git commit -m "提交大文件"<br />8.git push<br />9.git lfs ls-files --查看管理的大文件<br/> |

| <font color="#dd0000">linux(安装指定版本)</font>                                      |
| :----------------------------------------------------------------- |
| 1.git --version --查看版本<br />2.yum remove git --删除低版本git<br />3.wget https://[www.kernel.org/pub/software/scm/git/git-2.9.5.tar.gz](http://www.kernel.org/pub/software/scm/git/git-2.9.3.tar.gz) --下载新版本<br />4.tar -xf [git-2.9.5.tar.gz](http://www.kernel.org/pub/software/scm/git/git-2.9.3.tar.gz) && cd [git-2.9.](http://www.kernel.org/pub/software/scm/git/git-2.9.3.tar.gz)5 && ./configure --prefix=/usr/local/git-2.9.5 && make -j && make install<br />5.ln -fs /usr/local/git-2.9.5/bin/* /usr/bin/ --添加软连接<br />6.wget https://packagecloud.io/github/git-lfs/packages/el/7/git-lfs-2.8.0-1.el7.x86_64.rpm/download --下载lfs<br />7.rpm -ivh --force --nodeps git-lfs-2.8.0-1.el7.x86_64.rpm --强制安装，不检查依赖<br />8.git lfs install |

| <font color="#dd0000">相关命令</font>                                                     |
| :------------------------------------------------------ |
| 1.普通安装： yum install git-lfs<br />2.拉取大文件：git lfs pull<br />3.查看大文件：git lfs ls-files |

