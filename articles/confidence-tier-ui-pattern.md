---
title: "同じ「計算結果」でも確信度が違う数値を、同じ見た目で出さないための実装パターン"
emoji: "🔍"
type: "tech"
topics: ["javascript", "個人開発", "設計", "フロントエンド", "css"]
published: true
published_at: "2026-10-01 08:00"
---

計算系のツールを作っていると、同じ「結果を表示する」機能の中に、確信度がまったく違う数値が混在することがあります。たとえば以下はどちらも「トークン数」や「休暇日数」という1つの数値ですが、裏側の確からしさはまったく違います。

- 公開されているアルゴリズム通りに、入力から一意に決まる値(確定値)
- 非公開の仕様や個別事情に依存するため、近似・参考にしかならない値(参考値)

この2つを同じフォントサイズ・同じ色で並べて出すと、ユーザーは参考値の方も確定値と同じ確からしさで受け取ってしまいます。これは「精度を盛る」ことと実質的に同じ結果を生む、地味だが見落としやすい失敗です。実装で確信度を区別する、汎用的なパターンを整理します。

## 3段階の確信度とその見た目

筆者が実装しているツールでは、結果表示を次の3状態のいずれかに分類しています。

1. **confirmed(確定値)**: 公開仕様通りに入力から一意に決まる。通常の見た目でそのまま出す
2. **estimate(参考値)**: 近似・非公開仕様への代替・前提条件付きの値。バッジと注記付きで出す
3. **unavailable(未対応)**: この入力では確信度のある値を出せない。数値を出さず、理由と代替手段(専門家に確認、等)だけを示す

これは表示スタイルの話であると同時に、実装を始める前に「この機能はどの状態になり得るか」を洗い出すための分類でもあります。

## 実装: data属性 + CSSで状態を分離する

JS側は「どの状態か」を判定するだけにして、見た目の差はCSS側に寄せると、状態が増えてもロジックが複雑になりません。

```html
<div class="result" data-confidence="confirmed">
  <span class="result__value">12,480</span>
  <p class="result__note"></p>
</div>
```

```css
.result[data-confidence="confirmed"] .result__value {
  color: var(--color-text);
}

.result[data-confidence="estimate"] .result__value::after {
  content: "参考値";
  margin-left: 0.5em;
  padding: 0.1em 0.5em;
  font-size: 0.75em;
  border-radius: 999px;
  background: var(--color-warning-bg);
  color: var(--color-warning-text);
}
.result[data-confidence="estimate"] .result__note {
  display: block; /* 参考値の根拠・前提条件をここに書く */
}

.result[data-confidence="unavailable"] .result__value {
  display: none;
}
.result[data-confidence="unavailable"] .result__note {
  display: block;
  color: var(--color-muted);
}
```

```js
function renderResult(el, { confidence, value, note }) {
  el.dataset.confidence = confidence;
  el.querySelector('.result__value').textContent = value ?? '';
  el.querySelector('.result__note').textContent = note ?? '';
}
```

状態遷移はJSの1関数、見た目の作り分けはCSSのセレクタ1組に閉じるため、確信度を判定するロジック(業務要件側)と見た目(UI側)が混ざりません。新しいツールを作るたびに同じ3クラスを使い回せます。

## 実装例1: 非公開トークナイザの参考値表示

トークンカウンターでは、GPT系(公開BPE仕様)は `confirmed` として実際にエンコードした値をそのまま出しますが、Claude/Geminiは公式トークナイザが非公開のため、別モデルのエンコーダによる近似値を `estimate` として、バッジと「参考値です」という注記付きで表示しています。数値自体は消さず、確信度が違うことだけを明示する形です。

## 実装例2: 前提条件が外れたら参考値に切り替える

有給休暇計算ツールでは、「出勤率8割以上」という適用条件が確認できない場合、`confirmed` から `estimate` へ表示状態を切り替え、「専門家に確認を」という注記を出します。前提条件のチェックボックス1つが、そのまま `data-confidence` の値を切り替えるトリガーになっています。

## まとめ

- 確信度が違う数値を同じ見た目で出すと、ユーザーは全部を同じ確からしさで受け取ってしまう
- confirmed / estimate / unavailable の3状態に分類し、`data-confidence` のようなJS側の1つの状態変数とCSS側の見た目を分離すると、ツールが増えても同じパターンを使い回せる
- 前提条件が外れたときも数値そのものは隠さず、確信度が下がったことを明示する方向で切り替える

この考え方は、以前書いた[計算していい範囲の線引き](https://zenn.dev/ykwlab/articles/tool-scope-boundary-regulated-domain)や[サーバーに送らない設計](https://zenn.dev/ykwlab/articles/client-side-tool-no-network-trust-design)と同じ「誠実さ」を、実装のレベルまで落とし込んだものです。

---

:::message
**PR**: この記事の筆者は、建設・製造など現場を持つ中小企業向けに、日報・点検・在庫管理の「書く・数える・転記する」を固定価格のカスタムアプリでなくすサービス **[現場革命](https://www.genbakakumei.com)** を運営しています。現場業務のDXに関心があればぜひ。
:::
