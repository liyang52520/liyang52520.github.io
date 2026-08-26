# YG's Blog

基于 [Hexo](https://hexo.io/) 的个人博客，使用 [Butterfly](https://github.com/jerryc127/hexo-theme-butterfly) 主题。

## 快速开始

```bash
npm install        # 安装依赖
npx hexo server    # 本地预览
```

浏览器打开 <http://localhost:4000>。

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

## 技术栈

- **Hexo** 7.x — 静态博客框架
- **Butterfly** 主题
- **Pandoc** 渲染器（支持 MathJax 数学公式）
- **highlight.js** 代码高亮

---

## 为什么选择 Pandoc + 本地 MathJax？

### 问题：Markdown 渲染器会破坏 LaTeX

Hexo 默认的 `marked` 渲染器不认识 LaTeX，会把 `\\` 转成 `<br>`、`\begin{bmatrix}` 当标题、`$n$` 当斜体。复杂公式一塌糊涂。

**解决方案：换 `hexo-renderer-pandoc`。** Pandoc 对 LaTeX 有原生支持，配合 `--mathjax` 参数，会把所有公式原封不动保留：

```yaml _config.yml
pandoc:
  args:
    - --mathjax
```

### 问题：CDN 加载 MathJax 太慢

jsdelivr / unpkg / cdnjs 都是境外 CDN，MathJax 加载 10 秒起步甚至超时。

**解决方案：本地加载。** 把 MathJax 复制到项目目录，从 localhost 直接返回：

```bash
cp node_modules/mathjax/tex-mml-chtml.js source/js/mathjax/
```

```yaml _config.butterfly.yml
inject:
  head:
    - '<script>MathJax={tex:{inlineMath:[["$","$"],["\\(","\\)"]]}};</script>'
  bottom:
    - <script src="/js/mathjax/tex-mml-chtml.js"></script>
```

### 渲染链路

```
Markdown 源码 → Pandoc (--mathjax) → HTML (公式原样保留) → 本地 MathJax → ✨完美公式✨
```

### Pandoc vs marked

| 特性 | `marked` | `pandoc` |
|------|----------|----------|
| LaTeX 感知 | ❌ 无 | ✅ 原生 |
| `\begin{}` 环境 | ❌ 被破坏 | ✅ 完整保留 |
| `\\` 换行 | ❌ 转成 `<br>` | ✅ 保留 |
| `$n$` 简单公式 | ❌ 可能转斜体 | ✅ 保留 |

### 本地 MathJax vs CDN

| 方案 | 加载时间 | 可靠性 |
|------|----------|--------|
| jsdelivr CDN | 30s ~ 超时 | 不稳定 |
| **本地文件** | **< 100ms** | **100%** |

现在写文章只需：

```markdown
行内公式：$E = mc^2$

块级公式：
$$
\int_{0}^{\infty} e^{-x^2} \, dx = \frac{\sqrt{\pi}}{2}
$$
```

一切就绪，公式秒出。🚀

---

## 部署

站点已托管在 GitHub Pages: <https://liyang52520.github.io>

仓库: <https://github.com/liyang52520/liyang52520.github.io>

### 自动部署（推荐）

项目已配置 GitHub Actions 工作流（`.github/workflows/deploy.yml`）：

1. 推送 `main` 分支后，工作流会自动执行 `npm ci` + `npx hexo generate`，
   并把生成的静态文件部署到 GitHub Pages。
2. 也可以到仓库 Actions 页面手动触发（`workflow_dispatch`）。

**首次使用**：在仓库 Settings → Pages 中，把 Source（来源）设为
**GitHub Actions**，之后每次推送 main 分支就会自动发布。

### 手动构建预览

```bash
npx hexo clean   # 清理缓存
npx hexo generate  # 生成静态文件到 public/
npx hexo server    # 本地预览 http://localhost:4000
```