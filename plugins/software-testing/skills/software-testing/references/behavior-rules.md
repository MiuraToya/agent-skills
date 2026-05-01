# 振る舞い中心テスト設計

## 1. 原則

公開契約と観測可能な振る舞いを検証する。
privateメソッド・内部状態の直接検証を主目的にしない。

## 2. テスト対象に含めるもの

- 公開関数・公開クラス・公開モジュール
- APIレスポンス、永続化結果、発行イベントなどの外部可視な結果
- 例外契約（型・メッセージ・ステータス・リトライ条件）

## 3. 避けるもの

- privateヘルパー呼び出し回数の検証を主目的にする
- 実装詳細をなぞる過剰モック
- 1テストに複数の無関係アサーションを詰め込む
- 制御不能な待機（固定sleepなど）

## 4. 内部依存テストの書き換え手順

1. 内部実装チェックを、観測可能な成果物へ置き換える。
2. 公開インターフェースから入力する。
3. 出力・副作用・契約を検証する。
4. 実装詳細の検証は補助情報に限定する。

## 5. 3A ひな型（Arrange-Act-Assert, 擬似コード）

```text
Test <condition> -> <expected_behavior>
  Arrange: preconditions and test data
  Act: invoke public API
  Assert: verify observable result
```

## 6. 例外検証ひな型（擬似コード）

```text
Test <condition> -> raises <error_type>
  Arrange: preconditions and test data
  Act: invoke public API
  Assert: verify error type and message/metadata contract
```

## 7. ドキュメンテーション規則

### 7.1 検証内容と期待結果を docstring / コメントで明示する

各テストの先頭に、検証する振る舞いと期待結果が一読で伝わる短い説明を付ける。
テスト名と本文だけでは意図が読み取れないケースを避け、レビュアーが目的を素早く把握できる状態を保つ。

```text
def test_<condition>_<expected_behavior>():
    """<検証する振る舞い>のとき、<期待結果>であることを確認する。"""
    ...
```

冗長な説明や実装手順の繰り返しは書かない。

### 7.2 Arrange / Act / Assert をコメントで区切る

3A の境界をコメントで明示し、テスト構造を視覚的に追えるようにする。

```text
def test_xxx():
    # Arrange
    <preconditions and test data>

    # Act
    <invoke public API>

    # Assert
    <verify observable result>
```

セクションが空になる場合は省略可。ただし Act と Assert は必ず明示する。

### 7.3 ユビキタス言語を使う

テストコード内の変数名・関数名・コメント・docstring では、プロダクションコードや設計書で使われている用語をそのまま使う。
独自略語・独自パラフレーズ・テストコード固有の造語は避ける。

- 避ける: `usr`, `cust`, `tmp_record`
- 使う: `user`, `customer`, `pending_order`

ドメイン用語の不一致は、レビュー時の認知負荷とリファクタリング時の不整合の主因になる。
