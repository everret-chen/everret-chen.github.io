---
title: SQL注入原理与防护
slug: sql-injection-principles-defense
date: 2026-11-17T00:00:00+08:00
lastmod: 2026-11-17T00:00:00+08:00
description: 从拼接查询讲起：注入点判断、有无回显的攻击分类、攻击思路三步，以及纵深防御与参数化查询修复。
categories:
    - Web 安全
tags:
    - OWASP TOP 10
    - SQL注入
    - python
---

## 什么是SQL注入

SQL注入是指在用户输入里拼接恶意查询语句，使数据库执行非预期的查询语句

## 常见SQL注入场景

以下是一个无任何拦截措施的SQL注入漏洞代码：

```python
def get_user(username):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    query = "SELECT * FROM users WHERE username = '" + username + "'"
    cursor.execute(query)
    result = cursor.fetchall()
    conn.close()
    return result
```

可以看到第五行代码的查询直接拼接username，闭合形式是**单引号闭合**。可以这样攻击：

```url
http://xxx.example.com/login.php?username=1' union select 1, 2, 3 --+
```

此时的SQL语句为：

```SQL
SELECT * FROM users WHERE username = '1' union select 1, 2, 3 --+'
```

可以看到后面的单引号 `'` 被 `--+` 注释掉了，也可以用 `-- -` 、`#`注释

## SQL注入的攻击类型

SQL按不同方式有不同的分类，这里以有无回显进行分类：
 - 有回显： 响应的内容有可能是数据的信息或是报错信息，攻击方式有 **Union注入** 、**报错注入**
 - 无回显： 不会返回响应结果，但会对成功与失败的SQL产生不同反应，攻击方式用 **布尔盲注**、**时间盲注**，这一类攻击建议使用脚本

> 按照http报文的请求方式也可以分为POST注入或GET注入，按照查询的数据类型也可以分为数字型注入和字符型注入

## SQL注入的攻击思路

面对一处可能存在SQL注入的页面，如上文的GET请求方式的漏洞代码，通常在url后面带有参数 `username`。那么就可以去判断它是数字型注入还是字符型注入，例如下面是对字符型注入判断，同时也在判断闭合形式

```
# 回显正常
?username=1" 
# 报错，确定是单引号闭合
?username=1' 
```

而数字型注入通常通过 `1=1` 和 `1=2` 进行判断

```
# 回显正常
1 and 1=1
# 无回显，因为1=2恒假，数据库查询不到数据，所以无结果返回
1 and 1=2
```

确定注入点和注入类型后，就可对通过拼接合适的查询语句查询库名、表名、字段名、数据，如：

```
?username=-1' union select 1, database(), 3 --+
```

综上所述，SQL注入漏洞的攻击思路是：
 - 寻找注入点
 - 判断注入类型
 - 数据查询和绕过手段

## SQL注入漏洞防御

通常要对四个层面形成纵深防御：
 - 前端拦截
 - WAF拦截(匹配报文字段和内容进行拦截)
 - 后端防护
 - 数据库最小权限原则

前端形同虚设，个人认为有总比没有好；WAF则是应用层防火墙，部署好并设置好匹配规则就行。数据库则是限制好执行SQL语句的用户的权限，无必要不给权限

接着讲一下后端的防护，主要就是代码的设计问题，不要信任用户的输入，做好代码与数据的分离，不要直接拼接！不要直接拼接！不要直接拼接！（重要的事情说三遍）（划掉）具体防护措施如下表所示。

|措施|说明|
|:--|:--|
|参数化查询|首选方案，输入与SQL语句分离|
|输入校验|使用白名单限制输入长度、字符集|
|最小权限|不要直接使用root之类的用户连接数据库|
|ORM 框架|占位符（我也不太懂）|
|错误信息隐藏|避免将SQL报错暴露给用户|

上文代码修复示例（参数化查询）

```python
def get_user_safe(username):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE username = ?", (username,))
    result = cursor.fetchall()
    conn.close()
    return result
```
