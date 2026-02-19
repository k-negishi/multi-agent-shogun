---
name: shogun-model-list
description: >
  全AI CLIツール×利用可能モデル×必要サブスクリプション×Bloom最大能力のリファレンス表。
  multi-agent-shogunでどのモデルを使うか選択するためのリファレンステーブル。
  トリガー: "model list", "what models", "model comparison", "which models can I use",
  "モデル一覧", "モデル比較", "どのモデルが使える"
---

# /shogun-model-list — モデル機能リファレンス

## 概要

全AI CLIツール、モデル、必要なサブスクリプション、
モデルごとの最大Bloomレベルの完全なリファレンステーブルを表示する。
`config/settings.yaml`で`capability_tiers`を設定する前に使用。

## 使用タイミング

- "自分のサブスクリプションでどのモデルが使える？"
- "どのモデルがL5タスクを処理できる？"
- "ClaudeとCodexのモデル階層を比較したい"
- "全モデルを見せて" / "モデル一覧"
- `/shogun-bloom-config`を実行する前に全体像を把握したい時

## 指示

以下のリファレンステーブルをユーザーに直接出力する。ツール呼び出しは不要。

---

## Bloomの分類学 — クイックリファレンス

| レベル | カテゴリ | タスク例 |
|-------|----------|---------------|
| L1 | 記憶 (Remember) | ファイルコピー、テンプレート適用、データフォーマット |
| L2 | 理解 (Understand) | 要約、説明、翻訳 |
| L3 | 適用 (Apply) | 既知パターンの実装、ボイラープレート生成 |
| L4 | 分析 (Analyze) | デバッグ、コードレビュー、根本原因分析 |
| L5 | 評価 (Evaluate) | アーキテクチャレビュー、設計トレードオフ判断 |
| L6 | 創造 (Create) | 新規アーキテクチャ、要件設計、戦略 |

---

## Claude Code (Anthropic)

### サブスクリプションプラン

| プラン | 月額 | Opus 4.6 | Sonnet 4.6 | Haiku 4.5 | Extended Thinking |
|------|---------|----------|------------|-----------|-------------------|
| Free | $0 | ✗ | ✓ | ✓ | ✗ |
| Pro | $20 | ✓ | ✓ | ✓ | ✓ |
| Max 5x | $100 | ✓ | ✓ | ✓ | ✓ |
| Max 20x | $200 | ✓ | ✓ | ✓ | ✓ |

> Pro/Max 5x/Max 20xはモデルアクセスは同じ。違いは使用量クォータ（5x/20x = Proの倍率）。

### Claudeモデル × Bloom能力

| モデル | Bloom最大 | 最適用途 | 備考 |
|-------|-----------|----------|-------|
| `claude-haiku-4-5-20251001` | **L3** | 大量L1-L3タスク、高速応答 | $1/$5/M; SWE-bench 73.3% (Sonnet 4.5比4pp低い); extended thinking利用可 |
| `claude-sonnet-4-6` | **L5** | コードレビュー、分析、オーケストレーション | 最高のバランス — $3/$15/M; SWE-bench 79.6%, 1Mコンテキスト |
| `claude-opus-4-6` | **L6** | 新規設計、戦略、アーキテクチャ | $5/$25/M; SWE-bench 80.8% (Sonnet 4.6比わずか1.2pp上); 真のL6のみに使用 |

> **Extended Thinking** (Pro以上で利用可): 複雑な推論タスクで実質的に約1 Bloomレベル分の能力向上。

### 固定エージェント割り当て（推奨）

| エージェント | 推奨モデル | Bloom用途 | 理由 |
|-------|------------------|-----------|--------|
| Shogun (あなた) | `claude-opus-4-6` | L6 | 戦略的決定、最終レビュー |
| Karo (家老) | `claude-sonnet-4-6` | L4-L5 | タスクオーケストレーション; Opusは過剰 |
| Gunshi (軍師) | `claude-opus-4-6` | L5-L6 | 深いQC、アーキテクチャ評価 |
| Ashigaru 1–7 | `capability_tiers`経由で設定 | L1-L3 | 作業者 — Bloomレベルでルーティング |

---

## OpenAI Codex CLI

### サブスクリプションプラン

| プラン | 月額 | Spark | gpt-5.3-codex | codex-mini | codex-max |
|------|---------|-------|---------------|------------|-----------|
| Free / Go ($8) | $0–$8 | ✗ | ✗ (制限あり) | ✗ | ✗ |
| Plus | $20 | ✗ (**Proのみ**) | ✓ | ✓ | ✓ |
| Pro | $200 | ✓ | ✓ | ✓ | ✓ |

> **gpt-5.3-codex-spark は ChatGPT Pro ($200) が必要。** ChatGPT Plus ($20) には Spark は含まれない。

### Codexモデル × Bloom能力

| モデル | Bloom最大 | 最適用途 | 備考 |
|-------|-----------|----------|-------|
| `gpt-5.3-codex-spark` | **L3** | 1000+ tok/secでの大量L1-L3タスク | gpt-5.3-codexとは独立クォータ; 超高速 |
| `gpt-5-codex-mini` | **L2** | 軽量タスクの最小クォータ消費 | Sparkの軽量代替 |
| `gpt-5.3-codex` | **L4** | 分析、デバッグ、コードレビュー | 標準的な主力モデル |
| `gpt-5.1-codex-max` | **L5** | 複雑な分析、設計評価 | 最高Codex能力 |

> **L6ギャップ**: どのCodexモデルも新規創造設計（L6）を確実に処理できない。L6タスクにはClaude Opus推奨。

---

## 能力サマリー（全モデル、CLI横断）

| モデル | CLI | Bloom最大 | 最小サブスク | 備考 |
|-------|-----|-----------|-----------------|-------|
| `gpt-5-codex-mini` | Codex CLI | L2 | ChatGPT Plus | 軽量、最小クォータ |
| `claude-haiku-4-5-20251001` | Claude Code | **L3** | Claude Free | 最高のClaude コスト効率; SWE-bench 73.3% |
| `gpt-5.3-codex-spark` | Codex CLI | L3 | **ChatGPT Pro** | 1000+ tok/s; Terminal-Bench 58.4% |
| `gpt-5.3-codex` | Codex CLI | L4 | ChatGPT Plus | Terminal-Bench 77.3%; 400K+ context |
| `claude-sonnet-4-6` | Claude Code | L5 | Claude Free | $3/$15/M; SWE-bench 79.6%; 1M context; 数学+27pt vs Sonnet 4.5 |
| `gpt-5.1-codex-max` | Codex CLI | L5 | ChatGPT Plus | 最高Codex能力 |
| `claude-opus-4-6` | Claude Code | L6 | Claude Pro | $5/$25/M; SWE-bench 80.8%; 真のL6タスク専用 |

---

## 次のステップ

あなたのサブスクリプション用のready-to-paste `capability_tiers` YAMLを生成するには:

```
/shogun-bloom-config
```

または将軍に: "自分のサブスクリプション用にcapability_tiersを設定して"
