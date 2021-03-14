---
layout: post
title: history命令
date: 2021-03-14 09:53:00 +0800
category: Linux
---
history命令用于显示历史上执行过的shell命令，原理是：当执行完shell命令后，系统会将命令报存在内存中，当shell退出时会将内存中的命令存入文件中，比如~/.bash_history，存储的文件及存储的个数都可以用配置项配置。
#### 清除历史
```bash
history -c #清除内存
history -w #将内存写文件
```
#### 显示时间
```bash
export HISTTIMEFORMAT="%F %T "
说明：
%F ----> 指定日期格式为：‘YYYY-M-D’ (Year-Month-Day)
%T ----> 指定时间格式为：‘HH:MM:S’ (Hour:Minute:Seconds)
```

#### 其他配置
```bash
export HISTSIZE=1000         # 设置内存中的history命令的个数
export HISTFILESIZE=1000     # 设置文件中的history命令的个数
export HISTCONTROL=erasedups    # 清除整个命令历史中的重复条目
export HISTCONTROL=ignoredups   # 忽略记录命令历史中连续重复的命令
export HISTCONTROL=ignorespace  # 忽略记录空格开始的命令
export HISTCONTROL=ignoreboth   # 等价于ignoredups和ignorespace
```

#### 重启生效(root)
```
echo 'export HISTTIMEFORMAT="%F %T "' >> ~/.bashrc
bash ~/.bashrc
```

#### 重启生效(all)
```
echo 'export HISTTIMEFORMAT="%F %T "' >> /etc/profile
bash /etc/profile
```



