---
title: "GarminデータをGoPro動画に重ねるMacアプリを作ってMac App Storeに提出した話"
emoji: "🎬"
type: "idea"
topics: ["swift", "macos", "個人開発", "avfoundation", "appstore"]
published: false
---

:::message
この記事の情報は 2026年5月時点のものです。
:::

## はじめに

トレイルランニングやスパルタンレースの映像を振り返るとき、「この坂のところ、心拍いくつだったんだろう」と思ったことはないだろうか。

GoPro で撮った動画と Garmin ウォッチのデータは別々に存在している。動画を見ながら Garmin Connect を開いて、タイムスタンプを見比べて……という作業が毎回発生する。

「これ、動画の上にリアルタイムで重ねたら最高じゃないか」

そう思って作り始めたのが **ActivityVideoStudio**。Garmin の `.FIT` ファイルと GoPro の `.MP4` を時刻同期して、心拍数・ペース・GPS マップ・標高プロファイルをオーバーレイ合成して書き出す macOS ネイティブアプリだ。

作り始めてから Mac App Store への提出まで辿り着いたので、その過程を書いておく。

---

## 作ったもの

**ActivityVideoStudio** は、Garmin ウォッチの `.FIT` ファイルと GoPro の `.MP4` 動画を組み合わせて、アクティビティデータ入りの動画を出力する Mac ネイティブアプリ。

- Garmin FIT ファイルと動画を**自動で時刻同期**
- 心拍数（ゾーン別カラー）・ペース・距離・標高・ケイデンス・コア体温をリアルタイム表示
- GPS トラックの**ミニマップ**と標高プロファイルグラフをオーバーレイ
- 複数の GoPro ファイル（チャプター分割された MP4 群）を一括処理して 1 本に結合
- YouTube 投稿用のチャプター概要テキストを自動生成
- 10GB 超の 4K 動画にも対応（ストリーミング処理）

### 技術スタック

| 領域 | 技術 |
|---|---|
| 言語 | Swift |
| UI | SwiftUI |
| 動画処理 | AVFoundation（AVComposition, AVVideoComposition, AVAssetExportSession）|
| オーバーレイ描画 | Core Graphics |
| 地図 | OpenTopoMap タイル |
| ターゲット | macOS 14（Sonoma）以上 |

---

## なぜ作ったか

市場には似たようなツールが存在する。**Telemetry Overlay** がその代表で、GoPro の GPMF データや各種センサーのオーバーレイに対応している。

ただ、自分の用途では 2 つの問題があった。

1. **Garmin の FIT 連携が弱い** — 心拍ゾーン設定や FIT 独自フィールド（CORE 体温など）を活かしきれない
2. **macOS ネイティブじゃない** — M シリーズ Mac での動作がもたつく場面がある

「自分が使いたいものを作ったほうが早い」という結論に至り、Swift で書くことにした。

---

## 設計で最初に決めたこと

### 絶対に動画全体をメモリに載せない

GoPro の 4K 動画は 1 ファイルで数 GB になる。レース 1 本分だと複数ファイルで 20〜30GB になることもある。

最初の設計から `AVAssetReader` / `AVAssetWriter` によるストリーミング処理を前提とした。メモリ上に動画データを展開しない。このルールを最初に決めておいたことで、後の大容量対応がかなり楽になった。

### FIT パーサーは自前で書く

Garmin が提供する Swift 向けの FIT SDK は存在するが、依存関係が増えること・SDK のアップデートへの追従が必要なことを嫌って、自前でバイナリパーサーを書いた。

FIT フォーマットは、ファイルの前半に「どんなフィールドが来るか」を定義する **Definition Message** があって、後半に実データの **Data Message** が並ぶ構造。タイムスタンプ変換にはまる人が多いが、Garmin エポック（1989-12-31 00:00:00 UTC 起点）を Unix 時刻に変換するだけ。

```swift
let garminEpoch: TimeInterval = 631065600 // 1989-12-31 00:00:00 UTC in Unix time
let timestamp = Date(timeIntervalSince1970: garminEpoch + Double(rawTimestamp))
```

緯度経度は semicircles 単位で記録されているので度数に変換する。

```swift
let degrees = Double(semicircles) * (180.0 / pow(2.0, 31.0))
```

---

## 時刻同期のロジック

GoPro の MP4 には `creationDate` がメタデータとして入っている。これと FIT の `record` メッセージのタイムスタンプを突き合わせる。

```
動画の再生位置（秒） + 撮影開始時刻 = 現実時刻
↓
現実時刻 ± 同期オフセット = Garmin データのどの時点か
```

これだけだと「撮影開始と Garmin 記録開始がずれている」場合に対応できない。UI でオフセット（秒）を手動調整できるスライダーを用意した。

実際に使ってみると、GoPro が録画開始から数秒のバッファを持っていることがあり、オフセット調整は必須だった。

---

## オーバーレイ描画の実装

各フレームに Core Graphics でオーバーレイを描く。AVFoundation の `AVVideoComposition` を使って毎フレームのコールバックに割り込む方式。

```swift
let composition = AVVideoComposition(asset: asset) { [weak self] request in
    guard let self else { return }
    let currentTime = request.compositionTime.seconds
    let fitData = self.fitDataPoint(at: currentTime)

    let overlay = self.overlayRenderer.render(
        fitData: fitData,
        size: request.renderSize
    )

    let output = request.sourceImage.composited(over: overlay)
    request.finish(with: output, context: nil)
}
```

ここで一つ重要な注意がある。**macOS 26（Tahoe）では `customVideoCompositorClass` がサイレントにバイパスされる**。カスタムコンポジターを使う実装は将来的に動かなくなる可能性があり、クロージャベースの API を採用した。

### MapKit をやめて地図タイルに切り替えた

最初は MapKit でミニマップを描いていた。が、`WKWebView` の描画サイクルとエクスポートの処理が噛み合わず、正しいフレームのマップが取れないケースが出た。

対策として、GPS トラックを事前に OpenTopoMap のタイル画像にレイアウトしておき、エクスポート時はその静的な画像を使う方式に切り替えた。地形図スタイルでトレイルランニングとの相性もよくなった。

---

## エクスポートで詰まった問題

### 出力ファイルが 0 バイトになる

初期実装で、長尺動画のエクスポートが完了しても出力ファイルが 0 バイトになるバグがあった。

原因は **temp ファイルの置き場所**。デフォルトでは `/tmp`（内蔵 SSD）に書いていたが、外付け SSD 上の動画を内蔵 SSD に書き出してから移動するという流れで、内蔵 SSD の空き容量が不足してサイレントに失敗していた。

修正は「temp ファイルを出力先と同じボリューム上に作る」だけ。

```swift
let tempDir = outputURL.deletingLastPathComponent()
    .appendingPathComponent(".tmp_\(UUID().uuidString)", isDirectory: true)
```

これで外付け SSD ユーザーでも安定して動くようになった。

### トリム後の時刻ズレ

「最初の 10 分をカットしてエクスポート」という操作をすると、オーバーレイのタイムスタンプがズレる問題があった。

動画の再生位置（例: 0 秒）と、実際の撮影時刻（例: 撮影開始から 600 秒後）が一致しなくなるため、FIT データを引くタイミングにオフセットを加算する必要があった。

```swift
let absoluteTime = playbackTime + trimStartOffset
let fitData = timeSync.fitData(at: absoluteTime)
```

単純な計算だが、これに気づくまでに時間がかかった。

---

## Mac App Store 提出

### Sandbox 対応

Mac App Store に出すにはアプリを App Sandbox に対応させる必要がある。このアプリの場合、ユーザーが選んだファイルへのアクセス権があればよく、最小限のエンタイトルメントで済んだ。

```xml
<key>com.apple.security.app-sandbox</key>
<true/>
<key>com.apple.security.files.user-selected.read-write</key>
<true/>
```

### プライバシーマニフェスト

2024 年から必須になった `PrivacyInfo.xcprivacy`。このアプリはユーザーデータを一切収集しないので、ほぼ空のマニフェストで通った。

### 提出作業で知らないと詰まるポイント

App Store Connect での提出作業は、初めてだといくつか引っかかる。

- **スクリーンショットのサイズ** — 2560×1600 か 2880×1800 が必要。他のサイズは弾かれる
- **暗号化コンプライアンスの申告** — 独自暗号化を使っていなければ「上記のどれでもない」を選ぶ
- **プライバシーポリシー URL が必須** — アプリの App Store ページに表示されるわけではないが、審査前に入力が必要。GitHub Pages で 1 ページ作れば十分

---

## GitHub

https://github.com/bokuno-studio/activity-video-studio

---

## 作ってみて

アクティビティ系ツールを自分で使う目的で作り始めたが、コードを書きながら「同じ悩みを持ってる人が絶対いる」という確信が強くなった。Garmin と GoPro を両方使っているアスリートは多い。

技術的に一番楽しかったのは FIT パーサーと時刻同期のロジック。バイナリフォーマットを手で読んで、秒単位で実データと突き合わせる作業は地味だが達成感があった。

一番しんどかったのは AVFoundation の挙動調査。ドキュメントが古いケースや、macOS バージョンによって挙動が違うケースがあり、実機で確認し続けるしかなかった。

審査が通ったら更新する。使った感想を聞かせてもらえると嬉しい。
