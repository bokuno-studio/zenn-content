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

## 参考: 先行事例

### Claude Code × 記憶システム（日本語）

- https://note.com/kazuto1027/n/n31c9c2e217dc — Obsidian を AI の長期記憶として設計、自動更新
- https://zenn.dev/stafes_blog/articles/d7457519a2daec — MCP で Notion/Calendar 連携、朝のルーティン自動化
- https://future-architect.github.io/articles/20260317a/ — 対話で思考を深め Obsidian に構造化保存
- https://zenn.dev/pepabo/articles/ffb79b5279f6ee — Claude Code のセッションログを Obsidian に自動追記
- https://zenn.dev/abalol/articles/claude-code-rag — DevRag を MCP として接続、99% トークン削減を実測
- https://zenn.dev/acntechjp/articles/c1296f425baf03 — Claude Code が RAG を捨てて Agentic Search（glob/grep）に移行した理由
- https://zenn.dev/knowledgesense/articles/d015f1b810c05a — Claude Code がベクトル検索を採用しなくなった理由の技術・経済的分析

### Claude Code × 記憶システム（英語）

- https://dev.to/suede/the-architecture-of-persistent-memory-for-claude-code-17d — memory-mcp の 2 層アーキテクチャ設計（hooks でセッション終了時に自動抽出、重要度別減衰）
- https://medium.com/@mbonsign/how-i-use-claude-a-practical-rag-system-for-long-term-ai-collaboration-9573cd343865 — RAG + MCP Files API で永続記憶を実現した実践例

### 既製の Memory MCP サーバー

| ツール | URL | 特徴 |
|---|---|---|
| mem0 / OpenMemory | https://github.com/mem0ai/mem0 | ハイブリッド検索（Vector + BM25）、40% トークン削減実績 |
| basic-memory | https://github.com/basicmachines-co/basic-memory | Markdown + SQLite 内蔵、外部 DB 不要、WikiLink で知識グラフ化 |
| mcp-memory-service | https://github.com/doobidoo/mcp-memory-service | SQLite-vec + ONNX でオンプレミス完結、5ms 検索 |
| cognee | https://github.com/topoteretes/cognee | グラフ DB + Vector DB のハイブリッド、Claude Code hooks 統合 |
| mcp-mem0 | https://github.com/coleam00/mcp-mem0 | Mem0 を MCP ツールとして公開するシンプルな実装 |

### Vector DB の MCP サーバー（公式）

- https://github.com/qdrant/mcp-server-qdrant — Qdrant 公式、`qdrant-store` / `qdrant-find` の 2 ツール
- https://github.com/chroma-core/chroma-mcp — ChromaDB 公式、Anthropic MCP の公式統合
- https://milvus.io/docs/milvus_and_mcp.md — Milvus 公式ガイド、セマンティック検索 + RAG

### Obsidian + Vector DB 事例（英語）

- https://towardsdatascience.com/i-replaced-vector-dbs-with-googles-memory-agent-pattern-for-my-notes-in-obsidian/ — Vector DB の代わりに LLM の大コンテキストを使う Memory Agent Pattern（脱 RAG の実験）
- https://github.com/alexhholmes/mcp-obsidian — ChromaDB 内蔵の Obsidian セマンティック検索 MCP サーバー
- https://github.com/jacksteamdev/obsidian-mcp-tools — Obsidian プラグインとして MCP ツールを追加
- https://motherduck.com/blog/obsidian-rag-duckdb-motherduck/ — DuckDB VSS で Obsidian Vault を RAG 化

### LLM エージェントの長期記憶実装例

- https://zenn.dev/givery_ai_lab/articles/6e27ba216e0387 — LangGraph + LangMem で意味記憶・エピソード記憶・手続き記憶を統合管理
- https://github.com/FareedKhan-dev/langgraph-long-memory — LangGraph Store + pgvector の参照実装
- https://dev.to/einarcesar/long-term-memory-for-llms-using-vector-store-a-practical-approach-with-n8n-and-qdrant-2ha7 — n8n + Qdrant のノーコード寄り実装

---

## 注目トレンド詳説（2025〜2026）

---

### 1. Claude Code の「RAG 離れ」— Agentic Search とは何か

Claude Code の開発者 Boris Cherny 氏は、初期に Vector DB ベースの RAG を試みたが実運用で断念した。

**RAG が抱えた問題:**

| 問題 | 内容 |
|---|---|
| 同期遅延 | 書いたコードがインデックスに反映されるまでタイムラグ → 最新コードが検索できない |
| 権限管理 | インデックスへのアクセス制御が複雑でセキュリティリスク増大 |
| コスト | 初期インデックス構築 + コード変更のたびに再インデックスが必要 |
| 精度 | 関数名や変数名がベクトル化で干渉し、意図しない結果が出やすい |

**Agentic Search が機能する理由:**

RAG は「検索者が賢くない（人間）」という前提で設計されていた。  
AI エージェントは検索者自身が賢いため、パイプラインの知性が不要になる。

```
従来 RAG:  ユーザーの曖昧なクエリ → 埋め込み → 類似検索 → 結果
Agentic:   Claude が自分で grep/glob の戦略を立て → 反復検索 → 推論
```

- 曖昧な意図を自ら `grep "deploy"` のような具体的クエリに変換できる
- 試行錯誤しながら目的情報にたどり着ける
- ディレクトリ構成からファイルの場所を推測できる（ベクトル検索では不可能）

**Glob/Grep が向いているデータ、RAG が向いているデータ:**

| データ種別 | 向いている手法 | 理由 |
|---|---|---|
| ソースコード | Agentic Search（glob/grep） | 参照関係が明示的、初期コストゼロ |
| ドキュメント・議事録 | Vector DB / RAG | 参照を辿れない、意味検索が有効 |
| 個人のメモ・記憶 | **どちらも有効**（規模次第） | 小規模なら大コンテキストで代替できる |

---

### 2. 大コンテキスト時代の「Vector DB 不要論」

Claude Haiku 4.5 / Sonnet の 200K トークン対応により、Vector DB を使わない選択肢が現実的になってきた。

**Google の Memory Agent Pattern（Towards Data Science の実験より）:**

Vector DB の代わりに SQLite + LLM 直接読み込みで Obsidian ノートの記憶システムを構築。

```
3エージェント構成:
IngestAgent    → テキストから要約・エンティティ・重要度スコアを抽出して保存
ConsolidateAgent → 複数記憶をまたいでパターンを発見・洞察を生成
QueryAgent     → 最近の記憶 + 洞察を合成して回答
```

- 1記憶あたり約 300 トークン、200K コンテキストで 650 件以上を直接読み込み可能
- 埋め込みモデルも外部 Vector DB も不要
- 「**Vector DB はコンテキストが小さかった時代の回避策だった**」という結論

ただし、数万件・数十万件規模になると結局 Vector DB が必要になる。**個人スケールかプロダクションスケールかで答えが変わる。**

---

### 3. Knowledge Graph（ナレッジグラフ）との組み合わせ

Vector DB だけでは「意味が似ている」はわかるが「どう関係しているか」が取れない。そこで Graph DB を組み合わせる動きが出てきている。

**Vector DB と Graph DB の違い:**

| | Vector DB | Graph DB |
|---|---|---|
| 検索の軸 | 意味の近さ（コサイン類似度） | 関係性（エッジをたどる） |
| 得意なこと | 「これに似た記憶を探せ」 | 「この概念から繋がる概念を辿れ」 |
| 苦手なこと | 因果関係・時系列の表現 | 意味的な類似検索 |

**cognee のハイブリッドアーキテクチャ:**

```
テキスト入力
  ↓ cognify
エンティティ抽出 → Graph DB（関係性・因果）
意味埋め込み   → Vector DB（類似検索）
  ↓ recall（クエリ時）
自動ルーティング → Vector / Graph / 両方 を状況に応じて使い分け
```

例えば「先週話した決定事項」なら Graph（時系列エッジ）を辿り、「この件に似た過去の議論」なら Vector で類似検索する。**クエリの性質に応じて自動で使い分けるのがポイント。**

---

### 4. ローカルファースト・外部依存ゼロ化

| ツール | Vector DB | 埋め込みモデル | 外部依存 |
|---|---|---|---|
| mcp-memory-service | SQLite-vec | ONNX（all-MiniLM-L6-v2） | ゼロ |
| basic-memory | SQLite 内蔵 | FastEmbed | ゼロ |
| mem0 / OpenMemory | Qdrant（ローカル） | OpenAI API | API キーのみ |
| cognee | 選択可能 | 選択可能 | 設定次第 |

SQLite-vec は単一ファイルで Vector DB が完結するため、**iCloud/Dropbox での同期と相性が良い**。

---

### まとめ：何を使うべきか（2026年時点）

```
記憶の件数が少ない（〜数百）
  → SQLite + LLM 大コンテキスト直接読み込み（Vector DB 不要）

記憶の件数が中程度（数百〜数万）
  → SQLite-vec + MCP サーバー（ローカルファースト）

プロダクション・チーム利用
  → mem0 / Qdrant + Knowledge Graph（cognee）

コードベースの検索
  → Agentic Search（glob/grep）一択、Vector DB は不要
```
