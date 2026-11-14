---
title: "現場写真4枚から「A4印刷報告書＋音声付き9:16縦動画」を約5秒・ローカル完結で同時生成する設計（Sharp＋ffmpeg＋別モデル監査）"
emoji: "🏗️"
type: "tech"
topics: ["nodejs", "ffmpeg", "sharp", "個人開発", "saas"]
published: true
---

外壁塗装・外構エクステリア・自動車板金などの現場では、施工が終わるたびに次の2つの手作業が発生します。

1. 施主様・元請けへ提出する **A4施工完了報告書**（ExcelやWordに写真を4枚貼って整形・PDF化）
2. 集客用の **Instagramリール / YouTubeショート用 Before/After 縦動画**（スマホアプリで写真を並べてテロップ・BGM・音声を入れて編集）

この2つを毎回手作業でやると1現場あたり 30〜60 分かかります。そこで、「施工前・下地処理・本施工・完了」の**現場写真4枚を入れるだけで、A4印刷用完了報告書・比較画像PNG・1080×1920 音声付き9:16縦ショート動画・SNS投稿文を1回で生成するツール（SiteProof Studio）**を作りました。

外部AIやクラウド動画APIへ現場写真（お客様宅の外観やナンバープレート等）を一切送らず、Node.js 環境内の `Sharp` + `ffmpeg` だけで約5秒（1080×1920 / 18秒動画）で完結するアーキテクチャと、別モデル監査で見つかった実戦的な罠をまとめます。

公開デモ・出力見本ギャラリー（Cloudflare Workers 静的プレビュー）:
- [全体ポートフォリオ（7つの業務用ガードレールツール）](https://zero-one-lab.ai-hack-lab.workers.dev/)
- [SiteProof Studio 画面プレビュー](https://zero-one-lab.ai-hack-lab.workers.dev/ventures/v006_siteproof_studio/dist/public_landing_preview.html)
- [外壁塗装 出力見本HTML（音声付き 1080×1920 再生可）](https://zero-one-lab.ai-hack-lab.workers.dev/samples/sample_v006_siteproof.html)

---

## 1. アーキテクチャ全体：写真4枚から4成果物を同時に出す

入力は「工事種別・工事名・施工期間・メモ」＋「写真4枚（Base64 PNG/JPEG/WebP、各10MB上限）」。これを1リクエストで次の順路で同時生成します。

```text
写真4枚 (PNG / JPEG / WebP)
 ├─▶ Sharp 画質検品（フォーマット強制検査・120px下限・平均輝度・端数ぼけ検査）
 ├─▶ Sharp 1200×640 比較シートPNG合成
 ├─▶ A4縦1ページ最適化 HTML 生成（@page { size: A4 portrait; }・break-inside: avoid）
 └─▶ ffmpeg 1080×1920 (30fps / 18.00s) 縦動画エンコード
       ├─ 0.0–2.5s : BEFORE / AFTER 左右分割スプリット（フック）
       ├─ 2.5–13.7s: 工程01〜04 Ken Burns (zoompan) ＋ 透過窓SVGオーバーレイ
       ├─ 13.7–18.0s: xfade=transition=wiperight 拭き切り比較＋CTA
       └─ AAC 音声 : 日本語ナレーション＋抑えめBGM(16%)＋切替効果音SE
```

---

## 2. 動画の品質を決める3つの実装ポイント

### ① 透過窓つき UI オーバーレイを Sharp（SVG→PNG）で描く

`ffmpeg` の `drawtext` フィルタだけで日本語フォント・角丸ステップバー・BEFORE/AFTER バッジ・SNS セーフゾーンを厳密に組むと、OSごとのフォントパス差異やエスケープ地獄にハマります。

そこで **中央の写真表示領域（`1032×1062px`）だけ透明にくり抜いた 1080×1920 の PNG オーバーレイ** を `Sharp` + インライン SVG で毎パート描画し、`ffmpeg` 側ではガウスぼかし背景＋`zoompan` で動かした写真の上に `overlay=0:0` で1発合成する方式にしました。

- 上部 `0〜108px` と下部・右側アイコン帯（`x > 910, y > 1610`）を避けて下部テロップ（`x=28, width=880` で右端 `908px <= 910px` を厳守）を配置。
- 日本語テロップが `.slice(0, 28)` で単語の途中（「高圧洗浄・シ」など）で切れないよう、区切り文字（`／`・`・`・`、`）を優先して丸める関数を通しています。

### ② 1枚目の「左右スプリット」と終盤の「wiperight」

縦ショート動画は最初の 1.5 秒で「何が変わるか」が見えないと離脱されます。  
冒頭 `2.5秒` で **施工前と施工後を中央線で左右に並べるスプリット画面** を出し、最後に `ffmpeg` の `xfade=transition=wiperight` で左から右へ拭き切るように変化を回収させています。

### ③ `anullsrc` ベースで音声長ズレ・無音欠落を防ぐ

ナレーション（`ja-JP-Chirp3-HD` 等で事前生成した業種別音声）、BGM、効果音（SE）を `amix` で混ぜる際、**無音の `anullsrc`（動画尺と同じ 18.00 秒）を `amix` の第1入力にして `duration=first`** にするのが重要です。これがないと、「BGMなし＋短いナレーションのみ」のパターンを選んだ瞬間に `-shortest` で動画映像まで途中で切れてしまいます。

---

## 3. プライバシー保護（写真・工事名を SQLite に一切残さない）

施工写真には施主様の自宅外観や表札、車両のナンバープレートが写り込みます。そこで本体台帳 `siteproof_studio.sqlite3` には **日次利用カウントのハッシュ化バケットだけを記録し、工事名・写真・生成HTML・動画ファイルは DB へ一切保存しない** 構成にしました。

生成物の一時配信（`/api/video/:jobToken` での `Range 206` ストリーミング対応）には 24 文字（96bit）のランダムトークンを発行し、`x-client-key` 所有者トークンの検証と IP バケット併用のレート制限でトークンローテーションによる上限回避を塞いでいます。

---

## 4. 別系統モデルの Clean-Diff 監査で刺さった実装ミス 2 選

動作確認テストを通したあとに、差分だけを別系統のモデルへ厳格にレビューさせて見つかった見落としです。

1. **`path.join(dir, preset.missingField || '')` がディレクトリ自身に化ける罠**  
   `preset.narrationFile` が未設定の新業種を足したとき、`path.join(AUDIO_DIR, '')` が `AUDIO_DIR` そのものを返し、`fs.existsSync(AUDIO_DIR)` が `true` になって `ffmpeg -i <ディレクトリ>` で失敗する穴が見つかりました。`fs.statSync(p).isFile()` を強制して潰しました。
2. **レスポンス JSON への `Buffer` 混入**  
   内部処理用の `normalizedPng`（`Buffer`）を含んだ `audits` 配列を API の `c.json()` へそのまま返すと、`{ type: 'Buffer', data: [...] }` に展開されて応答 JSON が数十 MB に膨れ上がります。返却前にメタ情報（サイズ・輝度・警告文のみ）へ明示的に射影して ~1KB に抑えました。

---

## おわりに

小規模事業者の現場では「ツールが外部クラウドへ写真を送るのか」「スマホでどう見えるか」「印刷してそのまま渡せるか」が導入可否を左右します。

今回公開した静的ショーケースでは、`SiteProof Studio` のほか請求書桁ズレ検知・二重送信防止ガードなど計7ツールと業種別出力見本をブラウザから確認できます。

- [業務自動チェックツール群 公開ポートフォリオ](https://zero-one-lab.ai-hack-lab.workers.dev/)
- [SiteProof Studio（現場写真4枚 → 報告書＆縦動画）](https://zero-one-lab.ai-hack-lab.workers.dev/ventures/v006_siteproof_studio/dist/public_landing_preview.html)
