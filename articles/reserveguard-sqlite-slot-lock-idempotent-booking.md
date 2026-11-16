---
title: "小規模店舗・サロンの予約二重ブッキングを SQLite 1分単位ロックと SHA-256 冪等キーで止める実装"
emoji: "🛡️"
type: "tech"
topics: ["nodejs", "sqlite", "hono", "typescript", "javascript"]
published: true
---

小規模サロン・治療院・貸会議室向けの予約システムや LINE ミニアプリ連携を作るとき、本番運用で一番怖いのは UI の見た目よりも次の 3 つです。

1. **同時アクセスで同じ枠に 2 件確定してしまう（二重ブッキング）**
2. **スマホ通信不調のボタン連打・Webhook 再送で同じ申込や前日リマインドが 2 通飛ぶ**
3. **「令和8年」「１４：００」「全角スペース」「電話番号ハイフンゆれ」で取り込みが落ちる**

Node.js 標準モジュール（`node:sqlite`・`node:crypto`）＋ Hono で組んだ検証済みモジュール **ReserveGuard（予約ガード）** の実装パターンと、ブラウザ上でそのまま動かせる公開デモを公開したのでポイントを共有します。

## 公開実機デモ（ブラウザでそのまま確認可）

- **3場面切り替えデモ（正常／表記ゆれ正規化／14:00 3人同時衝突）**: https://zero-one-lab.ai-hack-lab.workers.dev/ventures/v007_reserve_guard/showcase_preview.html
- **導入出力見本レポート**: https://zero-one-lab.ai-hack-lab.workers.dev/samples/sample_v007_reserveguard.html
- **7+1種 業務ガードツール総合ポータル**: https://zero-one-lab.ai-hack-lab.workers.dev/

## 1. 「予約テーブル」とは別に「1分単位の排他ロック表」を持つ

時間帯の重なりチェックを `SELECT ... WHERE start < ? AND end > ?` だけで済ませると、非同期リクエストや複数チャネルからの一括取り込み時に競合しやすくなります。

代わりに SQLite 内に「`リソースID × 日付 × 分インデックス`（例: `room_a|2026-11-20|840`）」の主キー表 `slot_locks` を用意し、**トランザクション内で `PRIMARY KEY` 一意制約を張って全分ロック**すると、受付時刻が同じ 14:00 枠に 3 件同時に入っても物理的に先着 1 件しか通らず、後着 2 件は安全に弾けます。

さらに準備インターバル（例: 清掃 15 分）も同じ占有分数へ含めてロックすることで、「施術終了の 5 分後に次のお客さんが入ってしまう」事故も防げます。

## 2. 連打・Webhook 再送を止める SHA-256 冪等キー

フォーム送信や LINE Webhook は同じペイロードが 2 回届く前提で組みます。

- `normalizedPhone` + `bookingDate` + `startMinute` + `resourceId` から SHA-256 の冪等キーを生成して `idempotency_keys` に UNIQUE 保持
- 同一顧客・同一時刻の重複申込はエラー終了ではなく `DUPLICATE_SUBMISSION_IDEMPOTENT` として安全遮断（すでに確定済みの予約を壊さない）
- 前日リマインド送信キューも「対象日 × 予約ID」単位で冪等台帳に記録し、バッチを 2 回走らせても **2 回目の生成通数を必ず 0 通** に抑える

## 3. ハッシュ連鎖（`prev_hash -> event_hash`）による変更証跡

「誰が・いつ確定/キャンセルしたか」を追うため、イベント記録時に直前行の `event_hash` を連結して SHA-256 を計算・保存しておくと、後から SQLite レコードを書き換えられた場合も `verifyAuditChain()` で即検知できます。

## 4. 「AI 補助が通信不能な時は嘘の合格を出さない」

表記ゆれ正規化や予約ルール判定の主系は **外部通信ゼロのローカル決定論実行** に置き、AI 補助を有効にした場合でも通信断・タイムアウト時は `CANNOT_JUDGE` として停止させ、決して推測で確定（`PASS`）に倒しません。

全8プロダクトのフロント・サンプル一式は [GitHub 公開リポジトリ](https://github.com/pikuto1125-pixel/zero-one-lab) でも閲覧できます。
