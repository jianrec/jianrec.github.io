# 周健的技术博客

这是一个基于 Jekyll 和 GitHub Pages 的技术博客，主要记录 LLM、LLM4Rec、Agent、论文阅读和技术随笔。

当前站点结构基于已授权的 `zirconeey.github.io` 源码整理，已移除对方文章、个人信息、文件资源和评论服务配置。

## 本地预览

GitHub Pages 当前运行环境使用 Ruby 3.3.4。建议先安装 Ruby 3.3.4，再执行：

```bash
bundle install
bundle exec jekyll serve
```

浏览器打开 `http://127.0.0.1:4000`。

## 发布到 GitHub Pages

1. 在 GitHub 创建仓库。如果想使用 `https://用户名.github.io`，仓库名必须是 `用户名.github.io`。
2. 修改 `_config.yml` 里的 `url`、`baseurl`、作者信息和联系方式。
3. 把本目录推送到 GitHub：

```bash
git init
git branch -M main
git add .
git commit -m "Initial Jekyll blog"
git remote add origin git@github.com:你的用户名/你的仓库名.git
git push -u origin main
```

4. 在仓库 `Settings -> Pages` 中选择 `GitHub Actions` 作为部署来源。
5. 等待 Actions 完成后，访问 Pages 给出的地址。

## 写新文章

在 `_posts` 下创建 Markdown 文件，文件名格式为：

```text
YYYY-MM-DD-title.md
```

文章开头示例：

```yaml
---
title: "文章标题"
date: 2026-04-28 10:00:00 +0800
categories: [分类]
tags: [标签一, 标签二]
description: "一句话摘要。"
---
```

正文直接使用 Markdown 编写即可。
