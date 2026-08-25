# YG's Blog

基于 [Hexo](https://hexo.io/) 的个人博客，使用 [Butterfly](https://github.com/jerryc127/hexo-theme-butterfly) 主题。

## 快速开始

```bash
# 安装依赖
npm install

# 本地预览
npx hexo server
```

浏览器打开 <http://localhost:4000> 即可预览。

## 目录结构

```
blog/
├── _config.yml              # Hexo 主配置
├── _config.butterfly.yml    # Butterfly 主题配置
├── scaffolds/               # 文章模板（post / draft / page）
├── source/
│   ├── _posts/              # 所有文章（Markdown）
│   ├── _drafts/             # 草稿
│   ├── images/              # 图片资源
│   └── categories/          # 分类页
├── themes/                  # 主题
└── public/                  # 生成的静态网站（hexo generate 后产生）
```

## 常用命令

| 命令 | 说明 |
|------|------|
| `npx hexo new "标题"` | 新建文章 |
| `npx hexo new draft "标题"` | 新建草稿 |
| `npx hexo new page "页面名"` | 新建独立页面 |
| `npx hexo server` | 启动本地预览 |
| `npx hexo server --draft` | 预览时包含草稿 |
| `npx hexo generate` | 生成静态文件 |
| `npx hexo clean` | 清理缓存 |
| `npx hexo deploy` | 部署到服务器 |

## 文章写法

文章位于 `source/_posts/`，使用 Markdown 编写。每篇文件顶部需要包含 Front Matter：

```yaml
---
title: 文章标题
date: 2025-01-01 12:00:00
tags: [标签1, 标签2]
categories: 分类名
---
```

正文从这里开始...

## 部署

当前部署方式未配置。如需部署到 GitHub Pages：

```bash
npm install hexo-deployer-git --save
```

然后在 `_config.yml` 中配置：

```yaml
deploy:
  type: git
  repo: https://github.com/你的用户名/你的仓库.git
  branch: gh-pages
```

执行部署：

```bash
npx hexo clean && npx hexo deploy --generate
```

## 技术栈

- **Hexo** 7.x — 静态博客框架
- **Butterfly** 主题
- **Pandoc** 渲染器（支持 MathJax 数学公式）
- **highlight.js** 代码高亮