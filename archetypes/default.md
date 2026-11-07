---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
# URL 用英文短名，避免中文进链接被百分号编码
slug: "{{ .File.ContentBaseName }}"
date: {{ .Date }}
lastmod: {{ .Date }}
description: ""
# 首页卡片不显示封面图。如果某篇想单独加封面，取消下面这行注释即可。
# image: cover.svg
categories: []
tags: []
draft: true
---

正文摘要写在这里，卡片列表会显示这一段。

<!--more-->

## 小标题

正文从这里开始。

> 提示：图片直接放同目录，用相对路径引用即可，例如 `![](screenshot.png)`。
