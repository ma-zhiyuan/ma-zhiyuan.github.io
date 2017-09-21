---
layout:     post
title:      SaltStack笔记--安装&配置&命令
subtitle:   入门级SaltStack学习笔记
date:       2017-09-21
author:     zhiyuan.ma
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - SaltStack
    - 自动化运维
---


# SaltStack笔记--安装&配置&命令

## 安装 -->[文档](https://docs.saltstack.com/en/latest/topics/installation/index.html)
>**安装SaltStack存储库和密钥**

```
sudo yum install https://repo.saltstack.com/yum/amazon/salt-amzn-repo-latest-2.amzn1.noarch.rpm
```
>**清除缓存**

```
sudo yum clean expire-cache
```
>**安装组件(按需安装)**

- `sudo yum install salt-master`
- `sudo yum install salt-minion`
- `sudo yum install salt-ssh`
- `sudo yum install salt-syndic`
- `sudo yum install salt-cloud`
- `sudo yum install salt-api`

### Salt-master
安装组件完成使用`sudo salt --version`查看是否成功

### Salt-minion
1. 线上集群仅需安装Salt-minion `sudo yum install salt-minion`
2. 完成后配置`/etc/salt/minion`文件，将`master：`属性设置为master地址
3. 重启 `minion`, `sudo /etc/init.d/salt-minion restart`

## 常用配置 -->[文档](https://docs.saltstack.com/en/latest/topics/configuration/index.html)
>**初始简单配置**

```
interface: 0.0.0.0                    绑定本机接口
publish_port: 4505                    发布端口
user: root                            Salt进程运行用户
ret_port: 4506                        返回服务器端口
pidfile: /var/run/salt-master.pid     指定主pidfile位置
root_dir: /                           要操作的系统根目录
conf_file: /etc/salt/master           主机配置文件路径
pki_dir: /etc/salt/pki/master         pki密钥存储目录
```

>**自动验证消息**

修改`/etc/salt/master`中`auto_accept`属性为`True`（PS：双方密钥文件分别存储在`/etc/salt/pki/`下的`master`和`minion`文件夹下）

## 常用命令 -->[文档](https://docs.saltstack.com/en/latest/ref/cli/index.html)
>**master接受请求**

```
salt-key -L               查看接收消息
salt-key -a salt-minion   接受某一个minion
salt-key -A               接受所有
salt-key -d salt-minion   删除某个minion的公钥
salt-key -D               删除所有minion的公钥
salt-key -P               打印公钥
```

>**验证通讯正常**

`sudo salt '*' test.ping`
>**执行Linux命令**

`sudo salt '*' cmd.run 'ls /home/'`

>**执行脚本**

`sudo salt '*' cmd.exec_code  bash 'for i in {1,2};do echo $i;done'`

>**执行脚本**(`salt://`路径为`/srv/salt/`)

`sudo  salt '*' cmd.script salt://scripts/test.py HelloWorld`

>**拷贝文件**

`sudo salt-cp '*' test.py /tmp/`

>**安装软件**

`sudo salt '*' pkg.install vim`

>**帮助文档**

查看cmd模块功能`sudo salt '*' -d cmd`


## 模块 -->[文档](https://docs.saltstack.com/en/latest/ref/index.html)


>### cp模块（实现远程文件、目录的复制，以及下载URL文件等操作）

将主服务器file_roots指定位置下的目录复制到被控主机

`salt '*' cp.get_dir salt://hellotest /data`
 
下载指定URL内容到被控主机指定位置
`salt '*' cp.get_url http://xxx.xyz.com/download/0/files.tgz /root/files.tgz`
 
>### cmd模块（实现远程的命令行调用执行）

`salt '*' cmd.run 'netstat -ntlp'`
 
>### cron模块（实现被控主机的crontab操作）

为指定的被控主机、root用户添加crontab信息
`salt '*' cron.set_job root '*/5' '*' '*' '*' '*' 'date >/dev/null 2>&1'`

`salt '*' cron.raw_cron root`
 
>### 删除指定的被控主机、root用户的crontab信息

`salt '*' cron.rm_job root 'date >/dev/null 2>&1'`

`salt '*' cron.raw_cron root`
 
>### dnsutil模块（实现被控主机通用DNS操作）

为被控主机添加指定的hosts主机配置项

`salt '*' dnsutil.hosts_append /etc/hosts 127.0.0.1 rocketzhang.qq.com`
 
>### file模块（被控主机文件常见操作，包括文件读写、权限、查找、校验等）

`salt '*' file.get_sum /etc/resolv.conf md5`

`salt '*' file.stats /etc/resolv.conf`
 
>### network模块（返回被控主机网络信息）

`salt '*' network.ip_addrs`

`salt '*' network.interfaces`
 
>### pkg包管理模块（被控主机程序包管理，如yum、apt-get等）

`salt '*' pkg.install nmap`
`salt '*' pkg.file_list nmap`
 
>### service 服务模块（被控主机程序包服务管理）

```
salt '*' service.enable crond
salt '*' service.disable crond
salt '*' service.status crond
salt '*' service.stop crond
salt '*' service.start crond
salt '*' service.restart crond
salt '*' service.reload crond
```
















