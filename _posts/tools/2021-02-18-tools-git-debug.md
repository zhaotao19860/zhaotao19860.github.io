---
layout: post
title: git debug
date: 2021-02-18 10:58:00 +0800
category: Tools
---

## 概述
&nbsp;&nbsp;&nbsp;&nbsp;git传输协议支持**http**(如：https://github.com/zhaotao19860/mem-monitor.git)和**ssh**(如：git@github.com:zhaotao19860/mem-monitor.git)两种方式，因为我开发过程中主要使用ssh传输协议，所以下面以ssh协议为例分析如何调试。git运行包括两部分**ssh连接+git命令运行(本机部分+远端部分)**，所以调试也分两部分。

## git部分

```bash
Git内嵌了相当完整的一组跟踪，可以用来调试Git问题。

要打开它们，您可以定义以下变量::

GIT_TRACE : 一般调试等级,
GIT_TRACE_PACK_ACCESS : packfile访问跟踪,
GIT_TRACE_PACKET : 网络访问包级别的跟踪,
GIT_TRACE_PERFORMANCE : 记录性能相关数据,
GIT_TRACE_SETUP : 有关查找与其交互的存储库和环境的信息,
GIT_MERGE_VERBOSITY : 用于调试递归合并策略 (取值: 0-5),
GIT_CURL_VERBOSE : 用于记录所有curl消息 (equivalent to curl -v),
GIT_TRACE_SHALLOW : 用于调试、获取/克隆浅存储库.

可能的取值如下:
1)输出到stderr: true/1/2;
2)输出到指定文件: 以/开头的绝对路径.

1.调试信息输出到stderr:
  GIT_TRACE=1 git pull 

2.调试信息输出到指定文件:
  GIT_TRACE=/export/tp-dns/log/git.log git pull

3.python脚本中使用:
  os.environ['GIT_TRACE'] = /export/tp-dns/log/git.log
  repo = Repo(git_local)
  type(repo.git).GIT_PYTHON_TRACE = 'full'
```

## ssh部分

```bash
1.用文件替换git命令中调用的ssh:
  echo 'ssh -vvv "$@"' > ssh && chmod +x ssh
  GIT_SSH="$PWD/ssh" git pull origin master

2.环境变量直接替换ssh:
  GIT_SSH_COMMAND="ssh -vvv" git clone <repositoryurl>

3.python脚本中使用:
  1)创建ssh替换文件
    cat > debug_ssh.sh <<\EOF
    #!/bin/sh

    SHELL_FOLDER=$(cd "$(dirname "$0")";pwd)
    LOG_PATH="${SHELL_FOLDER}/../../log/ssh.log"
    echo -e "\n\n$(date) ssh debug start ..." >> $LOG_PATH
    exec 2>>$LOG_PATH
    exec /usr/bin/ssh -v "$@"
    EOF
  2)增加可执行权限
    chmod a+x debug_ssh.sh
  3)修改python脚本
    repo = Repo(git_local)
    for item in repo.remotes:
            if item.url == remote_url:
                remote = item
                break
    ssh_executable = os.path.join(g_script_path, 'debug_ssh.sh')
    with repo.git.custom_environment(GIT_SSH_COMMAND=ssh_executable):
	    info_list = remote.pull()

```
## 注意
1. git低版本(如：1.8)日志中没有时间，不好定位问题，最好升级到比较新的版本(如2.9.5)。