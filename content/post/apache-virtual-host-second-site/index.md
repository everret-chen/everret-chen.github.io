---
title: 在一台服务器上搭建第二个网站（Apache 虚拟主机）
slug: apache-virtual-host-second-site
date: 2026-11-13T00:30:00+08:00
description: 通过虚拟主机在同一台 CentOS 服务器上挂载第二个站点，附完整配置与踩坑记录。
categories:
    - 服务器运维
tags:
    - Apache
    - 虚拟主机
    - CentOS
    - SSRF
---

做 SSRF 练习时需要一台能对外提供文件的外网主机，隔太久没碰建站，差点连第二站点怎么挂都忘了。这篇把过程完整记一遍，重点标出当时卡住的地方。

<!--more-->

## 原理：一台机器怎么跑多个站

核心机制是 **虚拟主机（Virtual Host）**。所有站点共享同一个公网 IP，服务器根据请求里的域名或端口，把流量分发到各自独立的目录。

Apache 支持两种匹配方式：

| 方式 | 依据 | 典型场景 |
| :--- | :--- | :--- |
| 基于名称 | HTTP 请求头里的 `Host` | 多个域名共用一个 80 端口 |
| 基于端口 | 请求到达的 TCP 端口 | 一个域名下开多个独立服务 |

这次采用端口方案：主站在 80，第二个站在 8888，互不干扰。

## 环境

- 系统：CentOS
- Web 服务：Apache（`httpd`）
- 配置目录：`/etc/httpd/conf.d/`

> [!NOTE]
> CentOS 上 Apache 的服务名是 `httpd`，不是 `apache2`（那是 Debian/Ubuntu 的叫法）。

## 第一步：写站点配置

```bash
cd /etc/httpd/conf.d/
vim site2.conf
```

配置内容：

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

> [!WARNING]
> **`Listen 8888` 这行最容易遗漏。**
> 当时参考的模板里没有它，配置看起来完整无缺，但 8888 端口始终不在监听状态，网站怎么都起不来。
> `<VirtualHost *:8888>` 只负责「匹配到达 8888 端口的请求」，
> 而**让 Apache 真正去监听这个端口的是独立的 `Listen` 指令**。

### 配置项逐条说明

| 配置项 | 含义 |
| :--- | :--- |
| `Listen 8888` | 让 Apache 监听 TCP 8888 端口 |
| `<VirtualHost *:8888>` | 定义虚拟主机，匹配所有网卡上 8888 端口的请求 |
| `ServerName web.com` | 指定主机名，用于基于 `Host` 头的精确匹配 |
| `DocumentRoot /var/www/webshell` | 网站根目录，静态文件从此路径读取 |
| `<Directory "/var/www/webshell">` | 针对该目录的访问控制块 |
| `AllowOverride All` | 允许该目录下的 `.htaccess` 覆盖服务器配置 |
| `Require all granted` | 允许所有客户端访问，无 IP 或用户限制 |

## 第二步：创建网站根目录

`DocumentRoot` 指向的目录不会自动创建，必须手动建好并放个测试文件：

```bash
mkdir -p /var/www/webshell
cd /var/www/webshell
echo 'hello,world' > index.html

# 确认属主，避免 Apache 读不到
chown -R apache:apache /var/www/webshell
```

## 第三步：重新加载配置

```bash
systemctl daemon-reload
systemctl reload httpd
```

`reload` 是平滑重载，不会断开现有连接；只有改了 `httpd.service` 这类 systemd 单元文件时，才需要先 `daemon-reload`。

## 第四步：验证

```bash
# 1. 端口是否在监听
ss -lntp | grep 8888

# 2. 本机能否访问
curl -I http://127.0.0.1:8888/

# 3. 从外部访问
curl -I http://<服务器公网 IP>:8888/
```

只有第 1 步通过，后面才有意义。如果 `ss` 看不到 8888，先回去检查 `Listen` 指令。

## 常见问题

**端口已监听，但外部访问超时**
云服务器的安全组或防火墙没放行。CentOS 7 以上用 firewalld：

```bash
firewall-cmd --permanent --add-port=8888/tcp
firewall-cmd --reload
```

**访问返回 403 Forbidden**
目录权限或 SELinux 拦截。临时排除可以执行：

```bash
chcon -R -t httpd_sys_content_t /var/www/webshell
```

**改了配置但没生效**
忘了 `reload`，或者配置文件路径写错。检查语法：

```bash
httpd -t
```

> [!TIP]
> 每次改完配置先跑 `httpd -t` 验证语法，再 `reload`。语法错误时 reload 会失败，但旧配置仍在运行，容易误以为改动已经生效。

## 小结

整件事的关键只有两点：

1. `Listen` 和 `<VirtualHost>` 是两件事，前者负责开端口，后者负责匹配请求
2. 配置、目录、权限、防火墙，四者缺一不可

记下来，下次就不用再尴尬一次了。
