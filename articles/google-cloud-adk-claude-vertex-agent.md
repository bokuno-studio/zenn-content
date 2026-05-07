---
title: "Google Cloud ADK + Claude で始めるAIエージェント本番運用【30分ワークショップ】"
emoji: "🤖"
type: "tech"
topics: ["googlecloud", "claude", "aiagent", "vertexai", "mcp"]
published: true
---

:::message
この記事は @angeldot_ さんによる以下の X ポストの動画（約30分）を日本語でまとめたものです。
登壇者: Ivan Nardini（Google Cloud Developer Relations Engineer, AI/ML）/ Anthropic 主催イベントにて収録
:::

@[tweet](https://x.com/angeldot_/status/2052104456846622957?s=20)

## はじめに

AIエージェントの開発はできた。でも本番に出せない——そんな壁を Google Cloud の Ivan Nardini が 30 分で突破する方法を実演したワークショップです。

ADK・MCP・Vertex AI Agent Engine・A2A Protocol の4本柱を使い、Claude を脳みそにしたマルチエージェントシステムを構築・デプロイするまでを一気通貫で解説しています。

---

## AIエージェントを本番化できない3つの理由

![Building AI agents - Powerful, but complex](/images/google-cloud-adk-claude-vertex-agent-00-01-30.jpg)

エージェントのプロトタイプはできたのに、本番化で詰まるのはなぜか。3つの根本的な原因があります。

| 課題 | 内容 |
|------|------|
| **断片化したエコシステム** | フレームワークが乱立し、何を使えばいいか不明確 |
| **統合の難しさ** | 異なるフレームワーク同士のエージェント間通信が困難 |
| **運用・ガバナンスの欠如** | モニタリング・ロギング・スケーリングをすべて自前実装する必要がある |

この3つを解決するために Google Cloud が用意したのが **Agentic Stack** です。

## Google Cloud Agentic Stack の全体像

![Our agentic stack](/images/google-cloud-adk-claude-vertex-agent-00-04-30.jpg)

4層で構成されるスタックで、それぞれが上記の課題に対応しています。

| レイヤー | 役割 |
|---------|------|
| **Agent Development Kit (ADK)** | OSS のコードファーストなエージェント開発フレームワーク |
| **Model Context Protocol (MCP)** | LLM へのコンテキスト提供を標準化するオープンプロトコル |
| **Vertex AI Agent Engine** | エージェントを本番スケールでデプロイ・管理するマネージドプラットフォーム |
| **Agent2Agent (A2A) Protocol** | 異なるフレームワーク間のエージェント通信を実現するオープン標準 |

---

## Demo 1: 最初のエージェントを3ファイルで作る

![ADK でのエージェント実装](/images/google-cloud-adk-claude-vertex-agent-00-11-45.jpg)

バースデープランナーエージェントを例に、ADK の最小構成を見ていきます。

```python
from google.adk.agents import LlmAgent
from google.adk.models.anthropic_llm import Claude
from google.adk.models.registry import LLMRegistry

root_agent = LlmAgent(
    name="birthday_planner",
    model="claude-3-7-sonnet@20250219",
    description="誕生日パーティーの計画を手伝うエージェント",
    instruction="ゲストリストの作成、会場の提案、スケジュール調整を行う..."
)
```

必要なのは `agent.py`・`.env`・`requirements.txt` の3ファイルだけ。起動は1コマンドです。

```bash
adk run birthday_planner    # CLI で対話
adk web                     # ブラウザ UI で対話＋デバッグ
```

ADK は `LlmAgent`・`SequentialAgent` など複数のエージェントパターンを提供しており、Claude をはじめ Gemini など複数モデルに対応しています。

---

## Demo 2: MCPでマルチエージェント化する

![MCP を使ったカレンダーエージェントのコード](/images/google-cloud-adk-claude-vertex-agent-00-16-20.jpg)

「誕生日を計画する」だけでなく「カレンダーに予定を入れる」まで自動化するには、エージェントを複数連携させます。

構成：
- **BirthdayPlannerAgent** — パーティーの提案
- **CalendarServiceAgent** — MCP サーバー経由でカレンダー操作
- **EventOrganizerAgent** — 上記2つを束ねるオーケストレーター

MCP サーバーの接続はわずか2行です。

```python
mcp_tools, exit_stack = await MCPToolset.from_server(
    connection_params=SseServerParams(url=MCP_CALENDAR_SERVER_URL)
)

agent = LlmAgent(
    name="CalendarServiceAgent",
    model="claude-3-7-sonnet@20250219",
    tools=mcp_tools,
    ...
)
```

既存の MCP サーバーであればどれでもそのままツールとして組み込めます。オーケストレーターは各エージェントをツールとして渡すだけで自動ルーティングしてくれます。

---

## Demo 3: Vertex AI Agent Engine にデプロイする

![Vertex AI Agent Engine アーキテクチャ](/images/google-cloud-adk-claude-vertex-agent-00-21-00.jpg)

エージェントをユーザーに公開するには、スケールと運用基盤が必要です。Agent Engine はそれをマネージドで提供します。

```python
agent_engines.create(
    agent=root_agent,
    requirements=["google-cloud-aiplatform[adk]"]
)
```

デプロイ後に自動で使えるようになるもの：

- Cloud Trace / Logging / Monitoring による **Observability**
- セッション管理（会話履歴の永続化）
- Vertex AI Evaluation Service との統合（ログを使ったエージェント改善サイクル）

コンソール上でレイテンシ・クエリ数・CPU/メモリをリアルタイム監視できます。LangGraph・LangChain・LlamaIndex・CrewAI など他フレームワーク製エージェントも同様にデプロイ可能です。

---

## ボーナス: A2A Protocol — 異フレームワーク間の通信

LangChain で作ったエージェントと ADK のエージェントを連携させたい場合に必要なのが **Agent2Agent (A2A) Protocol** です。

基本概念は2つ：

- **Agent Card**: エージェントのデジタル名刺。外部エージェントが「このエージェントは何ができるか」を知るための情報
- **Agent Skills**: エージェントの具体的な機能・API の記述

HTTP / JSON-RPC ベースのオープン標準で、エンタープライズ向けのセキュリティ機能も組み込まれています。この仕組みにより、異なるフレームワークで作られたエージェント同士がフレームワークの壁を超えて協調動作できます。

---

## まとめ

| ポイント | 内容 |
|---------|------|
| ADK は3ファイル・1コマンドで最初のエージェントが動く | 開発フローがシンプル |
| MCP 統合は2行 | 既存の MCP エコシステムをそのまま活用できる |
| Agent Engine でゼロ運用デプロイ | Observability・スケーリング・セッション管理が自動 |
| A2A でフレームワーク間の壁を超える | Claude・Gemini・LangChain・CrewAI が共存できる |

ADK・MCP・Agent Engine・A2A を組み合わせることで、開発から本番スケールまで一気通貫で対応できるスタックが揃っています。

---

## 📝 動画全文書き起こし（日本語訳）

**[0:00]** 皆さん、こんにちは。このセッションでは、Vertex AI 上の Claude を使ってAIエージェントを構築する方法を解説します。まず背景を整理しましょう。AIエージェントの構築は非常に強力ですが、プロトタイプが完成した後に本番化するのが極めて困難という現実があります。主な理由は3つです。第1に、エコシステムが断片化しており、多数のフレームワークやツールから何を選ぶべきかが不明確。第2に、異なるフレームワーク間でエージェントを連携させることが難しい。第3に、モニタリング・ロギング・ガバナンスといった運用面の管理が困難です。これらを解決するために、Google Cloud は独自のAgentic Stackを定義しました。スタックは4層構造で、ADK（Agent Development Kit）・MCP・Vertex AI Agent Engine・A2A プロトコルで構成されます。

**[5:00]** Vertex AI のモデルガーデンから Claude にアクセスする方法をお見せします。Model Garden はさまざまなモデルを発見・デプロイ・管理できる中央ハブで、今朝 Claude 4 もリリースされました。Vertex AI Studio を使えばモデルをすぐにテストして API に統合できます。今日は Claude 3.7 を使ってデモを進めます。

**[10:00]** ADK の基本コンセプトです。`LlmAgent`（エージェントの脳）、`Tool`（エージェントにスキルを付与する仕組み）、`Runner`（セッションを管理してエージェントを実行するコーディネーター）、`Session`（会話状態を保持する仕組み）の4つが柱です。`adk run` と `adk web` の2つの CLI で、プログラム的にも UI でもエージェントと対話できます。ADK は Claude にも Gemini にも対応しており、今日は Vertex AI の LLM レジストリ経由で Claude を使います。

**[15:00]** マルチエージェントシステムの実装を見せます。バースデープランナーエージェントに加え、MCP サーバー経由でカレンダーを操作するエージェントと、それらを束ねるオーケストレーターを追加します。MCP サーバーの接続は `MCPToolset.from_server()` を呼ぶだけで、自動的にツールに変換されます。Web UI では、どのエージェントが何をしているかをリアルタイムで確認できます。バースデーエージェントには Gemini を、カレンダーエージェントには Claude を使うハイブリッド構成も可能です。

**[20:00]** Vertex AI Agent Engine によるデプロイです。`agent_engines.create()` を呼ぶと、イメージのビルドとエージェントのデプロイが自動で行われます。コンソールでクエリ数・レイテンシ・CPU/メモリを監視でき、セッション情報もすべて記録されます。LangGraph・LangChain・LlamaIndex・CrewAI など他フレームワークで作ったエージェントも同様にデプロイ可能です。Vertex AI Evaluation Service と連携してエージェントを継続改善できます。

**[25:00]** A2A プロトコルのボーナスセクションです。異なるフレームワークで構築・デプロイされたエージェント同士が協調するには共通言語が必要です。それが Agent2Agent (A2A) Protocol です。`Agent Card`（エージェントのデジタル名刺）と `Agent Skills`（機能の記述）という概念を使い、HTTP/JSON-RPC ベースでエンタープライズグレードのセキュリティを持った通信を実現します。このプロトコルは今月末のウェビナーで詳しく解説予定です。

**[30:00]** ご参加ありがとうございました。30分で駆け足でしたが、ADK のサンプルコードと、A2A を含む詳細ウェビナーの QR コードを共有しました。質問があればいつでも声をかけてください。
