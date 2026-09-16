# パターン1: 複数プロジェクトの進捗集約・自分用サマリ生成

## 課題

複数プロジェクトを掛け持ちする EM は、各プロジェクトの状況（課題の滞留、期限接近、直近のステータス変化）を毎朝/毎週、自分の頭の中で手作業で集約している。本パターンは Jira（課題管理）と Confluence（レポートの格納先）だけで完結する構成を前提とする — Todoist/Google Calendar/Gmail/Slack/Granola/Linear のような特殊な MCP 連携は前提にしない。特殊連携なしでも、複数の Jira プロジェクトを横断して見る作業自体がボトルネックになり、後回しにされがちで、気づいたときには手遅れになる。

## ゴール

EM 本人だけが使う「自分専用のハーネス」に、担当プロジェクト一覧を定期的に自動で見に行かせ、要点だけをまとめたサマリを生成させる。チームへの展開は行わない（本人のワークフロー改善が目的）。

## 設計方針

- **横断先は Jira プロジェクトキー単位の宣言的リストで持つ** — 対象プロジェクトを Jira のプロジェクトキー（例: `PROJA`）で列挙し、追加・削除は設定ファイルの編集だけで完結させる。ハーネス本体のロジックに個別プロジェクト名をハードコードしない。
- **サブエージェントはユーザーレベルスコープ（`~/.claude/agents/`）に置く** — サブエージェント定義には `~/.claude/agents/`（ユーザーレベル）と `.claude/agents/`（プロジェクトレベル）の2スコープがあり、プロジェクトレベルは単一プロジェクト限定で複数リポジトリ横断には使えない。個人PM用の「進捗集約エージェント」は `~/.claude/agents/` に1つ定義し、どのディレクトリで起動した Claude Code からも同じエージェントを呼び出せるようにする（出典: [Subagents docs](https://code.claude.com/docs/en/sub-agents)）。
- **収集は Jira REST API の JQL 検索、書き込みは行わない** — 各プロジェクトの状態取得は `POST /rest/api/3/search/jql` に JQL を渡して行う（旧 `/rest/api/2|3/search` は2025年10月に廃止済み。出典: `research/03-jira-confluence-api-for-personal-pm.md` §1-1）。認証は Atlassian API トークンによる Basic 認証で、可能な限り**スコープ付き（読み取り専用）トークン**を使う（同 §1-2）。このハーネスの役割は「見て要約する」ことで、Jira 課題への書き込み（コメント追加・ステータス変更など）は本パターンのスコープ外。
- **集約はサブエージェント並列 + 突合** — プロジェクトごとの JQL 検索は独立なので、プロジェクト数分のサブエージェントを並列 spawn して現状を集め、最後に1つの統合ステップで要約する。生ログはそのまま出力せず、統合済みサマリだけを最終成果物とする。
- **定期実行は用途に応じてスケジュール実行手段を使い分ける** — 詳細は後述「スケジュール実行手段の使い分け」参照。手動で毎回起動しない。
- **差分ベースで書く** — 毎回ゼロから全件を洗うのではなく、前回サマリ（もしくは前回実行時刻）との差分に注目させる。変化がない項目は「変化なし」の一行で済ませ、変化があった項目に紙幅を割く。これによりサマリが「読めば数十秒で終わる」量に保たれる。
- **中間状態はローカル Markdown、確定版は Confluence** — 集約の作業状態（差分比較の基準になる前回サマリ）はローカル Markdown ファイルで持つ。EM 本人以外が見る必要のある確定版（週報・サマリ）だけを Confluence ページとして publish する。この2層を混ぜない — ローカル Markdown は使い捨て可能な作業用状態、Confluence は共有される最終成果物、という役割分担を保つ。

## 構成イメージ

```
personal-pm-harness/
├── config/
│   └── projects.yaml          # 追跡対象の Jira プロジェクトキー・publish先 Confluence スペース
├── cron/
│   └── jobs.json               # 毎朝/毎週の実行スケジュール定義
├── state/
│   └── rollup-YYYY-MM-DD.md    # 実行日ごとの集約結果（差分比較の基準は直近ファイル）
└── skills/
    └── progress-rollup/
        └── SKILL.md             # 集約〜publishの手順を書いたプレイブック
```

`config/projects.yaml` の例:

```yaml
projects:
  - name: project-a
    jiraProjectKey: PROJA
    jqlExtra: "AND labels = personal-pm"   # プロジェクト単位で絞り込みを追加したい場合
  - name: project-b
    jiraProjectKey: PROJB

publish:
  confluence:
    spaceId: "123456"        # 週報/サマリの格納先スペース
    parentPageId: "789012"   # 既存の「週報」親ページ（任意）
```

対象プロジェクトが増えても、この YAML にエントリを追加するだけで済む。JQL の組み立て時は `project in (PROJA, PROJB)` のように `projects[].jiraProjectKey` を機械的に連結する。

## スケジュール実行手段の使い分け

Claude Code には性質の異なる複数のスケジュール実行手段があり、個人PMハーネスでは目的に応じて選ぶ必要がある（出典: [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks)）。

| 手段 | 実行場所 | PC起動要否 | ローカルファイルアクセス | 最小間隔 | 個人PM用途での向き不向き |
|---|---|---|---|---|---|
| `/loop`（セッション内） | 自分のマシン | 必要（セッション起動中） | 可 | 1分 | ローカルリポジトリを直接見る集約タスクに向くが、セッションが閉じると消え7日で失効。常設運用には不向き |
| Routines（クラウド） | Anthropic管理クラウド | 不要 | 不可（都度クローン） | 1時間 | PCを閉じていても動く「毎朝のサマリ生成」の本命。GitHubイベントトリガーとも組み合わせ可能 |
| Desktop scheduled tasks | 自分のマシン | 必要 | 可 | 1分 | ローカルファイル・APIトークン設定を使う集約タスクを、`/loop`より安定して常設したい場合に向く |
| GitHub Actions（`on: schedule`） | GitHub | 不要 | リポジトリ内のみ | 任意 | 1リポジトリ単位。複数リポジトリを横断するには各リポジトリに同じワークフローを設置する必要がある |

個人の進捗集約ハーネスでは、**「毎朝のサマリ生成」は Routines、ローカル環境に依存する調査ステップがある場合は Desktop scheduled tasks**、を軸にするのが実用的。`/loop` はセッションを開きっぱなしにする用途（作業中の一時的な監視）に限定し、常設の定期実行には使わない。

## 実行フロー

1. Routines（または Desktop scheduled task）が定期起動し、`progress-rollup` スキルを呼ぶ。
2. スキルが `config/projects.yaml` を読み、Jira プロジェクトキーごとにサブエージェントを並列 spawn する。各サブエージェントは `POST /rest/api/3/search/jql` に `project = <key>` を含む JQL を投げ、課題一覧（`summary,status,updated,duedate,assignee` 程度に絞ったフィールド）を読み取り専用トークンで取得する。
3. 各サブエージェントの結果を本体プロセスが受け取り、`state/` 配下の直近の `rollup-YYYY-MM-DD.md` と突合して「新規に発生した変化」「解消した項目」「変化なし」に分類する。
4. 統合サマリを生成し、`state/rollup-YYYY-MM-DD.md`（本日分）としてローカルに書き出す。
5. 確定版として Confluence へ publish する — 週報ページが既存なら GET `/wiki/api/v2/pages/{id}` で現在の `version.number` を取得し、+1 した値を指定して PUT で本文を更新する。新規ページなら `spaceId` を指定して POST `/wiki/api/v2/pages` で作成する（`version.number` は自動インクリメントされないため、更新の直前に必ず GET する）。

## サマリのフォーマット例

```
## 進捗サマリ 2026-09-17

### 要対応（新規）
- project-a: PROJA-123 が2日レビュー未着手
- project-b: PROJB-45 の期限が本日中

### 進行中（変化なし）
- project-c: 予定通り

### 解消
- project-a: PROJA-45 クローズ
```

## 注意点・落とし穴

- **サマリの粒度が肥大化しやすい** — 差分ベースにしないと、プロジェクト数が増えるごとに読む時間が線形に増え、結局読まれなくなる。「変化なし」を積極的に圧縮する設計を崩さないこと。
- **publish先の一本化** — 確定版の出力先を分散させると「どこを見ればいいか」が曖昧になり、結局手動確認に戻ってしまう。Confluence の固定ページ（同じ親ページ配下、命名規則を統一）に集約する。
- **Confluence の version 番号を GET し忘れない** — `version.number` は自動インクリメントされないため、直前の GET を省略すると更新が 409 で失敗する。「GET→version確認→PUT」の3ステップを skill 側で必ず固定する。
- **JQL の日付境界のズレに注意** — `updated`/`duedate` の比較はタイムゾーン・カレンダー日の扱いで直感と異なる挙動が報告されている（`research/03-jira-confluence-api-for-personal-pm.md` §1-5）。「ちょうど14日目」のような境界値のブレは許容する設計にする。
- **権限スコープを広げすぎない** — 「ついでに書き込みもできると便利」という誘惑があるが、本パターンは集約・可視化が目的。Jira 課題への書き込み系アクションは別パターン（意思決定支援や委譲管理）に切り出す。Confluence への publish は「確定版の格納」という一点に限定し、それ以外の書き込みは行わない。
- **並列調査に Agent teams は使わない** — サブエージェント並列 spawn（本パターンの核）と、実験的機能の Agent teams（複数の Claude Code インスタンスが協調する仕組み）は別物。Agent teams はプランモード時にトークン消費が通常セッションの**約7倍**になるとコスト文書に明記されており、個人利用でも軽視できないコスト要因（出典: [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)）。本パターンの集約は素のサブエージェント並列で十分であり、Agent teams を使う理由はない。
- **コストは `/usage` の `Loops` 行で可視化する** — v2.1.242 以降の `/usage` コマンドには `Loops` 行があり、直近実行された `/loop`・スケジュールタスクごとのトークン消費・実行回数・最終実行時刻を確認できる。定期実行を増やしていくと「どの定期タスクがコストを食っているか」が見えにくくなるため、運用に組み込んで定期的にチェックする（出典: [Manage costs effectively](https://code.claude.com/docs/en/costs)）。

## 関連パターン

- [リスク・ブロッカーの早期検知](../risk-detection/README.md) — 本パターンで集約した情報を元に、放置・遅延の兆候をアラートするパターン。集約ステップは共通化できる。

---

> **改訂履歴**: 初版は一般知識ベースで作成。`research/01-official-features-for-personal-pm.md`（scout棚卸し）を反映し、スケジュール実行の使い分け・サブエージェントのスコープ・Agent teamsのコスト特性・`/usage` Loops行を出典付きで更新。その後、Todoist/Calendar/Gmail/Slack/Granola/Linear等の特殊MCP連携を前提としない方針（だいすけ指定）に合わせ、`research/03-jira-confluence-api-for-personal-pm.md` を反映し、収集をJira JQL検索・確定版のpublish先をConfluenceページ・中間状態をローカルMarkdownとする構成に全面改訂（2026-09-17）。
