---
title: "【Aqua Voice】カスタム指示の設定ガイド ― 音声入力をSlack用に最適化する"
emoji: "🎙️"
type: "tech"
topics: ["AquaVoice", "音声入力", "Slack", "生産性向上"]
published: true
---

## はじめに

この記事では、AI音声入力ツール「Aqua Voice」の**カスタム指示（Custom Instructions）**の設定方法と、私が実際にSlack用途で使っているカスタム指示の記載例を紹介します。

**対象読者**

- Aqua Voiceを既に使っているが、カスタム指示はまだ設定していない方
- 音声入力の出力品質を上げたい方
- Slackでの音声入力を実用レベルにしたい方

---

## Aqua Voiceとは

Aqua Voiceは、AIを搭載した高精度な音声入力ツールです。Mac / Windows のネイティブアプリとして動作し、テキスト入力ができる場所ならどこでも使えます。

### 主な特徴

| 特徴 | 説明 |
|------|------|
| **高精度な文字起こし** | AIが文脈を理解し、「えーと」「あのー」などのフィラーワードを自動除去 |
| **カスタム指示** | 出力テキストのスタイルやフォーマットを自然言語で制御可能 |
| **カスタム辞書** | 固有名詞や専門用語の「読み → 正しい表記」を登録可能 |
| **画面認識（ディープコンテキスト）** | 画面上のテキストを認識し、文脈に応じた変換を実現 |
| **高速レスポンス** | 起動50ms、出力450msの低レイテンシ |
| **49言語対応** | 日本語、英語など多言語に対応。自動言語検出も可能 |

### 料金プラン

| プラン | カスタム指示 |
|--------|-------------|
| Free（1,000ワードまで） | ✕ |
| Pro（有料） | ✔ |

カスタム指示は**Proプラン以上**で利用可能です。最新の料金は[公式サイト](https://withaqua.com/)を参照してください。

:::message
学生は `.edu` メールアドレスで登録すると、年額プランが70%オフになります。
:::

### 公式サイト

https://withaqua.com/

---

## カスタム指示とは

カスタム指示（Custom Instructions）とは、Aqua Voiceが音声をテキストに変換する際に「どのように整形・修正するか」をAIに伝える指示書です。

たとえば、以下のようなことが可能になります。

- フィラーワード（「えーと」「あのー」）の自動除去
- 句読点・改行の自動挿入
- 「ですます調」への統一
- カタカナ語の表記統一（例：サーバ → サーバー）
- 冗長な口語表現の簡潔化

### 設定場所

アプリ画面の **「Instructions」（インストラクション）** から設定します。

:::message alert
**重要**: カスタム指示を正しく動作させるには、設定画面で**ストリーミングモードを「Always（常に）」**に変更してください。デフォルトのままだとカスタム指示が反映されない場合があります。
:::

---

## なぜカスタム指示を英語で書くのか

本記事で紹介するカスタム指示は**英語で記述**しています。理由は以下の通りです。

1. **Aqua VoiceのAIは英語の指示処理精度が最も高い**
   - Aqua Voiceの開発元は米国のY Combinator出身のスタートアップであり、AIモデルのチューニングが英語を基準に行われている
   - 英語で書いた指示の方が、意図通りの動作をする確率が高い

2. **日本語の音声入力自体は日本語で正しく処理される**
   - カスタム指示が英語でも、話す言葉は日本語のままでOK
   - 「指示書は英語、入力は日本語」という使い分け

3. **参考にできる事例が多い**
   - 英語圏のユーザーが公開しているカスタム指示を流用・応用しやすい

:::message
もちろん日本語でカスタム指示を書いても動作します。ただし、複雑な条件分岐や細かいルール指定は英語の方が安定する傾向があります。
:::

---

## 前提：推奨する本体設定

カスタム指示を設定する前に、以下のAqua Voice本体の設定を確認してください。

| 設定項目 | 推奨値 | 理由 |
|----------|--------|------|
| ストリーミングモード | **Always（常に）** | カスタム指示が正しく動作する |
| 言語 | **Japanese（固定）** | Auto-Detectだと英語混在の可能性あり |
| アクティベーションキー | お好みで（右Altキーが人気） | - |

---

## カスタム指示の記載例（Slack用）

以下が、私が実際にSlack用途で使っているカスタム指示です。コピーしてそのまま使えます。

### 全文

```text
# Japanese Voice Input Instructions for Slack Communication

## PROCESSING ORDER (Must follow this exact sequence)
1. Filler Word Removal
2. Line Break and Bracket Handling
3. Text Formatting and Cleanup
4. Style and Expression Normalization
5. Final Output Generation

---

## 1. Filler Word Removal
Remove ALL filler words and hesitation sounds from the input:
- Remove: あー, えー, うー, えーと, あのー, そのー, まあ, ええ, うーん, んー
- Remove all interjections and verbal fillers
- When repeated words appear (e.g., "それで、それで"), keep only one occurrence
- When corrections are made mid-speech, use only the final version

---

## 2. Line Break and Bracket Handling

### Line Breaks
When the input includes "改行" (kaigyo), insert an actual line break at that point.
If similar-sounding words like "開業" appear but clearly mean a line break in context, treat them as "改行".

### Bracket Insertion
- "かっこ" → insert （ or ） based on context
- "かぎかっこ" → insert 「 or 」 based on context

Examples:
- Input: "注意事項かっこ以下の点かっこ" → Output: "注意事項（以下の点）"
- Input: "かぎかっこ了解ですかぎかっこと返事" → Output: "「了解です」と返事"

---

## 3. Text Formatting and Cleanup

### Punctuation
- End every sentence with 。(period)
- Insert 、(comma) at appropriate positions for readability
- Add line breaks when topics change
- Group 3 or more sentences into paragraphs

### Number and Symbol Rules
- Use half-width numbers (1, 2, 3... not １, ２, ３)
- Use half-width for alphanumeric characters in technical terms

### Katakana Standardization (USE LONG VOWEL MARK)
Always use the long vowel mark (ー) for katakana loanwords:
- サーバー (not サーバ)
- コンピューター (not コンピュータ)
- ユーザー (not ユーザ)
- プログラマー (not プログラマ)
- ブラウザー (not ブラウザ)
- マネージャー (not マネージャ)

### Space Removal
Remove all unnecessary half-width spaces, including those at the beginning, between, and at the end of sentences.

---

## 4. Style and Expression Normalization

### For Slack Messages
- Use casual-polite style (です・ます調, but natural and concise)
- Keep messages short and direct
- Do NOT use overly formal expressions

### Connector Word Cleanup
- Remove sentence-initial "で、" "それで、" "あと、" or replace with proper connectors
- "なので" → "そのため" or "ですので" (only when formal tone is needed)
- "させていただく" → "いたします" or "します"
- Fix double honorifics (二重敬語)

### Conciseness
- Eliminate verbose spoken-language patterns
- Convert to concise written-style Japanese
- Remove redundant polite expressions

---

## 5. Final Output
Produce clean, professional Japanese text suitable for Slack communication.
Remove all verbosity typical of voice input.
Output should be natural, concise, and immediately usable without editing.
```

---

## 各セクションの解説

### ① PROCESSING ORDER（処理順序）

AIに「この順番で処理してね」と伝える部分です。順序を明示することで、処理の安定性が上がります。

たとえば「フィラー除去 → 改行処理 → 整形」という順番を守らないと、フィラーワードの位置に余計な改行が入るなどの問題が起こり得ます。

### ② Filler Word Removal（フィラーワード除去）

音声入力で最も気になる「えーと」「あのー」などの言いよどみを自動除去します。

Aqua Voice自体にもフィラー除去機能がありますが、カスタム指示で明示することでより確実に除去されます。

また、以下の2つのルールが実用上とても重要です。

- **繰り返し除去**: 「それで、それで」→「それで」
- **言い直し対応**: 途中で言い直した場合は最後の発言のみ採用

### ③ Line Break and Bracket Handling（改行・括弧処理）

「改行」と声に出すことで、実際の改行が挿入されます。Slackでメッセージを段落分けしたいときに便利です。

括弧も同様に、「かっこ」「かぎかっこ」と声に出すことで自動挿入されます。開き・閉じの判定はAIが文脈から自動で行います。

:::message
この改行・括弧処理の仕組みは、kamikou氏がGitHubで公開しているカスタム指示を参考にしています。
参考: https://github.com/kamikou/aqua-voice-ja-instruct
:::

### ④ Katakana Standardization（カタカナ表記統一）

カタカナ語の長音記号（ー）を統一するルールです。

| ❌ 長音記号なし | ✅ 長音記号あり（本設定） |
|---------------|------------------------|
| サーバ | サーバー |
| コンピュータ | コンピューター |
| ユーザ | ユーザー |
| ブラウザ | ブラウザー |
| マネージャ | マネージャー |

JIS規格では長音省略（サーバ）が標準でしたが、近年はMicrosoftのガイドラインをはじめ**長音記号付き**が主流です。チームの表記ルールに合わせて変更してください。

### ⑤ Style and Expression Normalization（文体の正規化）

Slack用に「ですます調だけど堅すぎない」トーンに整形します。

**変換例:**

| 音声入力（話し言葉） | 変換後（Slack向け） |
|---------------------|-------------------|
| で、それでどうなったかっていうと | その結果、 |
| なので対応お願いします | そのため、対応をお願いします。 |
| 確認させていただきます | 確認いたします。 |
| あと、もう一個あるんですけど | また、もう1点あります。 |

---

## カスタム辞書も併用しよう

カスタム指示と合わせて、**カスタム辞書（Dictionary）** も設定すると精度がさらに上がります。

固有名詞や製品名が正しく変換されない場合は、カスタム辞書に登録するのが最も効果的です。

### 登録例

| 読み（声に出す言葉） | 変換後（出力される表記） |
|---------------------|--------------------------|
| あくあぼいす | Aqua Voice |
| くろーど | Claude |
| すらっく | Slack |
| じら | JIRA |
| こんふるえんす | Confluence |
| ぎっとはぶ | GitHub |
| いんてりじぇい | IntelliJ |
| すぷりんぐぶーと | Spring Boot |

### 設定方法

アプリ画面の **「Dictionary」** から登録できます。

---

## 自分用にカスタマイズするヒント

紹介したカスタム指示はベースとしてそのまま使えますが、自分の話し方や業務内容に合わせて調整すると、より精度が上がります。

### よくあるカスタマイズ

**1. 自分の口癖を追加除去する**

セクション1に追加:

```text
- Remove: "ちょっと", "基本的に", "一応" (when used as filler, not meaningful content)
```

**2. 業界用語の変換ルールを追加する**

セクション3に追加:

```text
## Special Term Conversion
- エスキューエル → SQL
- シーシャープ → C#
- ジェイエス → JS
```

**3. メンション挿入ルールを追加する**

セクション2に追加:

```text
### Mention Insertion
- When "メンション〇〇" is detected, output "@〇〇"
- Example: "メンション田中さん" → "@田中さん"
```

---

## 実際の変換例

### Before（カスタム指示なし）

```text
えーとあのー昨日のミーティングの件なんですけどえーサーバの設定変更
それでそれであやっぱりデプロイの件ですね確認させていただきたいんですが
明日対応可能でしょうか
```

### After（カスタム指示あり）

```text
昨日のミーティングの件ですが、デプロイの設定変更について確認したいです。
明日対応可能でしょうか。
```

フィラー除去・言い直し対応・表記統一・簡潔化がすべて自動で処理されます。

---

## トラブルシューティング

### カスタム指示が反映されない

- **ストリーミングモードを確認**: 「Always（常に）」に設定されているか
- **Proプランか確認**: Freeプランではカスタム指示は利用不可
- **指示の長さ**: 極端に長い指示は不安定になる場合あり。本記事の例程度の長さが目安

### 改行が「改行」という文字で出力される

- ストリーミングモードが「Always」になっていない可能性が高い

### 固有名詞が誤変換される

- カスタム指示ではなく**カスタム辞書（Dictionary）**に登録するのが最も効果的

---

## まとめ

| やること | 効果 |
|----------|------|
| ストリーミングモードを「Always」に変更 | カスタム指示が正しく動作する |
| カスタム指示を英語で設定 | 処理精度が最も高くなる |
| カスタム辞書に固有名詞を登録 | 製品名・人名の誤変換を防止 |

まずは本記事のカスタム指示をそのまま設定してみてください。1〜2週間使ってみて、自分の話し方のクセに合わせて調整していくのがおすすめです。

---

## 参考リンク

- [Aqua Voice 公式サイト](https://withaqua.com/)
- [Aqua Voice ユーザーガイド](https://aquavoice.com/guide)
- [kamikou氏のカスタム指示（GitHub）](https://github.com/kamikou/aqua-voice-ja-instruct)
- [Aqua Voice: 指示で更に洗練する（note）](https://note.com/sho7650/n/n28acf7557b33)
- [カスタムプロンプト付き Aqua Voiceガイド（note）](https://note.com/fumigoro_nomad/n/nc2eb9b18fbcf)
- [ストリーミングモード設定の解説（note）](https://note.com/feath/n/n0ac2f386c52a)
