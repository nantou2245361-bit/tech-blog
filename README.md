# blog

阿翔的個人部落格 — Hugo 靜態站，部署在 GitHub Pages。

- 網址：https://nantou2245361-bit.github.io/blog/
- 文章目錄：`content/posts/`
- 版型：`layouts/`（自製，無外部主題依賴）
- 部署：push 到 `main` → GitHub Actions 建置 → Pages 發布

## 寫一篇新文章

```bash
hugo new content/posts/<slug>.md
```

frontmatter 只放四個欄位：`title` / `date` / `tags` / `author`。

## 發布流程

```bash
git checkout -b post/<slug>
git add content/posts/<slug>.md
git commit -m "post: <標題>"
git push -u origin post/<slug>
# 開 PR → 合併 main → Actions 自動部署
```

## 本機預覽

```bash
hugo server -D
```
