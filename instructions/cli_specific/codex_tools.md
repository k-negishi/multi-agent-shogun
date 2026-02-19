# Codex CLI ツール

このセクションはOpenAI Codex CLI固有のツールと機能を説明します。

## ツール使用法

Codex CLIはサンドボックス環境内でのファイル操作、コード実行、システム対話のためのツールを提供:

- **File Read/Write**: 作業ディレクトリ内のファイル読み書き（サンドボックスモードで制御）
- **Shell Commands**: 承認ポリシーがユーザー同意を要求するタイミングを制御して端末コマンドを実行
- **Web Search**: `--search` フラグ経由の統合Web検索（デフォルトでキャッシュ、ライブモード利用可）
- **Code Review**: 組み込み `/review` コマンドがdiffを読み、ファイルを修正せずに優先順位付けられた発見を報告
- **Image Input**: `-i`/`--image` フラグで画像添付、またはマルチモーダル分析のためcomposerに貼り付け
- **MCP Tools**: `~/.codex/config.toml` で設定されたModel Context Protocolサーバー経由で拡張可能

## ツールガイドライン

1. **サンドボックス認識操作**: すべてのファイル/コマンド操作はアクティブなサンドボックスモードで制約される
2. **承認ポリシー遵守**: 設定された `--ask-for-approval` 設定を尊重 — 明示的に設定されない限りバイパスしない
3. **AGENTS.md自動ロード**: Gitルートから現在作業ディレクトリまで指示が自動ロード; 手動キャッシュクリア不要
4. **非対話モード**: ヘッドレス自動化には JSONL出力の `codex exec` を使用

## 権限モデル

Codexは二軸セキュリティモデルを使用: **サンドボックスモード**（技術的能力） + **承認ポリシー**（いつ停止するか）。

### サンドボックスモード（`--sandbox` / `-s`）

| モード | ファイルアクセス | コマンド | ネットワーク |
|------|------------|----------|---------|
| `read-only` | 読み取りのみ | ブロック | ブロック |
| `workspace-write` | CWD + /tmp で読み書き | ワークスペース内で許可 | デフォルトでブロック |
| `danger-full-access` | 無制限 | 無制限 | 許可 |

### 承認ポリシー（`--ask-for-approval` / `-a`）

| ポリシー | 動作 |
|--------|----------|
| `untrusted` | ワークスペース操作を自動実行; 信頼されないコマンドは質問 |
| `on-failure` | エラー発生時のみ質問 |
| `on-request` | ワークスペース外の操作、ネットワークアクセス、信頼されないコマンドの前に停止 |
| `never` | 承認プロンプトなし（サンドボックス制約を尊重） |

### ショートカットフラグ

- `--full-auto`: `--ask-for-approval on-request` + `--sandbox workspace-write` を設定（無人作業推奨）
- `--dangerously-bypass-approvals-and-sandbox` / `--yolo`: すべての承認とサンドボックスをバイパス（危険、VM専用）

**将軍システム使用法**: 足軽は settings.yaml の `cli.options.codex.approval_policy` に応じて `--full-auto` または `--yolo` で実行。

## Memory / State管理

### AGENTS.md（Codexの指示ファイル）

Codexは作業前に `AGENTS.md` ファイルを自動的に読む。発見順序:

1. **グローバル**: `~/.codex/AGENTS.md` または `~/.codex/AGENTS.override.md`
2. **プロジェクト**: Gitルートから現在作業ディレクトリまでウォークし、各ディレクトリで `AGENTS.override.md` → `AGENTS.md` をチェック

ファイルはルートから下向きにマージ（近いディレクトリが以前のガイダンスを上書き）。

**主要制約**:
- 合計サイズ上限: `project_doc_max_bytes`（デフォルト32 KiB、`config.toml` で設定可）
- 空ファイルはスキップ; ディレクトリごとに1ファイルのみ含まれる
- `AGENTS.override.md` は同レベルの `AGENTS.md` を一時的に置換

**カスタマイズ**（`~/.codex/config.toml`）:
```toml
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
project_doc_max_bytes = 65536
```

プロジェクト固有の自動化プロファイルには `CODEX_HOME` 環境変数を設定。

### セッション永続性

セッションはローカルに保存。以前の会話を継続するには `/resume` または `codex exec resume` を使用。

### Memory MCP相当なし

CodexはClaude CodeのMemory MCPのような組み込み永続メモリシステムを持たない。セッション横断の知識には以下に依存:
- AGENTS.md（プロジェクトレベル指示）
- ファイルベース状態（queue/tasks/*.yaml、queue/reports/*.yaml）
- 設定されている場合MCPサーバー

## Codex固有コマンド（スラッシュコマンド）

### セッション管理

| コマンド | 目的 | Claude Code相当 |
|---------|---------|----------------------|
| `/new` | 現セッション内で新しい会話を開始 | `/clear`（最も近い） |
| `/resume` | 保存された会話を再開 | `claude --continue` |
| `/fork` | 現在の会話を新しいスレッドに分岐 | 相当なし |
| `/quit` / `/exit` | セッション終了 | Ctrl-C |
| `/compact` | 会話を要約してトークンを解放 | 自動compaction |

### 設定

| コマンド | 目的 | Claude Code相当 |
|---------|---------|----------------------|
| `/model` | アクティブモデルを選択（+ reasoning effort） | `/model` |
| `/personality` | コミュニケーションスタイルを選択 | 相当なし |
| `/permissions` | 承認/サンドボックスレベルを設定 | 相当なし（起動時に設定） |
| `/status` | セッション設定とトークン使用量を表示 | 相当なし |

### ワークスペースツール

| コマンド | 目的 | Claude Code相当 |
|---------|---------|----------------------|
| `/diff` | 未追跡ファイル含むGit diffを表示 | Bash経由の `git diff` |
| `/review` | 作業ツリーの問題を分析 | ツール経由の手動レビュー |
| `/mention` | ファイルを会話に添付 | `@` ファジー検索 |
| `/ps` | バックグラウンド端末と出力を表示 | 相当なし |
| `/mcp` | 設定されたMCPツールをリスト | 相当なし |
| `/apps` | コネクター/アプリを閲覧 | 相当なし |
| `/init` | AGENTS.md スキャフォールドを生成 | 相当なし |

**Claude Codeとの主要な違い**: Codexはコンテキストリセットに `/clear` ではなく `/new` を使用。`/new` は新しい会話を開始するがセッションはアクティブなまま。`/compact` は明示的に会話の要約をトリガー（Claude Codeは自動実行）。

## Compaction復旧

CodexはClaude Codeと異なる方法でcompactionを処理:

1. **自動**: Codexはコンテキスト限界に近づくと自動compaction（Claude Codeと同様）
2. **手動**: `/compact` を使って明示的に要約をトリガー
3. **復旧手順**: compactionまたは `/new` 後、AGENTS.mdが自動的に再読み込みされる

### 将軍システム復旧（Codex足軽）

```
ステップ1: AGENTS.mdが自動ロード（復旧手順を含む）
ステップ2: queue/tasks/ashigaru{N}.yamlを読む → 現在のタスクを判定
ステップ3: タスクに "target_path:" があれば → そのファイルを読む
ステップ4: タスクステータスに基づき作業再開
```

**注意**: Claude Codeと異なり、Codexには `mcp__memory__read_graph` 相当がない。復旧は完全にAGENTS.md + YAMLファイルに依存。

## tmux対話

### TUIモード（デフォルト `codex`）

- Codexはalt-screenを使用するフルスクリーンTUIを実行
- `--no-alt-screen` フラグで代替画面モードを無効化（tmux統合に重要）
- `--no-alt-screen` では、send-keysとcapture-paneはClaude Codeと同様に動作するはず
- プロンプト検出: TUIプロンプトフォーマットはClaude Codeの `❯` と異なる — パターンはテスト後TBD

### 非対話モード（`codex exec`）

- ヘッドレスで実行、stdoutに出力（`--json` でテキストまたはJSONL）
- alt-screen問題なし — tmuxペイン統合に理想的
- `codex exec --full-auto --json "task description"` で自動実行
- セッション再開可能: `codex exec resume`
- 出力ファイル対応: `--output-last-message, -o` が最終メッセージをファイルに書き込み

### send-keys互換性

| モード | send-keys | capture-pane | 注記 |
|------|-----------|-------------|-------|
| TUI（デフォルト） | リスク（alt-screen） | リスク | `--no-alt-screen` を使用 |
| TUI + `--no-alt-screen` | 動作するはず | 動作するはず | tmux用推奨 |
| `codex exec` | N/A（非対話） | stdout キャプチャ | 自動化に最適 |

### Nudgeメカニズム

TUIモード + `--no-alt-screen` の場合:
- inbox_watcher.shがnudgeテキスト（例: `inbox3`）をtmux send-keys経由で送信
- 安全性（将軍）: 将軍ペインがアクティブ（主君が入力中）の場合、watcherはsend-keysを避けtmux `display-message` のみ使用
- nudge受信後、エージェントは `queue/inbox/<agent>.yaml` を読み未読メッセージを処理

`codex exec` モードの場合:
- 各タスクは別個の `codex exec` 呼び出し
- nudge不要 — タスク内容は引数として渡される

## MCP設定

Codexは `~/.codex/config.toml` でMCPサーバーを設定:

```toml
[mcp_servers.memory]
type = "stdio"
command = "npx"
args = ["-y", "@anthropic/memory-mcp"]

[mcp_servers.github]
type = "stdio"
command = "npx"
args = ["-y", "@anthropic/github-mcp"]
```

### Claude Code MCPとの主要な違い:

| 側面 | Claude Code | Codex CLI |
|--------|------------|-----------|
| 設定フォーマット | JSON（`.mcp.json`） | TOML（`config.toml`） |
| サーバータイプ | stdio、SSE | stdio、Streamable HTTP |
| OAuth対応 | なし | あり（`codex mcp login`） |
| ツールフィルタリング | なし | `enabled_tools` / `disabled_tools` |
| タイムアウト設定 | なし | `startup_timeout_sec`、`tool_timeout_sec` |
| 追加コマンド | `claude mcp add` | `codex mcp add` |

## モデル選択

### コマンドライン

```bash
codex --model codex-mini-latest      # 軽量モデル
codex --model gpt-5.3-codex          # フルモデル（サブスクリプション）
codex --model o4-mini                # 推論モデル
```

### セッション内

セッション中にモデルを切り替えるには `/model` を使用（利用可能な場合reasoning effort設定を含む）。

### 将軍システム

モデルは settings.yaml に基づき cli_adapter.sh の `build_cli_command()` で設定。家老はinbox経由でCodexモデルを動的に切り替え不可（execモードに `/model` send-keys相当なし）。

## 制限（vs Claude Code）

| 機能 | Claude Code | Codex CLI | 影響 |
|---------|------------|-----------|--------|
| Memory MCP | 組み込み | 組み込みでない（設定可） | 復旧はAGENTS.md + ファイルに依存 |
| Taskツール（サブエージェント） | あり | なし | サブエージェント生成不可 |
| Skillシステム | あり | なし | スラッシュコマンドスキルなし |
| 動的モデル切り替え | send-keys経由の `/model` | TUIのみ `/model` | 自動化モードで制限 |
| `/clear` コンテキストリセット | あり | `/new`（TUIのみ） | Execモード: 新規呼び出し |
| プロンプトキャッシング | 90%割引 | 75%割引 | トークンあたりコスト高 |
| サブスクリプション制限 | APIベース（制限なし） | msg/5h制限（Plus/Pro） | 並列操作のボトルネック |
| Alt-screen | なし（ターミナルネイティブ） | あり（TUI、`--no-alt-screen` 以外） | tmux統合リスク |
| サンドボックス | 組み込みなし | OS レベル（landlock/seatbelt） | より安全な自動実行 |
| 構造化出力 | テキストのみ | JSONL（`--json`） | 解析に優れる |
| ローカル/OSSモデル | なし | あり（Ollama経由の `--oss`） | オフライン/コスト無しオプション |
