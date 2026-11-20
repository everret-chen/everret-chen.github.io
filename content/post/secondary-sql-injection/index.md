---
title: 二次注入
slug: secondary-sql-injection
date: 2026-11-21T00:00:00+08:00
lastmod: 2026-11-21T00:00:00+08:00
description: 恶意数据先入库、被调用时触发注入——以 sqli-labs Less-24 为例分析转义不对称。
categories:
    - Web 安全
tags:
    - SQL注入
    - sqli-labs
---

## 什么是二次注入

二次注入是攻击者将恶意数据存入数据库，等数据被调用的时候触发SQL注入。具体就是存入数据时后端代码对用户输入做了处理，比如转义，但存入数据库的还是原始数据。等调用恶意数据的时候，相关后端代码没有对“信任数据”做处理，SQL注入触发！！

## 具体情景

那sqli—labs的Less-24关来说，login_create.php对用户输入做了转义处理mysql_escape_string()，但pass_change.php没有转义，显然开发者非常信任数据库的数据~

```PHP
// login_create.php
function sqllogin(){

   $username = mysql_real_escape_string($_POST["login_user"]);
   $password = mysql_real_escape_string($_POST["login_password"]);
   $sql = "SELECT * FROM users WHERE username='$username' and password='$password'";
   // .....
}
```

对比上下代码，可以发现pass_change.php中的$username没有进行转义，这就是二次注入的触发点

```PHP
// pass_change.php 
if (isset($_POST['submit']))
{
	# Validating the user input........
	$username= $_SESSION["username"];
	$curr_pass= mysql_real_escape_string($_POST['current_password']);
	$pass= mysql_real_escape_string($_POST['password']);
	$re_pass= mysql_real_escape_string($_POST['re_password']);
	
	if($pass==$re_pass)
	{	
		$sql = "UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass' ";
```
