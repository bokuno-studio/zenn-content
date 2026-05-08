---
title: "OpenAI Codexマスタークラス: プラグイン・サブエージェント・Hooks を62分で完全解説【AI Engineer Europe 2026】"
emoji: "🤖"
type: "tech"
topics: ["openai", "codex", "aiagent", "mcp", "githubactions"]
published: false
---

:::message
この記事は以下の YouTube 動画（約62分）を日本語でまとめたものです。
登壇者: Katia Gil Guzman（@kagigz）・Vaibhav (VB) Srivastav（@reach_vb）/ OpenAI Developer Experience チーム
イベント: AI Engineer Europe 2026（London）
:::

https://www.youtube.com/watch?v=MhHEGMFCEB0

## はじめに

「Codex はコーディングツールではない」——OpenAI の開発体験チームが AI Engineer Europe 2026（London）でそう言い切り、プラグイン・サブエージェント・Hooks・コードレビューまでを62分でデモしたワークショップです。

週間アクティブユーザーが1月から3倍に増えて300万人を超えた Codex の「正しい使い方」が凝縮されています。

---

## Codex とは何か

![AI Engineer Europe 2026 オープニング](/images/openai-codex-masterclass-vaibhav-katia-00-00-30.jpg)

Codex は **オープンエンドなソフトウェアエンジニアリングエージェント**です。コードを書くだけでなく、コマンドの実行・テスト・コードベース探索・PR レビューまで、ソフトウェアエンジニアがやること全般をこなします。

| レイヤー | 内容 |
|---------|------|
| **Foundation models** | GPT-5.2 → 5.3 → 5.4（+ mini / nano）系列 |
| **Unified Agent Harness** | ツール実行・環境セットアップ・安全機能を束ねるラッパー |
| **Surfaces** | App・IDE拡張・CLI・Slack・GitHub |
| **Integrations** | Figma・Linear・Notion などと接続可能 |

モデルは GPT-5.4 で WebSocket 導入により 1.75x 高速化、Fast Mode でさらに +2x が使えます。

---

## 最近リリースされた主な機能

![What's New スライド](/images/openai-codex-masterclass-vaibhav-katia-00-11-30.jpg)

| 機能 | 概要 |
|------|------|
| **Plugins** | Skills + Apps + MCP サーバーを1つにバンドルして再利用 |
| **Mini / Nano モデル** | サブエージェント向けの高速・低コストモデル |
| **Subagents** | タスクを並列・独立したエージェントに分割して実行 |
| **Automations** | バックグラウンドで定期実行するスケジュールタスク |
| **/review** | GitHub PR・CLI・Cursor から使えるコードレビュー |
| **Codex Security** | コミット単位で脆弱性を検出・パッチを自動生成 |

---

## Plugins: スキル・App・MCP をまとめて再利用

プラグインは **Skills（再利用可能な指示）+ Apps（外部サービス接続）+ MCP サーバー** を1つのバンドルにしたものです。毎回手動でセットアップしなくてよくなります。

### デモ1: Game Studio プラグインでゲームを自動生成

![Game Studio プラグイン + Image Gen + Playwright Interactive](/images/openai-codex-masterclass-vaibhav-katia-00-17-00.jpg)

```
「Game Studio プラグインを使って、ブロックのプラットフォームを飛び越える2Dプラットフォーマーゲームを作って」
```

これだけで Codex が:
1. **Image Gen** でスプライト（キャラクター・背景）を自動生成
2. **Playwright Interactive** でブラウザ上に表示・動作確認
3. バグを検出して自動修正

指示は一言で、スプライトも含めて完成したゲームが生成されました。

### デモ2: Google Drive プラグインでスプレッドシートを自動更新

```
「コードベースにある Codex イベント一覧を Google Drive のスプレッドシートに書き込んで」
```

Codex が YAML ファイルを解析 → 57件のイベントをスプレッドシートに書き込み（約2分で完了）。

### Automations: バックグラウンドで定期実行

Katia が実際に使っている自動化の例：

- **毎朝 Slack チェック**: 返信が必要なメッセージをフラグ、前日の要約をトピック別に整理
- **Gmail フィルタ**: 返信が必要なメールだけを抽出、時間的緊急度・スパム判定も

「これで1日に何時間も節約できている」とのこと。

---

## コードレビュー: OpenAI 全 PR を Codex が自動レビュー

![コードレビューのデモ](/images/openai-codex-masterclass-vaibhav-katia-00-28-00.jpg)

> **OpenAI の全従業員（Greg 含む）の全 PR が、デフォルトで Codex にコードレビューされています。**

使える場所:

| 場所 | 方法 |
|------|------|
| **GitHub** | ChatGPT アカウントを連携 → PR 作成時に自動レビュー |
| **CLI / App** | `/review` コマンド |
| **Cursor** | Cursor プラグイン（VB と Dom が先週リリース） |

レビューは差分だけでなく**リポジトリ全体のコンテキスト**を参照するため、変更していない別モジュールへの二次的影響まで指摘します。

---

## サブエージェント: タスクを並列化する

![openai/codex-agents リポジトリのサブエージェントペルソナ一覧](/images/openai-codex-masterclass-vaibhav-katia-00-35-30.jpg)

サブエージェントは**マスタータスクを分割して複数のエージェントに並列実行させる**仕組みです。

### デフォルトで用意される3ペルソナ

| ペルソナ | 用途 |
|---------|------|
| **default** | 汎用フォールバック |
| **worker** | 実装・機能開発に特化 |
| **explorer** | コードベース探索・分析に特化 |

### カスタムサブエージェントの作り方

TOML ファイルを1つ書くだけです（[openai/codex-agents](https://github.com/openai/codex-agents) に45件以上のサンプルあり）。

```toml
name = "architecture-critic"
description = "提案されたアーキテクチャをトレードオフの観点でレビューする"
nickname_candidates = ["Architecture Critic"]
sandbox_mode = "read-only"
model_reasoning_effort = "high"
developer_instructions = """
トレードオフ優先でアーキテクチャをレビューする。
不必要な抽象化・密結合・マイグレーションコスト・運用複雑性を優先的に指摘する。
"""
```

設定できるもの:
- `model`: 使用するモデル（mini / nano も指定可）
- `sandbox_mode`: `read-only` / `full-access`
- `model_reasoning_effort`: `low` / `medium` / `high`
- MCP サーバーの追加（Sentry・Linear など）
- Skills の追加

### デモ: 20サブエージェントが45ペルソナを並列レビュー

Codex が複雑タスクと判断すると**自動でプランモード**を起動し、タスクを分割。20エージェントが同時に走り、全ペルソナファイルをレビュー。「investigate-reporter は権限過多（P1）」「writer はサンドボックスの設定ミスマッチ」などの問題を検出しました。

---

## Bleeding Edge: 実験的な機能

![Hooks の設定例](/images/openai-codex-masterclass-vaibhav-katia-00-51-00.jpg)

### Gorge & Approvals（実験的）

`/experimental` で有効化できます。Codex が特権的な操作（ディレクトリ削除・サーバー公開など）を実行しようとするとき、**サブエージェントが自動で安全性を判断**して承認します。人間の承認疲れを減らすことが目的です。

### Hooks（実験的）

3種類のフックが使えます:

| フック | タイミング |
|--------|---------|
| `pre-tool-use` | ツール実行前 |
| `start` | セッション開始時 |
| `stop` | セッション終了時 |

VB が実際に使っている stop hook:

```python
# keep_going.py
# セッション終了のたびに「もう1パス続けて、バリデーションを1つ実行して止まれ」と指示する
# 長時間タスクをセルフで継続させるための仕組み
```

### Codex Security

GitHub リポジトリをコミット単位でスキャンして脆弱性を検出し、パッチ PR を自動作成します。

### Personality Changes

`/footics` の Personalization からカスタム人格（フレンドリー・実用的など）と追加指示を設定できます。

---

## Q&A: クラウド実行とスキル

**Q: ローカル作業中のタスクをクラウドに引き継げる？**

App の「Cloud」を選択すると可能。**Best of N**（例: 4並列で実行して最良を選ぶ）機能もあります。

**Q: クラウドモードでもローカルのスキルは使える？**

ローカルスキル（Python スクリプトを含むもの）はセキュリティ上の理由で実行されません。ただしリポジトリにチェックインされたスキルはコードベースの一部として読めます。改善は継続中とのこと。

---

## まとめ

| ポイント | 内容 |
|---------|------|
| Codex はコーディングツールではない | テスト・レビュー・セキュリティ・ゲーム生成まで対応 |
| プラグインで繰り返し作業をゼロに | Skills + Apps + MCP を1バンドルで再利用 |
| コードレビューは OpenAI 全社標準 | Greg の PR も Codex がレビューしている |
| サブエージェントで並列化 | TOML 1ファイルでカスタムペルソナを定義 |
| Hooks で長時間タスクを自律実行 | stop hook でセルフ継続が可能 |

openai/codex-agents リポジトリには accessibility-reviewer・architect・bug-reproducer・security-reviewer など45件以上のペルソナが公開されており、すぐに使えます。

---

## 📝 動画全文書き起こし（日本語訳）

**[0:00]** こんにちは。今日は Codex についてお話しします。私は Katia Gil Guzman、こちらは VB こと Vaibhav Srivastav です。私たちは OpenAI の Developer Experience チームで、ロンドンを拠点に活動しています。今日は Codex の概要から始まり、プラグイン・オートメーション・サブエージェント・最新機能まで順番にデモを見せていきます。Codex はオープンエンドなソフトウェアエンジニアリングエージェントです。コードを書くだけでなく、コマンド実行・テスト・コードベース探索など、ソフトウェアエンジニアがやること全般をこなします。GPT-5.x モデルを基盤に、ユニファイドエージェントハーネスが安全性を含めたツール実行を管理します。App・IDE拡張・CLI・Slack・GitHub などから利用でき、Figma・Linear・Notion とも連携できます。

**[5:00]** モデルの進化について。GPT-5.2 から始まり 5.3・5.4 と進化し、mini・nano もリリースしました。WebSocket 導入で約 1.75x 高速化、Fast Mode でさらに 2x 追加速度が得られます。週間アクティブユーザーは1月から3倍以上になり、先夜 300万人を突破しました。Codex App はプロジェクト管理とワークツリー対応で複数フィーチャーを同時並行で開発でき、Windows ネイティブサンドボックスも世界初でサポートしています。

**[10:00]** 最近リリースした機能の概要。プラグイン（Skills + Apps + MCP サーバーのバンドル）、ミニモデル（サブエージェント向け）、コードレビュー（GitHub・CLI・Cursor）、セキュリティ機能などが揃いました。

**[15:00]** プラグインのデモ。Game Studio プラグイン（Image Gen + Playwright Interactive を含む）で「2Dプラットフォーマーゲームを作って」と一言指示するだけで Codex がスプライト生成から動作確認まで自動で行います。Google Drive プラグインでコードベースの YAML から Codex イベント57件をスプレッドシートに自動書き込み（約2分）。

**[20:00]** オートメーションのデモ。Slack 連携で毎日の未返信メッセージを自動チェック・要約したり、Gmail で返信が必要なメールだけを抽出したりしています。

**[25:00]** コードレビューについて。OpenAI の全従業員（Greg 含む）の全 PR がデフォルトで Codex にレビューされています。差分だけでなくリポジトリ全体のコンテキストを踏まえて、変更の二次的影響まで指摘します。

**[30:00]** サブエージェントの説明。複雑なタスクをデコンポーズして複数エージェントに並列実行させます。デフォルト・ワーカー・エクスプローラーの3ペルソナが標準装備で、TOML ファイルを書けば独自ペルソナも作れます。openai/codex-agents リポジトリに45件以上のサンプルがあります。複雑タスクと判断すると Codex は自動でプランモードを起動します。

**[35:00]** サブエージェントのデモ。45件のペルソナファイルを20のサブエージェントが並列でレビューし、権限過多・サンドボックス設定ミスマッチなどを検出しました。

**[40:00]** サブエージェントの応用。セキュリティ脆弱性の多視点分析・機能設計の比較検討（5〜10通りを並列生成）など。各サブエージェントに MCP アクセスや Skills を付与してさらにカスタマイズできます。

**[45:00]** カスタムサブエージェントの作成デモ。「Docs Researcher サブエージェントを作って」と指示するだけで TOML ファイルを自動生成。docs MCP サーバーを組み込み、API リファレンスを参照できるエージェントが完成します。

**[50:00]** 実験的機能。Gorge & Approvals は `/experimental` で有効化でき特権操作を自動承認。Hooks（start・pre-tool-use・stop）でセッション開始時の最新コード pull や、長時間タスクの自律継続が可能です。

**[55:00]** Personality Changes でカスタム人格設定が可能。Codex Security はコミット単位で脆弱性を検出・パッチ PR を自動作成。Cursor プラグインで Cursor セッション内から Codex コードレビューを呼び出せます。

**[60:00]** Q&A。クラウド実行への引き継ぎは App の「Cloud」選択で可能、Best of N（例: 4並列）機能もあります。ローカルスキルのクラウド実行はセキュリティ上の理由で制限されていますが、リポジトリにチェックインされたスキルは参照できます。今後も改善予定とのこと。ありがとうございました。
