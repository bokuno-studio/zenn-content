---
title: "「記事公開して」だけでZennとdev.toに同時投稿できるClaude Codeプラグイン作った"
emoji: "📝"
type: "tech"
topics: ["claudecode", "zenn", "devto", "ai"]
published: true
---

Claude Codeに「記事公開して」と言うだけで、Zenn（日本語）とdev.to（英訳）に同時投稿してくれるプラグインを作りました。

## 作ったもの

**zenn-post** — Claude Code Plugin / Skill

https://github.com/bokuno-studio/zenn-post-cc-plugin

記事を書いて公開するまでの作業（下書き作成・Markdown整形・git push・dev.to API呼び出し）をClaude Codeに丸ごと任せられます。

## できること

- 「記事公開して」→ Zenn（日本語）と dev.to（英訳）に**同時公開**
- NotionページのURLを渡す → 内容を読み込んで記事に変換して投稿
- テーマを口頭で伝えるだけで執筆〜git pushまで全自動
- Mermaid図をローカルで自動PNG変換（外部サービス不要）
- 公開前にプレビュー確認 → OKで `git push` まで実行

```
「このNotionページをZennに上げて」
「記事公開して」          ← Zenn + dev.to 両方に投稿
「Zennだけに投稿して」    ← Zenn のみ
```

## なぜ作ったか

Zennへの投稿フローが地味に面倒でした。

1. Markdownファイルを作る
2. frontmatterを書く
3. 本文を書く
4. `published: true` に変える
5. git commit & push
6. （dev.toにも上げたいなら）英訳してAPIを叩く

これを毎回やるのが億劫で、「Claude Codeに全部やらせよう」と思ったのが動機です。

## 技術的なポイント

### Claude Code Skill（SKILL.md）として実装

Claude Code にはプラグイン・スキル機能があり、`SKILL.md` にプロンプトを書くだけでスキルとして登録できます。コードは一切書かずに、「どう動くか」を自然言語で定義するだけです。

```
skills/zenn-post/SKILL.md  ← これだけ
```

### dev.to には curl で直接 POST

dev.to の API はシンプルで、curl 一発で投稿できます。Claude Codeが `.env` からAPIキーを読み込んでリクエストを組み立てます。

```bash
curl -X POST https://dev.to/api/articles \
  -H "api-key: $DEVTO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "article": { "title": "...", "published": true, "body_markdown": "..." } }'
```

### Mermaid図はローカルで PNG 変換

dev.to は Mermaid をレンダリングしないので、`mermaid-cli` でローカル変換して画像化します。変換した PNG は zenn-content リポジトリの `images/` に置いて `raw.githubusercontent.com` 経由で配信するため、外部サービスは一切使いません。

```bash
npx @mermaid-js/mermaid-cli -i diagram.mmd -o diagram.png -t default -b white
```

## インストール

**前提:**
- Claude Code がインストール済み
- Zenn アカウントと zenn-content リポジトリ（public）が GitHub に存在すること
- dev.to アカウントと API キー

**手順:**

```bash
git clone https://github.com/bokuno-studio/zenn-post-cc-plugin ~/.claude/plugins/zenn-post
```

`skills/zenn-post/SKILL.md` の環境情報セクションを自分のパスに書き換えて、Claude Codeに登録するだけです。

```bash
claude plugin marketplace add ~/.claude/plugins/zenn-post
claude plugin install zenn-post
```

## 更新履歴

| バージョン | 日付 | 内容 |
|-----------|------|------|
| v0.1.0 | 2026-04-16 | 初回リリース（Zenn投稿のみ） |
| v0.2.0 | 2026-04-17 | dev.to同時投稿・Mermaid CLI対応を追加 |

## まとめ

Claude Code の Skill 機能を使うと、「コードを書かずにClaudeの動作を定義する」という新しい自動化のアプローチが取れます。記事投稿のような定型作業をスキルとして切り出すと、口頭指示だけで完結するのが気持ちいいです。

気になった方はぜひ試してみてください。フィードバック歓迎です！
