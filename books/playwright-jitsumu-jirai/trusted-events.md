---
title: "第1章 trusted event — JS の click は無視される"
free: true
---

## 症状：menu が開かない・「実行した」のに状態が変わらない

その日、私は記事プラットフォームの自分の記事 30 数本を一括で下書きに戻そうとしていました。各記事の「…」メニューを開いて「下書きに戻す」を押す。人間がやれば 1 件 3 秒の作業です。

最初の実装はこうでした。

```js
// ❌ 動かない実装
const opened = await page.evaluate(() => {
  const menuBtn = document.querySelector('button[aria-label="その他"]');
  menuBtn.click();           // ← JS からの click
  return true;
});
```

実行するとログ上は全件成功。しかしサイト側では**メニューすら開いていません**でした。さらに悪いことに、続けて書いた「下書きに戻す」の click も `true` を返し続け、スクリプトは 20 件「✅ 完了」と報告しながら、実際の処理件数は **0 件**でした。

## 原因：isTrusted

ブラウザのイベントには `isTrusted` というプロパティがあります。

- 人間の操作（とブラウザ自動化ツールの CDP 経由操作）→ `isTrusted: true`
- `element.click()` など JS から発火したイベント → `isTrusted: false`

モダンなフロントエンド（特に SPA）は、メニュー開閉や状態変更のような「ユーザーの意思」を要する操作で、untrusted なイベントを無視する実装になっていることがあります。フレームワーク側の挙動だったり、明示的な防御だったりしますが、結果は同じです。**`page.evaluate()` 内の `.click()` は、押したことになりません。**

## 解決：Playwright の locator click は trusted

Playwright の `locator.click()` は CDP（Chrome DevTools Protocol）経由で OS レベルに近い入力イベントを発火させるため、`isTrusted: true` になります。

```js
// ✅ 動く実装
await page.locator('button[aria-label="その他"]').first().click();
await page.waitForTimeout(900);
await page.locator('.menu-body').locator('text=下書きに戻す').first().click();
```

これだけで、さっきまで 0 件だった処理が通るようになります。

## では page.evaluate の click は全部ダメなのか

いいえ。**サーバーに直接届く操作**（`<a href>` の遷移、form の submit と同等の処理が onclick ハンドラで素直に書かれている場合など）は untrusted でも通ることがあります。実際、別のサービスの「dropdown 内の削除リンク」は CSS で隠れていて locator click が `not visible` で詰んだため、JS click で発火させました（これは通りました）。

使い分けの目安：

| 操作 | 推奨 |
|---|---|
| menu 開閉・toggle・選択状態の変更 | **locator click（trusted）一択** |
| CSS 非表示要素の発火（dropdown item 等） | locator click が `not visible` で詰む → JS click を試す |
| 入力 | `fill()` / `keyboard.insertText()`（後述の注意あり） |

## 落とし穴：成功したように見える理由

この地雷が厄介な本当の理由は、**JS click が例外を投げない**ことです。`element.click()` は要素さえあれば成功し、`true` を返します。スクリプトの「成功」とサイトの「変化」が完全に切断される。

だから実サイト自動化では、「click したか」ではなく「**click の結果、世界がどう変わったか**」を毎回確認する必要があります。それが次章の fail-closed 設計です。

## この章のまとめ

- SPA の menu / 状態変更は `page.evaluate` 内の `.click()` では発火しない（isTrusted: false が無視される）
- Playwright の `locator.click()` は trusted — 状態変更系はこれ一択
- JS click は例外を投げずに「成功」するので、失敗が静かに積み上がる
- 対策の本丸は「操作後の世界の変化を検証する」こと（→ 第 2 章）
