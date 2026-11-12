---
title: 如何搭建第二个网站
slug: apache-virtual-host-second-site
date: 2026-11-13T00:30:00+08:00
description: 在 CentOS + Apache 上通过虚拟主机搭建第二个网站的配置记录。
categories:
    - 服务器运维
tags:
    - Apache
    - 虚拟主机
    - CentOS
    - SSRF
---

## 前情提要

做 SSRF 练习时需要一台能对外提供文件的外网主机，隔太久没碰建站，差点连第二站点怎么挂都忘了。这篇把过程完整记一遍，重点标出当时卡住的地方。

## 正文

要在一台服务器上增加第二个网站，核心机制是 **虚拟主机**（Virtual Host）：所有网站共享一个公网IP，服务器根据访问的域名（或者是IP:端口），把请求分发到各自独立的目录。

好了，机制讲完，开始改配置。由于我的服务器（centos）是用apache搭建的，所以配置文件一般是在`/etc/httpd/conf.d/`

```bash
cd /etc/httpd/conf.d/
vim site2.conf
```

创建一个site2.conf用来写新网站的配置信息，一定要有`Listen 8888`！我问AI的时候，它给的模板没有这个，导致8888端口不在监听状态，网站起不来

```apache
Listen 8888
<VirtualHost *:8888>
    ServerName web.com
    DocumentRoot /var/www/webshell

    <Directory "/var/www/webshell">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

配置具体含义如下：

| 配置项 | 含义 |
| :--- | :--- |
| `Listen 8888` | 让 Apache 监听 TCP 8888 端口（此前缺失导致端口未开放） |
| `<VirtualHost *:8888>` | 定义一个虚拟主机，匹配所有网卡上 8888 端口的请求 |
| `ServerName webshell.com` | 指定该站点的主机名，用于基于 Host 头的精确匹配 |
| `DocumentRoot /var/www/webshell` | 设置网站根目录，所有静态文件从此路径读取 |
| `<Directory "/var/www/webshell">` | 针对该目录的访问控制块 |
| `AllowOverride All` | 允许该目录下的 `.htaccess` 文件覆盖服务器配置 |
| `Require all granted` | 允许所有客户端访问该目录（无 IP/用户限制） |
| `</Directory>` | 目录配置块结束 |
| `</VirtualHost>` | 虚拟主机配置块结束 |

写完配置文件后，需要重新加载apache配置文件

```bash
systemctl daemon-reload && systemctl reload httpd
```

另外，别忘了要创建网站根目录

```bash
mkdir /var/www/webshell
cd /var/www/webshell
echo 'hello,world' > index.html
```
