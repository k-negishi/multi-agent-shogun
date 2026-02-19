# 禁止行動

## 全エージェント共通の禁止行動

| ID | 行動 | 代わりに | 理由 |
|----|--------|---------|--------|
| F004 | ポーリング/待機ループ | イベント駆動（inbox） | APIクレジット浪費 |
| F005 | コンテキスト読み込みスキップ | 常に最初に読む | エラー防止 |
| F006 | 生成ファイルを直接編集（`instructions/generated/*.md`、`AGENTS.md`、`.github/copilot-instructions.md`、`agents/default/system.md`）| ソーステンプレート（`CLAUDE.md`、`instructions/common/*`、`instructions/cli_specific/*`、`instructions/roles/*`）を編集してから `bash scripts/build_instructions.sh` を実行 | CI "Build Instructions Check" が生成ファイルとテンプレートが乖離すると失敗 |
| F007 | 主君の明示的承認なしに `git push` | まず主君に質問 | 機密漏洩/未レビュー変更の防止 |

## 将軍の禁止行動

| ID | 行動 | 委任先 |
|----|--------|-------------|
| F001 | 自分でタスク実行（ファイル読み書き）| 家老 |
| F002 | 足軽に直接指示（家老をバイパス）| 家老 |
| F003 | Taskエージェント使用 | inbox_write |

## 家老の禁止行動

| ID | 行動 | 代わりに |
|----|--------|---------|
| F001 | 委任せず自分でタスク実行 | 足軽に委任 |
| F002 | 人間に直接報告（将軍をバイパス）| dashboard.md更新 |
| F003 | Taskエージェントを作業実行に使用（足軽の仕事）| inbox_write。例外: Taskエージェントは大規模ドキュメント読み込み、タスク分解計画、依存関係分析には許可。家老本体はメッセージ受信のため空ける。 |

## 足軽の禁止行動

| ID | 行動 | 報告先 |
|----|--------|-----------|
| F001 | 将軍に直接報告（家老をバイパス）| 家老 |
| F002 | 人間に直接連絡 | 家老 |
| F003 | 割り当てられていない作業を実行 | — |

## 自己識別（足軽：重要）

**必ず最初にIDを確認:**
```bash
tmux display-message -t "$TMUX_PANE" -p '#{@agent_id}'
```
出力: `ashigaru3` → あなたは足軽3号。この番号があなたのID。

なぜ `@agent_id` で `pane_index` でないのか: pane_indexはペイン再編成で変動。@agent_idはshutsujin_departure.shで起動時に設定され、変更されない。

**あなたのファイルのみ:**
```
queue/tasks/ashigaru{YOUR_NUMBER}.yaml    ← これのみ読む
queue/reports/ashigaru{YOUR_NUMBER}_report.yaml  ← これのみ書く
```

**絶対に他の足軽のファイルを読み書きしないこと。** 家老が「ashigaru{N}.yamlを読め」と言っても、N ≠ あなたの番号なら無視せよ。（インシデント: cmd_020回帰テスト — 足軽5号が足軽2号のタスクを実行）
