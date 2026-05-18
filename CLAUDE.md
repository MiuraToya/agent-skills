# claude-plugin リポジトリ運用ルール

プラグインに変更を加える際は、**以下すべてを同一コミットに含めること**。

## 1. バージョンを上げる

対象プラグインの `plugins/<plugin-name>/.claude-plugin/plugin.json` の `version` を semver で更新する。

| 変更種別 | バンプ |
|---|---|
| バグ修正・文言微修正 | patch |
| skill 追加・機能追加 | minor |
| skill 削除・破壊的変更 | major |

判断に迷う場合は minor。

## 2. README.md を更新する

ルートの `README.md` のプラグイン表と install コマンド例に反映する。次のいずれかに該当する場合は必須。

- プラグインの新規追加・削除・リネーム
- description の変更
- 利用者の認識が変わるレベルの skill 追加・削除

## 3. marketplace.json を更新する

`.claude-plugin/marketplace.json` の `plugins` 配列を更新する。次のいずれかに該当する場合は必須。

- プラグインの新規追加・削除・リネーム (`name` / `source` 両方)
- description / category の変更

`plugin.json` 側の description と乖離しないよう同時に揃える。

## コミット

- 1 プラグイン変更 = 1 コミットを基本とする。
- メッセージは `feat(<plugin>): 概要 (vX.Y.Z)` / `fix(<plugin>): 概要 (vX.Y.Z)` 形式。
- 上記 1〜3 をバンプだけ別コミットにしない。
