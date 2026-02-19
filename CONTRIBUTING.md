# multi-agent-shogun への貢献

multi-agent-shogun への貢献に興味を持っていただき、ありがとうございます！このドキュメントはプロジェクトへの貢献ガイドラインを提供します。

## 目次

1. [貢献方法](#貢献方法)
2. [プロジェクト構造](#プロジェクト構造)
3. [.gitignore ホワイトリストアプローチ](#gitignore-ホワイトリストアプローチ)
4. [コーディング規約](#コーディング規約)
5. [テスト](#テスト)
6. [プルリクエストガイドライン](#プルリクエストガイドライン)
7. [コミュニケーション](#コミュニケーション)

---

## 貢献方法

### Fork、Branch、PR ワークフロー

1. **リポジトリをForkする** on GitHub
2. **Forkをローカルにクローン**:
   ```bash
   git clone https://github.com/YOUR_USERNAME/multi-agent-shogun.git
   cd multi-agent-shogun
   ```
3. **機能ブランチを作成**:
   ```bash
   git checkout -b feature/your-feature-name
   ```
4. **変更を加えてコミット**（明確で説明的なメッセージで）
5. **Forkにプッシュ**:
   ```bash
   git push origin feature/your-feature-name
   ```
6. **プルリクエストを開く** on GitHub

### 開始前に

- 重複作業を避けるため、既存の[Issues](https://github.com/yohey-w/multi-agent-shogun/issues)を確認
- 大きな変更の場合、まず[Discussion](https://github.com/yohey-w/multi-agent-shogun/discussions)を開く
- 規約と要件を理解するため、このドキュメント全体を読む

---

## プロジェクト構造

ディレクトリレイアウトを理解すると、コードベースをナビゲートしやすくなります:

```
multi-agent-shogun/
│
├── scripts/              # コアユーティリティスクリプト
│   ├── inbox_write.sh    # エージェント間メッセージング（ファイルベースメールボックス）
│   ├── inbox_watcher.sh  # inotifywaitによるイベント駆動配信
│   ├── ntfy.sh           # スマホへのプッシュ通知
│   └── build_instructions.sh  # CLI固有の指示を生成
│
├── instructions/         # エージェント動作定義
│   ├── shogun.md         # 将軍（指揮官）の指示
│   ├── karo.md           # 家老（マネージャー）の指示
│   ├── ashigaru.md       # 足軽（ワーカー）の指示
│   ├── cli_specific/     # CLI固有のツール説明
│   │   ├── claude_tools.md
│   │   ├── codex_tools.md
│   │   └── copilot_tools.md
│   └── generated/        # テンプレートからビルド（手動編集禁止）
│
├── lib/                  # コアライブラリ
│   └── cli_adapter.sh    # マルチCLI抽象化レイヤー
│
├── templates/            # 報告とコンテキストテンプレート
│   ├── context_template.md  # 汎用7セクションプロジェクトコンテキスト
│   └── integ_*.md        # 統合報告テンプレート
│
├── queue/                # 通信とタスクデータ
│   ├── shogun_to_karo.yaml  # コマンドキュー
│   ├── inbox/            # エージェントごとのメールボックス
│   ├── tasks/            # ワーカーごとのタスク割当
│   └── reports/          # 完了報告
│
├── config/               # 設定ファイル
│   ├── settings.yaml     # 言語、CLI設定、ntfyトピック
│   └── projects.yaml     # プロジェクトレジストリ
│
├── tests/                # テストスイート
│   ├── unit/             # bats ユニットテスト
│   └── integration/      # bats 統合テスト
│
├── docs/                 # ドキュメント
│   └── philosophy.md     # 設計原則
│
├── .github/
│   └── workflows/        # CI/CDパイプライン
│       └── test.yml      # GitHub Actions テストスイート
│
├── shutsujin_departure.sh  # 日次デプロイスクリプト
├── first_setup.sh          # 初回セットアップ
├── CLAUDE.md               # コアシステム指示（自動ロード）
├── AGENTS.md               # Codex 自動ロードファイル
└── Makefile                # 開発コマンド
```

### 主要ディレクトリ

| ディレクトリ | 目的 | 重要な注意事項 |
|-----------|---------|-----------------|
| `scripts/` | コアシステムユーティリティ | すべてのスクリプトはshellcheckに合格すること |
| `instructions/` | エージェント動作 | CLI固有の指示は `cli_specific/` に配置 |
| `lib/` | 共有ライブラリ | `cli_adapter.sh` がCLI抽象化を処理 |
| `queue/` | ランタイムデータ | git-ignored、実行時に生成 |
| `templates/` | 再利用可能テンプレート | 報告とコンテキストファイルに使用 |
| `tests/` | テストスイート | bats形式、レベル別に整理（unit/integration） |

---

## .gitignore ホワイトリストアプローチ

**重要:** このプロジェクトは**ホワイトリストベースの.gitignore**戦略を使用しています。

### 仕組み

1. **Step 1**: デフォルトの `*` がすべてをgitから除外
2. **Step 2**: `!*/` がディレクトリトラバーサルを許可
3. **Step 3**: 個別のファイルとディレクトリを `!filename` で明示的に許可

### 新しいファイルをGitに追加する

**新しいファイルをgitに追加する前に、.gitignoreホワイトリストに追加する必要があります:**

```bash
# 例: 新しいスクリプトをgitに追加

# 1. ファイルを作成
touch scripts/new_script.sh

# 2. .gitignoreを編集してホワイトリストエントリを追加
echo '!scripts/new_script.sh' >> .gitignore

# 3. これでgitが追跡します
git add scripts/new_script.sh
git commit -m "feat: add new_script.sh"
```

### デフォルトで除外されるもの

以下は意図的に除外されています（これらはホワイトリストに追加しないでください）:

- `projects/` — 機密のクライアント情報を含む
- `queue/` — ランタイムデータ、動的に生成
- `memory/` — ユーザー固有の永続メモリ
- `.claude/commands/` — ユーザー固有のスキル（コミットされない）
- `saytask/streaks.yaml` — ユーザー固有のタスクデータ

### コミット前の確認

```bash
# 新しいファイルが追跡されているか確認
git status

# ファイルが表示されない場合、.gitignoreを確認
grep "your_file_name" .gitignore
```

---

## コーディング規約

### シェルスクリプト

すべてのシェルスクリプトはこれらの標準に準拠する必要があります:

1. **Shellcheck準拠**
   ```bash
   # コミット前にshellcheckを実行
   make lint
   ```
   - すべての警告とエラーを修正
   - 絶対に必要な場合のみ `# shellcheck disable=SCXXXX` を使用（説明付き）

2. **Shebang行**
   ```bash
   #!/usr/bin/env bash
   ```

3. **エラーハンドリング**
   ```bash
   set -euo pipefail  # エラー、未定義変数、パイプ失敗で終了
   ```

4. **関数ドキュメント**
   ```bash
   # 関数: send_message
   # 説明: エージェントのinboxにメッセージを書き込む
   # 引数:
   #   $1 - target_agent (shogun|karo|ashigaru1-8)
   #   $2 - メッセージ内容
   # 戻り値: 成功時0、エラー時1
   send_message() {
       local target_agent="$1"
       local message="$2"
       # ... 実装
   }
   ```

5. **変数命名**
   - `UPPERCASE` 定数と環境変数用
   - `lowercase` ローカル変数用
   - 関数スコープ変数には `local` を使用

6. **クォート**
   ```bash
   # 単語分割を防ぐため、常に変数をクォート
   echo "$VARIABLE"         # 良い
   echo $VARIABLE           # 悪い

   # スペースを含むパスをクォート
   cd "$PROJECT_PATH"       # 良い
   cd $PROJECT_PATH         # 悪い
   ```

### YAMLファイル

1. **インデント**: 2スペース（タブ禁止）
2. **ブール値**: `true`/`false` を使用（小文字）
3. **文字列**: 必要時にクォート、過剰なクォートは避ける
4. **コメント**: インライン説明に `#` を使用

例:
```yaml
# ashigaru1のタスク割当
task:
  task_id: subtask_001
  description: "React 19機能を調査"
  status: assigned
  blockedBy: []  # 依存関係なし
```

### Markdownファイル

1. **行の長さ**: ハードリミットなし、可読性を目指す（文章は80-120文字）
2. **ヘッダー**: ATXスタイル（`#` プレフィックス）を使用
3. **コードブロック**: 常にシンタックスハイライトのために言語を指定
4. **リンク**: 繰り返しリンクには参照スタイルを使用

---

## テスト

### テストレベル

プロジェクトは3層テスト戦略を使用:

| レベル | タイプ | ツール | 場所 | 実行コマンド |
|-------|------|------|----------|-------------|
| L1 | ユニット | bats | `tests/unit/` | `make test` |
| L2 | 統合 | bats | `tests/integration/` | `make test-int` |
| L3 | E2E | 手動 | N/A | 家老が実行 |

### SKIP = FAIL ポリシー

**重要ルール**: SKIP数 >= 1 のテストは失敗とみなされます。

- テストは実行するか明示的に失敗するかのどちらか
- テストがスキップされた場合、完了と報告しないこと
- テスト実行前に前提条件を確認

### テスト実行

```bash
# テスト依存関係をインストール（初回のみ）
make install-deps

# ユニットテスト実行
make test

# 統合テスト実行（Claude Codeのみ）
make test-int

# shellcheckリンター実行
make lint

# ビルド + 差分チェック（CI相当）
make check
```

### テスト作成

すべてのテストは**bats**（Bash Automated Testing System）を使用:

```bash
#!/usr/bin/env bats
# test_example.bats

setup() {
    # 各テスト前に実行されるセットアップコード
    TEST_TMP="$(mktemp -d)"
}

teardown() {
    # 各テスト後に実行されるクリーンアップコード
    rm -rf "$TEST_TMP"
}

@test "inbox_write.sh creates inbox file" {
    run bash scripts/inbox_write.sh karo "test message" cmd_new shogun
    [ "$status" -eq 0 ]
    [ -f "queue/inbox/karo.yaml" ]
}
```

### テストガイドライン

1. **事前確認**: テスト実行前にすべての前提条件を検証
   ```bash
   @test "check tmux is installed" {
       command -v tmux || skip "tmux not installed"
   }
   ```

2. **分離**: テストは相互に干渉してはならない
   - 一時ディレクトリを使用（`mktemp -d`）
   - `teardown()` で各テスト後にクリーンアップ

3. **アサーション**: 明確なエラーメッセージのためbats-assertを使用
   ```bash
   load 'test_helper/bats-assert/load'

   @test "example assertion" {
       run some_command
       assert_success
       assert_output --partial "expected text"
   }
   ```

4. **E2Eテスト**: 家老のみがE2Eテストを実行可能（マルチエージェント制御が必要）

---

## プルリクエストガイドライン

### 提出前に

- [ ] すべてのテストが合格（`make test`、`make test-int`）
- [ ] Shellcheckが合格（`make lint`）
- [ ] 生成された指示が同期されている（`make check`）
- [ ] 新しいファイルが `.gitignore` ホワイトリストに追加されている
- [ ] コミットが明確で説明的なメッセージを持つ
- [ ] ドキュメントが更新されている（該当する場合）

### PRタイトル形式

従来のコミットプレフィックスを使用:

```
feat: Kimi Code用の新しいCLIアダプタを追加
fix: atomic writeでのinbox_watcher rc=1を解決
docs: .gitignoreルールでCONTRIBUTING.mdを更新
test: cli_adapter.shのユニットテストを追加
refactor: inbox_write.shのメッセージハンドリングを簡素化
```

### PR説明テンプレート

```markdown
## 概要
このPRが行うことの簡単な説明。

## 動機
この変更がなぜ必要か？どんな問題を解決するか？

## 変更内容
- 主要な変更の箇条書きリスト
- コンテキストのためにファイルパスを含める

## テスト
- [ ] ユニットテストを追加/更新
- [ ] 統合テストが合格
- [ ] 手動テスト済み（方法を説明）

## スクリーンショット（該当する場合）
UI/UX変更のスクリーンショットを追加。

## 関連Issue
Closes #123
```

### レビュープロセス

1. **自動チェック**: GitHub Actionsがテストとリンターを実行
2. **コードレビュー**: 少なくとも1人のメンテナーレビューが必要
3. **テスト**: レビュアーが追加テストを要求する場合あり
4. **ドキュメント**: 変更がドキュメント化されていることを確認

---

## コミュニケーション

### GitHub Issues

GitHub Issuesを以下の用途で使用:
- **バグ報告** — 再現手順、期待される動作 vs 実際の動作、環境詳細を含める
- **機能リクエスト** — ユースケース、提案された解決策、検討した代替案を説明
- **質問** — 実装詳細、設計決定について質問

**日本語でのイシュー報告も歓迎します** (Issues in Japanese are welcome).

### GitHub Discussions

GitHub Discussionsを以下の用途で使用:
- 設計提案
- アーキテクチャに関する質問
- ベストプラクティス
- ワークフローの紹介

### バグ報告テンプレート

```markdown
**バグの説明**
バグが何であるかの明確な説明。

**再現手順**
動作を再現する手順:
1. '...' を実行
2. エラーを確認

**期待される動作**
何が起こることを期待していたか。

**実際の動作**
実際に何が起こったか。

**環境**
- OS: [例: WSL2 Ubuntu 22.04]
- Claude Code バージョン: [例: 1.2.3]
- Shell: [例: bash 5.1]

**追加のコンテキスト**
問題に関するその他のコンテキスト。
```

---

## ライセンス

multi-agent-shogunへの貢献により、あなたの貢献が[MITライセンス](LICENSE)の下でライセンスされることに同意したものとみなされます。

---

## クレジット

貢献はプロジェクトREADMEで認識されます。multi-agent-shogunをより良くしていただき、ありがとうございます！
