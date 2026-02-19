# 通信プロトコル

## メールボックスシステム（inbox_write.sh）

エージェント間通信はファイルベースのメールボックスを使用:

```bash
bash scripts/inbox_write.sh <target_agent> "<message>" <type> <from>
```

例:
```bash
# 将軍 → 家老
bash scripts/inbox_write.sh karo "cmd_048を書いた。実行せよ。" cmd_new shogun

# 足軽 → 家老
bash scripts/inbox_write.sh karo "足軽5号、任務完了。報告YAML確認されたし。" report_received ashigaru5

# 家老 → 足軽
bash scripts/inbox_write.sh ashigaru3 "タスクYAMLを読んで作業開始せよ。" task_assigned karo
```

配信は `inbox_watcher.sh`（インフラ層）が処理します。
**エージェントは tmux send-keys を直接呼び出してはいけません。**

## 配信メカニズム

2つのレイヤー:
1. **メッセージ永続化**: `inbox_write.sh` が `queue/inbox/{agent}.yaml` にflockで書き込み。保証されている。
2. **起床シグナル**: `inbox_watcher.sh` が `inotifywait` でファイル変更を検知 → エージェント起床:
   - **優先度1**: Agent self-watch（エージェント自身の inbox への `inotifywait`）→ nudge不要
   - **優先度2**: `tmux send-keys` — 短いnudgeのみ（テキストとEnterを別々に送信、0.3秒間隔）

nudgeは最小限: `inboxN`（例: `inbox3` = 3件未読）。それだけです。
**エージェントは inbox ファイルを自分で読みます。** メッセージ内容は tmux を経由しません — 短い起床シグナルのみ。

安全性注意（将軍）:
- 将軍ペインがアクティブ（主君が入力中）の場合、`inbox_watcher.sh` はキーストロークを注入してはいけません。tmux `display-message` のみ使用すべき。
- エスカレーションキーストローク（`Escape×2`、`/clear`、`C-u`）は将軍に対して抑制し、人間の入力を妨害しないこと。

特殊ケース（`tmux send-keys` で送信されるCLIコマンド）:
- `type: clear_command` → `/clear` + Enter を send-keys で送信
- `type: model_switch` → /model コマンドを send-keys で送信

## エージェント自己監視フェーズポリシー（cmd_107）

フェーズ移行はwatcherフラグで制御:

- **フェーズ1（ベースライン）**: 起動時 `process_unread_once` + `inotifywait` イベント駆動ループ + タイムアウトフォールバック。
- **フェーズ2（通常nudgeオフ）**: `disable_normal_nudge` 動作有効（`ASW_DISABLE_NORMAL_NUDGE=1` または `ASW_PHASE>=2`）。
- **フェーズ3（最終エスカレーションのみ）**: `FINAL_ESCALATION_ONLY=1`（または `ASW_PHASE>=3`）で通常 `send-keys inboxN` を抑制; エスカレーションレーンは復旧用に残る。

読み取りコスト制御:

- `summary-first` ルーティング: 完全inbox解析前にunread_count高速パス。
- `no_idle_full_read`: unread=0のタイムアウトサイクルは重い読み取りパスをスキップ必須。
- メトリクスフックを記録: `unread_latency_sec`、`read_count`、`estimated_tokens`。

**エスカレーション**（nudgeが処理されない場合）:

| 経過時間 | アクション | トリガー |
|---------|--------|---------|
| 0〜2分 | 標準pty nudge | 通常配信 |
| 2〜4分 | Escape×2 + nudge | カーソル位置バグ回避策 |
| 4分以上 | `/clear` 送信（5分に1回まで）| 強制セッションリセット + YAML再読込 |

## Inbox 処理プロトコル（家老/足軽/軍師）

`inboxN`（例: `inbox3`）を受信したとき:
1. `queue/inbox/{your_id}.yaml` を読む
2. `read: false` のエントリをすべて見つける
3. 各メッセージを `type` に応じて処理
4. 処理済みの各エントリを更新: `read: true`（Edit toolを使用）
5. 通常ワークフローに戻る

### 必須: タスク後のInbox確認

**いかなるタスク完了後も、アイドル状態になる前に:**
1. `queue/inbox/{your_id}.yaml` を読む
2. `read: false` のエントリがあれば → 処理する
3. その後にアイドル状態へ

これはオプションではありません。これをスキップしてredoメッセージが待機していると、
エスカレーションが `/clear` を送信するまでアイドル状態のままになります（約4分）。

## Redo プロトコル

家老がタスクをやり直す必要があると判断したとき:

1. 家老が新しいタスクYAMLを新しいtask_idで書く（例: `subtask_097d` → `subtask_097d2`）、`redo_of` フィールドを追加
2. 家老が `clear_command` タイプの inbox メッセージを送信（`task_assigned` ではない）
3. inbox_watcher が `/clear` をエージェントに配信 → セッションリセット
4. エージェントがセッション開始手順で復旧、新しいタスクYAMLを読んで新規開始

競合状態は排除: `/clear` が古いコンテキストを消去。エージェントは新しいtask_idでYAMLを再読込。

## 報告フロー（割り込み防止）

| 方向 | 方法 | 理由 |
|-----------|--------|--------|
| 足軽/軍師 → 家老 | Report YAML + inbox_write | ファイルベース通知 |
| 家老 → 将軍/主君 | dashboard.md 更新のみ | **将軍へのinbox禁止** — 主君の入力を中断しない |
| 家老 → 軍師 | YAML + inbox_write | 戦略タスク委任 |
| 上 → 下 | YAML + inbox_write | 標準的な起床 |

## ファイル操作ルール

**Write/Edit前に必ずRead。** Claude Codeは未読ファイルへのWrite/Editを拒否します。

## Inbox通信ルール

### メッセージ送信

```bash
bash scripts/inbox_write.sh <target> "<message>" <type> <from>
```

**sleep間隔不要。** 配信確認不要。複数送信を連続実行可能 — flockが並行性を処理。

### 報告通知プロトコル

報告YAML書き込み後、家老に通知:

```bash
bash scripts/inbox_write.sh karo "足軽{N}号、任務完了でござる。報告書を確認されよ。" report_received ashigaru{N}
```

これだけ。状態確認不要、リトライ不要、配信確認不要。
inbox_writeが永続性を保証。inbox_watcherが配信処理。
