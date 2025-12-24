+++
title = 'Build My Blog'
date = 2025-12-24T22:26:09+08:00
draft = false
summary = ""
tags = []
categories = []
series = []
cover = ""
+++

作为一个 Go 开发者，我始终认为**查错成本是最高的成本**。因此，在构建个人博客时，我选择了最稳健、最透明、自动化程度最高的方案：

- **核心框架**: [Hugo](https://gohugo.io/) (Go 语言编写，极速构建)
- **主题**: [PaperMod](https://github.com/adityatelange/hugo-PaperMod) (极简、高性能)
- **托管**: [GitHub Pages](https://pages.github.com/) (免费、稳定、版本控制)
- **自动化**: GitHub Actions (CI/CD，自动部署)
- **评论系统**: [Giscus](https://giscus.app/) (基于 GitHub Discussions，无数据库维护成本)

本文将详细复盘整个搭建过程，作为备忘，也供后来者参考。

## 1. 初始化环境与主题

首先，确保本地安装了 Hugo (Extended 版本) 和 Git。

### 创建站点

```bash
hugo new site myblog
cd myblog
git init
```

### 安装 PaperMod 主题

使用 `git submodule` 而不是直接下载文件，这样方便后续更新主题，也符合 Git 的最佳实践。

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 基础配置 (`hugo.toml`)

这是我的核心配置，启用了 Profile 模式和搜索功能：

```toml
baseURL = "https://hui-cyber.github.io/" # 关键：必须是你的最终域名，末尾带斜杠
languageCode = "zh-cn"
title = "Hui's Blog"
theme = "PaperMod"

[markup.goldmark.renderer]
  unsafe = true # 允许内联 HTML，为了灵活性

[outputs]
  home = ["HTML", "RSS", "JSON"] # 开启 JSON 输出供搜索功能使用

[params]
  defaultTheme = "auto"
  ShowReadingTime = true
  
  [params.profileMode]
    enabled = true
    title = "Huihui"
    subtitle = "Go Developer / Cryptography"
    image = "avatar.jpg" # 图片需放在 static/avatar.jpg
    imageWidth = 120
    imageHeight = 120

  [[params.socialIcons]]
    name = "github"
    url = "https://github.com/hui-cyber"

```

## 2. 自动化部署 (GitHub Actions)

为了避免手动 build 和 push `public` 文件夹这种容易出错的“史前方式”，我使用了 GitHub Actions 进行云端构建。

### 创建 Workflow 文件

创建文件 `.github/workflows/deploy.yaml`：

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive # 关键：必须递归拉取 PaperMod 主题
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4

```

### 仓库设置 (关键坑点)

1. **仓库名称**：必须严格命名为 `<username>.github.io`，否则无法部署在根域名。
2. **Pages 设置**：在仓库 Settings -> Pages 中，Source 必须选择 **GitHub Actions**。
3. **权限设置**：在 Settings -> Actions -> General 中，确保 Workflow permissions 选为 **Read and write permissions**。

## 3. 接入 Giscus 评论系统

为了不维护独立的数据库，我选择了利用 GitHub Discussions 存储评论的 Giscus。

### 配置步骤

1. 在 GitHub 仓库 Settings 中开启 **Discussions** 功能。
2. 访问 [Giscus 官网](https://giscus.app/) 获取 `data-repo-id` 和 `data-category-id`。
3. 在 Hugo 项目中创建自定义模板覆盖主题默认行为。

### 注入代码

新建文件 `layouts/partials/comments.html`，填入生成的脚本：

```html
<script src="https://giscus.app/client.js"
        data-repo="hui-cyber/hui-cyber.github.io"
        data-repo-id="<YOUR_REPO_ID>"
        data-category="Announcements"
        data-category-id="<YOUR_CATEGORY_ID>"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>

```

最后在 `hugo.toml` 中开启评论开关：

```toml
[params]
  comments = true

```

## 总结

至此，一个**无需服务器维护**、**全球 CDN 加速**、**带有评论功能**的现代化技术博客就搭建完成了。

以后发布文章只需要简单的三步：

1. `hugo new posts/new-post.md`
2. 写作并保存
3. `git push`

剩下的构建和发布工作，全部交给 GitHub Actions 自动完成。这正是自动化的魅力。

### 下一步建议

1. 保存文件
2. 运行 `git add .` -> `git commit -m "add build tutorial"` -> `git push`
3. 等 1 分钟，去你的博客看看这篇新文章，**顺便在底下用 Giscus 发第一条评论测试一下！**
