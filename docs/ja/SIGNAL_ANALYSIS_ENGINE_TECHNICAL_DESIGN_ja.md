# Signal Analysis Engine — 技術設計

## 1. Purpose

この文書は、Signal 分析エンジンの MVP 向け技術設計を定義する。

エンジンはテキストコンテンツを分析し、AI生成らしさに関する構造化された評価を返す責務を持つ。CLI と将来の API の両方から同じ実装を再利用できるよう、独立したコアとして設計する。

この文書が対象とする内容:

- エンジンの責務
- コアドメインモデル
- 検出戦略
- 分析パイプライン
- コンポーネント設計
- スコアリングと信頼度設計
- 決定性、プライバシー、テスト要件
- 将来拡張性

CLI フラグやユーザー向けコマンド挙動そのものは、エンジン契約に関係する部分を除き扱わない。

---

## 2. Design Goals

設計目標:

1. 分析ロジックを CLI や API などの提供インターフェースから独立させる。
2. Signal を単一の detector ではなく、複数の detector / analyzer を束ねるオーケストレータとして扱う。
3. ブラックボックスな単一スコアではなく、説明可能な結果を返す。
4. 単一ヒューリスティックではなく、複数の独立した signal を組み合わせる。
5. モデルベース signal とヒューリスティック signal を同じ結果契約で扱えるようにする。
6. MVP では決定論的でローカル計算可能な分析を優先する。
7. 実用価値がある場合に備えて、より強いローカル事前学習モデルを導入できる余地を残す。
8. 英語専用前提にせず、日本語も扱えるようにする。
9. デフォルトで分析対象コンテンツの保存や外部送信を避ける。
10. 任意の LLM ベース説明機能は、コア検出経路とは分離する。

---

## 3. Non-Goals

MVP エンジンの非目標:

- コンテンツが AI 生成かどうかを証明すること
- 画像、音声、動画入力のサポート
- コア検出でリモート LLM 呼び出しに依存すること
- 実行時にユーザー投稿から継続学習すること
- 「モデル X によって生成された」といった attribution を行うこと
- アーキテクチャが検証される前に全言語最適化を行うこと

---

## 4. Architectural Position

分析エンジンは Signal のコアドメインコンポーネントである。

Signal は、複数の detector と analyzer の出力を共通の signal モデルに正規化し、それを集約するシステムとして設計するべきである。

概念的には:

```text
Input Adapter
  -> Document Builder
  -> Preprocessor
  -> Detector/Analyzer Runners
  -> Score Aggregator
  -> Confidence Estimator
  -> Classification Mapper
  -> AnalysisResult
```

より具体的には:

```text
                    Input
                      │
                      ▼
              ┌──────────────┐
              │ Preprocessor │
              └──────┬───────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   AI Detector   Statistical   Linguistic
      Model        Signals       Signals
        │            │            │
        └────────────┼────────────┘
                     ▼
              Signal Aggregator
                     │
                     ▼
              Analysis Result
```

MVP では:

```text
                 ┌────────────────────────┐
                 │ Signal Analysis Engine │
                 └────────────┬───────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
     Input Contract      Result Contract     Config Contract
          │                   │                   │
          └───────────────────┴───────────────────┘
                              │
                    Thin Interface Adapters
                              │
                   CLI now / API in the future
```

インターフェース層はファイル読み込み、stdin 処理、表形式出力、JSON シリアライズ、終了コード判定を行ってよいが、エンジンはそれらに依存してはならない。

---

## 5. Responsibilities and Boundaries

### 5.1 Engine Responsibilities

エンジンは次を行う:

- 正規化済みテキスト入力と分析オプションを受け取る
- 内部文書表現を構築する
- 同一入力に対して複数の detector / analyzer コンポーネントを走らせる
- ヒューリスティック analyzer に必要な測定特徴量を抽出する
- 異種の出力を説明可能な signal に正規化する
- signal を集約して AI生成らしさスコアを算出する
- score とは独立に confidence を推定する
- score を classification に変換する
- 安定した言語非依存の結果オブジェクトを返す

### 5.2 Interface Responsibilities

CLI や将来の API は次を担当する:

- ファイルまたは stdin の読み込み
- コマンドライン引数の検証
- 人間向け出力の整形
- エンジン結果の JSON 化
- エンジンエラーをインターフェース向けメッセージに変換

### 5.3 Explicit Boundary Rule

スコアリング、しきい値マッピング、signal 生成ロジックを CLI 層に置いてはならない。

---

## 6. Detection Strategy

AI生成検出にはいくつか主要な系統がある。Signal はそれらをひとつのアルゴリズムに潰すのではなく、異なる signal source として明示的に扱うべきである。

### 6.1 Detection Families

アーキテクチャは次の4分類をサポートする:

1. 専用 AI 検出モデル
2. Perplexity などの統計的言語モデル signal
3. 言語的 / 文体的特徴 analyzer
4. 任意の LLM ベース説明

### 6.2 Dedicated AI-Detection Models

事前学習済みの AI 生成テキスト検出モデルは、最初期の高価値 signal 候補である。

```text
Input text
  -> pretrained detector
  -> AI-likelihood score
  -> DetectorModelSignal
```

これは Signal 全体そのものではなく、Signal を構成する signal source のひとつとして扱う。

利点:

- 実用的なベースラインを早く構築できる
- ひとつの寄与 source として説明しやすい
- 他の独立した signal と組み合わせると強い証拠になり得る

制約:

- モデルの provenance と version を engine metadata に露出する
- 出力を共通 signal 形式に正規化する
- 無効化や利用不可時にもエンジン全体が動くようにする

### 6.3 Statistical Signals

統計的 signal は、予測しやすさ、繰り返し、トークン分布などの性質を見る。

最重要例は perplexity である。

perplexity は最終的な AI 判定として扱うべきではない。`perplexity_pattern` のような独立 signal として扱うべきである。

```text
Input text
  -> language model scoring
  -> perplexity or related metric
  -> PerplexitySignal
```

整った人間の文章でも低 perplexity になり得るし、AI生成文章を人間が編集すれば単純な予測しやすさは崩れるためである。

### 6.4 Linguistic and Stylistic Signals

これらは feature-based analyzer であり、例えば次のようなものを含む:

- 文長の変動
- 語彙多様性
- フレーズの繰り返し
- 文体変動
- フォーマットの規則性

これらは単独で AI 生成を断定しない。モデルベースや統計ベースの証拠と組み合わされたときに、より有用になる。

### 6.5 LLM-Based Explanation

LLM-as-a-judge は MVP の主判定手段にすべきではない。

将来的には説明層として使う方が適切である。

```text
Detection
  -> Signals
  -> LLM explainer
  -> Human-readable narrative
```

これにより、コアエンジンの決定性を保ちつつ、後から説明力を強化できる。

---

## 7. Core Domain Model

分析ステップをテスト可能かつ合成可能に保つため、明示的な内部モデルを用いる。

### 7.1 Document

`Document` は入力テキストの正規化済み表現である。

```text
Document
- raw_text
- normalized_text
- language_hints
- character_count
- line_count
- paragraph_count
- sentence_units
- token_units
- metadata
```

補足:

- `raw_text` はエンジンが受け取った元入力を保持する
- `normalized_text` は正規化後の決定論的分析用表現
- `sentence_units` と `token_units` は言語依存 segmentation を抽象化する
- `metadata` には normalization version などの非機密派生情報を入れてよい

### 7.2 Feature Set

`FeatureSet` は文書から導出された不変の測定値集合である。

例:

- 文長分布
- 段落長分布
- 繰り返し n-gram 数
- type-token ratio
- unique phrase ratio
- 句読点分布
- Markdown の heading / list / code block 比率
- 文字クラス比率
- 隣接文間の遷移分散

Feature extraction は heuristic analyzer 実行前に行い、複数 analyzer が共通利用できるようにする。

モデルベース detector はこの層の大部分を経由せず、直接 normalized signal を出してもよい。

### 7.3 Signal

`Signal` は説明可能性の最小単位である。

```text
Signal
- type
- source_family
- source_name
- score
- weight
- description
- evidence
```

定義:

- `type`: `repetitive_structure` のような安定識別子
- `source_family`: `detector_model`、`statistical`、`linguistic` など
- `source_name`: `roberta_detector` や `perplexity` などのコンポーネント識別子
- `score`: `0.0..1.0` の正規化値
- `weight`: scorer が用いる任意の寄与重み
- `description`: 人間向け説明
- `evidence`: 根拠となる構造化指標

`evidence` は将来 API で詳細表示できるよう機械可読に保つ。

このモデルにより、性質の異なる signal producer を同一契約で束ねられる。

### 7.4 Analysis Result

`AnalysisResult` はエンジンの出力契約である。

```text
AnalysisResult
- score
- confidence
- classification
- signals
- input_stats
- engine_info
```

意味:

- `score`: AI生成らしさ。証明ではない
- `confidence`: 分析の信頼性
- `classification`: 設定しきい値に基づくラベル
- `signals`: 寄与した signal の順序付きリスト
- `input_stats`: 上位インターフェースに必要な入力統計
- `engine_info`: engine version、scoring profile、有効 component 識別子

---

## 8. Analysis Pipeline

エンジンパイプラインは決定論的かつ明示的であるべきである。

### 8.1 Step 1: Input Normalization

raw text を一貫した分析表現へ変換する。

MVP の normalization は次を行う:

- 改行コード正規化
- 不要な前後空白の除去
- 連続空行の扱いの標準化
- 意味を持つ句読点の保持
- 特定 feature extractor が必要としない限り大文字小文字を保持
- 可能な限り Markdown 構造を保持

小さな変更でも結果に影響するため、normalization は version 管理する。

### 8.2 Step 2: Language-Aware Segmentation

文書を分析単位に分割する。

必要単位:

- 文字
- 行
- 段落
- 文様単位
- トークン様単位

空白区切り前提は禁止する。日本語では専用 tokenizer または明示された代替戦略を用いる。実装選択は document-building 層の背後に隠す。

### 8.3 Step 3: Shared Preparation

前処理後、各 component に必要な共有 artifact を準備する。

例:

- tokenization 結果
- 文分割結果
- heuristic analyzer 用 feature extraction
- ローカル detector model 用 tensor / model input
- perplexity 計算用の normalized text view

重要なのは、準備処理を各インターフェースに重複実装しないことである。

### 8.4 Step 4: Detector and Analyzer Execution

有効 component 群を実行し、それぞれが 0 個以上の normalized signal を返す。

想定される2種類:

- 入力から直接 AI-likelihood 系 signal を返す direct detector
- feature から狭い証拠 signal を返す heuristic analyzer

```text
Component.run(document, prepared_inputs) -> []Signal
```

例:

- `DetectorModelComponent`
- `PerplexityComponent`
- `SentenceStructureAnalyzer`
- `VocabularyPatternAnalyzer`

各 component は次を満たす:

- 決定論的である
- 安定した component metadata を持つ
- 結果を共有 signal 契約に正規化する
- 証拠不足なら signal を返さない
- 将来の調査用に十分な evidence を残す

### 8.5 Step 5: Score Aggregation

scorer は signal 群を単一の AI-likelihood score に集約する。

要件:

- 可能な限りハードコードではなく設定を使う
- 欠損 signal を安全に扱う
- 最終 score を `0.0..1.0` に正規化する
- 説明可能で検査可能である

aggregator は、異なる signal family が部分的に独立した証拠を提供するという前提で設計する。

最低限、次を組み合わせられるべきである。

- detector model 出力
- perplexity / statistical 出力
- 複数の linguistic feature signal

これにより、どれか一つだけに依存するより防御可能な結果になる。

### 8.6 Step 6: Confidence Estimation

confidence は score とは独立に計算する。

考慮要素:

- 入力長の十分性
- 有効な evidence を出した component 数
- 結果に含まれる signal family の多様性
- signal 間の一致 / 不一致
- 適用可能 feature の coverage
- 検出された入力プロファイルに対する既知制約

score が高くても confidence が low であり得る。

### 8.7 Step 7: Classification Mapping

score 範囲を classification label に写像する後処理ステップである。

初期ラベル:

```text
likely_human
possibly_human
uncertain
possibly_ai_generated
likely_ai_generated
```

しきい値は設定値として保持し、評価データセットで検証する。

---

## 9. Component Design

detector model と heuristic analyzer が共存できるよう、汎用 component モデルを用いる。

### 9.1 Component Types

推奨カテゴリ:

- `DetectorComponent`
- `StatisticalComponent`
- `AnalyzerComponent`
- 将来の `ExplainerComponent`

すべて共通の `Signal` 構造を返す。

### 9.2 Design Principles

各 component は次を満たす:

1. 特定の仮説または証拠 source を測る。
2. 平易な言葉で説明可能な出力を返す。
3. 明示的に例外とされない限り、bounded かつ決定論的な scoring function を持つ。
4. 短文、壊れた入力、多言語入力で安全に degrade する。
5. 個別に unit test できる。
6. source family と version metadata を宣言する。

### 9.3 Initial MVP Components

MVP は、ローカル実装可能で説明可能な小さな component 集合から始める。

推奨構成:

- 適切なローカル pretrained model が検証できた場合の dedicated detector-model component 1つ
- 運用上現実的なら perplexity または近縁統計 component 1つ
- いくつかの linguistic analyzer

候補:

- Structure analyzer
- Repetition analyzer
- Vocabulary analyzer
- Style-variation analyzer
- Formatting-pattern analyzer

### 9.4 Candidate MVP Signals

候補:

- `detector_model_score`
- `perplexity_pattern`
- `repetitive_structure`
- `sentence_length_consistency`
- `phrase_repetition`
- `low_vocabulary_diversity`
- `low_stylistic_variation`
- `template_like_formatting`

これらは因果の断定ではなく、測定可能パターンの例として扱う。

### 9.5 Component Registration

component は registry または明示的 composition root を通じて生成する。

これにより:

- 有効 component の制御
- component 組み合わせ実験
- 決定論的な固定順序
- 将来の analysis profile 対応

が可能になる。

### 9.6 Signal Ordering

signal は、寄与度降順、次に stable type 名で決定論的にソートするのが望ましい。

理由:

- 人間向け出力の一貫性
- JSON 契約の安定性
- golden test

---

## 10. Scoring Design

### 10.1 Scoring Philosophy

scorer は、実際以上に数学的強さを装うことなく evidence を結合すべきである。

MVP の目標:

- 安定動作
- 説明可能性
- 較正しやすさ
- ラベル付き評価セットに対する改善のしやすさ

### 10.2 Recommended MVP Approach

weighted aggregation scorer が最も実用的な出発点である。

```text
final_score = normalize(sum(signal.score * signal.weight))
```

この方式は:

- 検査しやすい
- 調整しやすい
- version 管理しやすい
- ローカル決定論実行と両立しやすい

重み設計では、detector model、perplexity、linguistic signals が同質ではないことを明示的に考慮するべきである。

### 10.3 Calibration

将来的な統計較正の余地を残す。

候補:

- threshold tuning
- Platt scaling
- isotonic calibration
- trained meta-classifier への置換

内部実装が変わっても public result contract は維持できるようにする。

### 10.4 Score Versioning

比較可能性のため、result metadata に scoring profile または engine version を含める。

---

## 11. Confidence Design

confidence は score の写しであってはならない。

### 11.1 Proposed Inputs to Confidence

候補:

- 最小入力長ゲート
- 正常実行した component 数
- 有意味な出力を返した component 数
- signal score の分散
- 矛盾 evidence の有無
- 言語サポート確実性

### 11.2 Confidence Levels

MVP の値:

```text
low
medium
high
```

マッピングロジックは明示的かつテスト可能にする。

例:

- `low`: 入力が短すぎる、segmentation 品質が低い、evidence が疎
- `medium`: 入力は十分だが component 間一致が部分的
- `high`: 十分な入力に対し、広い component で整合した evidence が得られた

---

## 12. Configuration Design

インターフェース層を変更せずに制御可能な tuning を許す。

設定可能項目例:

- enabled components
- per-signal weights
- classification thresholds
- minimum input lengths
- language-specific tokenizer behavior
- detector-model selection and local model paths
- perplexity-model selection
- feature-extraction limits for performance protection

configuration は engine release または analysis profile と一緒に version 管理する。

MVP では static / compile-in でもよいが、CLI 層に埋め込まない。

---

## 13. Error Handling

エンジンは domain-relevant error を返し、インターフェース固有表現は持ち込まない。

例:

- Empty input
- Invalid UTF-8 after decoding boundary assumptions
- Unsupported analysis option
- Internal segmentation failure
- Inconsistent configuration

デフォルト出力向け error 値には raw sensitive content を含めない。

---

## 14. Determinism and Reproducibility

決定性は MVP の中核要件である。

同じ:

- 入力テキスト
- normalization rules
- enabled components
- configuration
- engine version

であれば、同じ結果を返すべきである。

再現性のために:

- component 順序を固定する
- 浮動小数処理を安定させる
- MVP のコア経路で randomness を使わない
- metadata で engine / scoring revision を識別する

将来非決定論モデルを導入する場合は、その挙動を説明できる metadata を契約に含める。

---

## 15. Performance Considerations

MVP はローカル CLI ファーストなので、起動コストと予測可能な実行時間が重要である。

### 15.1 Performance Principles

- 共通 feature は一度だけ計算する
- tokenization を重複しない
- メモリ使用量を入力サイズに比例させる
- 可能な限り線形または準線形アルゴリズムを使う
- 極端に大きな入力に対する高コスト処理を制限する

### 15.2 Practical Limits

安全制限例:

- 最大入力文字数
- 繰り返しパターン探索 window 上限
- signal ごとの evidence 保持件数上限

クラッシュするのではなく、穏当に失敗または詳細度を下げるべきである。

---

## 16. Privacy and Data Handling

分析対象コンテンツは機密を含む可能性がある前提で扱う。

したがって MVP エンジンは:

- デフォルトでローカル動作
- 入力内容を永続化しない
- raw analyzed text をログに書かない
- 明示的導入と文書化なしに remote call を行わない
- derived かつ最小限の evidence だけを保持する

将来 component が外部モデルや外部サービスを必要とする場合、それは opt-in のアーキテクチャ変更として明示文書化する。

---

## 17. Testing Strategy

エンジンは複数層でテスト可能であるべきである。

### 17.1 Unit Tests

独立にテストする:

- normalization
- segmentation
- feature extraction
- individual components
- score aggregation
- confidence mapping
- classification mapping

### 17.2 Contract Tests

`AnalysisResult` が downstream consumer に対して schema-stable であることを検証する。

### 17.3 Golden Tests

代表 fixture に対して決定論的 end-to-end 出力を検証する:

- human-written prose
- AI-generated prose
- human-edited AI text
- mixed text
- Markdown-heavy content
- Japanese text

### 17.4 Evaluation Tests

score 変化をリリース間で測るため、versioned evaluation dataset を使う。

追跡項目:

- threshold ごとの precision / recall
- false positive / false negative 傾向
- confidence distribution
- language-specific behavior
- prior engine versions に対する回帰

---

## 18. Extensibility

コアを書き直さず将来成長できる設計にする。

拡張点:

- 新しい detector model
- 新しい statistical component
- 新しい analyzer
- 新しい signal type
- 言語別 segmentation strategy
- 代替 scoring strategy
- 任意の LLM-based explanation
- 構造化 evidence の API 公開
- code、essay、Markdown-heavy document などの analysis profile

内部実装が heuristic aggregation から trained ensemble に移行しても、engine contract は安定に保つ。

---

## 19. Recommended Implementation Shape

正確なディレクトリ構造は言語依存だが、概念的には次のように対応づけられる。

```text
internal/
  engine/
    engine.*
  document/
    builder.*
    normalize.*
    segment.*
  features/
    extractor.*
    structure.*
    repetition.*
    vocabulary.*
    style.*
  analyzers/
    analyzer.*
    structure.*
    repetition.*
    vocabulary.*
    style.*
    formatting.*
  detectors/
    detector.*
    model.*
    perplexity.*
  scoring/
    scorer.*
    confidence.*
    classification.*
  model/
    document.*
    signal.*
    result.*
  config/
    profile.*
```

これは package 名の固定ではなく、ドメイン境界を保つための形である。

---

## 20. Open Technical Decisions

実装計画で決めるべき項目:

1. MVP の日本語 segmentation / tokenizer を何にするか
2. 最初に評価する既存 local pretrained detector model を何にするか
3. perplexity を MVP で運用可能か、可能ならどの local model を使うか
4. 初期リリースに十分信頼できる signal 定義はどれか
5. Markdown 構造を plain text として扱うか、軽量に構造考慮するか
6. JSON にどこまで evidence を出すか
7. confidence を `low` に制限する最小入力長をどうするか

これらは最初の具体実装と一緒に記録するべきである。

---

## 21. Summary

Signal 分析エンジンは、複数種の evidence を集約して次を返す、決定論的で説明可能かつインターフェース非依存のコアであるべきである。

- AI-likelihood score
- confidence level
- classification
- contributing signals
- stable structured metadata

Signal は単一 detector ではなく、次を組み合わせるシステムとして扱う。

- dedicated AI-detection model output
- perplexity などの statistical evidence
- linguistic / stylistic analyzers
- 将来の optional LLM-based explanation

MVP では、早すぎる複雑化よりも、少数で強く説明可能な component を優先するべきである。エンジン境界をきれいに保てれば、CLI ファーストのツールから API 中心の分析プラットフォームへ拡張できる。
