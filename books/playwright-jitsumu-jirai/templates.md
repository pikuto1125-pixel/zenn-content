---
title: "第5-6章 不可逆操作の安全設計 + 再利用テンプレート集"
---

# 第5章 不可逆操作の安全設計

削除・送信・課金のように取り返しのつかない操作を、人間が見ていない時間に自動化するための作法です。

## 原則1：サーキットブレーカー（2 回失敗で止まる）

同じアプローチで 2 回失敗したら、3 回目を試さずに人間にエスカレーションします。失敗の連打はクレジットと時間を溶かすだけで、たいてい根本原因が別にあります。

```js
let consecutiveFails = 0;
async function tryWithBreaker(fn) {
  try {
    const r = await fn();
    consecutiveFails = 0;
    return r;
  } catch (e) {
    consecutiveFails++;
    if (consecutiveFails >= 2) {
      await snapshot();
      throw new Error('2 連続失敗 → サーキットブレーカー作動。手動確認へ');
    }
  }
}
```

私自身、ある一括処理で 5 回別アプローチを試して全敗したことがあります。本来は 2 回でブレーカーを引いて設計を疑うべきでした。「もう一回パラメータ変えれば」は罠です。

## 原則2：対象の前 assert / 守るものの後 assert

```js
// 削除前：これは本当に消していい対象か
if (!isTargetSafe(item)) throw new Error('対象が想定外 — 停止');

await deleteItem(item);

// 削除後：守るべきものは無事か（巻き込み事故の検出）
const survived = await countProtected();
if (survived < EXPECTED_KEEP) throw new Error('本命が消えた疑い — 即停止');
```

不可逆操作では「やったこと」より「やってはいけなかったことをやっていないか」を確認するほうが大事です。

## 原則3：可逆を優先する

「非公開にする（下書きに戻す）」で済むなら削除しない。後から気が変わっても戻せます。削除はそれが不可能と分かってから、明示的な承認を取って行います。私の実例でも、まず「下書きに戻す」を試み、それが UI 上できないと分かってから、人間の確認を得て削除に切り替えました。

## 原則4：操作記録を残す

何を・いつ・なぜ実行したかを構造化して残します。後から「あの削除は妥当だったか」を検証できるようにするためです。承認者・対象件数・前後の状態を 1 レコードにします。

---

# 第6章 再利用テンプレート集

そのままコピーして使える雛形です。

## テンプレ1：安全な context（4 防御先回り）

```js
const { chromium } = require('playwright');

async function safeContext(browser, authPath) {
  return browser.newContext({
    storageState: authPath,
    viewport: { width: 1400, height: 1800 },
    userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 '
             + '(KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36',
  });
}
```

## テンプレ2：login 自動検知 capture（Enter 不要）

```js
// CEO/ユーザーが 1 回 login するだけ。ログイン後の要素を検知して自動保存。
const browser = await chromium.launch({ headless: false });
const ctx = await browser.newContext();
const page = await ctx.newPage();
await page.goto(LOGIN_URL);
const deadline = Date.now() + 10 * 60 * 1000;
let ok = false;
while (Date.now() < deadline) {
  await page.waitForTimeout(3000);
  try {
    ok = await page.evaluate(() => !!document.querySelector(LOGGED_IN_SELECTOR));
  } catch (e) {}
  if (ok) break;
}
if (ok) await ctx.storageState({ path: OUT });
```

## テンプレ3：公開コンテンツの第三者 verify

```js
async function verifyPublic(browser, url, { title, keyword, mustBeFree }) {
  const ctx = await browser.newContext();   // ログインなし
  const page = await ctx.newPage();
  await page.goto(url, { waitUntil: 'domcontentloaded' });
  await page.waitForTimeout(4000);
  const txt = await page.evaluate(() => document.body.innerText);
  const checks = {
    title: txt.includes(title),
    keyword: txt.includes(keyword),
    free: !mustBeFree || !/購入手続きへ|この続きをみるには/.test(txt),
  };
  await ctx.close();
  return checks;  // 呼び出し側で全 true を assert
}
```

## テンプレ4：状態カウントによるやったフリ検出

```js
async function processAllWithGuard(page, { countFn, doOne, expectDecrease = true }) {
  const before = await countFn(page);
  let done = 0;
  for (let i = 0; i < 50; i++) {
    const acted = await doOne(page);
    if (!acted) break;
    done++;
    if (done <= 2 || done % 5 === 0) {
      const now = await countFn(page);
      if (expectDecrease && done >= 2 && now > before - done + 3) {
        await page.screenshot({ path: `_stuck_${Date.now()}.png`, fullPage: true });
        throw new Error('やったフリ検出 — 停止');
      }
      if (now === 0) break;
    }
  }
  return done;
}
```

## テンプレ5：日本語入力（IME 回避・ファイル経由）

```js
// type() は IME で化ける → insertText
await editor.click();
await page.keyboard.insertText(fs.readFileSync('body.txt', 'utf8').trim());

// ⚠️ node -e で日本語インライン JSON を渡さない。必ず .js ファイルにして実行する
```

---

## おわりに

この本の内容は、特別な技術ではありません。全部「人間が普通にやっていること」を、機械にも分かるように明示的に書いただけです。

- 人間はメニューを「本当にクリック」する → trusted event
- 人間は「あれ、消えてないな」と目で確認する → 状態 verify
- 人間は不安なら手を止める → fail-closed

自動化が難しいのは技術が高度だからではなく、人間が無意識にやっている確認を、コードに書き起こす必要があるからです。その確認を省いた自動化は速いですが、見ていない時間に静かに壊れます。

「動いた」で終わらせず「何が起きたか」まで確認する。それだけで、あなたの自動化は本番で信頼できるものになります。

長い間お読みいただきありがとうございました。質問や「うちのサイトではこうだった」という報告は歓迎です。
