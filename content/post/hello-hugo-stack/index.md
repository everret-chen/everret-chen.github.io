---
title: 用 Hugo + Stack 搭一个静态博客
slug: hello-hugo-stack
date: 2025-11-02
lastmod: 2025-11-02
description: 从零建站、配置、写文章到部署到 GitHub Pages 的完整记录。
categories:
    - 建站笔记
tags:
    - Hugo
    - Stack
    - GitHub Pages
---

这篇是本站在建成后发布的第一篇文章，把整个流程记一遍，方便以后换机器重来。

<!--more-->

## 为什么选 Hugo

静态站点生成器里，Hugo 的单二进制、无依赖、秒级构建是最省心的组合。装完只有一个 `hugo.exe`，不用 Node、不用 Python 虚拟环境，几年后回来照样能编译。

```powershell
# 验证是否 extended 版（Stack 主题的硬性要求）
hugo version
# hugo v0.169.0+extended windows/amd64 ...
```

版本号里必须出现 `extended`。普通版在编译 SCSS 时会直接报：

```text
Error: TOCSS: failed to transform "scss/style.scss"
```

## 目录约定

文章用**叶子 bundle**：

```text
content/post/hello-hugo-stack/
├── index.md
└── cover.svg
```

好处是图片直接写相对路径 `![](cover.svg)`，不用关心 baseURL 是根路径还是子路径。

## 常用命令

{{< highlight bash >}}
hugo new content post/my-post/index.md   # 新建文章
hugo server -D                            # 本地预览（含草稿）
hugo --gc --minify                        # 生成发布产物到 public/
{{< /highlight >}}

## 部署

推送到 GitHub 后由 Actions 自动构建，详见仓库里的 `.github/workflows/hugo.yaml`。

> 这里有个容易踩的坑：官方 workflow 模板下载的是普通版 Hugo，需要把文件名改成 `hugo_extended_...`，否则 CI 会在 SCSS 编译阶段失败。
