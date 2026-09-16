# 一次資料の棚卸し: Jira/ConfluenceをEM個人のPMハーネスに使う

対象: EM個人のプロジェクト管理を、Todoist/Google Calendar/Gmail/Slack/Granola/Linear等の特殊MCP連携を前提にせず、**Jira と Confluence のみ**で構築する観点での一次資料調査。管理情報（進捗集約結果・検知したリスク・前回サマリ等）はローカルMarkdownで持つ前提（2026-09-17 だいすけ指定、`department.yaml` 反映済み）。
調査日: 2026-09-17

---

## 1. Jira REST API / JQL — 放置課題・期限接近の抽出

### 1-1. エンドポイントは `/rest/api/3/search/jql` に統一済み

出典: [Run JQL search query using Jira Cloud REST API](https://confluence.atlassian.com/jirakb/run-jql-search-query-using-jira-cloud-rest-api-1289424308.html)、[When are JQL search endpoints /rest/api/2/search and /rest/api/3/search really being removed?](https://community.atlassian.com/forums/Jira-questions/When-are-JQL-search-endpoints-rest-api-2-search-and-rest-api-3/qaq-p/3029221)

- 旧 `GET/POST /rest/api/2|3/search` は**2025年10月末までに完全停止済み**。2026年時点で新規実装するなら選択肢はない — 必ず新エンドポイントを使う。
  - GET `/rest/api/3/search` → GET `/rest/api/3/search/jql`
  - POST `/rest/api/3/search` → POST `/rest/api/3/search/jql`
- ページネーションは `startAt` 廃止、**`nextPageToken`** に統一（カーソル方式）。
- `maxResults` は最大10000だが、実運用では数百件程度に抑えてページングする想定（レスポンスサイズ・レート制限の観点）。

### 1-2. 認証

出典: [Basic auth for REST APIs - Jira Cloud platform](https://developer.atlassian.com/cloud/jira/platform/basic-auth-for-rest-apis/)、[Manage API tokens for your Atlassian account](https://support.atlassian.com/atlassian-account/docs/manage-api-tokens-for-your-atlassian-account/)

- Basic認証: `email:APIトークン` を base64 化して `Authorization: Basic ...`、または多くのHTTPクライアントは `--user email:token` で自動化可能。
- APIトークンは Atlassianアカウント設定（プロフィール → Manage account → Security → Create and manage API tokens）から発行。**スコープ付きトークン**（読み取り専用など）の作成が推奨されている — EM個人利用でも「読み取り専用トークン」を使えば誤操作でJiraを書き換えるリスクを下げられる。

### 1-3. curl例（GET、URLエンコード必須）

出典: [Run JQL search query using Jira Cloud REST API](https://confluence.atlassian.com/jirakb/run-jql-search-query-using-jira-cloud-rest-api-1289424308.html)

```bash
curl --request GET \
  --url 'https://<site>.atlassian.net/rest/api/3/search/jql?jql=project%3DOP%20and%20created%20%3E%3D%20-7d&maxResults=100&fields=summary,status,updated,duedate,assignee' \
  --user 'email@example.com:<api_token>' \
  --header 'Accept: application/json'
```

- JQLが長い/複雑な場合は **POST方式**でボディにJQLを直接書ける（URLエンコード不要）。EM個人が複数プロジェクトを横断する複雑なJQLを組むならPOSTが安全。
- `fields` パラメータで返却フィールドを絞れる。個人ダッシュボード用途では `summary,status,updated,duedate,assignee,priority` 程度に絞ってレスポンスを軽量化するのが定石。

### 1-4. 「放置課題」「期限接近」抽出のJQL例

一次資料に厳密な公式サンプルは無いが、JQL構文自体は公式ドキュメント範囲内の標準関数（`now()`, `endOfWeek()` 等）の組み合わせ。

出典（JQL構文の基礎）: [Jira REST API examples](https://developer.atlassian.com/server/jira/platform/jira-rest-api-examples/)、[Rest API Filter based on created or updated date](https://community.atlassian.com/forums/Jira-questions/Rest-API-Filter-based-on-created-or-updated-date/qaq-p/1354197)

```
# 放置課題: 自分がアサインされ、未解決で、14日以上更新なし
assignee = currentUser() AND resolution = Unresolved AND updated <= -14d

# 期限接近: 未解決で、期限が今日〜7日以内
resolution = Unresolved AND duedate >= now() AND duedate <= 7d

# 期限超過: 未解決で期限切れ
resolution = Unresolved AND duedate < now()
```

複数プロジェクトを横断する場合は `project in (PROJ1, PROJ2, PROJ3)` を先頭に足す。EM個人が自分の関与プロジェクト一覧をローカルMarkdown（例: `state/projects.md`）で管理し、そこからJQLの `project in (...)` 句を組み立てる設計が「特殊MCP連携なし」の前提に合致する。

### 1-5. レスポンス形式・注意点

- レスポンスはJSON。`issues[]` 配列に `key`, `fields.{summary,status,updated,duedate,assignee,...}` を含む。`nextPageToken` があれば次ページが存在。
- JQLの日付比較 (`updated`, `duedate`) はタイムゾーンやカレンダー日の扱いで直感と異なる挙動が報告されている（[JQL on search with updated date time does not work correctly](https://community.developer.atlassian.com/t/jql-on-search-with-updated-date-time-does-not-work-correctly/58754)）。EM個人が閾値ベースの検知ロジックを組む際は、境界値（ちょうど14日目など）のズレを許容する設計にするのが無難。

---

## 2. Confluence REST API — ページ作成・更新（週報・サマリの格納先）

### 2-1. v2 API が現行の推奨系統

出典: [Confluence Cloud REST v2 page API](https://developer.atlassian.com/cloud/confluence/rest/v2/api-group-page/)、[Release of v2 Confluence REST API for Pages and Blogposts](https://community.developer.atlassian.com/t/release-of-v2-confluence-rest-api-for-pages-and-blogposts-experimental/62164)

- ベースパス: `/wiki/api/v2/`
- ページ作成: `POST /wiki/api/v2/pages`
- ページ更新: `PUT /wiki/api/v2/pages/{id}`
- ページネーションは `limit` + `cursor`（カーソル方式、Jiraと同じ思想）。

### 2-2. ページ作成のリクエストボディ例

出典: [Confluence REST API v2 - struggling to create a page with the new editor（Atlassian Developer Community、公式フォーラム上の実例）](https://community.developer.atlassian.com/t/confluence-rest-api-v2-struggling-to-create-a-page-with-the-new-editor/75235)

```json
{
  "spaceId": "<スペースID>",
  "status": "current",
  "title": "2026-09-17 週報",
  "parentId": "<親ページID（任意）>",
  "body": {
    "representation": "storage",
    "value": "<p>本文（Confluence storage形式のHTML風マークアップ）</p>"
  }
}
```

- `body.value` には `<body>` タグを含めない（内部要素のみ）。
- `body.representation` は `storage`（Confluence独自のXHTML風形式）と `atlas_doc_format`（ADF、Atlassian Document Format）の2系統がある。EM個人がシンプルなMarkdown由来のHTMLを流し込むだけなら `storage` の方が扱いやすい可能性が高い（ADFは構造化JSONで生成コストが高い）。

### 2-3. ページ更新は version 番号のインクリメントが必須

出典: [Page | go-atlassian docs](https://docs.go-atlassian.io/confluence-cloud/v2/page)（非公式SDKドキュメントだが公式APIの実例を引用）

```json
{
  "id": "<ページID>",
  "status": "current",
  "title": "2026-09-17 週報",
  "body": {
    "representation": "storage",
    "value": "<p>更新後の本文</p>"
  },
  "version": {
    "number": 5,
    "message": "週報を更新"
  }
}
```

- `version.number` は**現在のバージョン+1**を明示的に指定する必要がある（自動インクリメントではない）。更新の直前に該当ページをGETして現在のversion番号を取得するのが定石フロー。
- 「毎週同じページを更新し続ける週報」パターンを組むなら、GET→version確認→PUTの3ステップが必須になる点は設計上の考慮点。

### 2-4. 認証

Jiraと同じくAPIトークンによるBasic認証が使える（出典: [Basic auth for REST APIs - Confluence Cloud](https://developer.atlassian.com/cloud/confluence/basic-auth-for-rest-apis/)）。JiraとConfluenceは同一のAtlassianアカウント・同一のAPIトークンを使い回せる（Atlassian Cloud全体で共通のトークン基盤のため）。

---

## 3. Claude CodeからJira/Confluenceを叩く現実的な手段

### 3-1. 公式Atlassian Remote MCP Server（2026年2月GA、Claude初の公式パートナー）

出典: [Introducing Atlassian's Remote Model Context Protocol (MCP) Server](https://www.atlassian.com/blog/confluence/remote-mcp-server)、[Getting started with the Atlassian Remote MCP Server](https://support.atlassian.com/rovo/docs/getting-started-with-the-atlassian-remote-mcp-server/)

- サーバーURL: `https://mcp.atlassian.com/v2/mcp`
- **Claude Codeでの追加コマンド**:
  ```bash
  claude mcp add --transport http atlassian https://mcp.atlassian.com/v2/mcp
  ```
  追加後 `/mcp` コマンドでOAuth認証フローを開始する。
- 認証はOAuth 2.1（ユーザーの既存Jira/Confluence権限をそのまま尊重）。APIトークン認証もオプションとして選べる旨の記載あり（詳細な切替手順は本調査では未確認）。
- 対応範囲: Jira / Jira Service Management / Confluence / Bitbucket / Compass / Goals / Loom。検索・作成・更新・自動化が可能と明記。
- 要検証（本調査では裏取り未完了）: 「クレジット消費が組織の共有プールから引かれる」という記述がサポートページにあった。個人利用（Free/Standardプラン等）での課金体系・利用上限は次回調査で要確認。

### 3-2. カスタムツール実装（REST APIを直接叩く）という代替

MCPサーバーを使わず、Bashツール経由で `curl` を叩く、または軽量なスクリプト（Python/Node）をカスタムスキルとして実装する方法。

- 利点: 依存が増えない（追加のMCP接続・OAuth設定が不要）、`department.yaml` が要求する「ローカルMarkdown中心・特殊MCP連携なしの設計」により忠実。
- 欠点: APIトークンの管理（環境変数化、`.env` 等での秘匿）を自前で設計する必要がある。Jira/Confluenceそれぞれのページネーション（`nextPageToken` / `cursor`）・エラーハンドリングを自前実装する必要がある。
- 本パターン集の主設計としては、**この「カスタムツール（curl直叩き）」路線が department.yaml のガードレール（特殊MCP連携を前提にしない）に最も合致する**。公式MCPサーバーは「参考として触れる」代替案の位置づけが妥当。

### 3-3. 現時点での結論（次回パターン設計への示唆）

- 主設計: Jira REST API (`/rest/api/3/search/jql`) + Confluence REST API (`/wiki/api/v2/pages`) をAPIトークン+Basic認証で直接叩くBashスクリプト/カスタムスキルとして実装し、抽出結果・状態はローカルMarkdownに保存。
- 参考事例: 公式Atlassian Remote MCP Serverを使えば `claude mcp add` 一発でOAuth接続でき、実装コストは大幅に下がる。ただしクレジット消費体系が未確認なため、個人のコスト管理という観点で軽視できない検証事項が残る。

---

## 次回調査の候補（未着手）

- Atlassian Remote MCP Serverのクレジット消費・利用上限の詳細（個人アカウント/Free-Standardプランでの扱い）
- Confluence `atlas_doc_format`（ADF）の具体的なJSON構造（表・リストを含む週報テンプレートを作る場合に必要になる可能性）
- Jira Webhooks（ポーリングでなくプッシュ通知でリスク検知する代替手段になり得るか）
- Confluence REST API v2 の検索系エンドポイント（既存ページの検索・重複作成防止に必要）
