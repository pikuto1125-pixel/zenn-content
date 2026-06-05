---
title: "第3章 地雷カタログ — 症状・原因・回避コード 25 件"
---

実サイト 5 つを相手に 1 日で踏んだ地雷を、症状から逆引きできる形で並べます。「動かない」ときにここを上から眺めれば、だいたい該当するものが見つかるはずです。

## A. 入力系

### 地雷 1：本文を入れたが反映されない（contenteditable）

**症状**：`fill()` したのに本文が空、または一部しか入らない。

**原因**：リッチエディタは `<textarea>` ではなく `contenteditable` の div。`fill()` は value プロパティを持つ要素用なので効かない。

**回避**：
```js
const editor = page.locator('[contenteditable="true"]').last();  // 複数ある時は本文は最後尾が多い
await editor.click();
await page.keyboard.insertText(bodyText);   // 改行も保持される
// 反映 assert（95% 以上入ったか + キーワード存在）
const got = await editor.innerText();
if (got.replace(/\s/g, '').length < body.replace(/\s/g, '').length * 0.95) throw new Error('本文反映不足');
```

### 地雷 2：複数の textarea があり、どれがタイトルか分からない

**症状**：タイトル欄に本文が入る、AI アシスタント欄に誤入力。

**原因**：1 画面に textarea が 3 つ（AI アシスタント用・タイトル用・隠し用）あった。

**回避**：placeholder を dump してから placeholder 一致で取る。
```js
const dump = await page.evaluate(() =>
  Array.from(document.querySelectorAll('textarea')).map(t => t.placeholder));
// → ["AIにお願い…", "記事タイトル", ""]
await page.locator('textarea[placeholder*="タイトル"]').first().fill(title);
```

### 地雷 3：説明文の上限を超えてボタンが disabled

**症状**：「次へ」「公開」ボタンが押せない。エラーは小さな赤字でしか出ない。

**原因**：説明文に文字数上限（例：1000 字）があり、超過すると後続ボタンが disabled になる。だが上限の表示は目立たず、ボタンが押せない真因に見えない。

**回避**：disabled を検知したら、その場で「必須欠け / 文字数オーバー」を赤字テキストから dump して原因を特定する。テキストは上限内（例：980 字）で作る。

## B. 選択・チェックボックス系

### 地雷 4：check() してもカウンタが 0

**症状**：`checkbox.check()` は成功するのに「N 件選択中」が 0 のまま。

**原因**：見た目は checkbox だが実体は custom component。`<input>` への直接操作では内部状態（Vue/React の state）が動かない。

**回避**：人間と同じく**ラベルを click**する。
```js
const box = page.locator('input[type="checkbox"]').nth(i);
const label = box.locator('xpath=ancestor::label[1]');
await label.click({ force: true });   // trusted な label click で state が動く
```

### 地雷 5：再 click で選択が解除される

**症状**：複数選択ループで、選択数が増えたり減ったりする。

**原因**：click はトグル。既に checked の box をもう一度 click すると外れる。

**回避**：`if (await box.isChecked()) continue;` で checked skip。

### 地雷 6：一覧の「見えている 20 件」しか選べない

**症状**：33 件あるのに 20 件しか選択できない。

**原因**：lazy render。一覧は表示分しか DOM に存在しない。スクロールでも増えないページネーション型だった。

**回避**：「20 件選択 → 一括処理 → ページ再読込 → 残りを再処理」の周回方式。1 周で終わらせようとしない。

## C. メニュー・ボタン系

### 地雷 7：menu が JS click で開かない

→ 第 1 章参照（trusted event）。`locator.click()` 一択。

### 地雷 8：「削除」を押したつもりが visually-hidden の見出しを掴む

**症状**：footer の「削除」ボタンを click → タイムアウト。

**原因**：`text=削除` が `<h2 class="visually-hidden">選択中の記事を削除</h2>`（スクリーンリーダー用の不可視見出し）に誤解決していた。不可視なので click できずタイムアウト。

**回避**：要素種別で絞る。
```js
page.locator('[class*="Footer"] button').filter({ hasText: '削除' }).first();
// text= ではなく button を明示
```

### 地雷 9：ボタンが modal-background に覆われて click できない

**症状**：`element is visible` なのに `intercepts pointer events`。

**原因**：直前の操作で出た crop modal / confirm modal の背景レイヤーが、目的のボタンの上に被さっている。

**回避**：先に modal を確定（「この画像を使う」等）または Escape で閉じてから次へ。modal-background が残っていないかを assert してから進む。

## D. 認証・レンダリング系

### 地雷 10：ログインは通るのに編集画面が永遠に loading

**症状**：title「新規作成」のまま、本文エリアが描画されない（contenteditable が 0 個）。

**原因**：エディタの SPA が **HeadlessChrome の UserAgent を検出して描画を拒否**していた。ログイン自体は成立しているので、認証エラーには見えない。

**回避**：context に通常の Chrome UA を渡す。
```js
const ctx = await browser.newContext({
  storageState: AUTH,
  userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 '
           + '(KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36',
});
```
これは「ログインしてるのに動かない」系の最有力容疑。SPA エディタで詰まったらまず UA を疑う。

### 地雷 11：storageState はあるのに API が 401

**原因**：cookie のドメインスコープ。`note.com` と `editor.note.com` のようにサブドメインが分かれていると、保存時のドメインによっては届かない。capture 時に対象ドメインを一度踏んでおく。

### 地雷 12：CtoC マーケットの shop が curl で 403

**原因**：Cloudflare の bot challenge。`curl` や素の fetch は弾かれる。

**回避**：実ブラウザ（Playwright）+ UA 偽装 + challenge 通過待ち（数秒の waitForTimeout）。

## E. フォーム構造・遷移系

### 地雷 13：販売データが空だと後続フィールドが出ない

**症状**：サムネ・タイトル欄が画面に存在しない。

**原因**：商品ファイルをアップロードするまで、後続フィールドが非表示。

**回避**：必ず「ファイル → 後続入力」の順。かつ 1 セッション内で全入力 + 下書き保存（揮発防止）。

### 地雷 14：file input の nth が再レンダリングで変動

**症状**：サムネが商品本体の枠に入る事故。

**回避**：upload 後に**両セクションの filename を verify**。`nth(count-1)`（最後）を使い、入れた後で「本体側のファイル名が消えていないか」を確認。

### 地雷 15：下書き保存しても再オープンで内容が復元されない

**原因**：サービスによっては下書きの永続化が弱い。

**回避**：そのサービスでは「1 セッション内で公開まで完結」を絶対とする。途中保存を当てにしない。

### 地雷 16：カテゴリ選択が「その他 → 第2モーダル」の二段

**回避**：第1モーダルで「その他」→「選択する」→ 第2モーダルで検索 → leaf を選択 → 確定。モーダルが閉じたか（input が消えたか）を assert。

### 地雷 17：投稿後の URL が editor のまま

**症状**：公開 URL が取れない。

**原因**：投稿成功後も URL は `editor.../publish/` のまま、成功サインは「シェアしましょう」モーダルだった。

**回避**：成功サインをモーダル出現で判定し、公開 URL は別途 `domain/user/article-id` で組み立てて未ログイン verify。

## F. 一覧操作の地雷

### 地雷 18：一覧 URL の直叩きが 404

**回避**：URL 直叩きでなく、画面の nav リンクを辿る。あるいは正しい URL を nav の href から取得してから goto。

### 地雷 19〜25（要約）

- **19**：confirm ダイアログのボタン文言がサービスごとに違う（「削除する」「はい」「OK」）→ 複数候補を順に試す
- **20**：menu 要素は描画に時間差 → 開いた直後の取得は失敗。1.2 秒待ち + リトライ
- **21**：trusted な hover が必要な「hover で出るボタン」→ `locator.hover()` を先に
- **22**：日本語入力が `type()` だと IME 絡みで化ける → `keyboard.insertText()` を使う
- **23**：`page.evaluate` 内に複雑な日本語文字列を渡すとシェル経由でエスケープ崩壊 → スクリプトはファイル経由で実行（インライン `node -e` を避ける）
- **24**：スクリーンショット verify を怠ると「なぜ失敗したか」が永遠に分からない → 各停止点で必ず撮る
- **25**：成功時の後片付け（失敗 run が残す空下書き）→ 1 セッション完結 + 残骸の検出ロジック

次章では、この 25 件を 4 つの「防御カテゴリ」に抽象化して、未知のサイトに当たったときの当たり所を体系化します。
