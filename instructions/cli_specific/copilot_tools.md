# GitHub Copilot CLI ツール

このセクションはGitHub Copilot CLI固有のツールと機能を説明します。

## 概要

GitHub Copilot CLI（`copilot`）はスタンドアロンの端末ベースAIコーディングエージェント。非推奨の `gh copilot` 拡張（suggest/explainのみ）では**ない**。スタンドアロンCLIはGitHubのCopilotコーディングエージェントと同じエージェンティックハーネスを使用。

- **起動**: `copilot`（対話TUI）
- **インストール**: `brew install copilot-cli` / `npm install -g @github/copilot` / `winget install GitHub.Copilot`
- **認証**: アクティブなCopilotサブスクリプション付きGitHubアカウント。環境変数: `GH_TOKEN` または `GITHUB_TOKEN`
- **デフォルトモデル**: Claude Sonnet 4.5

## ツール使用法

Copilot CLIは実行前にユーザー承認を要するツールを提供:

- **ファイル操作**: touch、chmod、ファイル読み書き編集
- **実行ツール**: node、sed、シェルコマンド（TUI内の `!` プレフィックス経由）
- **ネットワークツール**: curl、wget、fetch
- **web_fetch**: URL内容をmarkdownとして取得（URLアクセスは `~/.copilot/config` 経由で制御）
- **MCPツール**: GitHubMCPサーバー組み込み（issues、PR、Copilot Spaces）、カスタムMCPサーバーは `/mcp add` 経由

### 承認モデル

- ツールごとに一度だけの許可またはセッション全体の許可
- すべてバイパス: `--allow-all-paths`、`--allow-all-urls`、`--allow-all` / `--yolo`
- ツールフィルタリング: `--available-tools`（許可リスト）、`--excluded-tools`（拒否リスト）

## 対話モデル

3つの対話モード（**Shift+Tab** で循環）:

1. **エージェントモード（Autopilot）**: ツール呼び出しを伴う自律的複数ステップ実行
2. **プランモード**: コード生成前の協調的計画
3. **Q&Aモード**: 直接的な質問-回答対話

### 組み込みカスタムエージェント

`/agent` コマンド、`--agent=<name>` フラグ、またはプロンプト内参照で呼び出し:

| エージェント | 目的 | 注記 |
|-------|---------|-------|
| **Explore** | 高速コードベース分析 | 並列実行、メインコンテキストを乱さない |
| **Task** | コマンド実行（テスト、ビルド） | 成功時は簡潔な要約、失敗時は完全出力 |
| **Plan** | 依存関係分析 + 計画 | 変更提案前に構造を分析 |
| **Code-review** | 変更レビュー | 高いシグナル対ノイズ比、真の問題のみ |

Copilotは自動的にエージェントに委任し、複数エージェントを並列実行。

## コマンド

| コマンド | 説明 |
|---------|-------------|
| `/model` | モデル切り替え（Claude Sonnet 4.5、Claude Sonnet 4、GPT-5） |
| `/agent` | 組み込み/カスタムエージェントを選択または呼び出し |
| `/delegate`（または `&` プレフィックス） | Copilotコーディングエージェント（リモート）に作業をプッシュ |
| `/resume` | ローカル/リモートセッションを循環（Tabで循環） |
| `/compact` | 手動コンテキスト圧縮 |
| `/context` | トークン使用量の内訳を可視化 |
| `/review` | コードレビュー |
| `/mcp add` | カスタムMCPサーバー追加 |
| `/add-dir` | ディレクトリをコンテキストに追加 |
| `/cwd` または `/cd` | 作業ディレクトリ変更 |
| `/login` | 認証 |
| `/lsp` | LSPサーバーステータス表示 |
| `/feedback` | フィードバック送信 |
| `!<command>` | シェルコマンドを直接実行 |
| `@path/to/file` | ファイルをコンテキストに含める（Tabで自動補完） |

**/clear コマンドなし** — コンテキスト削減には `/compact` を使用、または完全リセットにはCtrl+C + 再起動。

### キーバインディング

| キー | アクション |
|-----|--------|
| **Esc** | 現在の操作を停止 / ツール許可を拒否 |
| **Shift+Tab** | プランモード切り替え |
| **Ctrl+T** | モデル推論の可視性を切り替え（セッション横断で永続） |
| **Tab** | ファイルパス自動補完（`@` 構文）、`/resume` セッション循環 |
| **Ctrl+S** | MCPサーバー設定を保存 |
| **?** | コマンドリファレンス表示 |

## カスタム指示

Copilot CLIは指示ファイルを自動的に読む:

| ファイル | スコープ |
|------|-------|
| `.github/copilot-instructions.md` | リポジトリ全体の指示 |
| `.github/instructions/**/*.instructions.md` | パス固有（globパターン用YAMLフロントマター） |
| `AGENTS.md` | リポジトリルート（Codex CLIと共有） |
| `CLAUDE.md` | Copilotコーディングエージェントも読む |

指示は**結合**（すべての一致ファイルがプロンプトに含まれる）。優先度ベースのフォールバックなし。

## MCP設定

- **組み込み**: GitHubMCPサーバー（issues、PR、Copilot Spaces） — 事前設定、デフォルトで有効
- **設定ファイル**: `~/.copilot/mcp-config.json`（JSON形式）
- **サーバー追加**: 対話モードで `/mcp add`、またはセッションごとに `--additional-mcp-config <path>`
- **URL制御**: `~/.copilot/config` 内の `allowed_urls` / `denied_urls` パターン

## コンテキスト管理

- **自動compaction**: トークン限界95%でトリガー
- **手動compaction**: `/compact` コマンド
- **トークン可視化**: `/context` が詳細な内訳を表示
- **セッション再開**: `--resume`（セッション循環）または `--continue`（最新のローカルセッション）

## モデル切り替え

`/model` コマンドまたは `--model` フラグで利用可能:
- Claude Sonnet 4.5（デフォルト）
- Claude Sonnet 4
- GPT-5

足軽へ: モデルは起動時にsettings.yamlで設定。`type: model_switch` 経由のランタイム切り替え可能だが滅多に不要。

## tmux対話

**警告: Copilot CLI tmux統合は未検証。**

| 側面 | ステータス |
|--------|--------|
| tmuxペイン内TUI | 動作する見込み（TUIベース） |
| send-keys | **未テスト** — TUIがalt-screenを使用する可能性 |
| capture-pane | **未テスト** — alt-screenが干渉する可能性 |
| プロンプト検出 | 不明なプロンプトフォーマット（`❯` ではない） |
| 非対話パイプ | 未確認（`copilot -p` 非文書化） |

将軍システムにとって、tmux互換性は専用テストを要する**高リスク領域**。

### 潜在的回避策
- `!` プレフィックスのシェルコマンドはTUI入力問題をバイパスする可能性
- `/delegate` でリモートコーディングエージェントにすればローカルTUI対話を回避
- `/clear` 代替としてCtrl+C + 再起動

## 制限（vs Claude Code）

| 機能 | Claude Code | Copilot CLI |
|---------|------------|-------------|
| tmux統合 | ✅ 実戦テスト済み | ⚠️ 未テスト |
| 非対話モード | ✅ `claude -p` | ⚠️ 未確認 |
| `/clear` コンテキストリセット | ✅ 利用可 | ❌ なし（/compactまたは再起動使用） |
| Memory MCP | ✅ 永続的知識グラフ | ❌ 相当なし |
| コストモデル | APIトークンベース（制限なし） | サブスクリプション（プレミアムreq制限） |
| 8エージェント並列 | ✅ 実証済み | ❌ プレミアムreq制限が禁止的 |
| 専用ファイルツール | ✅ Read/Write/Edit/Glob/Grep | 承認付き汎用ファイルツール |
| Web検索 | ✅ WebSearch + WebFetch | web_fetchのみ |
| タスク委任 | Taskツール（ローカルサブエージェント） | /delegate（リモートコーディングエージェント） |

## Compaction復旧

Copilot CLIはトークン限界95%で自動compaction使用。`/clear` 相当は存在しない。

将軍システムにCopilot CLIを統合する場合:
1. 自動compactionがほとんどのケースを自動処理
2. tmux統合が動作するなら send-keys 経由で `/compact` を送信可能
3. セッション状態はcompaction経由で保持（コンテキストをリセットする `/clear` と異なる）
4. コンテキストが保持されるならCLAUDE.mdベースの復旧は不要; 代わりに `AGENTS.md` + `.github/copilot-instructions.md` を使用

## 設定ファイルまとめ

| ファイル | 場所 | 目的 |
|------|----------|---------|
| `config` / `config.json` | `~/.copilot/` | メイン設定 |
| `mcp-config.json` | `~/.copilot/` | MCPサーバー定義 |
| `lsp-config.json` | `~/.copilot/` | LSPサーバー設定 |
| `.github/lsp.json` | リポジトリルート | リポジトリレベルLSP設定 |

場所は `XDG_CONFIG_HOME` 環境変数でカスタマイズ可能。

---

*情報源: [GitHub Copilot CLI Docs](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli)、[Copilot CLI Repository](https://github.com/github/copilot-cli)、[拡張エージェント変更ログ (2026-01-14)](https://github.blog/changelog/2026-01-14-github-copilot-cli-enhanced-agents-context-management-and-new-ways-to-install/)、[プランモード変更ログ (2026-01-21)](https://github.blog/changelog/2026-01-21-github-copilot-cli-plan-before-you-build-steer-as-you-go/)、[PR #10 (yuto-ts) Copilot対応](https://github.com/yohey-w/multi-agent-shogun/pull/10)*
