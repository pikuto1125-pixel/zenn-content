---
title: "ffmpeg EBU R128 実測と日本語「日付・助数詞」誤読ガードで教材・研修音声の検収やり直しを止める"
emoji: "🎙️"
type: "tech"
topics: ["ffmpeg", "nodejs", "audio", "ebur128", "japanese"]
published: true
---

eラーニング教材・企業コンプライアンス研修・社内マニュアル音声を 50〜500 トラックのロットで制作・納入するとき、後から大量再収録になりやすい失敗パターンは決まっています。

1. **章ごとの音量バラつき（EBU R128 の Integrated Loudness が -16 LUFS と -28 LUFS で混在）**
2. **語尾の無音マージン不足（ブツ切れ）や True Peak 超過（`-1.0 dBTP` 超の音割れ）**
3. **日本語テキスト固有の読み揺れ（「8日」ようか／はちにち、「1日」ついたち／いちにち、「10本」じっぽん／じゅっぽん、英字略語 KPI / SLA）**

Node.js 標準モジュール（`node:child_process`・`node:sqlite`・`node:crypto`）＋ `ffmpeg`（`ebur128`・`silencedetect`・`volumedetect`）で、納入前の検疫ゲート **EduAudio Guard** と公開デモを作成した実装の要点をまとめます。

## 公開実機デモ（ブラウザでそのまま確認可）

- **3場面切り替えデモ（辞書指定済み正常ロット／表記ゆれ正規化／低音圧・語尾切れ・日付誤読の停止）**: https://zero-one-lab.ai-hack-lab.workers.dev/ventures/v008_eduaudio_guard/showcase_preview.html
- **導入出力見本レポート**: https://zero-one-lab.ai-hack-lab.workers.dev/samples/sample_v008_eduaudio.html
- **全9ツール統合ポータル**: https://zero-one-lab.ai-hack-lab.workers.dev/

## 1. `ffmpeg` は引数配列のみで呼び、実測出力を読む

音声の良し悪しを波形サムネイルや推測で判定せず、`spawnSync('ffmpeg', [...], { encoding: 'utf8' })` でシェルを介さず呼び、`ebur128`（Integrated Loudness / LRA / True Peak）と `silencedetect`（末尾無音ミリ秒）の実ログをパースして判定します。

- ロット内の最小/最大 LUFS 差が **2.5 LU を超えたら `LOT_LOUDNESS_DRIFT` で停止**
- 末尾無音が **80ms 未満なら語尾切れリスク（`TAIL_CLIPPING_RISK`）で停止**
- True Peak が **-1.0 dBTP 超なら音割れ（`TRUE_PEAK_CLIPPING`）で停止**

また `ffmpeg` が利用不能・異常終了したときは、推測で合格（`PASS`）に倒さず `CANNOT_JUDGE`（判定不能）として安全停止し、バッチ内の無用な連続リトライも初回失敗で打ち切ります。

## 2. 日本語の日付・助数詞・略語は「読みピン」がない行を先に止める

収録や音声合成の前に、台本バッチ（CSV / TSV / JSON / プレーン行）を通して次の曖昧語を機械抽出します。

- 日付（`1日`, `4日`, `8日`, `10日`, `20日` 等）
- 助数詞・単位（`[0-9]+(本|匹|階|個|回|件|%)` 等）
- 英字略語（`KPI`, `B2B`, `SLA`, `PDF` 等）

台本内のインライン記法（例: `[8日=ようか]`）またはカスタム読み辞書（CSV / TSV / JSON）に明示登録されていない場合は収録前に停止して辞書補完を促すことで、完成後の再収録コストを大幅に下げられます。

## 3. 無認証公開 API 向けの DoS ガード

外部からのバッチ検査 API を公開する場合、次の防衛線も最初から入れています。

- CSV/TSV の展開前に **列数上限（32列）・行数上限（200行）** を検証して同期 CPU 枯渇を防止
- 各行 ID の最大長（40文字）制限とパストラバーサル系キー検知の 1行1件抑制による応答膨張防止
- 公開向けエンドポイントの台帳永続化（`commit`）は既定 `false` に固定し、匿名データ汚染を防止

ソース構造および 9 プロダクト全体の公開プレビューは [GitHub 公開リポジトリ](https://github.com/pikuto1125-pixel/zero-one-lab) にも掲載しています。
