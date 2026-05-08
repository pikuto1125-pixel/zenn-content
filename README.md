# zenn-content

[Zenn](https://zenn.dev/pikuto1125-pixel) の記事を管理するリポジトリ。zenn-cli + GitHub 連携で自動公開。

## 構成

- `articles/` — 記事（slug.md）
- `books/` — 本（未使用）

## 投稿フロー

```bash
npx zenn new:article --slug my-slug
# 編集 → preview → git push で Zenn が自動公開
git add articles/
git commit -m "post: my-slug"
git push origin main
```

## 関連

- [Zenn dashboard](https://zenn.dev/dashboard)
- [AI Hack Lab](https://ai-hack-lab.com/) — Arena Blueprint 公開中
- [Qiita](https://qiita.com/pikuto1125-pixel)

## CLI ガイド
* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)
