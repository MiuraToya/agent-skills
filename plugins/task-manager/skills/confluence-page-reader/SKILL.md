---
name: confluence-page-reader
description: Confluence Cloud のページを URL またはページ ID から REST API で取得し、本文を要約して提示するスキル。「このコンフル読んで」「<URL> の内容まとめて」「Jira チケットに貼ってある Confluence ページを見てほしい」などの依頼で使用する。読み取り専用。
---

# Confluence Page Reader

## 目的

Confluence Cloud のページ本文を REST API 経由で取得し、Claude Code セッション上で要約・参照できる状態にする。**読み取り専用**スキル。ページの作成・編集・コメント投稿は行わない。

## 前提条件

`jira-my-tasks` と同じ Atlassian アカウントの認証情報を使う (Jira と Confluence は同一 API トークンで認証できる)。

| 変数 | 内容 | 例 |
|---|---|---|
| `JIRA_BASE_URL` | Atlassian サイト URL (`/wiki` は含めない) | `https://your-domain.atlassian.net` |
| `JIRA_EMAIL` | Atlassian アカウントのメール | `you@example.com` |
| `JIRA_API_TOKEN` | API トークン | `ATATT...` |

未設定の場合はユーザーに案内し、API は呼び出さない。トークンは https://id.atlassian.com/manage-profile/security/api-tokens で発行する。

Confluence のエンドポイントベースは `"$JIRA_BASE_URL/wiki"` になる。

## 入力パターンの解決

ユーザーの入力からページ ID を特定する。

| パターン | 例 | 解決方法 |
|---|---|---|
| ページ ID 直接 | `1234567890` | そのまま使う |
| 標準 URL | `.../wiki/spaces/<SPACE>/pages/<ID>/<title>` | URL からページ ID を正規表現 `/pages/(\d+)` で抽出 |
| タイトル付き URL (ID なし) | `.../wiki/display/<SPACE>/<title>` (旧形式) | CQL 検索で解決 (下記) |
| 短縮 URL | `.../wiki/x/<short>` | curl で 1 度叩いてリダイレクト先 URL を取得 → 再判定 |

タイトル経由の解決には CQL を使う。

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -G "$JIRA_BASE_URL/wiki/rest/api/content/search" \
  --data-urlencode 'cql=space = "<SPACE>" AND title = "<TITLE>" AND type = page'
```

候補が複数返ったときは一覧をユーザーに提示して選んでもらう。**勝手に先頭を採用しない**。

## ページ取得

v2 API を使い、本文は `view` (HTML レンダー済み) で取る。

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -G "$JIRA_BASE_URL/wiki/api/v2/pages/<ID>" \
  --data-urlencode "body-format=view"
```

レスポンスの主なフィールド:

- `title`
- `spaceId` (必要ならスペース名は `/wiki/api/v2/spaces/<spaceId>` で別途取得)
- `version.number`, `version.createdAt`
- `body.view.value` (HTML 本文)
- `_links.webui` (相対パス。`"$JIRA_BASE_URL/wiki" + webui` でブラウザ URL になる)

HTML 本文はタグを素直に除去してプレーンテキストにする。`<table>` や `<ul>` 等の構造を残したい場合は Markdown 化する。

## 出力フォーマット

```text
## <title>
<page URL>

- Space: <space name (or ID)>
- 最終更新: <version.createdAt> (v<version.number>)
- 取得ページ ID: <id>

### 要約
<本文を読み、3-7 行で要点をまとめる>

### 主な見出し
- <h2/h3 の階層を箇条書きで示す。長文ページのナビ用>

### 注意点 (該当時のみ)
- <添付ファイル参照あり / 子ページが多数ある / 他ページへのリンクが多い 等>
```

本文を**そのまま全文貼り付けない**。長文ページでも要約 + 構造提示に留め、ユーザーが「全文ほしい」と明示したら追加で渡す。

## エラーハンドリング

- HTTP 401 / 403: 認証失敗、または当該ページへの閲覧権限なし。トークン値そのものは出力しない。
- HTTP 404: ページ ID が誤り、または削除済み。URL の再確認を促す。
- HTTP 429: レート制限。少し待って再試行する旨を伝える。自動リトライはしない。
- CQL 候補 0 件: タイトルの揺れ (全角/半角、スペース) を疑い、`title ~ "<部分一致>"` での再検索を提案する。

## 他スキルとの連携

- `jira-my-tasks` の詳細モードで Confluence URL が検出された際、本スキルに引き継いで本文を取得できる。**ユーザーから依頼があったときのみ**起動する (Jira 詳細表示時に自動連鎖はしない)。

## やってはいけないこと

- ユーザー依頼なしにページを取得する (Jira の Description に URL があるだけでは取得しない)。
- ページの作成・編集・削除・コメント投稿 (POST / PUT / DELETE は本スキルでは使わない)。
- API トークンを出力やコミット対象に含める。
- CQL 候補が複数あるときに先頭を勝手に採用する。
- 取得した本文を丸ごとそのまま貼り付ける (要約してから提示する)。
