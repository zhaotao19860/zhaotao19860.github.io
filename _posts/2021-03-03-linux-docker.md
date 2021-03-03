---
layout: post
title: Linux Docker
date: 2021-03-03 09:50:00 +0800
category: Linux
---
[docker](https://github.com/docker)是一种轻量级的虚拟化技术，用go语言实现，充分利用了linux的cgroups/namespaces/UnionFS三大技术，解决了开发环境和生产环境环境不一致和资源共享的问题。
1. 安装
   ```bash
   yum install docker
   ```
2. 启动
   ```bash
   systemctl enable docker
   systemctl start docker
   ```
3. 制作镜像
   ```bash
   1)获取系统基础包
     https://github.com/CentOS/sig-cloud-instance-images/tree/CentOS-7.2.1511/docker
   
   2)生成dockerfile
     cat Dockerfile7.2.base
     FROM scratch
     MAINTAINER The CentOS Project <cloud-ops@centos.org>
     ADD centos-7.2.1511-docker.tar.xz /
   
     LABEL name="CentOS Base Image"
     LABEL vendor="CentOS"
     LABEL license=GPLv2
   
     # Volumes for systemd
     # VOLUME ["/run", "/tmp"]
   
     # Environment for systemd
     # ENV container=docker
   
     # For systemd usage this changes to /usr/sbin/init
     # Keeping it as /bin/bash for compatability with previous
     CMD ["/bin/bash"]
   
   3)生成镜像
     docker build -t hub.ark.jcloud.com/zhaotao1/centos7.2 -f Dockerfile7.2.base .
   ```
4. 命令
   ```
   1)登录仓库：docker login hub.ark.jcloud.com 
   2)推镜像：docker push hub.ark.jcloud.com/proxy-dns/centos7.2
   3)拉镜像：docker pull hub.ark.jcloud.com/proxy-dns/centos7.2:latest
   4)启动(交互)：docker run -it hub.ark.jd.com/zhaotao1/centos7.1-adns /bin/bash
   5)启动(后台)：docker run -d --name none-nework --network none centos7-ssh:v1
   6)ssh连接container: ssh zhaotao@localhost -p 23
   7)交互式连接container: docker exec -it h2o /bin/sh
   8)非交互式执行命令：docker exec -t h2o netstat -antp|grep h2o
   7)查看container日志：docker logs --tail 50 --follow --timestamps h2o
   8)文件拷贝：docker cp ./MfConfig/ h2o:/opt/MfGameServer/cgi-bin
   9)删除关闭的container: 
     docker rm $(sudo docker ps -aq)
     docker ps -a | grep Exit | cut -d ' ' -f 1 | xargs docker rm
     docker container prune
   11)删除指定镜像：docker rmi docker.microfun.cn/mfprod_h2o:1.0.0
   12)删除dangling 镜像：
     docker rmi $(docker images | grep "^<none>" | awk '{print $3}')
     docker image prune
     docker rmi $(docker images -qf "dangling=true")
   13)查看磁盘占用：docker system df
   14)清理磁盘：docker system prune
   15)删除dangling数据卷：
     docker volume rm $(docker volume ls -qf dangling=true)
     docker volume prune
   16)查看images: docker images
   17)查看container: docker ps -a
   18)查看container配置：docker inspect container-id
   ```
5. 问题
   ```bash
   1)443端口访问拒绝
     现象：Get https://hub.ark.jcloud.com/v1/_ping: dial tcp 172.19.4.61:443: connect: connection refused;
     原因：仓库使用的是http，而docker模式访问https(443);
     方案：
       修改daemon.json
       cat daemon.json
       {
         "insecure-registries":["hub.ark.jd.com","hub.ark.jcloud.com"]
       }
     重载配置:
       systemctl daemon-reload
     重启docker:
       systemctl restart docker
   2)基础镜像的作用
     实际内容包含的是linux发行版本的rootfs文件系统。
     可以下载：参考3.1；
     可以制作：tar --numeric-owner --exclude=/proc -zcvf /mnt/centos-7.2.1511-docker.tar.gz /
   ```