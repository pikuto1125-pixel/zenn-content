---
title: "AIが書いた15本の自動化スクリプト、全部同じ場所でバグっていた — 別のAIに監査させて分かったこと"
emoji: "🔍"
type: "tech"
topics: ["claude", "ai", "openai", "gemini", "個人開発"]
published: true
---

20 歳の大学生です。Claude Code に会社（っぽい何か）を回させていて、メール監視・自動返信ドラフト・カレンダー登録・収益集計など、外部に副作用を持つ自動化スクリプト（sender と呼んでいます）を 15 本ほど運用しています。

先日、ChatGPT Plus に課金したのを機に **Codex CLI（GPT 系）と Gemini CLI に、Claude が書いた 15 本を全部監査させてみた**ところ、想像よりずっと面白い結果が出たので共有します。

# 結果：15 本中 15 本が「同じ 3 箇所」でバグっていた

個別のバグが散らばっていたのではありません。**全部が同じ 3 パターン**でした。

## ① non-atomic な read-modify-write（check-then-write race）

```js
// よくある形（15 本中 9 本がコレ）
const state = JSON.parse(fs.readFileSync(STATE_FILE, 'utf8'));
if (!state[key]) {
  await sendSomething();        // ← 副作用
  state[key] = { sent: true };
  fs.writeFileSync(STATE_FILE, JSON.stringify(state));
}
```

単体では正しく見えます。でも cron の重複起動や watchdog の再起動で **2 プロセスが同時に走ると、両方が「未送信」を読んで両方送る**。実際うちでは過去に、この形が原因で同じ相手に招待メールが 4 重送信される事故を起こしています。

## ② parse 失敗の fail-open

```js
function loadState() {
  try { return JSON.parse(fs.readFileSync(STATE_FILE, 'utf8')); }
  catch { return {}; }  // ← 破損 = 空 = 「全部未処理」扱い
}
```

state ファイルが壊れた時に空オブジェクトを返すと、**「全件処理済み」が「全件未処理」に化けて全部再送**されます。初回起動（ファイル不存在）と破損は区別して、破損は throw で止める（fail-closed）べきでした。

## ③ 副作用失敗の握りつぶし

```js
try { await sendLineMessage(msg); } catch (_) {}
```

通知や送信の失敗を握りつぶすと、**呼び出し元は成功したと思って状態を確定**します。再送もされず、失敗が観測すらされない。

# なぜ「全部同じ」なのか — correlated blind spot

ここが一番の学びでした。15 本は数ヶ月にわたって別々の文脈で書かれたコードです。それが同じ場所でバグる。

つまりこれは個別のミスではなく **モデルの「書き癖」**です。Claude は happy path のロジックは綺麗に書くけど、「並行プロセスが同時に走ったら」「state が破損していたら」「通知が落ちたら」を**確率的に同じ優先度で省略**する。

そして重要なのは、**この 15 本を何ヶ月もレビューしてきたのも Claude 自身**だったこと。書いた本人と同じ盲点を持つレビュアーは、同じ場所を素通りします（self-preference bias / correlated failure）。GPT と Gemini という**別系統のモデル**に見せた瞬間、初回の監査で全部出てきました。

面白かったのは severity 判定まで割れたこと：同じ二重実行バグを **Codex は BLOCKING、Gemini は SHOULD-FIX** と付けました。独立した目は、間違い方まで独立です。

# 直し方：advisory lock + 並行プロセステスト

処方箋自体は古典的です。外部依存ゼロの file lock（`fs.openSync` の `wx` フラグ = atomic create）で mutex を作り、check と write を lock 内に入れる：

```js
// advisory_lock.js（概略）
function acquire(target) {
  const lockPath = target + '.lock';
  try {
    const fd = fs.openSync(lockPath, 'wx');  // 既存なら EEXIST で throw = atomic
    fs.writeSync(fd, String(process.pid));
    fs.closeSync(fd);
    return true;
  } catch (e) {
    if (e.code !== 'EEXIST') throw e;
    return false;  // 他プロセスが保持中
  }
}
```

予約パターン（reserve → 副作用 → confirm / 失敗時 revoke）に組み替えて、**12 プロセス同時に reserve を叩いて勝者がちょうど 1 になるか**を実テストしました。修正前は普通に 2-3 プロセスが「勝って」いました。

```
race test: 12 並列 reserve → WON=1 ✅ / 15 並列 RMW → lost-update 0 ✅
```

# まとめ：AI にコードを書かせるなら

1. **AI が書いたコードのレビューを同じ AI にやらせない**。別系統のモデル（GPT ⇄ Claude ⇄ Gemini）を 1 つ挟むだけで、系統的盲点が初回で出てくる
2. 外部副作用を持つコードは「**並行に 2 つ走ったら**」を必ず聞く。AI は聞かれない限りこれを書かない
3. `catch { return {}; }` と `catch (_) {}` を grep するだけでも、かなりの確率で時限爆弾が見つかります

普段の運用記録は [AI Hack Lab](https://ai-hack-lab.com/blog/) に書いています。Claude Code に会社を回させる実録は[こちら](https://ai-hack-lab.com/blog/2026-06-05-claude-code-kaisha-jitsuroku/)。
