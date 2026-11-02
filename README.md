# 我的博客

Hugo + [Stack](https://github.com/CaiJimmy/hugo-theme-stack) 主题的静态博客，部署在 GitHub Pages。

---

## 一、本地启动

Hugo 本体放在项目外层 `.tools/hugo/hugo.exe`（不入库）。如果没有，按下面任一方式装：

```powershell
# 方式一：winget
winget install -e --id Hugo.Hugo.Extended

# 方式二：手动下载（注意文件名里要有 extended）
# https://github.com/gohugoio/hugo/releases
```

> **必须是 extended 版**。验证：`hugo version` 输出里要含 `extended` 字样。

启动预览：

```powershell
# 把 hugo 加进 PATH（本项目内置位置）
$env:PATH = "F:\project\1\.tools\hugo;" + $env:PATH

cd F:\project\1\blog
hugo server -D
# 打开 http://localhost:1313/
```

`-D` 表示连草稿一起渲染，写草稿时用。正式构建：

```powershell
hugo --gc --minify
```

---

## 二、写一篇新文章

```powershell
hugo new content post/my-article/index.md
```

生成的目录结构：

```text
content/post/my-article/
├── index.md
└── screenshot.png   ← 图片放这里
```

`index.md` 的 front matter：

```yaml
---
title: "文章标题"
slug: "my-article"          # 决定 URL：/p/my-article/
date: 2025-11-02
lastmod: 2025-11-02
description: "一句话摘要，显示在卡片上"
image: cover.svg             # 卡片封面，可选
categories: ["Web 安全"]     # 分类，会出现在「分类」页
tags: ["SSRF", "Checklist"]  # 标签
draft: true                  # 写完改成 false
---
```

要点：

- **图片用相对路径**：`![](screenshot.png)`，因为文章是叶子 bundle，不用管 baseURL。
- **`slug` 用英文**：中文会进 URL 变成 `%E6%B8%97%E9%80%8F` 那种编码链接。
- **`<!--more-->`** 之后的内容不会出现在卡片摘要里。
- **`draft: true`** 的文章本地能看到，线上不会发布。

---

## 三、目录说明

```text
blog/
├── hugo.yaml                # 全站配置（站名、菜单、分类、评论）
├── content/
│   ├── post/                # 文章
│   ├── page/                # 独立页面：关于/归档/搜索/链接
│   ├── categories/          # 分类页标题（自动汇总）
│   └── tags/                # 标签页标题
├── assets/img/avatar.svg    # 头像
├── themes/hugo-theme-stack/ # 主题（v4.0.3）
└── .github/workflows/       # 自动部署
```

改站名：`hugo.yaml` 里的 `title`。
改简介：`params.sidebar.subtitle`。
改菜单：各页面 front matter 的 `menu.main`，或在 `hugo.yaml` 的 `menu:` 下加条目。

---

## 四、分类怎么用

分类是 Hugo 的 taxonomy，不需要手动建页面：

1. 文章 front matter 里写 `categories: ["Web 安全"]`
2. 「分类」页自动汇总所有分类
3. 每个分类有自己的列表页：`/categories/web-安全/`

目前的分类：

| 分类 | 内容 |
|---|---|
| 建站笔记 | Hugo、工具、博客相关 |
| Web 安全 | 漏洞原理、检测清单、WriteUp |

想加新分类，直接在文章里写名字就行，Hugo 会自动创建。

---

## 五、启用评论（Giscus）

评论用 Giscus，基于 GitHub Discussions，免费且无广告。**现在配置里是空的，所以评论区不会显示**，需要三步：

### 1. 仓库必须是 public

Giscus 读的是公开仓库的 Discussions。

### 2. 开启 Discussions 并安装 Giscus App

- 仓库 **Settings → General → Features → 勾选 Discussions**
- 访问 <https://github.com/apps/giscus>，点 Install，授权到你的博客仓库

### 3. 拿到 repoID 和 categoryID

打开 <https://giscus.app/zh-CN>，填入仓库名（如 `yourname/yourname.github.io`），
页面会生成一段配置，里面包含 `data-repo-id` 和 `data-category-id` 两个值。

把它们填进 `hugo.yaml`：

```yaml
params:
  comments:
    enabled: true
    provider: giscus
    giscus:
      repo: "yourname/yourname.github.io"
      repoID: "R_kgDOxxxxxxx"        # ← 填这里
      category: "Announcements"
      categoryID: "DIC_kwDOxxxxxxxx"  # ← 填这里
      mapping: pathname
      lang: zh-CN
```

改完重新构建即可，评论框会出现在每篇文章底部。

> `mapping: pathname` 表示按 URL 路径关联讨论。**改了文章 slug 会导致评论丢失**，所以要改 URL 时谨慎。

---

## 六、部署到 GitHub Pages

### 1. 建仓库

仓库名必须是 **`<你的用户名>.github.io`**（用户站，访问地址是根路径，最省事）。

### 2. 改 baseURL

`hugo.yaml` 第一行：

```yaml
baseURL: https://<你的用户名>.github.io/
```

### 3. 推送

```powershell
cd F:\project\1\blog
git init
git add .
git commit -m "init: hugo + stack blog"
git branch -M main
git remote add origin https://github.com/<用户名>/<用户名>.github.io.git
git push -u origin main
```

### 4. 打开 Pages

仓库 **Settings → Pages → Source** 选 **GitHub Actions**（不是 "Deploy from a branch"）。

推送后 Actions 会自动构建，约 40 秒后访问 `https://<用户名>.github.io/`。

---

## 七、常见问题

**构建报 `TOCSS: failed to transform "scss/style.scss"`**
装成了普通版 Hugo，换成 extended。

**Actions 里同样的报错**
workflow 里下载的文件名必须是 `hugo_extended_...`，不是 `hugo_...`。

**文章日期差一天**
workflow 的 `TZ` 没设成 `Asia/Shanghai`。

**评论框不显示**
`repoID` / `categoryID` 没填，或仓库是 private，或没装 Giscus App。

**改主题样式**
不要直接改 `themes/` 里的文件（升级会覆盖）。把要改的模板复制到项目根的 `layouts/` 同名路径再改。

**升级主题**

```powershell
cd themes/hugo-theme-stack
git fetch --tags
git checkout v4.x.x
```
