# YG's Blog

基于 [Hexo](https://hexo.io/) + [Butterfly](https://github.com/jerryc127/hexo-theme-butterfly) 的个人博客，托管于 GitHub Pages。

- 站点：<https://liyang52520.github.io>
- 仓库：<https://github.com/liyang52520/liyang52520.github.io>

## 快速开始（本地开发）

```bash
npm install        # 安装依赖（若报缓存权限错误，见文末"常见问题"）
npx hexo server    # 本地预览
```

浏览器打开 <http://localhost:4000>。

## 日常发布流程

部署已全自动：**推送 `main` 分支 → GitHub Actions 自动构建 → 部署到 GitHub Pages**。
不需要手动执行 `hexo generate` / `hexo deploy`，也不需要把 `public/` 提交上去。

### 发一篇新文章

```bash
# 1. 新建文章（生成到 source/_posts/ 下）
npx hexo new "文章标题"

# 2. 编辑文章（Markdown + Front Matter，见下文"文章写法"）
#    可选：npx hexo server 本地预览，确认效果后 Ctrl+C 停掉

# 3. 提交并推送
git add -A
git commit -m "发布: 文章标题"
git push
```

推送后 1~2 分钟，访问 <https://liyang52520.github.io> 即可看到新文章。
构建状态在仓库 **Actions** 页面查看，绿色 ✓ 表示部署成功。

### 常见场景速查

| 场景 | 操作 |
|------|------|
| 发新文章 | `npx hexo new "标题"` → 编辑 → `git add -A && git commit -m "..." && git push` |
| 修改文章/配置 | 直接改 → `git add -A && git commit -m "..." && git push` |
| 只改了一部分文件 | `git add 具体文件路径` 替代 `git add -A` |
| 内容没改，只想重新部署 | 仓库 Actions 页面 → **Run workflow**（工作流支持手动触发） |
| 本地预览 | `npx hexo server`（想看草稿加 `--draft`） |

### 部署原理

```
git push (main)
  → GitHub Actions 运行 .github/workflows/deploy.yml
  → npm ci 安装依赖 → npx hexo generate 构建静态文件
  → 部署到 GitHub Pages → 站点更新
```

## 常用命令

| 命令 | 说明 |
|------|------|
| `npx hexo new "标题"` | 新建文章 |
| `npx hexo new draft "标题"` | 新建草稿 |
| `npx hexo new page "页面名"` | 新建独立页面 |
| `npx hexo server` | 本地预览（默认 4000 端口） |
| `npx hexo generate` | 生成静态文件到 `public/` |
| `npx hexo clean` | 清理缓存（改了配置不生效时先跑这个） |

## 目录结构

```
blog/
├── _config.yml              # Hexo 主配置（站点信息、搜索数据等）
├── _config.butterfly.yml    # Butterfly 主题配置
├── .github/workflows/       # GitHub Actions 部署工作流（deploy.yml）
├── scaffolds/               # 文章模板（post / draft / page）
├── source/
│   ├── _posts/              # 所有文章（Markdown）
│   ├── css/custom.css       # 自定义样式入口（搜索框等外观定制）
│   ├── images/              # 图片资源
│   └── categories/          # 分类页
└── public/                  # 生成的静态网站（hexo generate 产生，已 gitignore）
```

## 文章写法

文章位于 `source/_posts/`，使用 Markdown 编写，顶部带 Front Matter：

```yaml
---
title: 文章标题
date: 2025-01-01 12:00:00
tags: [标签1, 标签2]
categories: 分类名
---
```

数学公式直接写 LaTeX 即可（Pandoc + 本地 MathJax，见下文"公式渲染"）：

```markdown
行内公式：$E = mc^2$

$$
\int_{0}^{\infty} e^{-x^2} \, dx = \frac{\sqrt{\pi}}{2}
$$
```

## 已启用功能

| 功能 | 说明 |
|------|------|
| 自动部署 | 推送 `main` 自动构建发布到 GitHub Pages |
| 搜索 | 本地搜索（hexo-generator-search），无第三方依赖 |
| 评论 | Giscus（基于 GitHub Discussions），评论数据存在仓库 Discussions 里 |
| 数学公式 | Pandoc 渲染 + 本地 MathJax |
| 自定义样式 | 编辑 `source/css/custom.css` 即可覆盖主题外观（无需改主题源码） |

## 技术栈

- **Hexo** 8.x — 静态博客框架
- **Butterfly** 5.7 — 主题
- **Pandoc** 渲染器 + **MathJax** — 数学公式
- **highlight.js** — 代码高亮
- **GitHub Actions** — 自动构建部署

---

## 公式渲染：为什么用 Pandoc + 本地 MathJax

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

**解决方案：本地加载。** 把 MathJax 复制到项目目录，从本地直接返回：

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

---

## 常见问题

### npm install 报权限错误（EPERM）

本机 `~/.npm` 缓存目录里有历史遗留的 root 属主文件，导致 npm 无法写入。绕过或修复：

```bash
npm install <包名> --cache /tmp/npm-cache-blog   # 临时绕过（推荐）
sudo chown -R 501:20 ~/.npm                       # 一次性修复
```

### 推送失败 / 认证问题

仓库 remote 用的是 SSH（`git@github.com:liyang52520/liyang52520.github.io.git`），
可随时验证：`ssh -T git@github.com`。推送前先 `git pull --rebase` 拉取远程最新，避免冲突。
