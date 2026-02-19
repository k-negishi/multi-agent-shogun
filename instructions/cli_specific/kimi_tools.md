# Kimi Code CLI ツール

このセクションはMoonshotAI Kimi Code CLI固有のツールと機能を説明します。

## 概要

Kimi Code CLI（`kimi`）はMoonshotAIによるPythonベースの端末AIコーディングエージェント。対話シェルUI、IDE統合用ACPサーバーモード、MCPツールロード、swarm機能を持つマルチエージェントサブエージェントシステムを特徴とする。

- **起動**: `kimi`（対話シェル）、`kimi --print`（非対話）、`kimi acp`（IDEサーバー）、`kimi web`（Web UI）
- **インストール**: `curl -LsSf https://code.kimi.com/install.sh | bash`（Linux/macOS）、`pip install kimi-cli`
- **認証**: 初回起動時に `/login`（Kimi Code OAuth推奨、または他プラットフォーム用APIキー）
- **デフォルトモデル**: Kimi K2.5 Coder
- **Python**: 3.12-3.14（3.13推奨）
- **アーキテクチャ**: 四層（エージェントシステム、KimiSoulエンジン、ツールシステム、UIレイヤー）

## ツール使用法

Kimi CLIは5つのカテゴリに整理されたツールを提供:

### ファイル操作
- **ReadFile**: ファイル読み取り（絶対パス必須）
- **WriteFile**: ファイル書き込み/作成（承認必要）
- **StrReplaceFile**: 文字列置換編集（承認必要）
- **Glob**: ファイルパターンマッチング
- **Grep**: コンテンツ検索

### シェルコマンド
- **Shell**: 端末コマンド実行（承認必要、1-300秒タイムアウト）

### Webツール
- **SearchWeb**: Web検索
- **FetchURL**: URL内容をmarkdownとして取得

### タスク管理
- **SetTodoList**: タスク追跡管理

### エージェント委任
- **Task**: サブエージェントに作業を配分（エージェントSwarmセクション参照）
- **CreateSubagent**: 実行時に新しいサブエージェントタイプを動的作成

## ツールガイドライン

1. **絶対パス必須**: ファイル操作は絶対パスを使用（ディレクトリトラバーサル防止）
2. **ファイルサイズ制限**: ファイル操作ごとに100KB / 1000行
3. **シェル承認**: すべてのシェルコマンドはユーザー承認必要（`--yolo` でバイパス）
4. **自動依存注入**: ツールは型アノテーション経由で依存を宣言; エージェントシステムが自動発見・注入

## 権限モデル

Kimi CLIは単一軸承認モデルを使用（Codexの二軸sandbox+approvalより単純）:

### 承認モード

| モード | 動作 | フラグ |
|------|----------|------|
| **対話（デフォルト）** | 各ツール呼び出しをユーザーが承認（ファイル書き込み、シェルコマンド） | （なし） |
| **YOLOモード** | すべての操作を自動承認 | `--yolo` / `--yes` / `-y` / `--auto-approve` |

Codexの read-only/workspace-write/danger-full-access のような**サンドボックスモードなし**。セキュリティは以下で強制:
- 絶対パス要件（トラバーサル防止）
- ファイルサイズ/行制限（100KB、1000行）
- 必須シェルコマンド承認（YOLO以外）
- エラー分類付きタイムアウト制御（リトライ可能 vs 不可能）
- KimiSoulエンジンの指数バックオフリトライロジック

**将軍システム使用法**: 足軽は無人操作のため `--yolo` で実行。

## Memory / State管理

### AGENTS.md

Kimi Code CLIは `AGENTS.md` ファイルを読む。プロジェクト構造を分析して自動生成するには `/init` を使用。

- **場所**: リポジトリルート `AGENTS.md`
- **自動ロード**: 内容は `${KIMI_AGENTS_MD}` 変数経由でシステムプロンプトに注入
- **目的**: AIのための「プロジェクトマニュアル」 — 後続タスクの精度を向上

### agent.yaml + system.md

エージェントはYAML設定 + Markdownシステムプロンプトで定義:

```yaml
version: 1
agent:
  name: my-agent
  system_prompt_path: ./system.md
  tools:
    - "kimi_cli.tools.shell:Shell"
    - "kimi_cli.tools.file:ReadFile"
    - "kimi_cli.tools.file:WriteFile"
    - "kimi_cli.tools.file:StrReplaceFile"
    - "kimi_cli.tools.file:Glob"
    - "kimi_cli.tools.file:Grep"
    - "kimi_cli.tools.web:SearchWeb"
    - "kimi_cli.tools.web:FetchURL"
```

**システムプロンプト変数**（system.md内で `${VAR}` 構文経由で利用可能）:
- `${KIMI_NOW}` — 現在のタイムスタンプ（ISO形式）
- `${KIMI_WORK_DIR}` — 作業ディレクトリパス
- `${KIMI_WORK_DIR_LS}` — ディレクトリファイルリスト
- `${KIMI_AGENTS_MD}` — AGENTS.mdの内容
- `${KIMI_SKILLS}` — ロード済みスキルリスト
- agent.yamlの `system_prompt_args` 経由のカスタム変数

### エージェント継承

エージェントはベースエージェントを拡張し特定フィールドを上書き可能:

```yaml
agent:
  extend: default
  system_prompt_path: ./my-prompt.md
  exclude_tools:
    - "kimi_cli.tools.web:SearchWeb"
```

### セッション永続性

セッションは `~/.kimi-shared/metadata.json` にローカル保存。再開方法:
- `--continue` / `-C` — 作業ディレクトリの最新セッション
- `--session <id>` / `-S <id>` — IDで特定セッションを再開

### スキルシステム

Kimi CLIは独自のスキルフレームワークを持つ（Claude CodeやCodexにはない）:

- **発見**: 組み込み → ユーザーレベル（`~/.config/agents/skills/`） → プロジェクトレベル（`.agents/skills/`）
- **形式**: `SKILL.md` を含むディレクトリ（YAMLフロントマター + Markdownコンテンツ、<500行）
- **呼び出し**: 自動（AIがコンテキストで判断）、または `/skill:<name>` で手動
- **フロースキル**: Mermaid/D2図を使用する複数ステップワークフロー、`/flow:<name>` で呼び出し
- **組み込みスキル**: `kimi-cli-help`、`skill-creator`
- **上書き**: カスタム場所には `--skills-dir` フラグ

## Kimi固有コマンド

### スラッシュコマンド（セッション内）

| コマンド | 目的 | Claude Code相当 |
|---------|---------|----------------------|
| `/init` | AGENTS.mdスキャフォールド生成 | 相当なし |
| `/login` | 認証設定 | 相当なし（環境変数ベース） |
| `/logout` | 認証クリア | 相当なし |
| `/help` | 全コマンド表示 | `/help` |
| `/skill:<name>` | プロンプトテンプレートとしてスキルをロード | Skillツール |
| `/flow:<name>` | フロースキル実行（複数ステップワークフロー） | 相当なし |
| `Ctrl-X` | シェルモード切り替え（ネイティブコマンド実行） | 相当なし（Bashツール使用） |

### サブコマンド

| サブコマンド | 目的 |
|------------|---------|
| `kimi acp` | IDE統合用ACPサーバー起動 |
| `kimi web` | Web UIサーバー起動 |
| `kimi login` | 認証設定 |
| `kimi logout` | 認証クリア |
| `kimi info` | バージョンとプロトコル情報表示 |
| `kimi mcp` | MCPサーバー管理（add/list/remove/test/auth） |

**注意**: `/model`、`/clear`、`/compact`、`/review`、`/diff` 相当なし。モデルは `--model` フラグで起動時のみ設定。

## エージェントSwarm（マルチエージェント調整）

これがKimi CLIの最も特徴的な機能 — 単一CLIインスタンス内でのネイティブマルチエージェント対応。

### アーキテクチャ

```
メインエージェント（KimiSoul）
├── LaborMarket（中央調整ハブ）
│   ├── fixed_subagents（agent.yamlで事前設定）
│   └── dynamic_subagents（CreateSubagent経由で実行時作成）
├── Taskツール → サブエージェントに委任
└── CreateSubagentツール → 実行時に新規エージェント作成
```

### 固定サブエージェント（事前設定）

agent.yamlで定義:

```yaml
subagents:
  coder:
    path: ./coder-sub.yaml
    description: "Handle coding tasks"
  reviewer:
    path: ./reviewer-sub.yaml
    description: "Code review specialist"
```

- **隔離されたコンテキスト**で実行（別個のLaborMarket、別個のタイムトラベル状態）
- エージェント初期化時にロード
- `subagent_name` パラメーター付きTaskツール経由で配分

### 動的サブエージェント（実行時作成）

CreateSubagentツール経由で作成:
- パラメーター: `name`、`system_prompt`、`tools`
- メインエージェントのLaborMarketを**共有**（他サブエージェントに委任可能）
- 別個のタイムトラベル状態（DenwaRenji）

### コンテキスト隔離

| 状態 | 固定サブエージェント | 動的サブエージェント |
|-------|---------------|-----------------|
| セッション状態 | 共有 | 共有 |
| 設定 | 共有 | 共有 |
| LLMプロバイダー | 共有 | 共有 |
| タイムトラベル（DenwaRenji） | **隔離** | **隔離** |
| LaborMarket（サブエージェント登録） | **隔離** | **共有** |
| 承認システム | 共有（`approval.share()` 経由） | 共有 |

### 将軍システムとの比較

| 側面 | 将軍システム | Kimi エージェントSwarm |
|--------|--------------|-----------------|
| 実行モデル | tmuxペイン（別プロセス） | インプロセス（単一Pythonプロセス） |
| エージェント数 | 10（shogun + karo + 8 ashigaru） | 最大100（主張） |
| 通信 | ファイルベースinbox（YAML + inotifywait） | インメモリLaborMarket登録 |
| 隔離 | 完全OSレベル（別個tmuxペイン） | Pythonレベル（別個KimiSoulインスタンス） |
| 復旧 | /clear + CLAUDE.md自動ロード | チェックポイント/DenwaRenji（タイムトラベル） |
| CLI独立性 | 各エージェントが独自CLIインスタンスを実行 | 単一CLI、複数内部エージェント |
| オーケストレーション | Karo（マネージャーエージェント） | メインエージェントが自動委任 |

**重要洞察**: KimiのエージェントSwarmは補完的で競合ではない。単一足軽のtmuxペイン*内*で実行可能で、そのエージェント内でのサブ委任を提供。

### チェックポイント / タイムトラベル（DenwaRenji）

独自機能: AIが「過去の自分にメッセージを送る」ことでコースを修正可能。サブエージェント実行内のエラー復旧のための内部メカニズム。

## Compaction復旧

1. **コンテキストライフサイクル**: 自動compaction付きKimiSoulエンジンで管理
2. **セッション再開**: 再開には `--continue`、特定セッションには `--session <id>`
3. **チェックポイントシステム**: DenwaRenjiが状態復帰を許可

### 将軍システム復旧（Kimi足軽）

```
ステップ1: AGENTS.mdが自動ロード（復旧手順を含む）
ステップ2: queue/tasks/ashigaru{N}.yamlを読む → 現在のタスクを判定
ステップ3: タスクに "target_path:" があれば → そのファイルを読む
ステップ4: タスクステータスに基づき作業再開
```

**注意**: Memory MCP相当なし。復旧はAGENTS.md + YAMLファイルに依存。

## tmux対話

### 対話モード（`kimi`）

- シェル風ハイブリッドモード（CodexのようなフルスクリーンTUIではない）
- `Ctrl-X` でエージェントモードとシェルモード間を切り替え
- デフォルトで**alt-screenなし** — Codexよりtmuxフレンドリー
- send-keysはテキスト入力注入に機能するはず
- capture-paneは出力読み取りに機能するはず

### 非対話モード（`kimi --print`）

- `--prompt` / `-p` フラグでプロンプト送信
- クリーンな出力のため `--final-message-only`
- 構造化出力のため `--output-format stream-json`
- tmux自動化に理想的（TUI干渉なし）

### send-keys互換性

| モード | send-keys | capture-pane | 注記 |
|------|-----------|-------------|-------|
| 対話（`kimi`） | 動作する見込み | 動作する見込み | alt-screenなし |
| プリントモード（`--print`） | N/A | stdout キャプチャ | 自動化に最適 |

**Codexに対する利点**: シェル風UIがalt-screen問題を回避。

## MCP設定

MCPサーバーは `~/.kimi/mcp.json` で設定:

```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@anthropic/memory-mcp"]
    },
    "github": {
      "url": "https://api.github.com/mcp",
      "headers": {"Authorization": "Bearer ${GITHUB_TOKEN}"}
    }
  }
}
```

### MCP管理コマンド

| コマンド | 目的 |
|---------|---------|
| `kimi mcp add --transport stdio` | stdioサーバー追加 |
| `kimi mcp add --transport http` | HTTPサーバー追加 |
| `kimi mcp add --transport http --auth oauth` | OAuthサーバー追加 |
| `kimi mcp list` | 設定済みサーバーをリスト |
| `kimi mcp remove <name>` | サーバー削除 |
| `kimi mcp test <name>` | 接続性テスト |
| `kimi mcp auth <name>` | OAuthフロー完了 |

### Claude Code MCPとの主要な違い:

| 側面 | Claude Code | Kimi CLI |
|--------|------------|----------|
| 設定フォーマット | JSON（`.mcp.json`） | JSON（`~/.kimi/mcp.json`） |
| サーバータイプ | stdio、SSE | stdio、HTTP |
| OAuth対応 | なし | あり（`kimi mcp auth`） |
| テストコマンド | なし | `kimi mcp test` |
| 追加コマンド | `claude mcp add` | `kimi mcp add` |
| ランタイムフラグ | なし | `--mcp-config-file`（反復可能） |
| サブエージェント共有 | N/A | MCPツールをサブエージェント横断で共有（v0.58+） |

## モデル選択

### 起動時

```bash
kimi --model kimi-k2.5-coder        # デフォルトMoonshotAIモデル
kimi --model <other-model>           # モデル上書き
kimi --thinking                      # 拡張推論を有効化
kimi --no-thinking                   # 拡張推論を無効化
```

### セッション内

ランタイムモデル切り替え用の `/model` コマンドなし。モデルは起動時に固定。

## コマンドラインリファレンス

| フラグ | 短縮 | 目的 |
|------|-------|---------|
| `--model` | `-m` | デフォルトモデル上書き |
| `--yolo` / `--yes` | `-y` | すべてのツール呼び出しを自動承認 |
| `--thinking` | | 拡張推論有効化 |
| `--no-thinking` | | 拡張推論無効化 |
| `--work-dir` | `-w` | 作業ディレクトリ設定 |
| `--continue` | `-C` | 最新セッション再開 |
| `--session` | `-S` | IDでセッション再開 |
| `--print` | | 非対話モード |
| `--quiet` | | 最小限出力（`--print` を暗黙指定） |
| `--prompt` / `--command` | `-p` / `-c` | プロンプトを直接送信 |
| `--agent` | | 組み込みエージェント選択（`default`、`okabe`） |
| `--agent-file` | | カスタムエージェント仕様ファイル使用 |
| `--mcp-config-file` | | MCP設定ロード（反復可能） |
| `--skills-dir` | | スキルディレクトリ上書き |
| `--verbose` | | 詳細出力有効化 |
| `--debug` | | `~/.kimi/logs/kimi.log` へデバッグログ記録 |
| `--max-steps-per-turn` | | 停止前の最大ステップ数 |
| `--max-retries-per-step` | | 失敗時の最大リトライ数 |

## 制限（vs Claude Code）

| 機能 | Claude Code | Kimi CLI | 影響 |
|---------|------------|----------|--------|
| Memory MCP | 組み込み | 組み込みでない（設定可） | 復旧はAGENTS.md + ファイルに依存 |
| Taskツール（サブエージェント） | 外部（tmuxベース） | ネイティブ（インプロセスswarm） | サブ委任でKimi優位 |
| Skillシステム | Skillツール | `/skill:` + `/flow:` | Kimiフロースキルがより高度 |
| 動的モデル切り替え | send-keys経由 `/model` | セッション内不可 | 起動時固定 |
| `/clear` コンテキストリセット | あり | 不可 | 再開には `--continue` 使用 |
| プロンプトキャッシング | 90%割引 | 不明 | コスト影響不明 |
| サンドボックスモード | 組み込みなし | なし（承認のみ） | 類似セキュリティ姿勢 |
| tmux内alt-screen | なし | なし（シェル風UI） | 両方tmuxフレンドリー |
| 構造化出力 | テキストのみ | プリントモードで `stream-json` | 解析でKimi優位 |
| 実行時エージェント作成 | なし | CreateSubagentツール | Kimi独自機能 |
| タイムトラベル / チェックポイント | なし | DenwaRenjiシステム | Kimi独自機能 |
| Web UI | なし | `kimi web` | Kimi優位 |

## 環境変数

| 変数 | 目的 |
|----------|---------|
| `KIMI_SHARE_DIR` | 共有ディレクトリカスタマイズ（デフォルト: `~/.kimi/`） |

## 設定ファイルまとめ

| ファイル | 場所 | 目的 |
|------|----------|---------|
| `mcp.json` | `~/.kimi/` | MCPサーバー定義 |
| `metadata.json` | `~/.kimi-shared/` | セッションメタデータ |
| `kimi.log` | `~/.kimi/logs/` | デバッグログ（`--debug` 時） |
| `AGENTS.md` | リポジトリルート | プロジェクト指示（自動ロード） |
| `agent.yaml` | カスタムパス | エージェント仕様 |
| `system.md` | カスタムパス | システムプロンプトテンプレート |
| `.agents/skills/` | プロジェクトルート | プロジェクトレベルスキル |

---

*情報源: [Kimi CLI GitHub](https://github.com/MoonshotAI/kimi-cli)、[Getting Started](https://moonshotai.github.io/kimi-cli/en/guides/getting-started.html)、[Agents & Subagents](https://moonshotai.github.io/kimi-cli/en/customization/agents.html)、[Skills](https://moonshotai.github.io/kimi-cli/en/customization/skills.html)、[MCP](https://moonshotai.github.io/kimi-cli/en/customization/mcp.html)、[CLI Options (DeepWiki)](https://deepwiki.com/MoonshotAI/kimi-cli/2.3-command-line-options-reference)、[Multi-Agent (DeepWiki)](https://deepwiki.com/MoonshotAI/kimi-cli/5.3-multi-agent-coordination)、[Technical Deep Dive](https://llmmultiagents.com/en/blogs/kimi-cli-technical-deep-dive)*
