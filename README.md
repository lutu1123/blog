# 路途的博客

GitHub Pages + Hugo 静态博客。文章位于 `content/posts/`(Markdown,frontmatter 字段:title/date/tags/author),合并到 `main` 后由 GitHub Actions 自动构建并部署到 https://lutu1123.github.io/blog/ 。

## 发布流程

1. 新增/修改 `content/posts/<slug>.md`
2. 推送到分支并开 PR 合并 `main`
3. GitHub Actions 自动部署,无需本地安装 Hugo

## 目录结构

```
content/posts/   文章(Markdown)
layouts/         站点模板
static/          静态资源(CSS 等)
.github/workflows/  Pages 部署工作流
```
