---
title: 运维工具ansible安装与使用
date: 2021-06-27 11:26:00 +0800
category: Tools
---

[ansible](https://github.com/ansible/ansible)是一款IT自动运维工具，使用python实现，通过插件实现批量命令执行。要求：ssh + python(>=2.4)，这个要求基本所有的系统都是默认满足的。

#### 执行过程
```bash
1.加载自己的配置文件，默认/etc/ansible/ansible.cfg；
2.查找对应的主机配置文件，找到要执行的主机或者组，默认/etc/ansible/hosts;
3.加载自己对应的模块文件，默认command；
4.通过ansible将模块或命令生成对应的临时py文件(python脚本)， 并将该文件传输至远程服务器；
5.对应执行用户的家目录的.ansible/tmp/XXX/XXX.PY文件；
6.给文件 +x 执行权限；
7.执行并返回结果；
8.删除临时py文件，sleep 0退出；
```
#### 安装
可以参考[官方文档](https://ansible-tran.readthedocs.io/en/latest/docs/intro_installation.html)
```bash
#主控端安装ansible(yum方式)
yum install -y epel-release  //安装epel源
yum install ansible -y
#主控端安装ansible(pip方式)
yum install python-pip
python -m pip install --upgrade "pip < 21.0"
pip install ansible
pip install paramiko PyYAML Jinja2 httplib2 six
#主控端生成ssh-key
ssh-keygen -t rsa -C "xxx@xxx.com"
#公钥推送给受控端
ssh-copy-id user@host
```
#### ansible.conf
默认配置路径/etc/ansible/anible.cfg
#### 受控主机列表配置
格式为ini，支持分组，默认路径/etc/ansible/hosts，可通过-i指定其他路径
```bash
cat /etc/ansible/hosts
[dev]
abc.host1
abc.host2
[test]
test.host1
test.host2
```
#### 使用
```bash
#命令格式
ansible -i ${hosts文件} ${hosts中的section名称} -m {功能模块} -a {模块参数}
ansible-doc -l //列出已安装的模块，按q退出
ansible-doc -s shell //列出shell模块的描述信息和操作动作
#测试某个section连通性
ansible -i /etc/ansible/hosts dev -m ping (hosts文件在标准路径，可以省略)
#某个主机执行shell命令
ansible abc.host1 -m shell -a "pwd"
#指定hosts的所有主机创建test01用户
ansible -i ./hosts all -m user -a 'name="test01"'
#将文件copy到test分组
ansible test -m copy -a 'src=/etc/fstab dest=/opt/fstab owner=root mode=640'
#dev分组创建一个文件
ansible dev -m file -a "path=/opt/test state=touch"
#test分支执行脚本
ansible test -m script -a "test.sh"
```
#### 问题
1.pip安装时，需要版本对应，[参考文档](https://stackoverflow.com/questions/65869296/installing-pip-is-not-working-in-python-3-6/65871131#65871131)
```bash
#pip for 2.7
curl -O https://bootstrap.pypa.io/pip/2.7/get-pip.py
python get-pip.py
#pip for 3.4
curl -O https://bootstrap.pypa.io/pip/3.4/get-pip.py
python get-pip.py
#pip for 3.5
curl -O https://bootstrap.pypa.io/pip/3.5/get-pip.py
python get-pip.py
#pip for latest
curl -O https://bootstrap.pypa.io/pip/get-pip.py
python get-pip.py
```
2.批量推送公钥
```bash
cat updateAuthorizedKey.sh 
#!/bin/sh
#usage: bash ./updateAuthorizedKey.sh /etc/ansible/hosts
HOSTS_FILE=$1
grep -qxF 'StrictHostKeyChecking no' /root/.ssh/config || echo 'StrictHostKeyChecking no' >> /root/.ssh/config
function for_hosts(){
    for i in `cat $1`
    do
		if [[ $i ==  \#* || $i == \[* || !($i == *\.*) ]]
		then
			echo $i
		else
			ssh-copy-id -i ~/.ssh/id_rsa.pub $i
		fi
    done
}
for_hosts $HOSTS_FILE
```