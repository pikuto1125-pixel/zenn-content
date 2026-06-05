---
title: "第2章 fail-closed 設計 — 「やったフリ」を機械的に検出する"
free: true
---

## 自動化の本当の敵は、エラーではなく「静かな成功」

前章で見たように、実サイト自動化の失敗はエラーを出しません。スクリプトは正常終了し、ログには成功が並ぶ。これが一番怖い。エラーで落ちてくれれば気づけますが、静かに 0 件処理して「完了しました」と報告されると、人間はそれを信じてしまいます。

対策の原則は 1 つです。**操作したかどうかではなく、操作の結果として観測可能な状態がどう変わったかを、毎回検証する。** そして検証が通らなければ、続行せずに止まる。これを fail-closed（安全側に倒れる）と呼びます。

## パターン1：操作のたびに「真の状態」を数える

一括処理では、UI のカウンタや内部 API を使って、操作前後の実数を比較します。

```js
async function publishedCount(page) {
  // サイトの内部 API で「公開中」の実数を取る（DOM の見た目ではなく）
  return await page.evaluate(async () => {
    let total = 0;
    for (let p = 1; p <= 10; p++) {
      const r = await fetch(`/api/contents?page=${p}`, { credentials: 'include' });
      if (!r.ok) break;
      const j = await r.json();
      const items = j.data?.contents ?? [];
      // ⚠️ status で必ず絞る。一覧 API は下書きも混ぜて返すことがある
      total += items.filter(n => n.status === 'published').length;
      if (!items.length || j.data?.isLastPage) break;
    }
    return total;
  });
}
```

DOM の `.item-card` を数えるのではなく、サイト自身が持っている状態（API レスポンスの `status`）を数えるのがポイントです。見た目は嘘をつきますが、状態は嘘をつきません。

## パターン2：減らなければ止まる（やったフリ検出）

```js
const before = await publishedCount(page);
let done = 0;
for (let i = 0; i < 40; i++) {
  await doOneOperation(page);   // 1 件処理
  done++;

  if (done <= 2 || done % 5 === 0) {
    const now = await publishedCount(page);
    // 2 件処理したのに公開数が減っていない → click が効いていない
    if (done >= 2 && now > before - done + 3) {
      await page.screenshot({ path: '_stuck.png', fullPage: true });
      throw new Error('やったフリ検出: 状態が変化していない。停止して目視へ');
    }
    if (now === 0) break;       // 完了
  }
}
```

私はこの仕組みのおかげで、「20 件✅と報告しながら実際は 0 件」のスクリプトを、本番データを一切壊さずに検出できました。安全装置が `_stuck.png` を吐いて止まり、スクリーンショットを見て初めて「メニューが開いていない」と分かったのです。

## パターン3：未ログイン状態で「公開側」を確認する

コンテンツ公開の自動化では、ログイン状態の管理画面で「公開しました」と出ても信用しません。**別の、ログインしていないブラウザコンテキスト**で公開 URL を開き、第三者から見えているかを確認します。

```js
const ctx2 = await browser.newContext();   // ログインなし
const p2 = await ctx2.newPage();
await p2.goto(publicUrl);
const txt = await p2.evaluate(() => document.body.innerText);
const ok = txt.includes(expectedTitle)
        && txt.includes(expectedKeyword)
        && !/購入手続きへ|この続きをみるには/.test(txt);  // 無料で読めるか
if (!ok) throw new Error('公開 verify 不一致');
```

「自分には見えるが、客には見えない」を防ぐ唯一の方法です。

## パターン4：不可逆操作は二段構え

削除のような取り返しのつかない操作は、必ず「対象が想定通りか」を assert してから実行し、実行後に「巻き込み事故がないか」を確認します。

```js
// 削除前：対象が本当に「下書き」か
if (!/下書き/.test(rowText)) throw new Error('対象が下書きでない — 停止');

// 削除実行 …

// 削除後：守るべきもの（公開中の本命）が残っているか
const survived = await publishedCount(page);
if (survived < EXPECTED_KEEP) throw new Error('本命が消えた疑い — 即確認');
```

## なぜここまでやるのか

自動化は「人間が見ていない時間」に動く前提です。見ていない時間に静かに失敗し、しかも「成功」と報告するシステムは、何もしないより危険です。間違ったデータの上に翌日の判断が積み上がるからです。

fail-closed の合言葉は「**疑わしきは止まれ**」。止まれば人間が気づけます。進めば、気づいた時には手遅れです。

## この章のまとめ

- 実サイト自動化の失敗は例外を出さない → 結果の状態を毎回検証する
- 見た目（DOM）ではなく状態（API の status 等）を数える
- 「2 件やって減らなければ止まる」やったフリ検出を入れる
- 公開はログインなしの別コンテキストで第三者視点 verify
- 不可逆操作は前 assert + 後 assert の二段構え

---

ここまでの 2 章が無料パートです。第 3 章からは、この 2 つの原則を実際の 25 件の地雷にどう当てたか、症状・原因・回避コードの形でカタログ化していきます。
