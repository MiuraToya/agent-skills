# miura-claude-plugins

[Claude Code](https://claude.com/claude-code) 用のプラグインを配布するマーケットプレイスです。

## インストール

Claude Code 上で次を実行します。

```text
/plugin marketplace add MiuraToya/claude-plugin
/plugin install software-testing@miura-claude-plugins
```

## 同梱プラグイン

### software-testing

任意の言語・フレームワークで自動テストを決定論的に設計・実装・改善するスキルを提供します。

- 戦略選定（Pyramid / Diamond）
- 振る舞いベースの設計ルール
- Python + pytest プロファイル
- 拡張: `references/runtime-<lang>-<framework>.md` を追加することで他言語環境にも対応可能

詳細は [`plugins/software-testing/skills/software-testing/SKILL.md`](plugins/software-testing/skills/software-testing/SKILL.md) を参照。

## ディレクトリ構成

```text
.
├── .claude-plugin/
│   └── marketplace.json
└── plugins/
    └── software-testing/
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            └── software-testing/
                ├── SKILL.md
                └── references/
```

## ライセンス

MIT
