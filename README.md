# agent-skills

Claude 用のプラグインを配布するマーケットプレイスです。

## インストール

Claude Code 上で次を実行します。

```text
/plugin marketplace add MiuraToya/claude-plugin
/plugin install software-testing@miura-claude-plugins
/plugin install pr-review-response@miura-claude-plugins
/plugin install task-manager@miura-claude-plugins
```

## プラグイン

| プラグイン | 概要 | 詳細 |
|---|---|---|
| `software-testing` | 任意の言語・フレームワークで自動テストを決定論的に設計・実装・改善するスキル | [SKILL.md](plugins/software-testing/skills/software-testing/SKILL.md) |
| `pr-review-response` | GitHub の PR レビューコメントへ体系的に対応するワークフロー (取得 → 方針合意 → コミット → 返信) | [SKILL.md](plugins/pr-review-response/skills/pr-review-response/SKILL.md) |
| `task-manager` | Jira の自分担当タスク一覧・詳細取得と、Confluence ページの参照を行う読み取り専用スキル群 | [jira-my-tasks](plugins/task-manager/skills/jira-my-tasks/SKILL.md) / [confluence-page-reader](plugins/task-manager/skills/confluence-page-reader/SKILL.md) |

## ライセンス

MIT
