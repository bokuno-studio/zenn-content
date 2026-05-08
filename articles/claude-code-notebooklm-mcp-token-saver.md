---
title: "Claude Code + NotebookLM MCPでトークン節約しながらAI調査を自動化する"
emoji: "📓"
type: "tech"
topics: ["claudecode", "notebooklm", "mcp", "ai", "automation"]
published: false
---

:::message
この記事は @saidstetic さんによる以下の X ポストの動画（約8分）を日本語でまとめたものです。
個人の学習・整理目的のメモです。内容の正確性はご自身でご確認ください。なお、元動画はスペイン語のため、日本語への翻訳を含みます。
:::

https://x.com/saidstetic/status/2052469867173781817

## はじめに

Claude Code は強力だが、長い調査タスクでトークンを大量消費してしまう——これを解決するのが **NotebookLM MCP** との連携です。

Claude Code から直接 NotebookLM のノートブックを操作できるようになると、PDF や文書をまるごと Claude に読ませる代わりに、NotebookLM に問い合わせさせることができます。つまり **Claude 側のコンテキストウィンドウを消費せずに** 大量の情報を扱えます。

---

## NotebookLM MCP でできること

![NotebookLM MCPの統合デモ画面](/images/screenshot-notebooklm-mcp-01.jpg)

Claude Code のターミナルから NotebookLM に対して、以下のような操作ができます：

| 操作 | 内容 |
|------|------|
| ノートブック一覧取得 | 最新のノートブックや更新日を確認 |
| ノートブック作成 | Claude Code から新規ノートブックを作成 |
| インフォグラフィック生成 | ノートブック内に視覚的なまとめを自動作成 |
| コンテンツ照会 | 特定のノートブックに質問して結果を取得 |
| ファイルダウンロード | 生成されたインフォグラフィックをローカルに保存 |

デモでは「最新のノートブックを教えて」「そのノートブックでインフォグラフィックを作って」「ダウンロードして」という流れを Claude Code のターミナルだけで完結させています。

---

## 実際の操作フロー

![NotebookLMのノートブック一覧](/images/screenshot-notebooklm-mcp-02.jpg)

### 1. ノートブックに問い合わせる

```
> dime cuál es el último cuaderno en NotebookLM
（最新のノートブックを教えて）
```

Claude Code が MCP 経由で NotebookLM に問い合わせ、ノートブック名・作成日・更新日を返します。Claude 自身がコンテンツを読むわけではないので、**トークン消費がほぼゼロ**です。

### 2. インフォグラフィックを生成する

```
> ¿puedes crear una infografía en ese cuaderno?
（そのノートブックでインフォグラフィックを作って）
```

MCP がバックグラウンドで NotebookLM の生成機能を呼び出し、完成したら URL を返します。

### 3. ファイルをダウンロードする

```
> descarga la infografía realizada
（作成したインフォグラフィックをダウンロードして）
```

![Claude CodeがMCP経由でインフォグラフィックURLを取得](/images/screenshot-notebooklm-mcp-03.jpg)

ローカルフォルダに自動保存されます。NotebookLM の Web UI を開かずにすべて完結します。

---

## MCP のインストール方法

使用する MCP サーバーは OSS の `notebooklm-mcp-cli` です。

![notebooklm-mcp-cli GitHubリポジトリ](/images/screenshot-notebooklm-mcp-04.jpg)

**GitHub**: [jacob-bd/notebooklm-mcp-cli](https://github.com/jacob-bd/notebooklm-mcp-cli)

### Claude Code に自己インストールさせる方法（おすすめ）

```
> ¿me puedes dar los pasos para instalar el MCP de NotebookLM?
（NotebookLM の MCP をインストールする手順を教えて）
```

Claude Code 自身にセットアップさせるのが最も簡単です。指示するだけで必要なコマンドを実行してくれます。

### 手動でインストールする場合

```bash
# MCP サーバーをインストール
npx notebooklm-mcp-cli setup

# Google 認証（必須ステップ）
nlm login

# Claude Code に接続
nlm connect claude-code
```

**重要**: `nlm login` で Google アカウントと連携するステップは必須です。ブラウザが開くので、NotebookLM で使っているアカウントでログインしてください。

### CLI コマンド一覧

```bash
nlm notebook list                    # ノートブック一覧
nlm notebook create "Project Name"   # ノートブック作成
nlm source add <notebook> --url "URL"  # ソース追加
nlm audio create <notebook> --confirm  # ポッドキャスト生成
nlm download audio <notebook> <id>   # 音声ダウンロード
```

---

## なぜトークン節約になるのか

通常の Claude Code ワークフローでは、PDF や長文ドキュメントを調査するとき Claude のコンテキストに直接読み込みます。これがトークンを大量消費する原因です。

NotebookLM MCP を使うと：

- **Claude**: 「NotebookLM に○○について聞いて」と指示するだけ
- **NotebookLM**: 自分のノートブック内で検索・回答を生成
- **Claude**: 結果だけを受け取って次のアクションに使う

ドキュメントの内容を Claude のコンテキストに乗せる必要がないため、**同じ月次トークン予算でより多くの調査・作業**ができます。また NotebookLM はソースに基づいて回答するため、**ハルシネーションのリスクも低減**できます。

---

## まとめ

| ポイント | 内容 |
|---------|------|
| Claude Code からノートブックを直接操作 | リスト・作成・照会・ファイル生成すべて対応 |
| トークン消費をほぼゼロに抑える | コンテンツは NotebookLM が保持・検索 |
| 自己インストールが可能 | Claude Code に「インストールして」と言うだけ |
| ハルシネーション低減 | ソースに基づいた回答のみを返す |

大量の資料をもとに調査・整理するワークフローがある方には、特に効果的な組み合わせです。

---

## 📝 動画全文書き起こし（日本語訳）

**[0:00]** NotebookLM の全機能を Claude Code の中から使えると想像してみてください。ノートブックの照会、新規作成——しかも重要なのは、Claude Code がリサーチや PDF・書籍の調査をするとき、トークンを消費しなくていいということです。特定のノートブックに問い合わせさせ、その結果を Claude が受け取って作業を続けられます。それを実演します。ターミナルを開いて Claude に「NotebookLM の最新ノートブックを教えて」と入力しました。最新は N8N 比較のノートブックです。「そのノートブックにインフォグラフィックを作れる？」と聞くと、「生成中」と応答し始めました。NotebookLM の Web 画面を見ると、実際に生成が始まっています。ノートブックへの問い合わせや操作だけでなく、新しいノートブックをまるごと作成させることもできます。インフォグラフィックが完成したと通知が来ました。Claude Code に「インフォグラフィックにアクセスできる？」と聞くと URL を返してくれます。「ダウンロードして」と言うとローカルフォルダに保存——Downloads フォルダに実際に保存されていました。Claude Code から離れることなく、すべて完結しています。

**[5:00]** このセットアップの仕組みを説明します。まず `notebooklm-mcp-cli` の GitHub リポジトリを開きます。より詳しい Claude Code 入門動画も別に用意していますが、今回のポイントはこの MCP をどう接続するかです。ドキュメントには接続コマンドが載っています。技術に詳しくない方向けにおすすめなのは、Claude Code 自身にセットアップさせることです。「NotebookLM の MCP インストール手順を教えて」と言えば、必要なコマンドを自動で実行してくれます。手動でやりたい方のためにコマンドも示されています。**重要なステップは `nlm login`** です。ブラウザが開くので、Google アカウントで NotebookLM にログインします。これだけで認証が完了し、あとは簡単です。このツールは大量の情報を扱いながらも Claude のトークンを節約したい場面で非常に強力です。気に入ったら、次に作ってほしい動画のアイデアをコメントで教えてください。またお会いしましょう。
