# Signal CLI — MVP Specification

## 1. Purpose

This document defines the implementation-level specification for the first version of the Signal CLI.

The goal is to provide a small, usable, testable CLI that can analyze text and return an AI-generation likelihood, confidence, classification, and contributing signals.

The implementation should establish a clean foundation for the future Signal API.

---

## 2. MVP Goals

The MVP must:

1. Accept text from a file or standard input.
2. Analyze the content.
3. Produce an AI-generation likelihood score.
4. Produce a confidence level.
5. Produce a classification.
6. Produce human-readable output.
7. Produce machine-readable JSON output.
8. Return meaningful exit codes.
9. Be usable in shell scripts and CI.
10. Keep the analysis engine independent from the CLI layer.
11. Include an evaluation/test dataset.
12. Make the analysis result schema reusable by the future API.

The MVP does **not** need:

- A web UI.
- A hosted API.
- Authentication.
- MCP.
- Browser extensions.
- URL fetching.
- PDF/DOCX parsing.
- Image/audio/video analysis.
- User accounts.
- Persistent storage.

---

## 3. User Stories

### 3.1 Analyze a file

As a developer, I want to analyze a text file so that I can determine whether it contains signals associated with AI-generated content.

```bash
signal article.md
```

### 3.2 Analyze standard input

As a developer, I want to pipe content into Signal so that I can integrate it into shell workflows.

```bash
cat article.md | signal
```

### 3.3 Consume JSON

As a developer, I want machine-readable output so that I can integrate Signal with other tools.

```bash
signal article.md --json
```

### 3.4 Use Signal in CI

As a developer, I want Signal to return meaningful exit codes so that I can use it as a quality gate.

```bash
signal article.md --threshold 0.8
```

### 3.5 Understand the result

As a developer, I want to know why Signal produced a result rather than receiving only a single score.

---

## 4. CLI Interface

## 4.1 Command

```bash
signal <input>
```

Where `<input>` is:

- A local file path
- `-` for standard input
- Omitted when stdin is available

Examples:

```bash
signal article.md
```

```bash
signal -
```

```bash
cat article.md | signal
```

If both an explicit file and stdin are available, the explicit file takes precedence.

---

## 4.2 Options

Initial options:

```text
--json
--threshold <float>
--quiet
--version
--help
```

### `--json`

Returns machine-readable JSON.

```bash
signal article.md --json
```

### `--threshold`

Sets the AI-likelihood threshold used by the exit code.

```bash
signal article.md --threshold 0.8
```

Default:

```text
0.8
```

### `--quiet`

Suppresses human-readable analysis output.

This is useful when only the exit code matters.

```bash
signal article.md --quiet --threshold 0.8
```

### `--version`

Displays the CLI version.

### `--help`

Displays command usage.

---

## 5. Input Handling

### 5.1 Supported Formats

MVP:

- Plain text
- Markdown

The implementation should treat Markdown as text for analysis.

No Markdown rendering is required.

### 5.2 Encoding

UTF-8 is the required encoding.

If input is not valid UTF-8, the CLI should return an input error.

### 5.3 Empty Input

Empty input is invalid.

The CLI should return exit code `2`.

### 5.4 Minimum Input

The engine should accept short inputs but indicate insufficient data where appropriate.

The CLI should not silently classify insufficient input as human or AI.

---

## 6. Result Model

The analysis engine should expose a language-independent result model.

Conceptual schema:

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

The exact schema should be defined as a versioned internal contract.

---

## 7. Score

`score` represents the estimated likelihood that the analyzed content is AI-generated.

Type:

```text
float
```

Range:

```text
0.0 <= score <= 1.0
```

Interpretation:

```text
0.0 = very low AI-generation likelihood
1.0 = very high AI-generation likelihood
```

The score must not be presented as a probability unless the underlying algorithm has been statistically calibrated as such.

Therefore, user-facing wording should initially be:

```text
AI likelihood: 87%
```

rather than:

```text
87% probability
```

unless calibration supports that interpretation.

---

## 8. Confidence

Allowed values:

```text
low
medium
high
```

Future values may be added only through an explicit schema change.

Confidence should reflect the reliability of the analysis, considering factors such as:

- Input length
- Number of available signals
- Agreement between analyzers
- Quality of the input
- Other model-specific constraints

Score and confidence must remain independent.

---

## 9. Classification

Initial values:

```text
likely_human
possibly_human
uncertain
possibly_ai_generated
likely_ai_generated
```

A recommended initial mapping is:

```text
0.00–0.19 → likely_human
0.20–0.39 → possibly_human
0.40–0.59 → uncertain
0.60–0.79 → possibly_ai_generated
0.80–1.00 → likely_ai_generated
```

These thresholds are provisional.

They must be treated as configuration and validated against the evaluation dataset before being considered final.

---

## 10. Signal Model

Each analyzer produces zero or more signals.

Conceptual interface:

```text
Analyzer.analyze(document) -> []Signal
```

A Signal contains:

```text
type
score
description
```

Example:

```json
{
  "type": "repetitive_structure",
  "score": 0.82,
  "description": "Sentence structures show unusually low variation."
}
```

### Initial Signal Types

The MVP should start with a small number of explainable signals.

Candidate types:

```text
repetitive_structure
sentence_length_consistency
vocabulary_diversity
phrase_repetition
stylistic_variation
```

The final list should be determined during implementation based on what can be measured reliably.

Do not add signals merely to increase the number of signals.

Every signal should have:

1. A clear definition.
2. A deterministic calculation where possible.
3. Unit tests.
4. Documentation.
5. An explanation suitable for CLI output.

---

## 11. Analysis Engine Architecture

The CLI should be a thin adapter around the analysis engine.

Recommended conceptual architecture:

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

The CLI must not implement scoring logic directly.

---

## 12. Suggested Project Structure

The exact structure depends on the implementation language, but the conceptual separation should be preserved.

Example:

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

If the chosen language has different idioms, adapt the structure while preserving the separation of concerns.

---

## 13. Human-Readable Output

Default output:

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

The exact wording can evolve.

The output should prioritize:

1. Result
2. Confidence
3. Signals
4. Input statistics

It should remain concise enough for terminal use.

---

## 14. JSON Output

`--json` must output JSON only.

No banners, progress indicators, logs, or diagnostic messages should be written to stdout in JSON mode.

Example:

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

Errors should be written to stderr.

---

## 15. Exit Codes

Initial specification:

```text
0 = Analysis completed and score is below threshold
1 = Analysis completed and score is at or above threshold
2 = Invalid input
3 = Analysis failure
```

This makes the CLI usable as a CI quality gate.

Example:

```bash
signal README.md --threshold 0.8
```

A score of `0.82` returns:

```text
exit 1
```

A score of `0.65` returns:

```text
exit 0
```

The exit code must not imply that the classification is scientifically certain.

---

## 16. Threshold Handling

Default threshold:

```text
0.8
```

The threshold must:

- Be between `0.0` and `1.0`.
- Reject invalid values.
- Not change the underlying analysis score.
- Affect only threshold-based CLI behavior.

For example:

```bash
signal article.md --threshold 0.9
```

must return the same analysis score as:

```bash
signal article.md --threshold 0.8
```

Only the exit code may change.

---

## 17. Insufficient Data

If the content is too short for reliable analysis, the engine should return an explicit result.

Example:

```json
{
  "score": 0.51,
  "confidence": "low",
  "classification": "uncertain",
  "signals": [],
  "input": {
    "characters": 120,
    "words": 23
  }
}
```

Do not fabricate confidence.

The CLI should communicate the limitation clearly.

---

## 18. Privacy

The MVP should not persist analyzed content.

Requirements:

- Do not write input content to logs.
- Do not include content in error messages.
- Do not create hidden cache files containing analyzed content.
- Do not send content to external services without explicit implementation and documentation.
- Document data handling in the README.

If the analysis implementation requires an external service or model, this must be explicitly documented.

---

## 19. Determinism

The MVP should produce deterministic results when the same input and same analysis-engine version are used.

If a non-deterministic external model is introduced, the architecture must expose the model/version information and document the effect on reproducibility.

---

## 20. Testing Strategy

Testing is a first-class requirement.

### 20.1 Unit Tests

Each analyzer must have unit tests.

Test:

- Normal inputs
- Edge cases
- Empty input
- Very short input
- Long input
- Unicode
- Japanese text
- Markdown
- Repeated text
- Highly structured text

### 20.2 Integration Tests

Test the complete CLI flow:

```text
file → input reader → engine → result → formatter → exit code
```

### 20.3 JSON Contract Tests

The JSON output should be validated against the result schema.

### 20.4 Golden Tests

Where appropriate, maintain expected outputs for representative fixtures.

### 20.5 Evaluation Tests

Maintain a versioned evaluation dataset containing:

```text
human
ai_generated
human_edited_ai
ai_edited_human
mixed
```

Evaluation results should be reproducible.

---

## 21. Japanese Language Support

Japanese text should be treated as a first-class input language.

The MVP should not assume whitespace-separated words.

Therefore, word-count logic must support languages where words are not separated by spaces.

For Japanese input, the implementation should use an appropriate tokenizer or clearly define an alternative measurement.

The analysis engine should not be limited to English-only assumptions.

---

## 22. Error Handling

Errors must be actionable.

Examples:

```text
Error: file not found: article.md
```

```text
Error: input is empty
```

```text
Error: invalid threshold: 1.5
```

```text
Error: input must be valid UTF-8
```

Do not expose stack traces by default.

A debug mode may be added later.

---

## 23. CLI UX Requirements

The CLI should feel like a normal developer tool.

Requirements:

- Fast startup.
- Clear output.
- Useful errors.
- Unix pipeline support.
- Stable exit codes.
- Stable JSON format.
- No unnecessary telemetry.
- No interactive prompts during normal operation.

---

## 24. Versioning

The CLI should expose a version:

```bash
signal --version
```

Example:

```text
signal 0.1.0
```

The result should include the analysis-engine version:

```json
{
  "engine": {
    "version": "0.1.0"
  }
}
```

Changes to the result schema should follow explicit versioning rules.

---

## 25. API Compatibility Requirement

Although the API is not part of the MVP implementation, the analysis result must be designed with future API use in mind.

The following objects should not depend on terminal formatting:

```text
AnalysisResult
Signal
Classification
Confidence
InputMetadata
EngineMetadata
```

The CLI formatter should convert these objects into terminal output.

This makes the future architecture:

```text
Analysis Engine
      │
      ├── CLI formatter
      │
      └── API serializer
```

rather than:

```text
CLI
 └── analysis logic
```

---

## 26. Documentation Requirements

The repository must contain:

### README

Include:

- What Signal is.
- Limitations.
- Installation.
- Basic usage.
- JSON usage.
- CI example.
- Example output.
- Development instructions.

### CONTRIBUTING

Explain:

- Local development.
- Running tests.
- Adding an analyzer.
- Adding evaluation data.
- Code style.

### Architecture Documentation

Document:

- Analysis engine
- Analyzer interface
- Scoring
- Result model
- CLI architecture

### Evaluation Documentation

Document:

- Evaluation dataset
- Metrics
- Evaluation procedure
- Current benchmark results

---

## 27. CI Requirements

The repository should run automated checks on every pull request.

Minimum checks:

```text
Build
Unit tests
Integration tests
Lint
Format check
JSON schema validation
```

Evaluation benchmarks may run separately if they are expensive.

---

## 28. Security Requirements

The CLI should treat input as untrusted content.

Requirements:

- Do not execute input.
- Do not interpret embedded shell commands.
- Do not follow URLs in MVP.
- Do not load arbitrary local resources referenced from Markdown.
- Do not write files unless explicitly requested.
- Avoid path traversal vulnerabilities in any future file-output feature.

---

## 29. Out of Scope for MVP

The following are explicitly out of scope:

```text
Web UI
Hosted API
Authentication
Accounts
Billing
MCP
Browser extension
URL crawling
PDF parsing
DOCX parsing
Image analysis
Audio analysis
Video analysis
AI model attribution
Content provenance
Persistent storage
```

They should not complicate the initial architecture.

---

## 30. Definition of Done

The MVP is complete when all of the following are true:

- `signal file.md` works.
- `cat file.md | signal` works.
- `signal file.md --json` produces valid JSON.
- `--threshold` changes threshold-based exit behavior.
- Exit codes are stable and documented.
- Empty and invalid input are handled correctly.
- Japanese text is supported.
- Analysis logic is independent from CLI presentation.
- Each analyzer has unit tests.
- CLI integration tests exist.
- Evaluation fixtures exist.
- README contains installation and usage instructions.
- CI runs the required automated checks.
- No analyzed content is persisted or logged by default.
- The result model is suitable for reuse by the future Signal API.

---

## 31. Implementation Principle

Keep the first version small.

The primary objective of the MVP is not to build the most sophisticated AI detector.

The primary objective is to establish a reliable foundation:

```text
                 Signal
                    │
              Analysis Engine
                    │
              ┌─────┴─────┐
              │           │
             CLI       Result Model
                          │
                          ▼
                     Future API
```

Build the smallest useful implementation, measure its accuracy, and improve the analysis engine based on evidence.

Do not add architectural complexity before there is a concrete requirement for it.
