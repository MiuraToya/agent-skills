# claude-plugin

Claude 用のプラグインを配布するマーケットプレイスです。

## インストール

Claude Code 上で次を実行します。

```text
/plugin marketplace add MiuraToya/claude-plugin
/plugin install software-testing@miura-claude-plugins
/plugin install pr-review-response@miura-claude-plugins
```

## プラグイン

| プラグイン | 概要 | 詳細 |
|---|---|---|
| `software-testing` | 任意の言語・フレームワークで自動テストを決定論的に設計・実装・改善するスキル | [SKILL.md](plugins/software-testing/skills/software-testing/SKILL.md) |
| `pr-review-response` | GitHub の PR レビューコメントへ体系的に対応するワークフロー (取得 → 方針合意 → コミット → 返信) | [SKILL.md](plugins/pr-review-response/skills/pr-review-response/SKILL.md) |

## ライセンス

MIT
