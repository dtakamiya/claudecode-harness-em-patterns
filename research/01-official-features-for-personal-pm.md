# 一次資料の棚卸し: 個人のプロジェクト管理に使えるClaude Code機能

対象: EM/マネージャーが「自分個人のプロジェクト管理」にClaude Code / Claude Agent SDKを転用する観点での公式機能棚卸し。チーム展開・権限付与・オンボーディング系はスコープ外。
調査日: 2026-09-17

---

## 1. スケジュール実行（cron/定期実行）

Claude Codeには**3種類**のスケジュール実行手段があり、性質が大きく異なる。個人のPMハーネスを設計する際は用途に応じて使い分ける必要がある。

出典: [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks)

| | Cloud (Routines) | Desktop | `/loop`（セッション内） |
|---|---|---|---|
| 実行場所 | Anthropic管理クラウド | 自分のマシン | 自分のマシン |
| PC起動要否 | 不要 | 必要 | 必要（かつセッション起動中） |
| 再起動後も継続 | 継続 | 継続 | `--resume`で復元（例外あり） |
| ローカルファイルアクセス | 不可（都度クローン） | 可 | 可 |
| 最小間隔 | 1時間 | 1分 | 1分 |

### 1-1. `/loop`（セッション内スケジュール）
出典: [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks)

- `/loop 5m check the deploy` のように間隔＋プロンプトを指定すると内部でcron式に変換されて定期実行される。
- プロンプトを省略すると Claude が「短い/長い待ち時間」を動的に選ぶ自己ペース型ループになる（ビルド中は短く、静かなら長く）。
- `.claude/loop.md`（プロジェクト）または `~/.claude/loop.md`（ユーザー）にデフォルトプロンプトを定義でき、bareな `/loop` はこれを実行する。**個人のPM用途では「今日の未完了タスクを確認し、放置3日以上のものをフラグする」のような内容をここに書くパターンが有効**。
- 内部的には `CronCreate` / `CronList` / `CronDelete` ツールを使用。1セッションで最大50個のスケジュールタスクを保持可能。
- 制約: セッションが起動・アイドル状態でないと発火しない。新しい会話を始めるとタスクは消える。7日で自動失効。

### 1-2. Routines（クラウド）
出典: [Automate work with routines](https://code.claude.com/docs/en/routines)

- プロンプト・対象リポジトリ・コネクタ（MCP）をパッケージ化し、Anthropic管理クラウドで実行。PCを閉じていても動く。
- トリガーは Scheduled（定期）/ API（HTTP POST）/ GitHub イベントの3種、組み合わせ可能。
- 個人PM用途の具体例（公式サンプルより）: 「バックログ整理を毎晩実行」「デプロイ後にAPIトリガーでスモークチェック＆go/no-go判定をSlack投稿」。
- `/schedule` コマンドでCLIから会話的に作成可能（例: `/schedule daily PR review at 9am`）。一度きりのリマインダーも `/schedule in 2 weeks, open a cleanup PR` のように作成できる。
- Pro/Max/Team/Enterpriseで利用可、1日あたりのルーチン実行回数に上限あり。
- 制約: 最小間隔1時間、実行はGitHubアカウントの自分の身元で行われる（コミット・PRは本人名義）。

### 1-3. Desktop scheduled tasks
出典: [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks) 内比較表（詳細ページ `/docs/en/desktop-scheduled-tasks` は本調査では個別確認していない）

- ローカルマシン上で動く定期タスク。ローカルファイル・MCP設定ファイルにアクセス可、権限プロンプトをタスクごとに設定可能。

### 1-4. GitHub Actionsのcronトリガー
出典: [Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions)

- `on: schedule` でGitHub Actions上でClaude Codeをcron実行できる。公式サンプルは「毎日09:00 UTCに前日のコミット・オープンイシューのサマリーをワークフローログに生成」。
- 個人の複数リポジトリ横断ではなく1リポジトリ単位のワークフローだが、複数リポジトリそれぞれに同じワークフローを設置すれば疑似的に横断できる。
- 注意点: GitHubはデフォルトブランチからのみスケジュール実行し、パブリックリポジトリでは60日間活動がないと自動的に無効化される。

---

## 2. 複数プロジェクト/リポジトリ横断のサブエージェント

### 2-1. Subagents（ユーザーレベルスコープ）
出典: [Subagents docs](https://code.claude.com/docs/en/sub-agents)（WebFetch要約。原文の章立ては別途確認要）

- サブエージェント定義は4つのスコープを持つ: `~/.claude/agents/`（ユーザーレベル・**全プロジェクト横断で使える**）、`.claude/agents/`（プロジェクトレベル・単一プロジェクト限定）、managed settings（組織全体）、`--agents` CLIフラグ（そのセッション限定）。
- **個人のPM用途で重要なのはユーザーレベルスコープ**: 例えば「進捗集約エージェント」を `~/.claude/agents/` に1つ定義すれば、どのプロジェクトのディレクトリで起動したClaude Codeからも同じエージェントを呼び出せる。
- 優先順位: managed settings > `--agents` CLIフラグ > プロジェクトレベル > ユーザーレベル > プラグイン。

### 2-2. Agent teams（実験的機能）
出典: [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)

- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` で有効化する実験的機能。複数のClaude Codeインスタンス（チームメイト）が共有タスクリストとメッセージングで協調する。
- 個人のPM用途としては「並列調査・レビュー」に向く（例: PRを security/performance/test-coverage の3観点で同時レビュー、5つの仮説を競わせてデバッグ）。単一プロジェクト内の並列化が主眼で、**複数リポジトリを横断する明示的な機能ではない**（各チームメイトは同じプロジェクトコンテキストをロードする）。
- トークン消費が通常セッションの約7倍（プランモード時）になるとコスト文書に明記されており、個人利用でも軽視できないコスト要因。
- 制約: 1セッション1チームのみ、ネストしたチーム不可、リード役の交代不可。

### 2-3. Cross-session messaging（補足・複数セッション横断の軽量手段）
出典: [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams) 内での言及

- Agent teamsより軽量な代替として、「自分が起動した複数のセッション間でClaudeが知見を渡し合う」cross-session messaging機能がある。複数プロジェクトをそれぞれ別セッションで開いている個人ユーザーが、プロジェクト間で情報を受け渡す用途に近い。詳細ページ（`/docs/en/cross-session-messaging`）は本調査では未取得、次回サイクルでの深掘り候補。

---

## 3. レポート生成

### 3-1. `/insights`（使い方の分析レポート）
出典: [Manage costs effectively](https://code.claude.com/docs/en/costs) の「Analyze your usage patterns」セクション

- 直近セッション（最大200件、未分析のもの）を分析し、「何に取り組んでいるか」「誤解やバグ等のフリクションポイント」「改善提案」を含むHTMLレポートを `~/.claude/usage-data/report.html` に生成。タイムスタンプ付きで過去分も保存される。
- 個人のPM文脈では「自分がどのプロジェクトにどれだけ時間を使っているか」の振り返りに転用できる一次データ。

### 3-2. Routines/GitHub Actionsのプロンプトによるレポート生成
出典: [Automate work with routines](https://code.claude.com/docs/en/routines)、[Claude Code GitHub Actions](https://code.claude.com/docs/en/github-actions)

- 「レポート生成」自体の専用機能は無く、Routines/`/loop`/GitHub Actionsのプロンプトに「サマリーを作って」と指示する形で実現する。専用テンプレートやAPIは公式には存在しない（後述タスク2のAcademy記事が実務パターンとして詳しい）。

---

## 4. usage/cost可視化

出典: [Manage costs effectively](https://code.claude.com/docs/en/costs)

個人利用（Pro/Max、組織を持たない）でも使える機能:

- **`/usage`コマンド**: セッションごとのトークン使用量・推定コストをローカルで表示。Session blockの他、Pro/Max/Team/Enterpriseでは直近24時間/7日間の「何に使われたか」の内訳（skills/subagents/plugins/MCPサーバー別の割合）も見られる（`d`/`w`キーで切替）。
  - 重要な注意点: 表示コストはローカル計算の**見積もり**であり、正式な請求根拠ではない（正式には Claude Console の Usage ページ）。
  - v2.1.242以降の `Loops` 行では、直近実行された `/loop`やスケジュールタスクごとのトークン消費・実行回数・最終実行時刻が見られる。個人が「どの定期タスクがコストを食っているか」を把握するのに直接使える。
- **OpenTelemetry (OTel) エクスポート**: 環境変数 `CLAUDE_CODE_ENABLE_TELEMETRY=1` + `OTEL_METRICS_EXPORTER=console`（または`otlp`で自前バックエンドへ）でセッションコスト・トークン使用量・アクティブ時間などをリアルタイムに取得できる。個人でも `console` エクスポーターだけなら追加インフラ不要ですぐ試せる。
  - 主要メトリクス: `claude_code.cost.usage`、`claude_code.token.usage`、`claude_code.active_time.total`、`claude_code.lines_of_code.count`。
- **Claude Agent SDKのコストトラッキング**: `query()` 呼び出しごとに `total_cost_usd` / `modelUsage`（モデル別内訳）を返す。自前のハーネス（例: 複数プロジェクトを横断して回すカスタムスクリプト）を書く場合、このAPIでプロジェクト別・タスク別のコスト集計を自作できる。
  - 出典: [Track cost and usage (Agent SDK)](https://code.claude.com/docs/en/agent-sdk/cost-tracking)
  - 注意: `total_cost_usd` はSDK内蔵の価格表によるクライアント側推定であり、正式な請求データではない。
  - サブエージェントを使うと `usage` フィールドはサブエージェント分を含まないため、`total_cost_usd` または `modelUsage` を使う必要がある（ここを間違えると個人の複数プロジェクト集計で過少カウントする落とし穴になる）。

---

## 次回調査の候補（未着手）

- `/docs/en/desktop-scheduled-tasks` の原文確認（比較表からの推測のみで未フェッチ）
- `/docs/en/cross-session-messaging` の原文確認
- `/docs/en/goal`（`/goal`機能）— 「セッションを条件に向けて回し続ける」機能。個人のPM用途（目標駆動のタスク遂行）に転用できる可能性
- Claude Code Analytics API（Console/Enterprise向け）が個人利用でも使えるか否かの確認
