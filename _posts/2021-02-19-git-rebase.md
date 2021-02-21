---
layout: post
title: git rebase
date: 2021-02-19 10:52:00 +0800
category: GIT
---
当一个功能分支开发完毕后，一般会有若干次的编译问题，优化问题及代码调整问题的提交，而这些提交没有必要体现在主干分支的log里面，为了保持主干分支log的简洁性，有必要提前合并一下，然后再merge到主干。

```bash
1. git rebase -i  (startpoint])  [endpoint]
```

其中`-i`的意思是`--interactive`，即弹出交互式的界面让用户编辑完成合并操作，`[startpoint] [endpoint]`则指定了一个编辑区间，如果不指定`[endpoint]`，则该区间的终点默认是当前分支HEAD所指向的commit(注：该区间指定的是一个**前开后闭**的区间)。

```bash
2. git log
```

运行命令查看历史提交，确定合并的起止点。

```bash
3. git rebase -i 36224db
```

出现以下画面

<img src="\public\img\git-rebase\git-rebase-1-20200318101000666.png" alt="git-rebase-1-20200318101000666" style="zoom:50%;" />

下面未被注释的部分列出的是我们本次rebase操作包含的所有提交，下面注释部分是git为我们提供的命令说明。每一个commit id 前面的pick表示指令类型，git 为我们提供了以下几个命令:

```
pick：保留该commit（缩写:p）

reword：保留该commit，但我需要修改该commit的注释（缩写:r）

edit：保留该commit, 但我要停下来修改该提交(不仅仅修改注释)（缩写:e）

squash：将该commit和前一个commit合并（缩写:s）

fixup：将该commit和前一个commit合并，但我不要保留该提交的注释信息（缩写:f）

exec：执行shell命令（缩写:x）

drop：我要丢弃该commit（缩写:d）
```

```
4. 修改pick为其他命令字符 ----》wq 保存退出；
```

<img src="\public\img\git\git-rebase-2-20200318101308971.png" alt="git-rebase-2-20200318101308971" style="zoom:50%;" />

```
5. 进入commit信息编辑界面，重新编辑commit信息，wq保存退出；
```

<img src="\public\img\git\git-rebase-3-20200318101643441.png" alt="git-rebase-3-20200318101643441" style="zoom:50%;" />

```
6. git push -f (如果待合并的commit已经提交到远端，需强制覆盖提交)
```

