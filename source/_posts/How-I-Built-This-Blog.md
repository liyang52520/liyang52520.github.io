---
title: How I Built This Blog — Pandoc + MathJax + Hexo 完美渲染 LaTeX
date: 2026-08-24 14:19:09
tags:
  - Hexo
  - LaTeX
  - Pandoc
  - MathJax
  - 博客
categories:
  - 技术
  - 教程
mathjax: true
---

## 前言

搭建这个博客的过程中，最折磨人的一件事就是 **LaTeX 数学公式的渲染**。看似简单——不就是 `$...$` 和 `$$...$$` 嘛——结果踩了无数坑：CDN 太慢加载一分钟、`\begin{bmatrix}` 报错 `Unknown environment`、公式显示两遍、矩阵被截断……

这篇文章记录最终的解决方案，以及**为什么它能工作**。

---

## 一、问题拆解

LaTeX 在 Hexo 里的渲染链路是这样的：

```
Markdown 源码 → [Markdown 渲染器] → HTML → [MathJax/KaTeX] → 渲染后的公式
                  ↑ 问题 1             ↑ 问题 2
```

### 问题 1：Markdown 渲染器破坏 LaTeX

Hexo 默认使用 `hexo-renderer-marked`。它会把 `\\` 当成换行符、把 `\begin{bmatrix}` 当成标题、把 `$n$` 当成斜体。像这种复杂矩阵：

$$
\begin{bmatrix}
1 & 2 & 3 \\
4 & 5 & 6
\end{bmatrix}
\times
\begin{bmatrix}
7 & 8 \\
9 & 10 \\
11 & 12
\end{bmatrix}
=
\begin{bmatrix}
58 & 64 \\
139 & 154
\end{bmatrix}
$$

在 `marked` 眼里完全是一团乱码——`\\` 被转成 `<br>`，`\begin{bmatrix}` 被当成 `<h1>`，`\times` 成了纯粹的文本。

**解决方案：换 `hexo-renderer-pandoc`。** Pandoc 是 Haskell 写的通用文档转换器，对 LaTeX 有原生级别的理解。配合 `--mathjax` 参数，它会**原封不动**地把所有 `$...$` 和 `$$...$$` 块保留在 HTML 中，不碰任何 LaTeX 语法。

```yaml _config.yml
# pandoc renderer - keep all math as raw TeX
pandoc:
  args:
    - --mathjax
```

### 问题 2：MathJax 加载太慢

保留公式只是第一步，渲染还得靠 MathJax。Butterfly 主题默认从 jsdelivr CDN 加载 MathJax：

```
https://cdn.jsdelivr.net/npm/mathjax@4.1.3/tex-mml-chtml.min.js
```

这个 URL 在国内加载速度极慢，甚至超时。换成 unpkg、cdnjs 也一样——都是境外 CDN。

**解决方案：把 MathJax 下载到本地，从项目目录直接加载。**

```bash
cp node_modules/mathjax/tex-mml-chtml.js source/js/mathjax/
```

然后在 Butterfly 配置中关闭内置 MathJax，用 `inject` 手动注入本地脚本：

```yaml _config.butterfly.yml
math:
  use:         # 关闭 Butterfly 内置 MathJax
  per_page: false

inject:
  head:
    - <script>MathJax={tex:{inlineMath:[['$','$'],['\\(','\\)']]}};</script>
  bottom:
    - <script src="/js/mathjax/tex-mml-chtml.js"></script>
```

这样 MathJax 从 `localhost:4000` 直接加载，零网络延迟，瞬间渲染。

---

## 二、完整架构

```
┌─────────────────────────────────────────────────┐
│                  Markdown 源码                    │
│  $E=mc^2$  │  $$ \int_0^\infty $$  │  \begin{}  │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│          hexo-renderer-pandoc --mathjax          │
│  · 原生理解 LaTeX 语法                            │
│  · 所有公式原样保留为 \(...\) 和 \[...\]            │
│  · \begin{bmatrix} 等环境完整通过                  │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│                   HTML 输出                       │
│  <span class="math inline">\(E=mc^2\)</span>     │
│  <span class="math display">\[...\]</span>       │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│        MathJax 4 (本地 /js/mathjax/)             │
│  · 零 CDN 依赖，秒加载                            │
│  · 支持 \(...\) / \[...\] / $...$ / $$...$$     │
│  · AMSmath 扩展完整支持                           │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
              ✨ 完美公式 ✨
```

---

## 三、为什么 Pandoc 比 marked 好？

| 特性 | `marked` | `pandoc` |
|------|----------|----------|
| LaTeX 感知 | ❌ 无 | ✅ 原生 |
| `\begin{}` 环境 | ❌ 被破坏 | ✅ 完整保留 |
| `\\` 换行 | ❌ 转成 `<br>` | ✅ 保留 |
| `$n$` 简单公式 | ❌ 可能转成斜体 | ✅ 保留 |
| 转义处理 | 需要 `\\\\` 等 hack | 不需要 |

Pandoc 的 `--mathjax` 参数告诉它：**"不要碰任何数学公式，全部留给 MathJax 处理"**。这就是关键。

---

## 四、为什么本地 MathJax 比 CDN 好？

| 方案 | 加载时间 | 可靠性 |
|------|----------|--------|
| jsdelivr CDN | 30s ~ 超时 | 境外，不稳定 |
| unpkg CDN | 10s ~ 超时 | 同上 |
| cdnjs | 10s ~ 超时 | 同上 |
| **本地文件** | **< 100ms** | **100%** |

`tex-mml-chtml.js` 大约 1.5MB，放在本地后 Hexo 开发服务器直接返回，不需要任何外部网络请求。

---

## 五、其他踩过的坑

### 5.1 AMSmath 扩展缺失

`hexo-filter-mathjax` 默认不加载 `ams` 扩展，导致 `\begin{bmatrix}`、`\dfrac`、`\implies` 等报错 `Unknown environment`。需要配置 `packages: [ams]`。

但后来发现直接用 Pandoc + Butterfly 内置 MathJax 更简单，就不需要 `hexo-filter-mathjax` 了。

### 5.2 公式显示两遍

同时装了 `hexo-filter-mathjax`（服务端渲染）和 `markdown-it-texmath`（客户端 KaTeX），导致每一条公式被渲染两次。**只保留一个渲染引擎**即可。

### 5.3 `\\(` 分隔符

Pandoc 的 `--mathjax` 输出 `\(...\)` 和 `\[...\]` 而不是 `$...$` 和 `$$...$$`。MathJax 默认同时支持两者，但需要确保配置中没有覆盖 `inlineMath` 和 `displayMath`。

---

## 六、最终配置清单

```yaml _config.yml
theme: butterfly

pandoc:
  args:
    - --mathjax
```

```yaml _config.butterfly.yml
math:
  use:
  per_page: false

inject:
  head:
    - <script>MathJax={tex:{inlineMath:[['$','$'],['\\(','\\)']]}};</script>
  bottom:
    - <script src="/js/mathjax/tex-mml-chtml.js"></script>
```

```bash
# 一次性安装
npm install hexo-renderer-pandoc mathjax
mkdir -p source/js/mathjax
cp node_modules/mathjax/tex-mml-chtml.js source/js/mathjax/
```

---

## 结语

Hexo + Pandoc + 本地 MathJax 这个组合，经过反复试错终于稳定了。现在写文章只需要：

```markdown
行内公式：$E = mc^2$

块级公式：
$$
\int_{0}^{\infty} e^{-x^2} \, dx = \frac{\sqrt{\pi}}{2}
$$
```

一切就绪，公式秒出。希望这篇文章能帮你少走弯路！🚀