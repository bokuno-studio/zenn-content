# Claude Code Auto Memory の RAG 化 設計メモ

## 動機・背景

現状の auto memory（markdown ファイルベース）の限界:
- ファイルが増えると全読み込みのコストが増大
- MEMORY.md インデックスを人間が管理する必要がある（スケールしない）
- 意味的な類似検索ができない

→ Vector DB + RAG 化で解決できる

---

## アーキテクチャ

```
Claude Code CLI / Claude Desktop
  ↓ MCP tool call（ローカル起動）
MCP サーバー（各PCにインストール）
  ├── save_memory(text) — 埋め込んで保存
  └── search_memory(query) — 類似検索して返す
  ↓
sqlite-vec（単一ファイルDB）← iCloud / Dropbox で同期
  ↑ 同期
Obsidian Vault（.md ファイル）← 人間が読み書きする UI
```

---

## 役割分担

| レイヤー | 担当 |
|---|---|
| sqlite-vec | 機械が使う記憶ストア（ベクトル検索） |
| Obsidian Vault | 人間が読む・編集する UI |
| MCP サーバー | Claude と DB の橋渡し（ローカル起動） |

Vector DB と Obsidian は**同じ内容の二重持ち**。  
ノートを編集・削除 → 再インデックスで DB にも反映。

---

## PC をまたいだ同期

**データ（DBファイル + Vault）だけをクラウドに置く。サーバーは各PCでローカル起動。**

| 同期方法 | 対応OS |
|---|---|
| iCloud Drive | Mac のみ |
| Dropbox | Mac / Windows |
| Git（GitHub） | どこでも、ただし中身の公開に注意 |

新PCでやることは「MCP サーバーを入れ直す」だけ。データはそのまま引き継ぎ可能。

---

## フロントエンド（UIクライアント）

| クライアント | ローカルMCP接続 | 備考 |
|---|---|---|
| Claude Code CLI | ✅ | |
| Claude Desktop | ✅ | GUI で MCP 設定できる |
| Claude.ai ブラウザ | ❌ | 外部からローカルサーバーに届かない |
| カスタム Web UI | △ | サーバー公開すれば可 |
| Slack / Discord Bot | △ | サーバー公開すれば可 |

### ブラウザ・スマホからも使いたい場合

MCP サーバーをクラウド（VPS 等）に公開する必要がある。  
その場合は「ローカル完結」ではなくなる。

---

## 実装の現実的な進め方

1. sqlite-vec + MCP サーバーをローカルで作る
2. Vault と DB ファイルを iCloud/Dropbox に移す
3. CLI / Desktop で使えることを確認
4. 必要になったらサーバー公開を検討

---

## 参考: 先行事例（同系統の記事）

- https://note.com/kazuto1027/n/n31c9c2e217dc — Obsidian を AI の長期記憶として設計
- https://zenn.dev/stafes_blog/articles/d7457519a2daec — MCP で Notion/Calendar 連携、朝のルーティン自動化
- https://future-architect.github.io/articles/20260317a/ — 対話で思考を深め Obsidian に構造化保存
- https://zenn.dev/pepabo/articles/ffb79b5279f6ee — Claude Code のセッションログを Obsidian に自動追記
