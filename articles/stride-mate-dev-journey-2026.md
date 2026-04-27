---
title: "1週間でパーソナルトレーニングコーチをゼロから作った話"
emoji: "🏃"
type: "idea"
topics: ["claudecode", "linebot", "nextjs", "supabase", "個人開発"]
published: true
---

:::message
この記事の情報は 2026年4月時点のものです。
:::

## はじめに

「自分のGarminデータを全部持ったAIコーチがいたら最高じゃないか？」

そんな思いつきから、**Stride Mate**というパーソナルトレーニングコーチアプリを作り始めた。4月17日に最初のコミットを打ち、その1週間後の今日（4月24日）にはSquare課金・Garminオンライン同期・使用量ダッシュボードまで動いている。

この記事では、ゼロから立ち上げた7日間の開発記録を振り返る。

---

## プロダクト概要

**Stride Mate**（https://stride-mate.vercel.app ）は、LINEで話しかけると自分のGarminトレーニングデータを踏まえて答えてくれるAIコーチ。

- 「先週の練習どうだった？」
- 「来月のレースに向けて今週何をすべき？」
- 「最近HRVが低いけど休むべき？」

こういった質問に、実際の活動履歴・心拍・睡眠データを参照しながら答えてくれる。

### 技術スタック

| レイヤー | 技術 |
|---|---|
| チャットUI | LINE Messaging API |
| Web（ダッシュボード） | Next.js 15 / Vercel |
| バックエンド同期 | Node.js / Railway |
| DB | Supabase (PostgreSQL) |
| AI | Claude API (Haiku / Sonnet) |
| 課金 | Square Subscriptions |
| Garmin連携 | garmin-connect（非公式API）|

---

## Day 1（4/17）: MVPを動かす

### 設計から実装まで

午後1時にリポジトリを作成。最初にやったのは設計ドキュメントの整理だった。

```
docs: システム設計書を追加
docs: フロントエンドをLINEに変更
docs: 認証をLINE Login一本化
```

3時間で設計を固め、17時には「training-coach-bot MVP実装」コミット。

このMVPで動いていたもの:
- LINE Webhook受信 → Claude API呼び出し → 返信
- Supabase Auth（LINE OAuth）
- Garmin ZIPアップロード（ダッシュボードから手動）
- アクティビティのDB格納・RAG検索

### デプロイで詰まった時間

MVPのコードが動いても、本番に出すまでが地獄だった。

```
fix: move Dockerfile to repo root for correct Railway build context
fix: use full npm install in Dockerfile
fix: include all workspace package.json files in Dockerfile
fix: set builder=DOCKERFILE in railway.toml to bypass nixpacks
fix: railway.toml dockerfilePath without leading ./
```

RailwayのNixpacksがモノレポの`@tcb/shared`パッケージを解決できず、結局Dockerfileを使う構成に切り替えた。20時過ぎにようやく本番デプロイ成功。

---

## Day 2-3（4/18-19）: 認証バグとZIPインポートの修正

### LINE OAuthのES256問題

翌日、実際にログインしようとすると認証が通らない。

調査すると、Supabase AuthのカスタムOIDC実装がES256（楕円曲線）の署名アルゴリズムを扱えない問題だった。SupabaseのカスタムOAuthはHS256前提で作られており、LINEのJWKSエンドポイントが返すES256には非対応だった。

解決策: **LINE OAuthを手動実装**。Supabaseの組み込み機能を使わず、LINEのOAuthフローをNext.js APIルートで自前で書いた。

```
fix: implement LINE OAuth manually to bypass Supabase OIDC ES256 issue
```

### ZIPインポートのOOM問題

GarminのデータエクスポートZIPは数GB になることがある。最初の実装はZIP全体をメモリに展開していたため、OOMでクラッシュした。

```
fix: stream ZIP processing to avoid OOM on large Garmin exports
fix: stream ZIP via fetch to avoid Supabase download() buffering entire file
```

Supabaseの`storage.download()`がバッファリングしてしまうため、署名付きURLをfetchしてストリームのまま処理するよう変更した。

---

## Day 4-5（4/20-21）: アーキテクチャの見直し

### RAGをやめた

当初はベクトル埋め込みでRAG検索する設計だったが、実際に使ってみるとSQLで直接引いたほうが正確だった。

Garminデータは構造化されていて時系列が明確。「先週の走行距離」「直近30日のHRV推移」は、ベクトル類似度検索より`WHERE date BETWEEN`のほうが圧倒的に正確に取れる。

```
feat: replace RAG with direct SQL fetch + improve coach tone
refactor: embedding削除・Garmin即時同期・アプリ名stride-mate変更
```

embeddingテーブルとRAGロジックを全部削除してコードがすっきりした。

### コーチの口調調整

「コーチとして」というシステムプロンプトを書いたが、最初のClaude応答はやけに丁寧すぎた。ランニングコーチらしい、少し距離感のある口調に調整した。

### Garminオンライン連携

ZIP手動アップロードに加え、Garmin Connectから自動同期する機能を追加。毎朝3時にRailwayのsync-serverが動いて新着データを取ってくる。

garmin-connectライブラリはGarminの非公式APIを使っており、2026年3月にGarminの認証APIが変更されたことで多くのライブラリが動かなくなっていた。動作確認できたライブラリを選んで使っている。

ただし、Garminオンライン連携には課題がある。**初期の全件取得は動くが、差分取得が安定しない**。セッションが切れたり、取得できる期間に制限があったりで、毎朝確実に昨日分だけ取ってくるという動作が難しい。

### Strava連携で差分を補う

この差分取得問題の解決策として、Stravaと連携する方法を取った。

StravaはOAuth対応の公式APIがあり、Webhookで新着アクティビティの通知を受け取れる。GarminデバイスとStravaを連携しておけば、走るたびにStravaにアクティビティが同期され、そこからStride Mateにも自動で入ってくる。

- 初期の全件取得 → Garmin ZIP（一括エクスポート）
- 日々の差分 → Strava（Webhook + 公式API）

この2本立てで、信頼性の高いデータ取り込みを実現した。

---

## Day 5-6（4/21-22）: Square課金の実装

### プラン設計

FreeプランとStandardプランの2段階構成。

AI利用枠は「実際のAPI費用の何倍まで許容するか」で決める。月額料金を価格係数で割り、それに原価倍率をかけた値をAI利用枠にする設計にした。

正直なところ、VercelやSupabase・Railwayなどのインフラ固定費を考えると、この設計では赤字になる。サブスク型AIプロダクトの適切な価格設計がどうあるべきかはまだよくわかっていない。とりあえず「使いすぎたら止まる」仕組みだけ作っておいて、あとは実態を見ながら調整するつもりでいる。

### Square Webhookがそもそも登録されていなかった

課金を実装したはずなのにプランが切り替わらない。調査すると、**Square Webhookが一度も登録されていなかった**。

コードは完璧に書いてあっても、Squareのダッシュボードで登録しないと通知が来ない。Square APIで手動登録し、署名キーを更新してようやく動いた。

```
fix: Square webhookサブスクリプション登録＋署名キー更新・解約ハンドラ修正
```

### キャンセル処理のSquareイベント仕様

最初、`subscription.canceled`というイベントを処理しようとしていた。ところがそんなイベントタイプは存在しない。

Squareの仕様では、解約も`subscription.updated`で来て、`status === "CANCELED"`で判別する。ドキュメントを読まないと気づかない落とし穴だった。

```typescript
case "subscription.updated": {
  if (sub.status === "CANCELED" || sub.status === "DEACTIVATED") {
    // freeに戻す
    await db.from("users").update({ plan: "free", plan_expires_at: null })
      .eq("square_customer_id", sub.customer_id);
    break;
  }
  // 通常のプラン更新処理...
}
```

---

## Day 7（4/24）: ダッシュボード統合

### /usage と /upgrade を /dashboard に統合

最初は `/dashboard`・`/usage`・`/upgrade` を別ページとして作っていた。使ってみると「ページをまたぐのが面倒」という問題が明らかになった。

一度タブUIに変えてみたが、「タブは別ページと概念的に同じ」というフィードバックで全廃。すべての情報を1ページにスクロールで並べる構成にした。

### 使用量表示の改善

- 円グラフの中: 金額 → パーセント表示
- 「無料枠」→「利用枠」（有料ユーザーにも表示されるので）
- Standardプランの利用枠を実態に合わせて修正

---

## 7日間で学んだこと

### 1. 外部サービスはWebhook登録を忘れない

当たり前すぎるが、「コードが動いている＝動作している」ではない。Squareのような課金サービスはダッシュボード側の設定が必要。

### 2. Garminデータは構造化データ。RAGより直クエリ

時系列・定量的なデータに対してはSQL直クエリが勝る。RAGは非構造化テキスト向け。設計段階で気づければよかった。

### 3. ストリーミング処理は最初から考慮する

大きなファイルを扱うなら最初からストリームで書く。「動いたらリファクタ」は後から高くつく。

### 4. 認証の非標準部分は自前実装が早い

SupabaseのカスタムOAuthがES256に対応していないなど、エッジケースで詰まると解決策の見通しが立たない。手動実装のほうが制御しやすい。

---

## 現在の状況

動いているもの:
- LINEからチャット（Garminデータ参照）
- Garmin ZIP一括インポート
- Garminオンライン自動同期（毎朝3時）
- Strava連携
- Square課金（Standardプラン）
- 使用量ダッシュボード

現在は自分と数名の知人だけが使っている段階だが、Garminユーザーであれば誰でも使える。興味があればぜひ試してみてほしい。反応次第でもっと作り込んでいく予定。

---

## おわりに

Claude Codeを使いながら1人で作っていると、「コードは書けるけど設定ミスに気づかない」「外部サービスの仕様を把握しきれない」という場面が多かった。特にSquare Webhookの未登録は数日気づかなかった。

プロダクトの骨格はすぐ作れる時代になったが、本番で動かし続けるための「つなぎ目」の設定や仕様確認が相変わらずハマりやすいポイントだと感じた。


