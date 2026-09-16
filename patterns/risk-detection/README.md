# パターン2: リスク・ブロッカーの早期検知

## 課題

複数プロジェクトを抱える EM は、個々のタスクの遅延・放置に気づくタイミングが遅れがちになる。「1週間レビュー待ちの PR」「2週間更新のない Issue」「期限が近いのに進捗のないタスク」は、能動的に探しに行かない限り見えない。EM 本人がすべてのボードを毎日目視するのは非現実的であり、気づいたときにはすでにスケジュールへの影響が出ている。

本パターンは [パターン1: 進捗集約](../personal-progress-rollup/README.md) の発展形で、「集約して見せる」から一歩進み、「閾値を超えたものだけを能動的にアラートする」ことを目的とする。

## ゴール

EM 自身のハーネスに、担当プロジェクト群の中から「放置」「遅延兆候」を定義済みのルールで検知させ、対応が必要なものだけを浮き上がらせる。ノイズを増やさず、見逃しを減らす。

## 設計方針

- **検知ルールは JQL に直接落とし込む** — 「N 日更新されていない」「期限が過ぎている/近い」は Jira の JQL 標準関数（`now()` との比較）でそのまま表現できる。後段のコードでフィルタするのではなく、ルールごとに JQL 文字列として `config/risk-rules.yaml` に持たせ、Jira 側に検索させる（出典: `research/03-jira-confluence-api-for-personal-pm.md` §1-4）。曖昧な「なんとなく怪しい」判定に頼ると誤検知が増え、アラート自体が信用されなくなる。
- **自己申告の進捗を鵜呑みにしない** — Claude Academy の公式ユースケースでは、147件のメール・312件のSlackメッセージから状況を抽出した際、「80%完了」という自己申告と実際の証跡から推定した約45%の実態との乖離が報告されている（出典: [Generate project status reports](https://academy.claude.com/use-cases/generate-project-status-reports)）。本パターンでは外部ツール（メール・Slack）を前提にしないため、この乖離検知は Jira 課題自身の一次証跡——ステータス遷移履歴（`GET /rest/api/3/issue/{id}/changelog`）——とコメント上の自己申告を突き合わせる応用ルールとして扱う（changelog APIの詳細は本調査では未検証、次回調査候補）。
- **進捗集約ステップを再利用する** — 情報収集（各プロジェクトの JQL 検索）は[パターン1](../personal-progress-rollup/README.md)と共通化できる。本パターンは集約結果に対してルールを適用する「フィルタ層」として設計し、二重に情報を取りに行かない。
- **重要度を分けて出力する** — すべてを同じ扱いで並べると埋もれる。「今すぐ対応が要る」「様子見でよい」の最低2段階に分け、上位だけを即時扱い、下位は定期サマリの末尾に留める。
- **誤検知の許容度を明示する** — 完璧な検知は目指さない。閾値は最初は緩めに設定し、運用しながら EM 本人がチューニングする前提で設計する（閾値も `config` に外出しし、コード変更なしで調整できるようにする）。
- **「即時通知」は Slack 等の外部チャンネルに頼らず、ローカル/Confluence 側で完結させる** — Todoist/Calendar/Gmail/Slack/Granola/Linear のような特殊 MCP 連携は前提にしない。重要度「高」は検知した瞬間に `state/alerts-today.md`（ローカル Markdown）へ追記し、次回セッション起動時・次回スキャン時に必ず目に入るようにする。重要度「中」は定期サマリ（パターン1で Confluence へ publish するページ）にまとめて記載する。
- **書き込み・自動アクションは行わない** — 本パターンは検知とローカル/Confluenceへの記録が範囲。「Jira課題にコメントを書く」「自動でアサインし直す」といった Jira 側への書き込み系アクションは、EM の意思決定を経ずに実行すべきでないため、明確にスコープ外とする。

## 検知ルール例

| ルール | JQL | 重要度 |
|---|---|---|
| 課題放置 | `resolution = Unresolved AND updated <= -14d` | 中 |
| 期限接近 | `resolution = Unresolved AND duedate >= now() AND duedate <= 7d` | 高 |
| 期限超過 | `resolution = Unresolved AND duedate < now()` | 高 |
| 担当者不明 | `resolution = Unresolved AND assignee is EMPTY AND priority = High` | 中 |
| 自己申告と証跡の乖離（応用） | JQLだけでは表現できず、対象課題の `changelog` を別途取得してステータス遷移の停滞とコメント上の自己申告を突き合わせる | 高 |

いずれも先頭に `project in (PROJA, PROJB)` を足して対象プロジェクトを絞る（[パターン1](../personal-progress-rollup/README.md)の `config/projects.yaml` から機械的に組み立てる）。閾値（`-14d`, `7d` など）は設定ファイルに外出しし、プロジェクトごとに上書き可能にする。

## 構成イメージ

```
personal-pm-harness/
├── config/
│   ├── projects.yaml          # パターン1と共通
│   └── risk-rules.yaml         # 検知ルール（JQLフラグメント）と閾値の定義（プロジェクト単位で上書き可）
├── cron/
│   └── jobs.json               # 定期スキャン + 即時アラート追記トリガー
├── state/
│   ├── known-risks.json        # 既に検知済みのリスク項目（ローカルJSON、重複通知の抑止）
│   └── alerts-today.md         # 重要度「高」の即時アラート（ローカルMarkdown）
└── skills/
    └── risk-detection/
        └── SKILL.md             # 検知〜記録の手順を書いたプレイブック
```

`config/risk-rules.yaml` の例:

```yaml
defaults:
  stale_days: 14        # 課題放置
  duedate_window_days: 7 # 期限接近

overrides:
  project-a:
    stale_days: 7   # 対応スピードが求められるプロジェクトは閾値を厳しく
```

`known-risks.json`・`alerts-today.md` はいずれもローカルファイルであり、外部の通知サービス（Slack等）には依存しない。

## 実行フロー

1. cron が定期起動し（例: 平日朝、または数時間おき）、`risk-detection` スキルを呼ぶ。
2. `config/risk-rules.yaml` の閾値から、検知ルールごとの JQL を組み立てる（`project in (...)` はパターン1の `config/projects.yaml` から機械的に生成）。
3. 各 JQL を `POST /rest/api/3/search/jql` に投げ、条件を満たす課題を直接取得する（後段でのフィルタリングは行わない — Jira 側の検索に閾値判定を委ねる）。
4. `state/known-risks.json`（ローカルJSON）と突合し、**新規に検知したリスクのみ**を処理する（同じ項目を毎回再処理しない）。解消された項目は state から取り除く。
5. 重要度「高」は検知した瞬間に `state/alerts-today.md`（ローカルMarkdown）へ追記する。重要度「中」は[パターン1](../personal-progress-rollup/README.md)の定期サマリ（Confluenceへpublishする週報）にまとめて記載する。

## アラートフォーマット例（`state/alerts-today.md`）

```
⚠️ リスク検知 2026-09-17

[高] project-a: PROJA-123 が3日レビュー未着手（期限接近）
[高] project-b: PROJB-45 の期限が本日中、進捗更新なし
```

## 参考コラム: MCP全面接続によるさらに広い実例（本パターンの主設計ではない）

本パターンの主設計は Jira/Confluence/ローカルファイルのみで完結させることだが、より広いツール群にMCP接続できる環境であれば以下のような先進事例もある（参考として紹介するのみで、本パターンの構成には組み込まない）。

あるEMが Claude Code を MCP 経由で Todoist（タスク）・Google Calendar・Gmail・Slack・Granola（会議メモ）・Linear（エンジニアリングPM）に接続し、詳細な `CLAUDE.md`（チーム構造・タスクシステム・優先順位の指示書）をもとに、毎朝自動で以下をフラグさせている（出典: [How I Use AI as an Engineering Manager](https://mentorcruise.com/blog/how-i-use-ai-as-an-engineering-manager-8b46a/)）。

1. **3日以上放置されている項目**
2. **近づいている締め切り**
3. **仕事タスクに押し出されて後回しになっている個人タスク**
4. **注目されていない成長目標（1on1で話した育成目標など）**

「3日放置」のようなシンプルな閾値ベースのフラグの考え方自体は、本パターンの検知ルール（前掲テーブル）と同じ発想である。ただし本パターンでは、これらを Todoist/Calendar/Slack 等への個別MCP接続ではなく、Jira の JQL 検索とローカルファイルの範囲で実現する。

## 注意点・落とし穴

- **通知疲れ（アラート疲労）** — 閾値を厳しくしすぎると `alerts-today.md` への追記が多発し、無視されるようになる。運用初期は緩めに始め、実際の見逃し/誤検知を見ながら調整する。
- **重複記録** — 状態を保持せずに毎回全件記録すると、同じ問題を何度も書き足してノイズになる。`known-risks.json`（ローカルJSON）のような検知済み状態の記録は必須。
- **ルールの陳腐化** — プロジェクトのフェーズが変わる（例: リリース直前は許容閾値を下げたい）のに閾値が固定だと、精度が落ちる。閾値を設定ファイル化し、EM 本人が容易に調整できることを維持する。
- **検知だけで終わらせない** — アラートを出しても EM 本人が見なければ意味がない。`alerts-today.md` はセッション開始時に必ず読む固定パス、定期サマリは [パターン1](../personal-progress-rollup/README.md) と同じ Confluence の固定ページに集約する。
- **JQLに閾値判定を委ねすぎて隠れコストを見落とさない** — Jira側で `updated`/`duedate` の判定をさせる設計は境界値のブレを含む（`research/03-jira-confluence-api-for-personal-pm.md` §1-5）。境界付近の課題を「検知漏れ」として過信しない。

## 関連パターン

- [複数プロジェクトの進捗集約・自分用サマリ生成](../personal-progress-rollup/README.md) — 情報収集の基盤。本パターンはこの上にルールベースのフィルタを載せる。

---

> **改訂履歴**: 初版は一般知識ベースで作成。`research/01-official-features-for-personal-pm.md`・`research/02-em-personal-pm-practices.md`（scout調査）を反映し、自己申告vs証跡の突き合わせルール・MCP接続による実例を出典付きで更新。その後、Todoist/Calendar/Gmail/Slack/Granola/Linear等の特殊MCP連携を前提としない方針（だいすけ指定）に合わせ、`research/03-jira-confluence-api-for-personal-pm.md` を反映し、検知ルールをJQLフラグメントとして直接表現し、状態管理・即時アラートをローカルMarkdown/JSONに一本化する構成に全面改訂。MCP全面接続の実例は参考コラムとして残置（2026-09-17）。
