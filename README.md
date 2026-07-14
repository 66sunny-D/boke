# boke

📝 个人博客 — Hugo 静态站点，GitHub Pages 自动部署

## 发布流程

1. 在 `content/posts/` 下创建新的 markdown 文章
2. 文章 frontmatter 格式：
   ```yaml
   ---
   title: "文章标题"
   date: 2024-01-01
   tags: ["标签1", "标签2"]
   author: "66sunny-D"
   ---
   ```
3. 提交 Pull Request
4. 合并到 main 分支后自动部署到 GitHub Pages
