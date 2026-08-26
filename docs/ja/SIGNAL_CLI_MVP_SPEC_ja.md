# Signal CLI — MVP 仕様書

## 1. Purpose

この文書は、Signal CLI の最初のバージョンに関する実装レベルの仕様を定義する。

目的は、小さく、実用的で、テスト可能な CLI を提供し、テキストを分析して AI生成らしさ、信頼度、分類、寄与シグナルを返せるようにすることである。

この実装は、将来の Signal API に向けたクリーンな基盤にならなければならない。

---

## 2. MVP Goals

MVP は次を満たす:

1. ファイルまたは標準入力からテキストを受け取る。
2. コンテンツを分析する。
3. AI生成らしさスコアを返す。
4. 信頼度を返す。
5. 分類を返す。
6. 人間が読める出力を返す。
7. 機械可読な JSON 出力を返す。
8. 意味のある終了コードを返す。
9. シェルスクリプトや CI で使える。
10. 分析エンジンを CLI 層から独立させる。
11. 評価 / テスト用データセットを含む。
12. 分析結果スキーマを将来の API で再利用可能にする。

MVP で不要なもの:

- Web UI
- ホストされた API
- 認証
- MCP
- ブラウザ拡張
- URL フェッチ
- PDF / DOCX 解析
- 画像 / 音声 / 動画解析
- ユーザーアカウント
- 永続ストレージ

---

## 3. User Stories

### 3.1 Analyze a file

開発者として、テキストファイルを分析し、AI生成コンテンツに関連するシグナルが含まれているかを確認したい。

```bash
signal article.md
```

### 3.2 Analyze standard input

開発者として、Signal をシェルワークフローに組み込むため、標準入力からコンテンツを流し込みたい。

```bash
cat article.md | signal
```

### 3.3 Consume JSON

開発者として、Signal を他ツールと統合するため、機械可読な JSON 出力を使いたい。

```bash
signal article.md --json
```

### 3.4 Use Signal in CI

開発者として、品質ゲートとして使えるよう、意味のある終了コードを得たい。

```bash
signal article.md --threshold 0.8
```

### 3.5 Understand the result

開発者として、単一スコアではなく、なぜその結果になったのかを知りたい。

---

## 4. CLI Interface

## 4.1 Command

```bash
signal <input>
```

`<input>` は次のいずれか:

- ローカルファイルパス
- 標準入力を表す `-`
- stdin が利用可能な場合は省略

例:

```bash
signal article.md
```

```bash
signal -
```

```bash
cat article.md | signal
```

明示的なファイル指定と stdin の両方がある場合は、明示的なファイルを優先する。

---

## 4.2 Options

初期オプション:

```text
--json
--threshold <float>
--quiet
--version
--help
```

### `--json`

機械可読な JSON を返す。

### `--threshold`

終了コード判定に使う AI生成らしさのしきい値を設定する。

デフォルト:

```text
0.8
```

### `--quiet`

人間向け分析出力を抑制する。

終了コードだけが重要な場合に有用である。

### `--version`

CLI バージョンを表示する。

### `--help`

使用方法を表示する。

---

## 5. Input Handling

### 5.1 Supported Formats

MVP:

- プレーンテキスト
- Markdown

Markdown は分析上はテキストとして扱う。

Markdown レンダリングは不要である。

### 5.2 Encoding

UTF-8 を必須とする。

入力が UTF-8 として不正な場合、CLI は入力エラーを返す。

### 5.3 Empty Input

空入力は不正である。

終了コード `2` を返す。

### 5.4 Minimum Input

短い入力も受け付けるが、必要に応じてデータ不足を示すべきである。

CLI は不十分な入力を人間 / AI のいずれかに黙って分類してはならない。

---

## 6. Result Model

分析エンジンは言語非依存の結果モデルを公開するべきである。

概念スキーマ:

```json
{
  "score": 0.87,
  "confidence": "high",
  "classification": "likely_ai_generated",
  "signals": [
    {
      "type": "repetitive_structure",
      "score": 0.82,
      "description": "Sentence structures show unusually low variation."
    }
  ],
  "input": {
    "characters": 4821,
    "words": 823
  },
  "engine": {
    "version": "0.1.0"
  }
}
```

正確なスキーマは、バージョン管理された内部契約として定義する。

---

## 7. Score

`score` は、分析対象コンテンツが AI によって生成されたらしさの推定値を表す。

型:

```text
float
```

範囲:

```text
0.0 <= score <= 1.0
```

解釈:

```text
0.0 = AI生成らしさが非常に低い
1.0 = AI生成らしさが非常に高い
```

基盤アルゴリズムが統計的に較正されていない限り、この値を probability と表現してはならない。

したがって初期のユーザー向け表現は:

```text
AI likelihood: 87%
```

であり、

```text
87% probability
```

ではない。

---

## 8. Confidence

許容値:

```text
low
medium
high
```

追加は明示的なスキーマ変更を通じてのみ行う。

信頼度は次のような要素を反映する:

- 入力長
- 利用可能なシグナル数
- analyzer 間の一致度
- 入力品質
- モデル固有の制約

score と confidence は独立したままでなければならない。

---

## 9. Classification

初期値:

```text
likely_human
possibly_human
uncertain
possibly_ai_generated
likely_ai_generated
```

初期マッピング推奨:

```text
0.00–0.19 → likely_human
0.20–0.39 → possibly_human
0.40–0.59 → uncertain
0.60–0.79 → possibly_ai_generated
0.80–1.00 → likely_ai_generated
```

これらのしきい値は暫定であり、設定値として扱い、評価データセットで検証する。

---

## 10. Signal Model

各 analyzer は 0 個以上の signal を生成する。

概念インターフェース:

```text
Analyzer.analyze(document) -> []Signal
```

Signal は次を持つ:

```text
type
score
description
```

例:

```json
{
  "type": "repetitive_structure",
  "score": 0.82,
  "description": "Sentence structures show unusually low variation."
}
```

### Initial Signal Types

MVP は少数の説明可能な signal から始めるべきである。

候補:

```text
repetitive_structure
sentence_length_consistency
vocabulary_diversity
phrase_repetition
stylistic_variation
```

最終リストは、実装時に何が信頼して測定できるかに基づいて決める。

Signal 数を増やすためだけに signal を追加してはならない。

すべての signal は次を持つ:

1. 明確な定義
2. 可能なら決定論的な計算
3. ユニットテスト
4. ドキュメント
5. CLI 出力に適した説明文

---

## 11. Analysis Engine Architecture

CLI は分析エンジンの薄いアダプターであるべきである。

推奨概念アーキテクチャ:

```text
                 ┌───────────────┐
                 │ Signal Engine │
                 └───────┬───────┘
                         │
              ┌──────────┴──────────┐
              │                     │
           Analyzer              Scorer
              │                     │
       ┌──────┼──────┐              │
       │      │      │              │
   Structure Style Vocabulary       │
       │      │      │              │
       └──────┴──────┴──────────────┘
                         │
                   AnalysisResult
                         │
                  ┌──────┴──────┐
                  │             │
                 CLI          Future API
```

CLI にスコアリングロジックを直接実装してはならない。

---

## 12. Suggested Project Structure

正確な構造は実装言語に依存するが、概念上の責務分離は保つべきである。

例:

```text
signal/
├── cmd/
│   └── signal/
│       └── main.*
├── internal/
│   ├── analyzer/
│   │   ├── analyzer.*
│   │   ├── structure.*
│   │   ├── vocabulary.*
│   │   ├── phrase.*
│   │   └── style.*
│   ├── scoring/
│   │   └── scorer.*
│   ├── model/
│   │   └── result.*
│   └── input/
│       └── reader.*
├── tests/
│   ├── fixtures/
│   └── evaluation/
├── docs/
├── README.md
├── LICENSE
└── ...
```

言語固有の慣習に合わせて変えてよいが、責務分離は維持する。

---

## 13. Human-Readable Output

デフォルト出力:

```text
Signal

AI likelihood    87%
Confidence       High

Signals
  ✓ Repetitive sentence structure
  ✓ Consistent sentence length
  ✓ AI-like phrase patterns

Analyzed
  4,821 characters
  823 words
```

文言は今後調整してよい。

優先順位:

1. 結果
2. 信頼度
3. シグナル
4. 入力統計

端末で読みやすい簡潔さを保つ。

---

## 14. JSON Output

`--json` は JSON のみを stdout に出力する。

バナー、進捗表示、ログ、診断メッセージを stdout に出してはならない。

Errors は stderr に出力する。

---

## 15. Exit Codes

初期終了コード:

```text
0 = 分析成功
1 = AI生成らしさがしきい値以上
2 = 入力エラー
3 = 分析エラー
```

意味は安定していなければならない。

---

## 16. Threshold Handling

`--threshold` は終了コード判定にのみ影響するべきであり、score 自体を変更してはならない。

条件:

- `0.0 <= threshold <= 1.0`
- 不正な値は入力エラー
- threshold 未指定時はデフォルトを使う

---

## 17. Insufficient Data

入力が短すぎる場合、エンジンはそれを明示的に表現できるべきである。

少なくとも:

- confidence を低くする
- 必要に応じて signal 数を減らす
- データ不足を隠さない

confidence を捏造してはならない。

---

## 18. Privacy

MVP では分析対象コンテンツを永続化しない。

要件:

- 入力内容をログに書かない
- エラーメッセージに内容を含めない
- 分析対象コンテンツを含む隠しキャッシュを作らない
- 明示的な実装とドキュメントなしに外部サービスへ送信しない
- README にデータ取り扱いを記載する

---

## 19. Determinism

同じ入力と同じ分析エンジンバージョンであれば、MVP は決定論的な結果を返すべきである。

将来、非決定論的な外部モデルを導入する場合は、モデル / バージョン情報と再現性への影響を公開する。

---

## 20. Testing Strategy

テストは第一級要件である。

### 20.1 Unit Tests

各 analyzer に対して次をテストする:

- 通常入力
- エッジケース
- 空入力
- 非常に短い入力
- 長い入力
- Unicode
- 日本語テキスト
- Markdown
- 繰り返しテキスト
- 強く構造化されたテキスト

### 20.2 Integration Tests

完全な CLI フローをテストする:

```text
file → input reader → engine → result → formatter → exit code
```

### 20.3 JSON Contract Tests

JSON 出力を結果スキーマに対して検証する。

### 20.4 Golden Tests

代表的 fixture の期待出力を管理する。

### 20.5 Evaluation Tests

バージョン管理された評価データセットを保守する:

```text
human
ai_generated
human_edited_ai
ai_edited_human
mixed
```

評価結果は再現可能でなければならない。

---

## 21. Japanese Language Support

日本語テキストは第一級の入力言語として扱う。

MVP は単語が空白で区切られることを前提にしてはならない。

したがって、word count ロジックは、空白区切りを持たない言語にも対応する必要がある。

日本語では適切な tokenizer を使うか、代替測定を明確に定義する。

分析エンジンは英語専用前提に制限されてはならない。

---

## 22. Error Handling

エラーは実行可能な内容であるべきである。

例:

```text
Error: file not found: article.md
Error: input is empty
Error: invalid threshold: 1.5
Error: input must be valid UTF-8
```

デフォルトではスタックトレースを露出しない。

将来的に debug mode を追加してよい。

---

## 23. CLI UX Requirements

CLI は普通の開発者ツールのように振る舞うべきである。

要件:

- 起動が速い
- 出力が明確
- エラーが有用
- Unix pipeline をサポート
- 終了コードが安定
- JSON 形式が安定
- 不要なテレメトリがない
- 通常動作で対話プロンプトを出さない

---

## 24. Versioning

CLI は次を提供する:

```bash
signal --version
```

例:

```text
signal 0.1.0
```

---

## 25. API Compatibility Requirement

API は MVP 実装範囲外だが、分析結果モデルは将来の API 利用を見据えて設計する必要がある。

つまり:

- 結果スキーマは CLI 専用表現に閉じない
- JSON フィールド名は API にも流用しやすい
- engine 情報や signal 情報を後から拡張しやすい

---

## 26. Documentation Requirements

### README

README には少なくとも以下を記載する:

- Signal が何か
- インストール方法
- 基本的な使い方
- JSON 出力例
- 終了コード
- 制約事項
- プライバシー方針

### CONTRIBUTING

CONTRIBUTING には以下を含める:

- 開発方法
- テスト実行方法
- fixture / evaluation data の扱い

### Architecture Documentation

CLI と分析エンジンの責務分離を説明する。

### Evaluation Documentation

評価方法とその限界を説明する。

---

## 27. CI Requirements

CI は次を実行するべきである。

- unit tests
- integration tests
- JSON contract tests
- formatting / linting

可能であれば、代表 fixture に対する golden test も含める。

---

## 28. Security Requirements

MVP は最低限次を満たす:

- 入力ファイルを意図しない場所に書き出さない
- 任意コード実行を伴う処理を持ち込まない
- 外部送信をデフォルトで行わない
- dependency を明示的に管理する

---

## 29. Out of Scope for MVP

MVP の対象外:

- Web UI
- Hosted API
- MCP server
- Browser extension
- URL crawling
- PDF / DOCX parsing
- Image / audio / video detection
- Remote LLM explanations
- User accounts
- Billing
- Persistent storage

これらは初期アーキテクチャを複雑にしないよう後続フェーズに回す。

---

## 30. Definition of Done

MVP 完了条件:

- CLI がファイルと stdin を処理できる
- score / confidence / classification / signals を返せる
- `--json` が安定した JSON を返す
- 終了コードが定義どおりに動く
- テストスイートが存在する
- 評価データセットが存在する
- 結果モデルが将来の API に再利用できる

---

## 31. Implementation Principle

主目的は、最も高度な AI detector を最初から作ることではない。

最小限で有用な実装を作り、測定し、評価データに基づいて分析エンジンを改善することが重要である。

```text
                 Signal
                   │
                   ▼
         Build small, measure, improve
```
