# Galleons 的个人主页

基于 [Quartz 5](https://quartz.jzhao.xyz/) 构建的个人博客 / 数字花园，用于直接发布 Obsidian 笔记。站点地址：<https://galleons2029.github.io>

## 写作与发布流程

1. 把 Obsidian 库中的主题文件夹（连同其中的 `附件` 图片文件夹）直接拖入本仓库的 `content/` 目录，文件夹层级会原样成为网站的目录结构。
2. 给想要公开的笔记在 frontmatter 中加上发布标记：

   ```yaml
   ---
   title: 笔记标题
   publish: true
   ---
   ```

   **没有 `publish: true` 的笔记不会出现在网站上**（显式发布模式）。
3. 提交并 push 到 `master` 分支，GitHub Actions 会自动构建并部署到 GitHub Pages。

参考 `content/示例笔记/` 文件夹，它演示了预期的目录结构、附件图片引用（`![[demo.png]]`）、wikilink 双链、数学公式和代码高亮的写法。

### 日期机制

文章的发布/修改时间自动记录：frontmatter 中的 `date` 字段优先；没有写 `date` 时自动取该文件的 git 提交历史时间。

### 注意事项

- `publish` 过滤只作用于 Markdown 文档，`附件` 中的图片等静态资源会原样输出。**不想公开的图片不要放进 `content/`**。
- `content/private/`、`content/templates/`、`.obsidian/` 目录默认被忽略（见 `quartz.config.yaml` 的 `ignorePatterns`），Obsidian 库的配置文件夹可以放心一起拖进来。

## 本地预览

需要 Node.js ≥ 22：

```bash
npm ci
npx quartz plugin install
npx quartz build --serve
```

然后访问 <http://localhost:8080>。

## 站点配置

- `quartz.config.yaml`：站点标题、语言、插件开关（如 explicit-publish、KaTeX、评论等）
- `.github/workflows/deploy.yml`：自动部署流水线

## 一次性初始化设置

首次启用需在 GitHub 仓库 **Settings → Pages** 中把 **Source** 设置为 **GitHub Actions**。
