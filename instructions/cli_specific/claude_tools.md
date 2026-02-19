# Claude Code ツール

このセクションはClaude Code固有のツールと機能を説明します。

## ツール使用法

Claude Codeはファイル操作、コード実行、システム対話のための専用ツールを提供:

- **Read**: ファイルシステムからファイルを読む（画像、PDF、Jupyter notebookに対応）
- **Write**: 新規ファイル作成または既存ファイル上書き
- **Edit**: ファイル内の文字列を正確に置換
- **Bash**: タイムアウト制御付きbashコマンド実行
- **Glob**: glob パターンによる高速ファイルパターンマッチング
- **Grep**: ripgrepを使用したコンテンツ検索
- **Task**: 複雑な複数ステップタスクのための専用エージェント起動
- **WebFetch**: Webコンテンツの取得と処理
- **WebSearch**: Web情報の検索

## ツールガイドライン

1. **Write/Edit前にRead**: ファイルを書き込みまたは編集する前に必ず読む
2. **専用ツールを使う**: 専用ツールが存在する場合、ファイル操作にBashを使わない（Read、Write、Edit、Glob、Grep）
3. **並列実行**: 最適性能のため、単一メッセージで複数の独立ツールを呼び出す
4. **過剰な設計を避ける**: 直接要求された、または明らかに必要な変更のみ行う

## Taskツール使用法

Taskツールは複雑な作業のための専用エージェントを起動:

- **Explore**: コードベース探索に特化した高速エージェント
- **Plan**: 実装計画の設計のためのソフトウェアアーキテクトエージェント
- **general-purpose**: 複雑な質問の調査と複数ステップタスク用
- **Bash**: コマンド実行専門家

Taskツールを使うべき場合:
- コードベースを徹底的に探索する必要がある（medium または very thorough）
- 複雑な複数ステップタスクが自律的処理を要する
- 実装戦略の計画が必要

## Memory MCP

重要情報をMemory MCPに保存:

```python
mcp__memory__create_entities([{
    "name": "preference_name",
    "entityType": "preference",
    "observations": ["Lord prefers X over Y"]
}])

mcp__memory__add_observations([{
    "entityName": "existing_entity",
    "contents": ["New observation"]
}])
```

使用対象: 主君の好み、重要決定+理由、プロジェクト横断的な洞察、解決済み問題。

保存しないもの: 一時的タスク詳細（YAMLを使用）、ファイル内容（読めばよい）、進行中の詳細（dashboard.mdを使用）。

## モデル切り替え

足軽のモデルは `config/settings.yaml` で設定され、起動時に適用。
ランタイム切り替えは可能だが滅多に不要（L4+タスクは軍師が処理）:

```bash
# 手動オーバーライドのみ — Bloomベースの自動切り替えではない
bash scripts/inbox_write.sh ashigaru{N} "/model <new_model>" model_switch karo
tmux set-option -p -t multiagent:0.{N} @model_name '<DisplayName>'
```

足軽へ: モデルは自分で切り替えない。家老が管理する。

## /clear プロトコル

家老のみ: 足軽にコンテキストリセット用 `/clear` を送信:

```bash
bash scripts/inbox_write.sh ashigaru{N} "タスクYAMLを読んで作業開始せよ。" clear_command karo
```

足軽へ: `/clear` 後、CLAUDE.md の /clear 復旧手順に従え。最初のタスクでは instructions/ashigaru.md を読まないこと（コスト削減）。

## Compaction復旧

全エージェント: CLAUDE.md のSession Start / Recovery手順に従う。主要ステップ:

1. 自己識別: `tmux display-message -t "$TMUX_PANE" -p '#{@agent_id}'`
2. `mcp__memory__read_graph` — ルール、好み、教訓を復元
3. 指示ファイルを読む（shogun→instructions/shogun.md、karo→instructions/karo.md、ashigaru→instructions/ashigaru.md）
4. プライマリYAMLデータ（queue/、tasks/、reports/）から状態を再構築
5. 禁止行動をレビュー、その後作業開始
