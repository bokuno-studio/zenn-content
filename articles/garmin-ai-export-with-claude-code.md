---
title: "Claude Codeでガーミンの過去データをAIに食わせるWebアプリを1日で作った"
emoji: "⌚"
type: "tech"
topics: ["claudecode", "nextjs", "typescript", "garmin", "square"]
published: false
---

:::message
この記事の情報は 2026年5月時点のものです。
:::

## 作ったもの

**Garmin AI Export** — Garmin Connectからエクスポートした活動・睡眠・健康データのZIPを、ChatGPT / Gemini / Claude が読みやすいCSV形式に一発変換するWebアプリ。

🔗 https://garmin-ai-export.vercel.app

ガーミンの腕時計を何年も使ってると、走ったログ・寝た記録・心拍・ストレス・Body Batteryまで膨大なデータが溜まる。Garmin Connect公式のエクスポート機能で全部ダウンロードできるんだけど、出てくるのは謎の入れ子構造のJSON / バイナリFITファイルの塊で、そのままChatGPTに「分析して」と渡しても歯が立たない。

このアプリは：

1. Garmin Connectからエクスポートした生ZIPをアップロードすると
2. **ブラウザ内で**全部展開・解析して
3. `activities.csv` / `sleep.csv` / `daily_health.csv` / `laps.csv` の4ファイル + AIプロンプトテンプレートをまとめたZIPを返す

ChatGPTやGeminiやClaudeのCode Interpreterに食わせるとそのまま「ペースの推移を分析して」「睡眠と次の日のパフォーマンス相関見て」といった分析が動く。

そしてこれを **Claude Code（Opus 4.7）と一緒に半日くらいで作ってデプロイ・決済まで載せた** という話。

## 技術スタック

- **Next.js (App Router)** + Tailwind CSS
- **JSZip** — ZIP展開・再圧縮
- **@streamparser/json** — 巨大JSONのストリーミングパース
- **fit-file-parser** — Garminバイナリの`.fit`ファイル読み取り
- **PapaParse** — CSV生成
- **Web Worker** — メインスレッドを止めない
- **Square Payment Links** — 決済（Apple Pay / Google Pay対応）
- **Vercel** — ホスティング

ポイントは **「ブラウザ完結」**。Garminのエクスポートは数百MB〜GBになることがあって、これをサーバーに送るのは現実的じゃない（Vercelのリクエストボディ上限4.5MBで一発アウト）。それに健康データをサーバーに置きたくないというプライバシー観点もある。

なのでアップロードもパースも変換もダウンロードも、全部ユーザーのブラウザの中で完結させた。サーバーは決済リンクの発行APIだけ。

## Claude Codeとの開発フロー

今回は完全に **チケット駆動** で進めた。

```
要望を伝える → Claudeが GitHub Issue を作成（triageラベル付き）
→ ぼくが「OK」と承認 → ready ラベル
→ ぼくが「#XX 実装して」と指示
→ Claudeが実装してDraft PRを上げる
→ ぼくが「レビューお願い」とリクエスト
→ Claudeが辛口レビューしてGitHubコメント投稿
→ codex（自分自身）が修正
→ 再レビュー → LGTMでマージ
→ Vercel自動デプロイ
```

つまり Claude Code は **PdM・実装者・レビュアーの3役** をこなしている。ぼくは「やりたい」「OK」「マージして」を言うだけ。

「実装」と「レビュー」を別人格としてやらせるのが結構効いていて、実装側でハードコードしたconstantsをレビュー側がしっかり指摘して直させる、みたいな自浄作用が出た。

## ハマりどころ

何回か地雷を踏んだので残しておく。

### 1. 135MB JSONをメインスレッドで処理して画面が固まる

Garminの`*_summarizedActivities.json`は、ヘビーユーザーだと **100MB超** になる。素朴に `JSON.parse(text)` するとメインスレッドが20秒くらい固まり、ブラウザが「このページを終了しますか？」を出す。

対策は2段構え：

**(a) Web Workerに逃がす**

```ts
// src/workers/garmin-converter.worker.ts
self.onmessage = async (event) => {
  const result = await convertGarminExportCoreBuffer(event.data.file, postProgress);
  self.postMessage({ type: "done", result }, { transfer: [result.buffer] });
};
```

`transfer` リストに `ArrayBuffer` を入れると **ゼロコピー** でメインスレッドに渡せる。100MB超のbufferをコピーしないので一瞬で戻る。

**(b) JSON自体をストリーミングパースする**

`@streamparser/json` を使って、巨大配列を **要素単位で逐次取り出す**：

```ts
import { JSONParser } from "@streamparser/json";

const parser = new JSONParser({
  paths: ["$.*.summarizedActivitiesExport.*"],
});

parser.onValue = ({ value }) => {
  rows.push(extractActivityRow(value));
};
```

`paths` の `$.*` プレフィクスが地味な落とし穴で、Garminの実際のJSON構造は `[{"summarizedActivitiesExport": [...]}]` という **配列にラップされた形** なので、ルートが配列であることを示す `$.*` を入れないとマッチしない。最初これを入れ忘れて空CSVが出力されて頭を抱えた。

### 2. JSZipのプライベートAPIを掴んで動かなくなる

最初Claudeが書いたコードでは `zip.file().internalStream("uint8array")` を使っていた。動くんだけどこれはJSZipの非公開API。型定義もない。レビューで「これはマズい」と指摘して書き直した：

```ts
const buffer = await entry.async("uint8array");
const chunkSize = 1 << 20; // 1MB
for (let offset = 0; offset < buffer.length; offset += chunkSize) {
  // 1MBずつ処理してイベントループに譲る
  await yieldToEventLoop();
  processChunk(buffer.subarray(offset, offset + chunkSize));
}
```

`yieldToEventLoop()` は `new Promise(resolve => setTimeout(resolve, 0))`。これを挟まないと巨大ファイルで再びUIが止まる。

### 3. iOS Safari の Blob + IndexedDB の罠

決済ゲートを実装するとき、ユーザーがSquareの決済画面に飛んでから戻ってくる間、変換済みZIPを **どこかに保存しておく必要がある** 。普通は IndexedDB に Blob を入れれば済むんだけど、

> 一番のリスクは iOS Safari で支払い後に Blob が復元できないケース

これが本当に起きる。iOS Safariには **IndexedDBに保存した Blob を取り出すと壊れている** という長年の不具合があり、特に大きい Blob で顕著に出る。¥1,200払ったのにダウンロードできない、は最悪のUXなので、

```ts
// Blob → ArrayBuffer に変換してから保存
const buffer = await result.blob.arrayBuffer();
await store.put({ buffer, files, filename, ... });

// 取り出すときは ArrayBuffer から Blob を再構築
const blob = new Blob([record.buffer], { type: "application/zip" });
```

ArrayBuffer なら iOS Safari でも問題なく往復できる。これはレビュー段階で気付けて助かった。

### 4. Workerが終わらない

ユーザーが変換中に「Reset」を押した場合、走ってるWorkerをちゃんと止めないとメモリリークするしCPUも食い続ける。

```ts
let activeWorker: Worker | null = null;

export function abortConversion() {
  activeWorker?.terminate();
  activeWorker = null;
}
```

シンプルだけど忘れがち。「リセットボタン押しても裏で計算続けてるよね？」とレビューで突っ込まれて気付いた。

## 決済を載せる

ダウンロード時に **¥1,200の課金** を載せた。

### なぜSquare？

候補は Stripe / PayPal / Square / Paddle あたり。今回 Square を選んだ理由：

- **日本円のオンライン決済が普通にできる**（Stripeはできるけど審査が重め）
- **Apple Pay / Google Pay がデフォルトで載る**（追加実装ゼロ）
- **Payment Links API でホスト型チェックアウトが秒で作れる** — 自分のサイトにカード入力フォームを実装する必要がない（PCI DSS対応も不要）
- **手数料が3.6%**（オンラインの場合、JCBは3.95%）

実装は驚くほど簡単で、サーバー側はAPI経由で決済リンクを作るだけ：

```ts
// src/app/api/square/payment-link/route.ts
const squareResponse = await fetch(
  `${getSquareApiBaseUrl()}/v2/online-checkout/payment-links`,
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.SQUARE_ACCESS_TOKEN}`,
      "Content-Type": "application/json",
      "Square-Version": "2026-01-22",
    },
    body: JSON.stringify({
      idempotency_key: crypto.randomUUID(),
      quick_pay: {
        name: "Garmin AI Export",
        price_money: { amount: 1200, currency: "JPY" },
        location_id: process.env.SQUARE_LOCATION_ID,
      },
      checkout_options: {
        allow_tipping: false,
        redirect_url: "https://garmin-ai-export.vercel.app?paid=true",
      },
    }),
  },
);
```

返ってくる `payment_link.url` にユーザーをリダイレクトすれば、SquareのホストされたチェックアウトでカードでもApple Payでも決済できて、終わると `?paid=true` 付きで戻ってくる。

### ソフトゲート設計

サーバーサイドで「この人本当に払ったか」を検証していない。`?paid=true` が付いてれば信用してダウンロードを開放する。

これは意図的で：

- DBもユーザーアカウントもないので検証する手段がない（webhookで購入記録すれば可能だけど、ユーザー識別の仕組みが要る）
- 1人で個人用に使う前提なら、URL叩いて回避する人がいてもまあ良い
- 「払ってくれる気持ちのある人だけ払ってくれればいい」という優しい設計

### 通貨の話

Square は **アカウント所在国の通貨でしか売上を受け取れない** 。日本でアカウント開設したのでJPYのみ。海外ユーザーが Apple Pay で払うときは Apple 側が自動で為替変換してくれて、merchant 側には常に JPY で着金する。なので海外向けに英語UIにしつつJPY課金、で問題ない。

## デプロイとコスト

### Vercel Hobby → Pro にすぐ移行

Vercel Hobby プランは **「personal, non-commercial use」** 限定。決済を取って収益化するサイトを Hobby に置き続けると ToS 違反になり、最悪垢BAN。

→ 即 **Pro ($20/月)** に移行した。

### 転送量はほぼ食わない

「これ、ガーミンのZIPが数百MBになるけど転送量大丈夫なの？」と最初心配した。が、よく考えると：

- ガーミンZIPのアップロード → ブラウザ内で完結（Vercel経由しない）
- 変換済みZIPのダウンロード → ブラウザからローカル保存（Vercel経由しない）
- 決済画面 → Squareのドメイン（Vercel経由しない）

Vercel を通るのは **初回のページロード（〜2MB）** と Square API 呼び出し（〜1KB）だけ。Pro の 1TB 帯域なら 50万ユーザー分は余裕。クライアント完結アーキテクチャの恩恵がここで効く。

### 決済手数料

¥1,200 → 着金 ¥1,153〜1,157（カード種別による）。月3件売れれば Pro の固定費は回収できる。

## GA4 と ファビコン

最後に Google Analytics と独自ファビコンも追加。

GA4 は `next/script` で `afterInteractive` 戦略：

```tsx
{SHOULD_LOAD_GOOGLE_ANALYTICS ? (
  <>
    <Script
      src={`https://www.googletagmanager.com/gtag/js?id=${GA_MEASUREMENT_ID}`}
      strategy="afterInteractive"
    />
    <Script id="google-analytics" strategy="afterInteractive">
      {`window.dataLayer = window.dataLayer || [];
        function gtag(){dataLayer.push(arguments);}
        gtag('js', new Date());
        gtag('config', '${GA_MEASUREMENT_ID}');`}
    </Script>
  </>
) : null}
```

ファビコンは Next.js App Router の **規約ベース** で、`src/app/icon.tsx` と `src/app/apple-icon.tsx` を置けば自動的に `/icon` `/apple-icon` で配信される。`ImageResponse` から動的にPNGを生成するので、外部画像ファイル不要：

```tsx
// src/app/app-icon-image.tsx
export function createAppIconResponse(size: { width: number; height: number }) {
  return new ImageResponse(
    (
      <div style={{ background: "#104c3f", borderRadius: "20%", /* ... */ }}>
        <svg viewBox="0 0 24 24" fill="none" stroke="white">
          {/* FileArchive アイコンのSVGパス */}
        </svg>
      </div>
    ),
    size,
  );
}
```

## Claude Codeで作ってみての所感

**良かった点**

- 「ガーミンZIPのこの構造を解析するCSV変換アプリを」と要望を投げると、調査→Issue起票→実装→レビュー→修正→マージまで全部回せる
- レビューを別ターンで頼むと、本気で辛口になる。「L514 の `disabled={!result}` は dead code」みたいな細かい指摘までしっかり拾う
- ハマったときも「iOS Safari のIndexedDBでBlobが壊れる既知問題があるので ArrayBuffer に変換して保存する」みたいな解決策を即出せる

**気をつけた点**

- 「いいかも」「やりたい」と言うだけで実装を始めさせない。明示的に「#XX 実装して」と指示するルール（CLAUDE.mdに書いた）
- レビューは別ターンで頼むことで、実装者の自己肯定バイアスを切る
- デプロイは「マージして」ではなく「main に出して」「本番に出して」と明示する

「軽いノリで作りたい」をそのまま進められるのは強烈に楽。一方で **何も考えずにOKを出すと進みすぎる** ので、レビューと承認の儀式を挟む価値はある。

## まとめ

- 半日でちゃんと **ペイメント・分析・ファビコンまで載った状態のWebアプリ** が作れた
- ブラウザ完結アーキテクチャでサーバーコストもプライバシーも両立
- Claude Codeと **チケット駆動 + 実装/レビュー分離** で回すと、品質を犠牲にせず速度を出せる

Garminユーザーで自分の活動データをChatGPTやClaudeで分析してみたい人がいたら、ぜひ使ってみてください。

🔗 **https://garmin-ai-export.vercel.app**

ソースは GitHub に公開してます: https://github.com/bokuno-studio/garmin-ai-export
