# Signal — プロダクト要件定義書

## 1. 概要

### 1.1 プロダクト名

**Signal**

### 1.2 コンセプト

Signal はコンテンツを分析し、それが AI によって生成された可能性を示すシグナルを検出する。

Signal は単純に「AI生成」または「人間生成」と断定してはならない。代わりに、AI生成らしさのスコア、信頼度、その結果に寄与したシグナルを返す。

> **Signal — AI生成コンテンツのシグナルを可視化する。**

### 1.3 背景

生成AIの普及により、人間が作成したコンテンツとAIが生成したコンテンツを見分けることはますます難しくなっている。

既存の AI コンテンツ検出器には、主に次のような課題がある。

- なぜその結果になったのかを説明せず、スコアだけを返す。
- 技術的には不可能な確実性を示唆してしまう。
- Web UI 中心で提供されることが多い。
- 開発者ワークフローや他システムへの組み込みが難しい。

Signal はまず開発者向け CLI として提供し、その後 API を提供する。API は MCP、ブラウザ拡張、その他の統合機能の基盤となる。

---

## 2. プロダクト原則

### 2.1 断定ではなく分析

Signal は AI 生成検出を絶対的な証明として扱ってはならない。

結果には次を含めるべきである。

- AI生成らしさ
- 分類
- 信頼度
- 結果に寄与したシグナル

例:

```text
AI likelihood: 87%
Confidence: High

Signals:
- Repetitive sentence structure
- Consistent sentence length
- AI-like phrase patterns
- Low stylistic variation
```

### 2.2 結果を説明する

Signal の価値は単一のスコアに限定されるべきではない。

分析に寄与したシグナルを示すことで、システムは **なぜ** その結果になったのかを説明できるべきである。

### 2.3 API中心アーキテクチャ

分析エンジンは UI から独立していなければならない。

想定アーキテクチャ:

```text
                    ┌──────────────┐
                    │Signal Engine │
                    └──────┬───────┘
                           │
                 ┌─────────┴─────────┐
                 │                   │
             Signal CLI          Signal API
                                     │
                         ┌───────────┴───────────┐
                         │                       │
                    MCP Server            Browser Extension
```

CLI と API は同じ分析エンジンを利用し、検出ロジックが分岐しないようにする。

---

## 3. 対象ユーザー

### 3.1 初期ユーザー

主な対象は次のような開発者である。

- AI生成されたテキストやコードを調べたい開発者
- AIエージェントが生成したコンテンツを調べたい開発者
- GitHub 上のコンテンツを分析したい開発者
- CI/CD で AI コンテンツチェックを実行したい開発者

### 3.2 将来のユーザー

API 提供後は、以下のようなシステムをサポートする。

- AIエージェント
- MCPクライアント
- ブラウザ拡張
- CMSプラットフォーム
- 教育サービス
- コンテンツ管理システム
- エンタープライズ向けコンテンツ管理ツール
- AI利用ポリシーを適用するシステム

---

## 4. MVP スコープ

最初の MVP は **CLI を通じたテキストコンテンツの AI生成らしさ分析** に集中する。

### Inputs

MVP でサポートする入力:

- 標準入力
- テキストファイル
- Markdown ファイル

URL 入力は初期 MVP 後に追加してよい。

### Outputs

MVP が提供すべき出力:

- AI生成らしさスコア
- 分類
- 信頼度
- 結果に寄与したシグナル
- 入力文字数
- 入力語数
- JSON 出力

---

## 5. CLI 要件

### 5.1 基本コマンド

```bash
signal <input>
```

例:

```bash
signal article.md
```

標準入力もサポートする:

```bash
cat article.md | signal
```

URL サポートは後続フェーズで予定する:

```bash
signal https://example.com/article
```

### 5.2 人間向け出力

デフォルト出力は開発者向けに設計する。

例:

```text
Signal

AI likelihood    87%
Confidence       High

Signals
  ✓ Repetitive sentence structure
  ✓ Consistent sentence length
  ✓ AI-like phrase patterns
  ✓ Low stylistic variation

Analyzed
  4,821 characters
  823 words
```

### 5.3 JSON 出力

機械可読な JSON を提供しなければならない。

```bash
signal article.md --json
```

例:

```json
{
  "score": 0.87,
  "confidence": "high",
  "classification": "likely_ai_generated",
  "signals": [
    {
      "type": "repetitive_structure",
      "score": 0.82
    },
    {
      "type": "phrase_pattern",
      "score": 0.76
    }
  ],
  "input": {
    "characters": 4821,
    "words": 823
  }
}
```

CLI の JSON 形式は、可能な限り将来の API レスポンス形式と互換にする。

### 5.4 終了コード

CLI は CI/CD の用途をサポートするべきである。

初期案:

```text
0 = 分析成功
1 = AI生成らしさが設定しきい値以上
2 = 不正な入力
3 = 分析エラー
```

しきい値は設定可能にする:

```bash
signal article.md --threshold 0.8
```

---

## 6. 分析エンジン

### 6.1 分析粒度

エンジンは文書単位の分析を基本としつつ、将来的な細粒度分析にも対応できるよう設計する。

```text
Document
 ├── Paragraph
 │    ├── Sentence
 │    └── Sentence
 └── Paragraph
```

これにより、将来的には人間が書いた文書の一部にだけ AI らしい区間がある場合の検出にもつながる。

### 6.2 シグナル

分析エンジンは複数の独立した特徴を検出するべきである。

候補:

- 文構造
- 文長のばらつき
- 語彙多様性
- 表現の繰り返し
- テキスト構造
- フレーズパターン
- 不自然な一貫性
- AI に関連しやすい言語パターン

可能な限り、各シグナルは独立した analyzer として実装する。

```text
Analyzer
 ├── StructureAnalyzer
 ├── VocabularyAnalyzer
 ├── PhraseAnalyzer
 ├── StyleAnalyzer
 └── ...
```

アーキテクチャは、新しい analyzer を簡単に追加できるようにする。

### 6.3 総合スコア

総合的な AI 生成らしさは複数シグナルから導出する。

```text
AI likelihood = f(signal1, signal2, signal3, ...)
```

正確なスコアリングアルゴリズムは、実験と評価を通じて決定する。

---

## 7. 分類

Signal は単純な二値分類を避けるべきである。

初期分類:

```text
likely_human
possibly_human
uncertain
possibly_ai_generated
likely_ai_generated
```

結果には次も含める:

```text
score
confidence
```

score と confidence は独立した概念でなければならない。

例:

```text
AI likelihood: 85%
Confidence: Low
```

これは、観測されたシグナルは AI らしさを示しているが、入力が短すぎるなどの理由で高い確信を持てない場合に有効である。

---

## 8. 入力制約

非常に短いコンテンツでは信頼できる結果が出ない場合がある。

システムは入力サイズに応じて信頼度を調整すべきである。

初期しきい値例:

```text
< 100 words
  → insufficient_data

100–500 words
  → low confidence

500–1,000 words
  → medium confidence

> 1,000 words
  → normal analysis
```

最終しきい値は評価データに基づいて決める。

---

## 9. API

第2フェーズでは Signal を API として公開する。

### 9.1 エンドポイント

初期エンドポイント:

```http
POST /v1/analyze
```

Request:

```json
{
  "content": "..."
}
```

Response:

```json
{
  "score": 0.87,
  "confidence": "high",
  "classification": "likely_ai_generated",
  "signals": []
}
```

### 9.2 API 設計

API は CLI とは別実装の分析系ではない。

CLI と同じ分析エンジンを使う別インターフェースである。

```text
                 ┌───────────────┐
                 │ Analysis Core │
                 └───────┬───────┘
                         │
              ┌──────────┴──────────┐
              │                     │
             CLI                   API
```

### 9.3 認証

初期認証方式は API キーとする。

将来的には以下もサポートできるようにする。

- API keys
- OAuth
- Service accounts

### 9.4 API 制御

API は以下を管理できるべきである。

- リクエスト制限
- 入力サイズ制限
- 利用量
- レート制限
- API キー単位の利用量

---

## 10. MCP Server

API 提供後は Signal API を利用する MCP Server を提供する。

目的は AI エージェントが Signal を通じてコンテンツ分析できるようにすることである。

初期ツール:

```text
analyze_content
```

Input:

```json
{
  "content": "..."
}
```

Output:

```json
{
  "classification": "likely_ai_generated",
  "score": 0.87,
  "confidence": "high",
  "signals": []
}
```

概念アーキテクチャ:

```text
AI Agent
    │
    │ analyze content
    ▼
Signal MCP Server
    │
    ▼
Signal API
    │
    ▼
Analysis Engine
```

将来のツール候補:

```text
analyze_url
analyze_document
compare_content
explain_result
```

---

## 11. Browser Extension

ブラウザ拡張は、Web ページ上のコンテンツを Signal で分析できるようにする。

初期フロー:

```text
Web page
   ↓
Select content
   ↓
Signal Extension
   ↓
Signal API
   ↓
Analysis Result
```

初期版では、ページ全体の自動分析より選択テキスト分析を優先する。

例:

```text
AI likelihood: 82%

Confidence: Medium

3 signals detected
```

---

## 12. API中心エコシステム

長期的には、Signal は CLI を超えて AI 時代のコンテンツ分析プラットフォームとして発展するべきである。

```text
                         Signal API
                             │
      ┌──────────────┬───────┼──────────────┬──────────────┐
      │              │       │              │              │
   Signal CLI    MCP Server  Browser Ext  GitHub Action   SDK
```

各インターフェースは共通の分析エンジンまたはその API を利用する。

---

## 13. OSS 戦略

Signal CLI はオープンソースとするべきである。

利点:

- 透明性
- 検証可能性
- 開発者採用のしやすさ
- 研究的な改善のしやすさ

一方で、学習済みモデル、評価データセット、商用 API などの一部コンポーネントは、将来別配布や別ライセンスとなる可能性がある。

OSS 戦略は次を明確にすべきである。

- 何が OSS か
- 何が将来ホスト提供の対象になり得るか
- 何が評価専用データか

---

## 14. 非機能要件

### 14.1 再現性

同じ入力と同じエンジンバージョンであれば、同じ結果を返すべきである。

### 14.2 性能

CLI として十分に高速な起動と実行時間を目指す。

### 14.3 プライバシー

Signal に送られるコンテンツには機密情報が含まれる可能性がある。

したがって:

- 分析対象コンテンツをログに書かない
- 明示的な仕様なしに外部送信しない
- 一時ファイルやキャッシュに勝手に保存しない

### 14.4 可観測性

将来的な API では以下の可観測性が必要になる。

- リクエスト数
- レイテンシ
- エラー率
- モデル/エンジンバージョン別利用状況

MVP CLI では過剰なテレメトリは不要である。

---

## 15. 精度評価

Signal は評価データセットを用いて継続的に評価されるべきである。

評価データセットには少なくとも以下を含める:

- human
- ai_generated
- human_edited_ai
- ai_edited_human
- mixed

特に `human_edited_ai` は重要である。実運用では、AI が生成したコンテンツが人間によって編集されることが多いためである。

評価プロセスでは以下を追跡する:

- threshold ごとの precision / recall
- false positive / false negative
- 言語別の挙動
- エンジン変更による回帰

評価データセットと評価手順はバージョン管理する。

---

## 16. 重要な制約

Signal は、コンテンツが AI によって生成されたこと、あるいはそうでないことを絶対的に証明しない。

> **Signal does not prove that content was generated by AI. Results are probabilistic estimates based on detected signals.**

Signal は、教育、採用、法務、規制など高インパクトな判断において、結果を唯一の根拠として使うことを明示的に避けるべきである。

---

## 17. 開発フェーズ

### Phase 1 — Signal CLI

CLI によって分析エンジンを構築・検証する。

```text
Input
  ↓
Signal CLI
  ↓
Signal Analysis Engine
  ↓
Score / Confidence / Classification / Signals
```

目的:

- 最小限で有用な分析実装を作る
- 使いやすい CLI を作る
- 評価データセットで精度と限界を把握する

### Phase 2 — Signal API

検証済みの分析エンジンを API として公開する。

```text
Signal CLI ──────┐
                 ├── Signal Analysis Engine
Signal API ──────┘
```

### Phase 3 — MCP Server

Signal を MCP として公開する。

```text
AI Agent
  ↓
Signal MCP Server
  ↓
Signal API
```

### Phase 4 — Browser Extension

ブラウザから直接分析できるようにする。

### Phase 5 — Ecosystem

GitHub Action、SDK、他システム統合へ拡張する。

---

## 18. MVP 成功条件

### Technical

- ファイル入力と標準入力を処理できる
- スコア、分類、信頼度、シグナルを返せる
- JSON 出力が安定している
- 評価データセットで継続測定できる

### User Experience

- 開発者にとって使いやすい CLI である
- エラーメッセージが明確である
- CI で使える終了コードがある

### Explainability

- 単一スコアだけでなくシグナルを提示できる
- 結果の理由をユーザーが理解できる

### Automation

- 他ツールから JSON と終了コードで利用できる

### Architecture

- 分析エンジンが独立して呼び出せる
- 将来の API 再利用に耐える結果モデルになっている

---

## 19. 長期的方向性

Signal は単なる「AI detector」ではなく、AI時代のコンテンツ分析プラットフォームへ進化するべきである。

```text
                   ┌──────────────┐
                   │   Signal     │
                   │ Analysis Core│
                   └──────┬───────┘
                          │
                        Signal API
                          │
      ┌───────────┬───────────┬───────────┬────────────┬───────────┐
      │           │           │           │            │           │
  Signal CLI   Signal MCP  Browser Ext  GitHub Action   SDK     Future Apps
```

長期的なメッセージは次のとおりである。

> **Make the signals behind AI-generated content visible.**
