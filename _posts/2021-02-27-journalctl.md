---
layout: post
title: journalctl
date: 2021-02-27 15:24:00 +0800
category: Linux
---
**journalctl**用于读取systemd-journalctld.service服务抓取的系统及应用的日志，默认日志只存储于内存中/run/log/journal中，服务器重启后丢失；如果需要持久化到硬盘，需要手工创建/var/log/journal文件夹。<br/>
journald及syslogd都是用于收集系统日志的后台进程，linux系统都会默认安装，syslogd起源于init.d，而journald是随systemd的流行开始的。<br/>
>主要区别如下：<br/>
1.journald的日志文件格式为二进制，不能直接读取(使用journalctl)，而syslogd是纯文本的，方便阅读；<br/>
2.journald的日志文件存在索引、结构化、访问控制等特点，访问速度更快；<br/>
3.journald默认日志路径/run/log/journal(内存)，而syslogd默认/var/log/(硬盘)；<br/>
相同点如下：<br/>
1.都能用于收集系统及应用日志；<br/>
2.都能将日志传输到远端的独立日志服务器方便处理；<br/>
两者的关系如下：<br/>
1.journald提供了一个[syslog api](https://manpages.debian.org/jessie/manpages-dev/syslog.3.en.html)可以将日志发送给syslogd;<br/>
2.syslogd同样也提供了插件可以[读取](https://www.rsyslog.com/doc/v8-stable/configuration/modules/imjournal.html)和[写入](https://rsyslog.readthedocs.io/en/latest/configuration/modules/omjournal.html)journald;<br/>
**推荐阅读**：[Tutorial: Logging with journald](https://sematext.com/blog/journald-logging-tutorial/#toc-journald-vs-syslog-14)

1. 查看实时日志
   ```
   journalctl -f
   ```
2. 查看UID为1000的用户今天以来的日志
   ```
   journalctl _UID=1000 --since today
   ```
3. 查看1分钟以前的日志
   ```
   journalctl --since "1 min ago"
   ```
4. 查看某个单元/服务的日志
   ```
   journalctl -u wsgidav --since today
   ```
5. 查看指定PID的日志
   ```
   journalctl _PID=1000 -f
   ```
6. 查看内核日志
   ```
   journalctl -k
   ```