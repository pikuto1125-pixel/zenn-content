---
title: "AI 自動化を 9ヶ月運用して気づいた、Validator 設計の 3 つの落とし穴"
emoji: "🛡"
type: "tech"
topics: ["claude", "validator", "ai", "個人開発"]
published: true
---

20 歳の大学生が、Claude Code を使って AI 自動化システムを 9 ヶ月運用してきた中で **Validator パイプライン**を構築しました。Qiita に[全体像の記事](https://qiita.com/pikuto1125-pixel/items/1655c87495424d4ba447)を書きましたが、こちらでは**実装中に踏んだ 3 つの落とし穴**に絞って書きます。

設計書（10 モジュール完全版）は [Arena Blueprint](https://ai-hack-lab.com/products/arena-blueprint/) として公開しています。Notion の[公開ページ](https://shared-cut-385.notion.site/arena_blueprint_complete_v1-358347a29814808b81c1efe8d81cebe7)で全文無料で読めます。

# 前提：Validator は何層あるか

8-9 層構造で運用しています。

| 層 | 名称 | 主検査 |
|---|---|---|
| 0 | Typography | 誤字脱字・末尾形 |
| 1 | 地雷ワード | NG ワード正規表現 |
| 2 | プレースホルダ | `{{xxx}}` 残留検出 |
| 3 | 重複送信 | append-only ログ照合 |
| 4 | brand_guard | 煽り訴求検出 |
| 5 | sanity | 文字数・無償オファー禁止 |
| 6 | reflection_preamble | 過去失敗教訓注入 |
| 8 | secrets_leak | API key / 個人情報検出 |
| 9 | sender_pattern_audit | 副作用前ガード（静的検査） |

各層は単一責務で fail-closed。1 層でも error なら送信/公開を停止します。

# 落とし穴 1：Validator drift

> Generator チームに Validator を持たせると、デプロイ pressure で Validator がじわじわ緩む。

## 何が起きたか

最初は「`劇的`」「`絶対`」を NG ワードに入れていました。ある日、Generator が「`ensured outcome`」「`大幅な向上`」のような **同義の婉曲表現**を出すようになりました。これは regex を素通りします。

更に問題なのは、自分自身がこの状況を見て「じゃあ regex に追加するか…でも記事として読みにくくなる…」と **「煽り表現の許容ライン」を緩める判断**をしかけたことです。

## 構造解

- **Validator のロジックを「進化禁止領域」に指定**：自分のメモリに「Layer 1-9 のロジックは編集禁止」と明記。pattern 追加は OK だが、緩和は CEO 承認必須に。
- **Generator チームと Validator チームを分離**：単一の人間でも、編集セッションを別にする。Generator 設計中は Validator に触らない。
- **brand_guard の dictionary を separate file**：prompt 埋め込みでなく txt/json で外出し。何を NG にしているか可視化。

```javascript
// 進化禁止領域（メモリで明示）
// Layer 1-9 のロジック / Layer 8 secrets_leak / Layer 9 sender_pattern_audit
// → CEO 承認なき編集禁止（自己評価盛りで漏洩許容に緩和されるリスク回避）
```

# 落とし穴 2：False positive fatigue（厳しすぎて無効化される）

> Validator が厳しすぎると、人間が「めんどくさい」と無効化する。それが一番危険。

## 何が起きたか

Layer 8 の `secrets_leak_validator` を導入した直後、誤検知が連発しました。例えば：

- 英語記事の `4-4-4-4 文字の小文字単語列` が **Gmail App Password regex** に偶然マッチ（`logs+that+made+each` 形式の typical 英文も）
- 日本語の `「今〜すぐ使える」` 系の言い回しが FOMO 煽り判定（実は中立的な意味）
- 普通のドメイン名が「URL 形式の認証情報」と誤検知

このとき自分が考えたのは「**Validator 自体を一時的にバイパスしよう**」でした。これが最も危険な発想です。

## 構造解

- **severity 階層を 3 段（info / warn / error）に**：error だけ送信ブロック・warn は通すが log。fatigue 軽減と保証維持の両立。
- **regex の精度向上**：`[a-z]{4} [a-z]{4} [a-z]{4} [a-z]{4}` を「前後に password / token / 認証 等のキーワードがある場合のみ検出」に変更。
- **誤検知ケースは即時 dictionary 更新**：「これは NG じゃない」を辞書化する。Validator 自体は触らない。

```javascript
// 改善前：単純 4-4-4-4 検出
/\b[a-z]{4}\s[a-z]{4}\s[a-z]{4}\s[a-z]{4}\b/g

// 改善後：context-aware（前後 50 字以内に password / token 系キーワード）
function detectAppPassword(text) {
  const candidates = text.matchAll(/\b[a-z]{4}\s[a-z]{4}\s[a-z]{4}\s[a-z]{4}\b/g);
  return [...candidates].filter(m => {
    const ctx = text.slice(Math.max(0, m.index - 50), m.index + 50);
    return /password|token|アプリパス|認証/i.test(ctx);
  });
}
```

# 落とし穴 3：Pattern matching vs Intent

> regex で「絶対成功」をブロックしても、Generator は同じ意図を「100% 達成」「必ず成果」「請け負います」に書き換える。

## 何が起きたか

Layer 1-3 の regex で煽り表現を弾いても、Generator が**意味的に同じ煽り**を別表現で出してきます。これは LLM の言い換え能力の必然です。

regex を増やしてもイタチごっこ。語彙のロングテールに勝てません。

## 構造解

- **Layer 4 LLM Judge を導入**：regex の最終層として、独立した LLM（できれば別モデル）に「**意図として煽り訴求になっていないか**」を 0-10 で採点させる。8 以上で blocking。
- **Generator と Judge は別エージェント**：Sonnet が generate なら Haiku が judge。同モデル self-check は agreement bias で甘くなる（論文証明あり）。
- **Layer 1-3（regex）は表面、Layer 4（LLM）は意図**：両方必要。regex だけだと言い換えに負ける、LLM だけだと cost が嵩む。

```javascript
// Layer 4: LLM judge
async function llmJudge(text) {
  const prompt = `以下のコピーが「煽り訴求」かを 0-10 で採点。
0=完全中立 / 5=やや過剰 / 8=明らかに煽り / 10=極端な煽り
理由を 1 行で添えて。

コピー：${text}`;
  const result = await haiku.complete(prompt);
  return parseScore(result);
}
```

# まとめ：3 つの落とし穴に対する設計原則

1. **Validator drift** → 進化禁止領域 + Generator/Validator チーム分離
2. **False positive fatigue** → severity 階層 + dictionary 即時更新
3. **Pattern vs Intent** → regex（表面）+ LLM judge（意図）の併用

これらは新規研究ではなく、9 ヶ月運用の中で「踏んで・痛い・直した」経験の整理です。

詳細な実装パターンは [Arena Blueprint](https://ai-hack-lab.com/products/arena-blueprint/) で 10 モジュールに整理しています。Module 4（Validator）+ Module 8（Generator + Validator 原則）が本記事のテーマと最も近いです。

# 関連リンク

- 🇯🇵 [Arena Blueprint LP](https://ai-hack-lab.com/products/arena-blueprint/) — 10 モジュール完全版
- 📖 [Notion 公開ページ（無料）](https://shared-cut-385.notion.site/arena_blueprint_complete_v1-358347a29814808b81c1efe8d81cebe7)
- 🐙 [Skills Bundle (MIT)](https://github.com/pikuto1125-pixel/claude-skills-pikuto)
- 📰 [AI Hack Lab Daily Digest](https://ai-hack-lab.com/insights/) — 毎日 06:00 公開
- 🆓 [AI 業務診断（5 分無料）](https://ai-hack-lab.com/diagnosis/)
- 📝 [Qiita 全体像記事](https://qiita.com/pikuto1125-pixel/items/1655c87495424d4ba447)

— pikuto / [AI Hack Lab](https://ai-hack-lab.com/)
