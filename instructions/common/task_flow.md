# タスクフロー

## ワークフロー: 将軍 → 家老 → 足軽

```
主君: command → 将軍: YAML書き込み → inbox_write → 家老: 分解 → inbox_write → 足軽: 実行 → 報告YAML → inbox_write → 家老: dashboard更新 → 将軍: dashboard読む
```

## ステータス定義（単一の情報源）

ステータスはYAMLファイルタイプごとに定義。**最小限に保つ。シンプルが最善。**

固定ステータス集合（安易に追加しない）:
- `queue/shogun_to_karo.yaml`: `pending`、`in_progress`、`done`、`cancelled`
- `queue/tasks/ashigaruN.yaml`: `assigned`、`blocked`、`done`、`failed`
- `queue/tasks/pending.yaml`: `pending_blocked`
- `queue/ntfy_inbox.yaml`: `pending`、`processed`

このセクションを更新せずに新しいステータス値を発明しないこと。

### コマンドキュー: `queue/shogun_to_karo.yaml`

意味と許可/禁止行動（短縮版）:

- `pending`: 未確認
  - 許可: 家老が読んで即座にACK（`pending → in_progress`）
  - 禁止: まだ `pending` のままサブタスク配分

- `in_progress`: 確認済みで作業中
  - 許可: 分解/配分/収集/統合
  - 禁止: ゴールポスト移動（acceptance_criteria編集）、または全基準未達成での `done` マーク

- `done`: 完了・検証済み
  - 許可: 読み取り専用（履歴）
  - 禁止: 古いcmdを編集して「再開」（代わりに新しいcmdを使用）

- `cancelled`: 意図的に停止
  - 許可: 読み取り専用（履歴）
  - 禁止: このcmd下で作業継続（代わりに新しいcmdを使用）

**家老ルール（高速ACK）**:
- 家老がcmd処理開始の瞬間（読み取り後）、そのcmdステータスを更新:
  - `pending` → `in_progress`
  - これにより「誰も作業していない」混乱を防ぎ、エスカレーションロジックを安定化。

### 足軽タスクファイル: `queue/tasks/ashigaruN.yaml`

意味と許可/禁止行動（短縮版）:

- `assigned`: 今すぐ開始
  - 許可: 割当先足軽が実行し `done/failed` に更新 + 報告 + inbox_write
  - 禁止: 他エージェントがその足軽YAMLを編集

- `blocked`: まだ開始しない（前提条件不足）
  - 許可: 家老が準備完了時に `assigned` に変更、次にinbox_write
  - 禁止: `blocked` のままnudgeまたは作業開始

- `done`: 完了
  - 許可: 読み取り専用; 統合に使用
  - 禁止: redoでtask_id再利用（redo protocol使用）

- `failed`: 理由付きで失敗
  - 許可: 報告に理由 + アンブロック提案を含める必須
  - 禁止: 無言の失敗

注意:
- 通常、「idle」はUIステート（アクティブタスクなし）であり、YAMLステータス値ではない。
- 例外（プレースホルダーのみ）: `status: idle` は `task_id: null` の場合のみ許可（`shutsujin_departure.sh --clean` が書くクリーンスタートテンプレート）。
  - その状態では、ファイルはプレースホルダーで「まだタスク割当なし」として扱うべき。

### 保留タスク（家老管理）: `queue/tasks/pending.yaml`

- `pending_blocked`: 待機エリア; **まだ割当不可**
  - 許可: 前提条件完了後、家老が `ashigaruN.yaml` に `assigned` として移動
  - 禁止: 準備前に足軽へ事前割当

### NTFY Inbox（主君のスマホ）: `queue/ntfy_inbox.yaml`

- `pending`: 処理が必要
  - 許可: 将軍が処理して `processed` に設定
  - 禁止: 理由なく pending のまま放置

- `processed`: 処理済み; 記録保持
  - 許可: 読み取り専用
  - 禁止: 新しいエントリ作成なしに pending に戻す

## 即時委任原則（将軍）

**即座に家老に委任してターン終了** し、主君が次のコマンド入力できるようにせよ。

```
主君: command → 将軍: YAML書き込み → inbox_write → ターン終了
                                        ↓
                                  主君: 次を入力可能
                                        ↓
                              家老/足軽: バックグラウンドで作業
                                        ↓
                              dashboard.mdが報告として更新
```

## イベント駆動待機パターン（家老）

**全サブタスク配分後: 停止。** バックグラウンドモニターやsleepループを起動しない。

```
ステップ7: cmd_N サブタスク配分 → 足軽にinbox_write
ステップ8: check_pending → 保留cmd_N+1があれば処理 → 次に停止
  → 家老がアイドル（プロンプト待機）
ステップ9: 足軽完了 → 家老にinbox_write → watcherが家老をnudge
  → 家老起床、報告スキャン、行動
```

**バックグラウンドモニター不要の理由**: inbox_watcher.shが足軽のinbox_writeを家老へ検知してnudge送信。これが真のイベント駆動。sleep不要、ポーリング不要、CPU浪費なし。

**家老の起床経路**: 足軽報告からのinbox nudge、将軍の新cmd、またはシステムイベント。それ以外なし。

## 「起床 = 全スキャン」パターン

Claude Codeは「待機」不可。プロンプト待機 = 停止。

1. 足軽を配分
2. 「ここで停止」と言って処理終了
3. 足軽がinbox経由で起床
4. すべての報告ファイルをスキャン（報告者だけでなく）
5. 状況評価、次に行動

## 報告スキャン（通信ロス安全網）

起床時（理由問わず）、すべての `queue/reports/ashigaru*_report.yaml` をスキャン。
dashboard.mdとクロス参照 — まだ反映されていない報告を処理。

**理由**: 足軽inboxメッセージが遅延する可能性。報告ファイルは既に書き込まれ、安全網としてスキャン可能。

## フォアグラウンドブロック防止（24分フリーズの教訓）

**家老ブロック = 全軍停止。** 2026-02-06、配信確認中のフォアグラウンド `sleep` が家老を24分フリーズ。

**ルール: フォアグラウンドで `sleep` 使用禁止。** タスク配分後 → 停止してinbox起床を待つ。

| コマンドタイプ | 実行方法 | 理由 |
|-------------|-----------------|--------|
| Read / Write / Edit | フォアグラウンド | 即座完了 |
| inbox_write.sh | フォアグラウンド | 即座完了 |
| `sleep N` | **禁止** | inboxイベント駆動を使用 |
| tmux capture-pane | **禁止** | 報告YAML読み取りを使用 |

### 配分後停止パターン

```
✅ 正しい（イベント駆動）:
  cmd_008配分 → inbox_write ashigaru → 停止（inbox起床待ち）
  → 足軽完了 → inbox_write karo → 家老起床 → 報告処理

❌ 誤り（ポーリング）:
  cmd_008配分 → sleep 30 → capture-pane → status確認 → sleep 30 ...
```

## タイムスタンプ

**常に `date` コマンド使用。** 推測禁止。
```bash
date "+%Y-%m-%d %H:%M"       # dashboard.md用
date "+%Y-%m-%dT%H:%M:%S"    # YAML用（ISO 8601）
```

## プレコミットゲート（CI整合）

ルール:
- コミット前にGitHub Actionsと同じチェック実行。
- チェックOK時のみコミット。
- `git push` 前に主君に質問。

最小限のローカルチェック:
```bash
# ユニットテスト（CIと同じ）
bats tests/*.bats tests/unit/*.bats

# 指示生成が同期されている必須（CI "Build Instructions Check"と同じ）
bash scripts/build_instructions.sh
git diff --exit-code instructions/generated/
```
