---
name: jira-my-tasks
description: Jira から自分 (currentUser) に割り当てられた未完了タスクを取得して概要一覧で見せ、チケットキー指定時は詳細 (Description / サブタスク / 最新コメント) を表示するスキル。「Jira のタスク見せて」「自分のチケット一覧」「ABC-123 詳しく」などの依頼で使用する。
---

# Jira My Tasks

## 目的

Jira REST API v3 を curl で叩き、ユーザーに割り当てられたタスクを把握するための一覧と詳細表示を提供する。**読み取り専用**スキル。チケットの更新・コメント投稿は行わない。

## 前提条件

以下の環境変数が必要。未設定の場合は、ユーザーに次の手順を案内し、API は呼び出さない。

| 変数 | 内容 | 例 |
|---|---|---|
| `JIRA_BASE_URL` | Atlassian サイト URL (末尾スラッシュなし) | `https://your-domain.atlassian.net` |
| `JIRA_EMAIL` | Atlassian アカウントのメール | `you@example.com` |
| `JIRA_API_TOKEN` | API トークン | `ATATT...` |

API トークンは https://id.atlassian.com/manage-profile/security/api-tokens で発行する。

## 認証方法

すべての呼び出しは Basic 認証 (email:token) で行う。

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  "$JIRA_BASE_URL/rest/api/3/..."
```

トークンを `echo` や標準出力に流したり、コミット対象に含めたりしない。

## モード判定

ユーザーの発言から次のいずれかに振り分ける。

- **モード A (一覧)**: チケットキーの指定がない、または「タスク一覧」「自分のチケット見せて」「Jira 今何ある？」など。
- **モード B (詳細)**: `[A-Z][A-Z0-9_]+-\d+` 形式のチケットキーが含まれる、または「○○ 詳しく」「○○ の中身」など。

どちらか判断に迷う場合はユーザーに確認する。

## モード A: 一覧表示

### 1. JQL を組み立てる

```text
assignee = currentUser() AND statusCategory != Done ORDER BY priority DESC, duedate ASC
```

### 2. 検索 API を叩く

`/rest/api/3/search/jql` (新エンドポイント) を GET で呼ぶ。`fields` を絞ってレスポンスを軽くする。

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -G "$JIRA_BASE_URL/rest/api/3/search/jql" \
  --data-urlencode "jql=assignee = currentUser() AND statusCategory != Done ORDER BY priority DESC, duedate ASC" \
  --data-urlencode "fields=summary,status,priority,duedate" \
  --data-urlencode "maxResults=50"
```

レスポンスに `nextPageToken` が含まれる場合は、`--data-urlencode "nextPageToken=..."` を付けて続きを取得する (50 件超のときのみ)。

### 3. 表形式で出力

各 Key は `JIRA_BASE_URL/browse/<KEY>` へのリンクにする。

| Key | Summary | Status | Priority | Due |
|---|---|---|---|---|
| [ABC-123](https://your-domain.atlassian.net/browse/ABC-123) | ログイン画面のリファクタ | In Progress | High | 2026-05-22 |

末尾に件数と、「詳細を見たいチケットキーがあれば指示してください」と一言添える。

### 4. 0 件のとき

JQL と件数 0 を明示し、原因切り分け候補（フィルタを緩める／別アカウントで認証していないか）を 1-2 行で提示する。

## モード B: 詳細表示

### 1. Issue 本体を取得

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -G "$JIRA_BASE_URL/rest/api/3/issue/<KEY>" \
  --data-urlencode "fields=summary,status,priority,duedate,assignee,reporter,labels,description,subtasks" \
  --data-urlencode "expand=renderedFields"
```

`renderedFields.description` に HTML 化された本文が入るので、ADF JSON を自力で展開せずにここからプレーンテキスト化する (タグ除去のみで十分)。

### 2. 最新コメントを取得

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Accept: application/json" \
  -G "$JIRA_BASE_URL/rest/api/3/issue/<KEY>/comment" \
  --data-urlencode "orderBy=-created" \
  --data-urlencode "maxResults=5" \
  --data-urlencode "expand=renderedBody"
```

### 3. 次の構成で出力

```text
## [<KEY>] <Summary>
<JIRA_BASE_URL/browse/<KEY>>

- Status: <status.name>
- Priority: <priority.name>
- Due: <duedate or "未設定">
- Assignee: <assignee.displayName>
- Reporter: <reporter.displayName>
- Labels: <labels join ", ">

### Description
<本文。長文 (概ね 600 字超) なら冒頭 + 要点に要約し、その旨を明記>

### サブタスク (n 件)
| Key | Summary | Status |
| --- | --- | --- |

### 最新コメント (m 件)
- <author.displayName> (<created>): <本文要約 (1-2 文)>
```

サブタスクやコメントが 0 件のセクションは省略する。

### 4. Confluence リンクの扱い

Description やコメントに `<JIRA_BASE_URL>/wiki/...` 形式の Confluence URL が含まれていた場合は、URL を一覧化して提示するに留める (本文の自動取得は行わない)。

```text
### Confluence リンク (n 件)
- <URL 1>
- <URL 2>
```

ユーザーから「このページ読んで」「コンフルも見て」等の依頼があったら、`confluence-page-reader` スキルへ引き継ぐ。

## エラーハンドリング

- HTTP 401 / 403: 認証失敗。env var の値が正しいか、トークンの有効期限が切れていないかをユーザーに確認させる。**トークン値そのものは出力しない**。
- HTTP 404 (詳細モード): チケットキーが存在しないかアクセス権がない。タイポ確認を促す。
- HTTP 429: レート制限。少し待って再試行する旨を伝える。自動リトライはしない。
- JQL 構文エラー (400): リクエストした JQL と Jira 側のエラー本文を提示する。

## やってはいけないこと

- ユーザーに依頼されていないのにチケットを更新する (POST / PUT / DELETE は本スキルでは一切使わない)。
- API トークンやレスポンスの認証ヘッダを出力やコミット対象に含める。
- 環境変数未設定のまま推測値で API を叩く。
- 取得した Description やコメントを丸ごとそのまま長文で貼り付ける (要約してから提示)。
- ページング途中で打ち切ったまま「これで全部」と説明する (`nextPageToken` がある場合は続きを取るか、その旨を明示する)。
