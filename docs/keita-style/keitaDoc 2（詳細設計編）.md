# ケイタ式 VPS完全自動化システム 詳細設計編

**ドキュメント番号：** Doc 2 / 3
**バージョン：** v2.0 (MAX)
**発行日：** 2026年5月3日
**対象読者：** Doc 1（設計原則編）を読了し、自身のVPS自動化の実装に着手する受講生
**前提：** Doc 1のチェックリストをすべてパスしていること

---

## 0. このドキュメントの位置づけ

本書はDoc 1で確立した設計原則を、実装可能な仕様書まで落とし込んだものである。Doc 2を読みながら自分のリポジトリに各ファイルを配置すれば、Doc 3（実装チェックリスト編）でゼロ実装検証→デプロイへ進める構成にしてある。

| 章 | 内容 |
|---|---|
| 第1章 | リポジトリ全体構造（ファイル配置） |
| 第2章 | CLAUDE.md（User / Project / Directory 各階層）完全版 |
| 第3章 | .claude/rules/ 全ファイル |
| 第4章 | 5体のエージェント完全定義 |
| 第5章 | JSON Schema完全版（7種） |
| 第6章 | Hooks YAML完全版 |
| 第7章 | Slash Commands完全版 |
| 第8章 | MCP server実装スケルトン |
| 第9章 | systemd unit / timer完全版 |
| 第10章 | ログ・モニタリング設計 |
| 第11章 | 障害復旧手順 |

各章のコード・YAML・JSONはそのままコピーして使えるよう、placeholderは`<YOUR_*>`形式で明示している。

---

## 第1章 リポジトリ全体構造

````
~/keita-vps-auto/
├── CLAUDE.md                                # Project層
├── README.md
├── .claude/
│   ├── rules/
│   │   ├── ebay-tos-compliance.md           # alwaysApply: true
│   │   ├── ketai-shiki-philosophy.md        # alwaysApply: true
│   │   ├── profit-margin-15pct-strict.md    # alwaysApply: true
│   │   ├── vero-blacklist.md                # alwaysApply: true
│   │   ├── phase-strategy.md                # globs: agents/profit_judge/**
│   │   ├── rotation-longtail-criteria.md    # globs: agents/profit_judge/**
│   │   └── outsource-protocol.md            # globs: agents/outsource_dispatcher/**
│   ├── commands/
│   │   ├── keita-daily-cycle.md
│   │   ├── keita-supply-check.md
│   │   ├── keita-listing-now.md
│   │   ├── keita-position-check.md
│   │   ├── keita-status.md
│   │   ├── keita-emergency-halt.md
│   │   ├── keita-fx-refresh.md
│   │   └── keita-blacklist-add.md
│   └── hooks.yaml
├── agents/
│   ├── coordinator/
│   │   ├── CLAUDE.md
│   │   └── system_prompt.md
│   ├── seller_watcher/
│   │   ├── CLAUDE.md
│   │   └── system_prompt.md
│   ├── supply_checker/
│   │   ├── CLAUDE.md
│   │   └── system_prompt.md
│   ├── profit_judge/
│   │   ├── CLAUDE.md
│   │   └── system_prompt.md
│   ├── ebay_lister/
│   │   ├── CLAUDE.md
│   │   └── system_prompt.md
│   ├── outsource_dispatcher/
│   │   ├── CLAUDE.md
│   │   └── system_prompt.md
│   └── profit_verifier/
│       ├── CLAUDE.md
│       └── system_prompt.md
├── mcp/
│   ├── ebay-read-mcp/
│   │   ├── server.py
│   │   ├── pyproject.toml
│   │   └── tools/
│   │       ├── fetch_seller_sold_listings.py
│   │       ├── fetch_seller_active_listings.py
│   │       └── fetch_marketplace_insights.py
│   ├── ebay-write-mcp/
│   │   ├── server.py
│   │   └── tools/
│   │       ├── create_inventory_item.py
│   │       ├── submit_feed.py
│   │       ├── update_listing.py
│   │       └── end_listing.py
│   ├── buyee-mcp/
│   │   ├── server.py
│   │   └── tools/
│   │       └── search.py
│   ├── rakuten-mcp/
│   │   ├── server.py
│   │   └── tools/
│   │       └── search.py
│   ├── yahoo-shopping-mcp/
│   │   ├── server.py
│   │   └── tools/
│   │       └── search.py
│   └── fx-mcp/
│       ├── server.py
│       └── tools/
│           └── get_current_rate.py
├── hooks/
│   ├── check_profit_margin.py
│   ├── check_phase_compliance.py
│   ├── check_vero_blacklist.py
│   ├── check_rate_limit.py
│   ├── trim_responses.py
│   ├── update_candidate_status.py
│   └── log_structured_error.py
├── schemas/
│   ├── seller.schema.json
│   ├── listing.schema.json
│   ├── sold_listing_digest.schema.json
│   ├── supply_match.schema.json
│   ├── listing_draft.schema.json
│   ├── order_event.schema.json
│   └── outsource_task.schema.json
├── data/
│   ├── jsonl/
│   │   ├── sellers.jsonl
│   │   ├── digests.jsonl
│   │   ├── matches.jsonl
│   │   ├── drafts.jsonl
│   │   ├── orders.jsonl
│   │   └── fx_rates.jsonl
│   ├── sqlite/
│   │   └── keita.db
│   └── logs/
├── config/
│   ├── config.yaml
│   ├── phase_strategy.yaml
│   ├── shipping_rates.yaml
│   └── secrets.env                          # .gitignore対象
├── tests/
│   ├── test_profit_calculator.py
│   ├── test_phase_compliance.py
│   ├── test_vero_blacklist.py
│   └── fixtures/
│       └── sample_sold_listing_digest.json
└── systemd/
    ├── keita-daily-cycle.service
    ├── keita-daily-cycle.timer
    ├── keita-position-check.service
    ├── keita-position-check.timer
    ├── keita-inventory-check.service
    ├── keita-inventory-check.timer
    ├── keita-seller-rescore.service
    ├── keita-seller-rescore.timer
    └── keita-order-webhook.service
````

---

## 第2章 CLAUDE.md完全版

### 2-1. User層 `~/.claude/CLAUDE.md`

````markdown
# Anthropic公式準拠・全プロジェクト共通ルール

## 10の鉄則（毎セッション冒頭で確認）

1. `stop_reason == "end_turn"` が唯一信頼できる完了サイン
2. サブエージェント間は直接通信しない（必ずCoordinator経由）
3. サブエージェントは親文脈を継承しない
4. ツールの選択基準はdescription（名前ではない）
5. 業務ルール強制はHookで（プロンプトは確率的）
6. 構造化出力は `tool_use + JSON Schema`
7. JSON Schemaは構文エラーを防ぐが意味エラーは防げない
8. Lost in the Middle：重要情報は冒頭か末尾
9. エスカレーションは明示的トリガー
10. CI/CDではHeadless（`-p`）モード必須

## 違反検出時

設計や実装の途中で上記10の鉄則への違反を発見したら：

1. 作業を即座に一時停止
2. 「⚠️ Anthropic公式ルール違反を検出」と明示
3. 違反した条項番号と理由を報告
4. 修正方針を提示してユーザーの合意を得る
5. 合意後に作業を再開

## 公式単語の優先使用

Hub-and-Spoke / Coordinator / Subagent / Principle of Least Privilege / 
Generator-Verifier Pattern / Fixed Pipeline / Dynamic Adaptive Decomposition /
tool_use + JSON Schema / Hook（PreToolUse / PostToolUse）/ Headless Mode /
Lost in the Middle / Prompt Caching / Batch API
````

### 2-2. Project層 `~/keita-vps-auto/CLAUDE.md`

````markdown
# ケイタ式VPS完全自動化システム — Project層ルール

## このプロジェクトの目的

24時間365日、ケイタ式に準拠した利益率15%以上の出品候補を eBay へ自動出品し、
注文受領時に外注メルカリ購入タスクを Notion へ自動生成するシステム。

## ケイタ式の絶対原則3つ

1. **利益率15%絶対基準**：出品候補は計算後の純利益率が15%以上でなければ出品しない
2. **評価フェーズ整合**：現在のアカウント評価値に対応するフェーズの「回転/ロング比率」を必ず守る
3. **規約遵守の絶対境界線**：人間の確認なしのend-to-endフロー禁止。仕入れ実行は外注さん（人間）が担当

## 評価フェーズ別戦略（参照：config/phase_strategy.yaml）

| フェーズ | 評価値 | 出品数 | 回転/ロング | 月売上目標 |
|---|---|---|---|---|
| Phase 0 | 0-30 | 30 | 85/15 | $300 |
| Phase 1 | 30-100 | 70 | 70/30 | $800 |
| Phase 2 | 100-300 | 150 | 70/30 | $2,000 |
| Phase 3 | 300-500 | 220 | 70/30 | $5,000 |
| Phase 4 | 500-700 | 270 | 70/30 | $9,000 |
| Phase 5 | 700+ | 300+ | 70/30 | $13,000+ |

## エスカレーション基準

- `confidence < 0.7` → Discord通知 + manual_reviewキューへ
- 3回連続API失敗 → Discord critical + パイプライン停止
- 0.13 ≤ profit_margin < 0.15 → Discord通知（推奨判断、自動出品はしない）
- VeRO警告検知 → Discord critical + blacklist追加

## 5フェーズ責務

| フェーズ | 担当 | モデル |
|---|---|---|
| ① Sold Listing監視 | SA-1 seller_watcher | Haiku 4.5 |
| ② 仕入れ可否確認 | SA-2 supply_checker | Haiku 4.5 |
| ③ 採算判定・分類 | SA-3 profit_judge | Sonnet 4.5 |
| ③v 独立検証（条件付き） | profit_verifier | Sonnet 4.5（別セッション）|
| ④ eBay自動出品 | SA-4 ebay_lister | Sonnet 4.5 |
| ⑤ 注文ハンドリング | SA-5 outsource_dispatcher | Haiku 4.5 |

## 参照

@.claude/rules/ebay-tos-compliance.md
@.claude/rules/ketai-shiki-philosophy.md
@.claude/rules/profit-margin-15pct-strict.md
@.claude/rules/vero-blacklist.md
@.claude/rules/phase-strategy.md
@.claude/rules/rotation-longtail-criteria.md
@.claude/rules/outsource-protocol.md
````

### 2-3. Directory層（各エージェント）

各`agents/<agent_name>/CLAUDE.md`の構造は以下に統一：

````markdown
# <Agent Name> — Directory層ルール

## 役割
（1行で）

## allowed_tools
- tool_name_1
- tool_name_2

## 禁止事項
（このエージェントが絶対にやってはいけないこと）

## Few-shot examples
（3〜5個の入出力例）

## 参照
@system_prompt.md
````

具体的な内容は第4章で各エージェントごとに掲載する。

---

## 第3章 .claude/rules/ 全ファイル

### 3-1. ebay-tos-compliance.md

````markdown
---
alwaysApply: true
---

# eBay規約遵守の絶対境界線

## 2026年2月20日施行のUser Agreement

eBay新User Agreementは、AIエージェントによる自動購入を明示的に禁止した：

> "use any robot, spider, scraper, data mining tools, data gathering and 
> extraction tools, or other automated means (including, without limitation 
> buy-for-me agents, LLM-driven bots, or any end-to-end flow that attempts 
> to place orders without human review) to access our Services for any 
> purpose, except with the prior express permission of eBay"

## システム上の絶対遵守事項

1. **購入実行ツールはシステムに搭載しない**
   - SA-5 outsource_dispatcher には Notion / Discord 通知ツールしか与えない
   - メルカリ・ヤフオク・楽天等での購入実行は、外注さん（人間）が担当する

2. **eBay公式API優先**
   - Browse API / Marketplace Insights API / Inventory API / Feed API を主軸
   - スクレイピングは Marketplace Insights 招待降下までの暫定運用

3. **rate limit遵守**
   - 各APIの公式上限の80%を上限に設定
   - 3回連続失敗で circuit breaker 発動

4. **VeRO（Verified Rights Owner）対応**
   - vero-blacklist.md のブランド・キーワードに該当する商品は出品しない
   - VeRO警告検知時は当該パターンを自動でblacklistへ追加

## 違反時の動作

PreToolUse Hookで検知 → 当該ツール呼び出しをblock → Discord critical通知。
````

### 3-2. ketai-shiki-philosophy.md

````markdown
---
alwaysApply: true
---

# ケイタ式の核心思想

## 需要逆算型

「売れている商品を確認 → 仕入れ可能か確認 → 出品」という順序を絶対に逆転させない。
最初の処理は必ず「ライバルセラーのSold Listing取得」から。
在庫を持つ前に需要を確認する。

## 回転商品 vs ロングテール

| 項目 | 回転商品 | ロングテール |
|---|---|---|
| 定義 | 月2個以上継続して売れる商品 | 月0〜1個・年に数回売れる商品 |
| 単価帯 | $30〜$150 | $100〜$500以上 |
| 役割 | 月売上の土台 | 利益を跳ねさせる |
| ポジション確認頻度 | 毎日〜3日に1回 | 1ヶ月に1回 |

## 評価フェーズ別比率

評価値が低いほど回転商品比率を高める。「売れる体験を先に作る」「アカウントの売買実績を早期に積む」のが目的。
詳細は phase-strategy.md 参照。

## ライバルセラー逆引き

「自分でジャンルを選ばない」。Ship from Japan・コンバージョン5〜10%以上・中古品中心セラーを
1000人規模で集め、優良10〜20人を抽出して真似る。
1人を監視するのと500人を監視するのとで自動化の手間はほぼ変わらない。これがVPSのレバレッジ。

## 利益率15%絶対基準

為替変動・送料変動・eBay手数料変動を吸収するためのバッファ。
出品時点で15%以上を確保することが絶対基準。
profit-margin-15pct-strict.md で物理的に強制する。
````

### 3-3. profit-margin-15pct-strict.md

````markdown
---
alwaysApply: true
---

# 利益率15%絶対基準（Hook強制）

## 計算式

````
revenue_jpy = sold_price_usd × fx_rate × (1 - 0.05_buffer)

ebay_fees_usd =
    sold_price_usd × 0.1315  (FVF)
  + sold_price_usd × 0.0149  (国際手数料)
  + 0.30                     (注文手数料)
  + sold_price_usd × promoted_rate  (Promoted Listings)

ebay_fees_jpy = ebay_fees_usd × fx_rate

total_cost_jpy =
    supply_cost_jpy
  + domestic_shipping_jpy
  + international_shipping_jpy
  + ebay_fees_jpy
  + fx_conversion_fee_jpy

tax_refund_jpy = supply_cost_jpy × 0.07

net_profit_jpy = revenue_jpy - total_cost_jpy + tax_refund_jpy

profit_margin = net_profit_jpy / revenue_jpy

判定：profit_margin >= 0.15 で出品候補に通す
````

## ボーダーライン処理

| profit_margin | 動作 |
|---|---|
| `>= 0.15` | 自動承認 → SA-4 ebay_lister へ |
| `0.13 ≤ x < 0.15` | Discord通知（推奨判断・自動出品はしない） |
| `< 0.13` | rejected（候補から除外、ログのみ） |

## Hook強制（hooks/check_profit_margin.py）

PreToolUse Hookとして、`create_inventory_item` および `submit_feed` の呼び出し前に必ず走る。
profit_margin が 0.15 未満なら、tool_useをblockする。
````

### 3-4. vero-blacklist.md

````markdown
---
alwaysApply: true
---

# VeRO（Verified Rights Owner）blacklist

eBayのVeROプログラムに参加しているブランドの商品を出品すると、
即座にアカウント警告→停止のリスクがある。以下は出品禁止：

## ブランドblacklist（例・各受講生は自分のリスクに応じて拡張）

- Disney（および Pixar / Marvel / Star Wars 関連）
- Nintendo（および Pokémon 関連）
- Supreme
- Louis Vuitton
- Chanel
- Hermès
- Rolex（中古市場でも警告対象になることあり）
- Apple（一部の中古品）
- Sony（PlayStation関連の一部）
- Nike Air Jordan（限定モデル・コラボ品）

## キーワードblacklist

タイトル・description・item_specifics に以下が含まれる場合も対象：

- "Replica" / "Inspired by"
- "Box Logo"（Supreme関連）
- "First Copy"
- ブランド名の typosquatting（"Adlbas"等）

## Hook強制（hooks/check_vero_blacklist.py）

PreToolUse Hookとして、`create_inventory_item` の前に走る。
タイトル・ブランド名・カテゴリのいずれかが blacklist hit したら block。

## 動的更新

VeRO警告を1回でも受けた商品パターンは、自動でblacklistへ追加。
受講生は週次でblacklistを目視レビューする。
````

### 3-5. phase-strategy.md

````markdown
---
globs:
  - agents/profit_judge/**
  - hooks/check_phase_compliance.py
---

# 評価フェーズ別戦略（profit_judge専用ルール）

## 現在の評価値→フェーズ判定

```python
def determine_phase(feedback_score: int) -> str:
    if feedback_score < 30:
        return "phase_0"
    elif feedback_score < 100:
        return "phase_1"
    elif feedback_score < 300:
        return "phase_2"
    elif feedback_score < 500:
        return "phase_3"
    elif feedback_score < 700:
        return "phase_4"
    else:
        return "phase_5"
```

## 各フェーズの出品ポリシー

| フェーズ | 出品数上限 | 回転% | ロング% | 単価上限 |
|---|---|---|---|---|
| phase_0 | 30 | 85 | 15 | $50 |
| phase_1 | 70 | 70 | 30 | $100 |
| phase_2 | 150 | 70 | 30 | $200 |
| phase_3 | 220 | 70 | 30 | $350 |
| phase_4 | 270 | 70 | 30 | $500 |
| phase_5 | 300+ | 70 | 30 | $1000 |

## 整合判定ロジック

ListingDraft が以下すべてを満たすときのみ`phase_fit: true`：

1. 単価が当該フェーズの上限以下
2. 当該フェーズの回転/ロング比率を維持できる
   （現在の出品ポートフォリオで、追加すると比率が崩れる場合はrejected）
3. 当該フェーズの出品数上限を超えない

## Hook強制

`hooks/check_phase_compliance.py` で物理的にblock。
````

### 3-6. rotation-longtail-criteria.md

````markdown
---
globs:
  - agents/profit_judge/**
---

# 回転/ロングテール分類基準

## 回転商品の判定条件

以下AND条件：
- `sold_30d_count >= 2`（過去30日で2個以上売れている）
- `sold_price_usd between 30 and 150`
- `category_id` not in [excluded_categories_for_rotation]

## ロングテール商品の判定条件

以下AND条件：
- 過去90日の sold_count を月平均化 → 月1個以下
- `sold_price_usd >= 100`
- 単発・希少品・コレクター品の傾向が item_specifics やキーワードから読み取れる

## unclear（要人間判断）の条件

以下のいずれか：
- 上記いずれにも該当しない
- 回転条件とロング条件の両方に部分的に該当する（境界線）
- confidence < 0.7

## excluded（除外）の条件

以下のいずれか：
- VeRO blacklist hit
- 過去赤字パターンDB hit
- 単価 $20 未満（送料負けの可能性大）
- 仕入れ価格 + 送料合計が想定売上を超える

## 出力スキーマ強制

profit_judge の出力は ListingDraft schema の `classification.type` に
`["rotation", "longtail", "unclear", "excluded"]` のいずれかが入る。
strict: true で文法レベル強制。
````

### 3-7. outsource-protocol.md

````markdown
---
globs:
  - agents/outsource_dispatcher/**
---

# 外注タスク自動生成プロトコル

## 注文受領時の自動処理

eBay Order Webhook受信 → SA-5 outsource_dispatcher 起動 → 以下を順次実行：

1. ListingDraftから supply_target を引き出す
2. Notionデータベースに新規カード作成
3. Discord に外注さん向け通知
4. Discord に運営者向け通知
5. OrderEvent をJSONLへ追記

## Notionカードの必須フィールド

- task_id（uuid）
- ebay_order_id
- 締切時刻（受注+24時間）
- 仕入れ先プラットフォーム
- 仕入れ先URL
- 仕入れ先タイトル（日本語）
- 仕入れ価格（円）
- 商品画像URL
- 購入指示書（日本語）
- 想定利益（円）
- 状態（created → notified → accepted → purchased → shipped → completed）

## SA-5の権限制限

SA-5の `allowed_tools` は以下に限定：
- create_notion_task
- send_discord_notification
- update_order_status

**購入実行ツール・支払い実行ツールは絶対に与えない。**
これによりend-to-endフロー禁止条項を物理的に遵守。

## 失敗時のエスカレーション

- Notionカード生成失敗 → Discord critical
- 外注さんが24時間以内に accepted ステータスにしない → Discord通知 + 別の外注さんへ転送
````

---

## 第4章 5体のエージェント完全定義

### 4-1. Coordinator

#### `agents/coordinator/CLAUDE.md`

````markdown
# Coordinator — Directory層ルール

## 役割

Fixed Pipelineの統括役。SA-1〜SA-5を順次起動し、各stop_reasonを判定。
エスカレーションが必要な場合のみ人間に通知。

## allowed_tools

- dispatch_subagent
- log_event
- escalate_to_human
- schedule_next_run
- send_discord_critical

## モデル
Sonnet 4.5

## 禁止事項

- Subagent を並列で複数起動しない（Fixed Pipelineの順序を崩さない）
- Subagent の出力を「テキスト解析」で完了判定しない（stop_reasonのみ）
- 自分でツール呼び出しをしてビジネスロジックを実行しない
  （Coordinatorは統括のみ。実装はSubagentが担う）

## 参照
@system_prompt.md
````

#### `agents/coordinator/system_prompt.md`

````markdown
# Coordinator System Prompt

You are the Coordinator agent in a Hub-and-Spoke architecture for the
ケイタ式 VPS automation system.

## Your role

Execute Fixed Pipeline: SA-1 → SA-2 → SA-3 → (profit_verifier conditional) 
→ SA-4 → (Webhook trigger) → SA-5

You orchestrate. You do NOT execute business logic yourself.

## Pipeline procedure

1. Dispatch SA-1 (seller_watcher) with target seller list
2. Wait for stop_reason == "end_turn"
3. Receive SoldListingDigest[] output
4. For each digest, dispatch SA-2 (supply_checker)
5. Wait for all SA-2 to return stop_reason == "end_turn"
6. Receive SupplyMatch[] output
7. For each (digest, match) pair, dispatch SA-3 (profit_judge)
8. Wait for stop_reason == "end_turn"
9. Receive ListingDraft[] output
10. For each draft where 0.7 <= confidence < 0.9, dispatch profit_verifier
11. For each draft where approval_status == "auto_approved", dispatch SA-4 (ebay_lister)
12. SA-5 (outsource_dispatcher) is event-driven, not invoked by you in this pipeline

## Escalation triggers

If any of the following occur, halt the pipeline and call escalate_to_human:
- 3 consecutive API failures from same source
- VeRO warning detected
- profit_margin between 0.13 and 0.15 (notify but don't auto-list)
- profit_judge.confidence < 0.7 (queue to manual_review)

## Memory hierarchy
@~/.claude/CLAUDE.md
@~/keita-vps-auto/CLAUDE.md
@.claude/rules/ebay-tos-compliance.md
@.claude/rules/ketai-shiki-philosophy.md

## Output format

Use tool_use for all dispatching. Never produce free text as output.
Final output of the pipeline is a structured PipelineRunSummary
(see schemas/pipeline_run_summary.schema.json).
````

### 4-2. SA-1: seller_watcher

#### `agents/seller_watcher/CLAUDE.md`

````markdown
# SA-1 seller_watcher — Directory層ルール

## 役割

ライバルセラーのSold/Active Listingを公式API経由で取得し、
SoldListingDigest[] として構造化して返す。

## allowed_tools
- fetch_seller_sold_listings        (ebay-read-mcp)
- fetch_seller_active_listings      (ebay-read-mcp)
- fetch_marketplace_insights        (ebay-read-mcp)
- score_seller                      (内部関数)

## モデル
Haiku 4.5

## 禁止事項

- eBay書き込み系ツールには絶対アクセスしない（capability confusion防止）
- スクレイピング系ツール（apify等）は本MCPに含めない（公式API優先）
- 親文脈を継承しない。Coordinatorから明示的に渡されたseller_idリストのみで動作

## Few-shot examples

例1（通常のSold Listing取得）:
  入力: {"seller_ids": ["japan_camera_2024"], "days_back": 30}
  処理: fetch_seller_sold_listings × 1回
  出力: SoldListingDigest[] (12件)

例2（rate limit到達）:
  入力: {"seller_ids": [...500件...], "days_back": 30}
  処理: 200件で daily quota の80%到達
  動作: 残り300件を翌日キューへ繰り延べ、Discord通知
  出力: 部分結果 + escalation_required: true

例3（Marketplace Insights使用）:
  入力: 招待降下済みの場合
  処理: fetch_marketplace_insights を優先、Browse APIはフォールバック
  出力: data_source: "marketplace_insights" を付与

## 参照
@system_prompt.md
````

#### `agents/seller_watcher/system_prompt.md`

````markdown
# SA-1 seller_watcher System Prompt

You are SA-1, the seller_watcher subagent. You retrieve sold/active 
listings from eBay via official APIs only.

## Your tools

- fetch_seller_sold_listings(seller_id, days_back) -> SoldListingDigest[]
- fetch_seller_active_listings(seller_id) -> ActiveListing[]  
- fetch_marketplace_insights(seller_id, days_back) -> SoldListingDigest[]
- score_seller(seller_id) -> SellerScore

## Decision logic

1. If Marketplace Insights API access is available (check config),
   prefer it over Browse API for sold listings.
2. For active listings, always use Browse API.
3. After every fetch, score the seller via score_seller and update
   the seller's priority_tier in JSONL.
4. Output strictly conforms to SoldListingDigest schema.

## Output schema

Use tool_use to return SoldListingDigest[]. Each digest must include:
- digest_id (uuid)
- generated_at (ISO 8601)
- seller_id, listing_id, title_en, category_id, sold_price_usd, sold_date
- primary_image_url, image_urls
- item_specifics (brand, model, mpn, year — nullable)
- supply_search_keys (auto-generated from title and item_specifics)
- data_source enum

## Escalation

If 3+ consecutive API errors from same source, return tool_use with
isError=true and structured error (see hooks/log_structured_error.py format).

## Stop condition

After processing all assigned seller_ids, return stop_reason="end_turn".
````

### 4-3. SA-2: supply_checker

#### `agents/supply_checker/CLAUDE.md`

````markdown
# SA-2 supply_checker — Directory層ルール

## 役割

SoldListingDigestを受け取り、Buyee/楽天/Yahoo!ショッピングの公式APIで
仕入れ可否を確認。画像マッチング後、SupplyMatch[]として返す。

## allowed_tools
- search_buyee
- search_rakuten
- search_yahoo_shopping
- match_image  (内部関数: 画像URLとhashを比較)

## モデル
Haiku 4.5

## 禁止事項

- メルカリ直接スクレイピング禁止（メルカリToS違反）
- Buyee以外のメルカリAPI呼び出し禁止
- 購入実行ツールは持たない（仮にAPIで可能でも実装しない）

## 並列実行ポリシー

3つの検索エンジンを並列実行可能（Coordinatorが並列ディスパッチ）。
ただしrate limit遵守のため、各APIは独立したsemaphoreで制御。

## Few-shot examples

例1（カメラ・Buyee hit）:
  入力digest: title_en="Vintage Canon AE-1 Camera", brand="Canon"
  処理: search_buyee("Canon AE-1 中古") + search_yahoo_shopping
  出力: SupplyMatch with platform="mercari" (Buyee経由)、image_match_score=0.92

例2（楽天hit）:
  入力digest: title_en="Roland D-10 Synthesizer", brand="Roland"
  処理: search_rakuten("Roland D-10 中古")
  出力: SupplyMatch with platform="rakuten"

例3（no match）:
  入力digest: title_en="Rare Limited Edition Boxed Set"
  処理: 3つのAPIすべて検索
  出力: SupplyMatch with supply_available=false, no_match_reason="No matching item found in JP marketplaces"

## 参照
@system_prompt.md
````

#### `agents/supply_checker/system_prompt.md`

````markdown
# SA-2 supply_checker System Prompt

You are SA-2, the supply_checker subagent. You verify if a sold-on-eBay
item can be sourced from Japanese marketplaces via official APIs.

## Your tools

- search_buyee(query, max_results=5) -> BuyeeResult[]
- search_rakuten(query, max_results=5) -> RakutenResult[]
- search_yahoo_shopping(query, max_results=5) -> YahooResult[]
- match_image(image_url_a, image_url_b) -> {score: float, hash_distance: int}

## Procedure

1. Extract supply_search_keys from input digest
2. Run 3 searches in parallel (or sequential if rate-limited)
3. For each candidate result, run match_image against digest's primary_image_url
4. Pick best_match (highest image_match_score AND lowest total_cost_jpy)
5. Output SupplyMatch

## Match threshold

- image_match_score >= 0.85 -> high confidence
- 0.70 <= score < 0.85 -> medium, include but flag confidence_lower
- score < 0.70 -> exclude

## Output schema

SupplyMatch with:
- match_id, digest_id, checked_at
- supply_available (boolean)
- best_match (object or null)
- alternative_matches[] (up to 3)
- no_match_reason (if applicable)
- confidence (0-1)

## Stop condition

After all searches and match scoring complete: stop_reason="end_turn"
````

### 4-4. SA-3: profit_judge

#### `agents/profit_judge/CLAUDE.md`

````markdown
# SA-3 profit_judge — Directory層ルール

## 役割

SoldListingDigest + SupplyMatchを受け取り、利益率15%判定・回転/ロング分類・
評価フェーズ整合チェックを実行し、ListingDraftを生成。

## allowed_tools
- get_current_fx_rate     (fx-mcp)
- calculate_profitability (内部関数)
- classify_rotation_or_longtail (内部関数)
- check_phase_fit (内部関数)
- check_blacklist (SQLite query)

## モデル
Sonnet 4.5

## 禁止事項

- 利益率閾値を勝手に下げない（15%固定。緩和したい場合はescalation）
- VeRO blacklistを無視しない（hookで物理的にblockされるが、自分でも判定）
- confidence を恣意的に高くしない（false confidenceの危険）

## Few-shot examples

例1（rotation・自動承認）:
  入力digest: sold_price_usd=$45, category="Cameras > Vintage"
  入力match: supply_cost_jpy=2800, image_match_score=0.95
  fx: 150 JPY/USD
  計算: revenue=6750-buffer=6413, costs(intl_ship+ebay_fee+supply)=4500
        net_profit=2090, margin=0.31
  分類: sold_30d_count=4 -> rotation
  フェーズ: 単価<$50 -> phase_0 fit
  出力: {type:"rotation", phase_fit:"phase_0", passes_15pct_threshold:true,
         confidence:0.95, approval_status:"auto_approved"}

例2（longtail・auto_approved）:
  入力digest: sold_price_usd=$320, category="Audio > Vintage"
  入力match: supply_cost_jpy=18000
  計算: margin=0.21
  分類: sold_30d=0, sold_total=2 over 6 months -> longtail
  フェーズ: 単価$320 -> phase_3 fit
  出力: {type:"longtail", phase_fit:"phase_3", passes_15pct_threshold:true,
         confidence:0.88, approval_status:"auto_approved"}

例3（borderline・human_review）:
  入力digest: sold_price_usd=$120, category="Watches"
  入力match: supply_cost_jpy=8000
  計算: margin=0.14 (borderline)
  動作: 0.13 <= margin < 0.15 -> notify but don't auto-list
  出力: {type:"unclear", phase_fit:"phase_2", passes_15pct_threshold:false,
         confidence:0.65, approval_status:"human_review_required",
         escalation_reason:"Borderline profit margin (14%)"}

例4（excluded・低単価）:
  入力digest: sold_price_usd=$15
  計算: margin=-0.20 (赤字)
  出力: {type:"excluded", phase_fit:"none", passes_15pct_threshold:false,
         confidence:0.99, approval_status:"rejected",
         rejection_reason:"Price too low for profitable export"}

例5（excluded・VeRO）:
  入力digest: title="Supreme Box Logo Tee"
  動作: check_blacklist -> hit
  出力: {type:"excluded", phase_fit:"none", passes_15pct_threshold:false,
         confidence:1.0, approval_status:"rejected",
         rejection_reason:"VeRO blacklist hit: Supreme"}

## 参照
@system_prompt.md
@.claude/rules/profit-margin-15pct-strict.md
@.claude/rules/phase-strategy.md
@.claude/rules/rotation-longtail-criteria.md
````

#### `agents/profit_judge/system_prompt.md`

````markdown
# SA-3 profit_judge System Prompt

You are SA-3, the profit_judge subagent. You determine whether a 
sourced item can be profitably listed on eBay according to ケイタ式 rules.

## Critical rules (冒頭配置 - Lost in the Middle対策)

1. **Profit margin must be >= 15% to auto-approve**
2. **Phase fit must match current account's evaluation phase**
3. **VeRO blacklist must be checked**
4. **Confidence < 0.7 must trigger escalation**

## Your tools

- get_current_fx_rate() -> FxRate
- calculate_profitability(...) -> ProfitabilityResult
- classify_rotation_or_longtail(digest, match) -> Classification
- check_phase_fit(classification, sold_price_usd, current_phase) -> bool
- check_blacklist(title, brand, category_id) -> BlacklistResult

## Procedure

1. Get current FX rate (must be < 6h old)
2. Calculate profitability with all fees and tax refund
3. Classify as rotation/longtail/unclear/excluded
4. Check phase fit
5. Check VeRO blacklist
6. Determine approval_status:
   - auto_approved: margin >= 0.15 AND phase_fit AND not blacklisted AND confidence >= 0.9
   - human_review_required: 0.13 <= margin < 0.15, OR confidence in [0.7, 0.9)
   - rejected: margin < 0.13 OR blacklist hit OR phase mismatch

## Output schema

ListingDraft (see schemas/listing_draft.schema.json)

## Critical rules (末尾再掲 - Lost in the Middle対策)

Remember:
1. 15% margin is absolute
2. Phase fit is non-negotiable
3. VeRO blacklist must be checked
4. Confidence < 0.7 escalates
````

### 4-5. SA-4: ebay_lister

#### `agents/ebay_lister/CLAUDE.md`

````markdown
# SA-4 ebay_lister — Directory層ルール

## 役割

auto_approved の ListingDraft を受け取り、DeepL翻訳・eBay Inventory API登録・
Feed APIバルク投入・広告率最適化を実行。

## allowed_tools
- translate_deepl
- create_inventory_item    (ebay-write-mcp)
- submit_feed              (ebay-write-mcp)
- update_listing           (ebay-write-mcp)
- end_listing              (ebay-write-mcp)
- set_promoted_rate

## モデル
Sonnet 4.5

## 禁止事項

- approval_status != "auto_approved" な ListingDraftには絶対に出品しない
- Hookでブロックされた呼び出しを再試行しない（ループ防止）
- eBay読み取り系ツールには触らない（責務分離）

## PreToolUse Hooks（強制）

create_inventory_item / submit_feed の前に以下が走る：
- enforce_profit_margin_15pct  -> margin < 0.15 なら block
- enforce_phase_compliance     -> phase違反なら block
- enforce_vero_blacklist       -> VeRO hit なら block

これらは informational ではなく、**物理的にtool_useを阻止する**。

## 翻訳品質基準

DeepL翻訳結果が以下を満たすかチェック：
- 80文字以内
- ブランド名・型番が原文と一致
- 不適切な表現を含まない

満たさない場合、再翻訳または手動修正キューへ。

## 広告率設定

カテゴリ別の標準広告率：
- Cameras: 6-8%
- Audio: 5-7%
- Musical Instruments: 7-10%
- Watches: 4-6%
- Default: 6%

過去の販売データから動的に最適化（30日窓）。

## 参照
@system_prompt.md
````

#### `agents/ebay_lister/system_prompt.md`

````markdown
# SA-4 ebay_lister System Prompt

You are SA-4, the ebay_lister subagent. You execute eBay listings
based on auto_approved ListingDrafts.

## Critical pre-conditions (冒頭)

Only process drafts with:
- approval_status == "auto_approved"
- profitability.passes_15pct_threshold == true
- classification.phase_fit matches current account phase

If any of these fail, immediately output an error structured response.
Do NOT attempt to list.

## Procedure

1. Translate title via DeepL (Japanese keywords -> English title, max 80 chars)
2. Validate translation (length, brand match, no inappropriate terms)
3. Create Inventory Item via ebay-write-mcp.create_inventory_item
4. Submit Feed (bulk) via ebay-write-mcp.submit_feed
5. Set Promoted Listings rate via set_promoted_rate
6. Verify listing went live (check listing_id)
7. Update candidate status in JSONL/SQLite (via PostToolUse hook)

## Hooks behavior

If a Hook blocks your tool_use, do NOT retry. Output a structured error:
{
  "isError": true,
  "error": {
    "type": "hook_blocked",
    "hook_name": "<hook_name>",
    "draft_id": "<uuid>",
    "reason": "<from hook output>"
  }
}

## Stop condition

After all auto_approved drafts processed (success or hook-blocked):
stop_reason="end_turn"
````

### 4-6. SA-5: outsource_dispatcher

#### `agents/outsource_dispatcher/CLAUDE.md`

````markdown
# SA-5 outsource_dispatcher — Directory層ルール

## 役割

eBay Order Webhook受信時に起動。注文情報からNotionタスクカードを自動生成し、
外注さん・運営者に Discord 通知。

## allowed_tools
- create_notion_task        (notion-mcp)
- send_discord_notification (discord-mcp)
- update_order_status

## モデル
Haiku 4.5

## 絶対的禁止事項

**購入実行ツール・支払い実行ツールは絶対に持たない。**

これがケイタ式VPS自動化のeBay規約遵守の根幹。
SA-5に購入実行能力を与えると、システムが「人間の確認なしのend-to-endフロー」と
判定されるリスクが発生する。

## トリガー

eBay Order Webhook → systemd-managed `keita-order-webhook.service` →
Coordinator → SA-5 dispatch

## Notionカード生成内容

@.claude/rules/outsource-protocol.md に準拠。

## 通知パターン

| 通知先 | チャネル | タイミング |
|---|---|---|
| 外注さん | Discord (#outsource) | カード生成時 |
| 運営者 | Discord (#orders) | カード生成時 |
| 運営者 | Discord (#orders) | 24時間以内に外注さんがacceptしない場合 |
| 運営者 | Discord (#orders) | 購入完了時 |

## 参照
@system_prompt.md
@.claude/rules/outsource-protocol.md
````

#### `agents/outsource_dispatcher/system_prompt.md`

````markdown
# SA-5 outsource_dispatcher System Prompt

You are SA-5, the outsource_dispatcher subagent. You bridge eBay orders
to human contractors who execute the actual mercari purchase.

## CRITICAL rule

**You do NOT have purchase execution tools. You ONLY notify humans.**

This is the cornerstone of eBay ToS compliance. Never request such tools.

## Procedure on order Webhook

1. Receive OrderEvent (validated against schema)
2. Look up corresponding ListingDraft (via draft_id)
3. Extract supply_target (platform, url, price, image_urls)
4. Calculate deadline_for_supply_purchase (received_at + 24h)
5. Create Notion task card with all required fields
6. Send Discord notification to #outsource channel (mention assigned outsourcer)
7. Send Discord notification to #orders channel (info to operator)
8. Append OrderEvent to data/jsonl/orders.jsonl

## Output schema

OrderEvent (see schemas/order_event.schema.json)

## Stop condition

After Notion card created and both Discord notifications sent:
stop_reason="end_turn"
````

### 4-7. profit_verifier（独立検証用・Generator-Verifier Pattern）

#### `agents/profit_verifier/CLAUDE.md`

````markdown
# profit_verifier — Independent Review Instance

## 役割

SA-3 profit_judge が出力した ListingDraft を、独立した文脈でレビューする。
SA-3のpromptは継承しない。CLAUDE.mdとListingDraftだけで判断。

## 起動条件

confidence が 0.7 〜 0.9 の範囲のときのみ Coordinator が起動。

## allowed_tools
- (Verifierは判定のみ。新規ツール呼び出しはせず、入力JSONとルールから判断)
- output_verification_result

## モデル
Sonnet 4.5（別セッション）

## 禁止事項

- SA-3のシステムプロンプトを参照しない
- 自分の以前の判定を参照しない（毎回独立判断）

## 参照
@system_prompt.md
@.claude/rules/profit-margin-15pct-strict.md
@.claude/rules/phase-strategy.md
@.claude/rules/rotation-longtail-criteria.md
@.claude/rules/vero-blacklist.md
````

#### `agents/profit_verifier/system_prompt.md`

````markdown
# profit_verifier System Prompt

You are an independent verifier. You receive a ListingDraft and the
ケイタ式 rules. You do NOT see the original profit_judge's reasoning.

## Your task

Re-evaluate the ListingDraft from scratch based on:
- profitability.calculated_profit_margin
- classification.type and phase_fit
- VeRO blacklist
- Phase strategy rules

## Output

verification_result:
{
  "draft_id": "<uuid>",
  "verifier_agrees": boolean,
  "verifier_classification": "rotation|longtail|unclear|excluded",
  "verifier_phase_fit": "phase_0|phase_1|...|none",
  "verifier_passes_15pct": boolean,
  "verifier_confidence": 0-1,
  "discrepancy_notes": "string|null"
}

## Decision

If verifier_agrees == false, the original ListingDraft is downgraded to
human_review_required, regardless of original confidence.
````

---

## 第5章 JSON Schema完全版

Doc 1で先行掲載済みの SoldListingDigest / SupplyMatch / ListingDraft / OrderEvent
に加え、以下の追加スキーマを定義する。

### 5-1. seller.schema.json

````json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "Seller",
  "type": "object",
  "properties": {
    "seller_id": {"type": "string", "minLength": 1, "maxLength": 50},
    "seller_url": {"type": "string", "format": "uri"},
    "store_url": {"type": ["string", "null"], "format": "uri"},
    "feedback": {
      "type": "object",
      "properties": {
        "score": {"type": "integer", "minimum": 0},
        "positive_percent": {"type": "number", "minimum": 0, "maximum": 100},
        "feedback_30d": {"type": "integer", "minimum": 0},
        "feedback_90d": {"type": "integer", "minimum": 0},
        "feedback_365d": {"type": "integer", "minimum": 0}
      },
      "required": ["score", "positive_percent"]
    },
    "location": {
      "type": "object",
      "properties": {
        "country": {"type": "string", "const": "JP"},
        "ship_from": {"type": "string", "const": "Japan"}
      },
      "required": ["country", "ship_from"]
    },
    "store_info": {
      "type": "object",
      "properties": {
        "is_store_subscriber": {"type": "boolean"},
        "store_tier": {"type": "string", "enum": ["none", "Starter", "Basic", "Premium", "Anchor", "Enterprise"]},
        "registered_since": {"type": "string", "format": "date"},
        "top_rated": {"type": "boolean"}
      }
    },
    "activity_metrics": {
      "type": "object",
      "properties": {
        "active_listings_count": {"type": "integer", "minimum": 0},
        "sold_listings_30d": {"type": "integer", "minimum": 0},
        "sold_listings_90d": {"type": "integer", "minimum": 0},
        "conversion_rate": {"type": "number", "minimum": 0, "maximum": 1},
        "avg_unit_price_usd": {"type": "number", "minimum": 0},
        "monthly_revenue_estimate_usd": {"type": "number", "minimum": 0}
      }
    },
    "ketai_shiki_filters": {
      "type": "object",
      "properties": {
        "is_japan_shipper": {"type": "boolean"},
        "conversion_rate_above_5pct": {"type": "boolean"},
        "used_item_majority": {"type": "boolean"},
        "feedback_in_target_range": {"type": "boolean"},
        "japan_seller_score": {"type": "number", "minimum": 0, "maximum": 100},
        "priority_tier": {"type": "string", "enum": ["S", "A", "B", "C", "EXCLUDED"]}
      },
      "required": ["priority_tier"]
    },
    "tracking_meta": {
      "type": "object",
      "properties": {
        "added_to_watchlist_at": {"type": "string", "format": "date-time"},
        "last_scraped_at": {"type": "string", "format": "date-time"},
        "next_check_at": {"type": "string", "format": "date-time"},
        "check_interval_hours": {"type": "integer", "minimum": 1, "maximum": 168},
        "data_source": {"type": "string", "enum": ["manual", "browse_api", "marketplace_insights", "apify", "oxylabs"]}
      },
      "required": ["last_scraped_at", "data_source"]
    }
  },
  "required": ["seller_id", "seller_url", "feedback", "location", "ketai_shiki_filters", "tracking_meta"],
  "additionalProperties": false
}
````

### 5-2. listing.schema.json

（Doc 1掲載のlisting.jsonをDraft 2020-12形式に揃えたもの。冗長になるため省略表記）

完全版は `~/keita-vps-auto/schemas/listing.schema.json` に配置。

### 5-3. outsource_task.schema.json

````json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "OutsourceTask",
  "type": "object",
  "properties": {
    "task_id": {"type": "string", "format": "uuid"},
    "ebay_order_id": {"type": "string"},
    "ebay_buyer_username": {"type": "string"},
    "created_at": {"type": "string", "format": "date-time"},
    "deadline_at": {"type": "string", "format": "date-time"},
    "task_type": {"type": "string", "const": "mercari_purchase"},
    "supply_target": {
      "type": "object",
      "properties": {
        "platform": {"type": "string", "enum": ["mercari", "yahoo_auctions", "rakuten", "yahoo_shopping"]},
        "url": {"type": "string", "format": "uri"},
        "title_jp": {"type": "string"},
        "price_jpy": {"type": "integer", "minimum": 1},
        "image_urls": {"type": "array", "items": {"type": "string", "format": "uri"}},
        "purchase_instructions": {"type": "string"}
      },
      "required": ["platform", "url", "price_jpy"]
    },
    "ebay_order_info": {
      "type": "object",
      "properties": {
        "buyer_name": {"type": "string"},
        "shipping_address": {"$ref": "OrderEvent#/properties/shipping_address"},
        "sold_price_usd": {"type": "number"},
        "expected_profit_jpy": {"type": "integer"}
      }
    },
    "assignee": {
      "type": "object",
      "properties": {
        "outsource_id": {"type": "string"},
        "discord_user_id": {"type": "string"},
        "notion_page_id": {"type": "string"}
      }
    },
    "status": {"type": "string", "enum": ["created", "notified", "accepted", "purchased", "shipped", "completed", "failed"]},
    "purchase_proof": {
      "type": ["object", "null"],
      "properties": {
        "receipt_image_url": {"type": "string", "format": "uri"},
        "purchase_confirmation_id": {"type": "string"},
        "actual_price_jpy": {"type": "integer"}
      }
    }
  },
  "required": ["task_id", "ebay_order_id", "created_at", "deadline_at", "task_type", "supply_target", "status"],
  "additionalProperties": false
}
````

### 5-4. pipeline_run_summary.schema.json

````json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "PipelineRunSummary",
  "type": "object",
  "properties": {
    "run_id": {"type": "string", "format": "uuid"},
    "started_at": {"type": "string", "format": "date-time"},
    "ended_at": {"type": "string", "format": "date-time"},
    "trigger": {"type": "string", "enum": ["scheduled", "manual", "webhook"]},
    "phases_executed": {"type": "array", "items": {"type": "string"}},
    "stats": {
      "type": "object",
      "properties": {
        "sellers_processed": {"type": "integer"},
        "digests_generated": {"type": "integer"},
        "supply_matches_found": {"type": "integer"},
        "drafts_auto_approved": {"type": "integer"},
        "drafts_human_review": {"type": "integer"},
        "drafts_rejected": {"type": "integer"},
        "listings_created": {"type": "integer"},
        "errors_encountered": {"type": "integer"}
      }
    },
    "escalations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "trigger": {"type": "string"},
          "occurred_at": {"type": "string", "format": "date-time"},
          "details": {"type": "object"}
        }
      }
    },
    "stop_reason": {"type": "string", "enum": ["end_turn", "max_turns", "error", "user_halt"]}
  },
  "required": ["run_id", "started_at", "trigger", "stop_reason"]
}
````

---

## 第6章 Hooks YAML完全版

`~/keita-vps-auto/.claude/hooks.yaml`

````yaml
hooks:
  # ============================================
  # PreToolUse Hooks
  # ============================================

  - name: enforce_profit_margin_15pct
    type: PreToolUse
    match:
      tool_name: ["create_inventory_item", "submit_feed"]
    run: ./hooks/check_profit_margin.py
    description: |
      ListingDraftのprofit_marginが15%未満ならtool_useをblock。
      0.13-0.15はnotifyのみ（block）、<0.13はrejected ステータス更新。
    on_failure: block
    on_block_message_template: |
      PROFIT_MARGIN_VIOLATION
      draft_id={draft_id}
      margin={margin}
      threshold=0.15
      action=blocked

  - name: enforce_phase_compliance
    type: PreToolUse
    match:
      tool_name: ["create_inventory_item"]
    run: ./hooks/check_phase_compliance.py
    description: |
      ListingDraftのphase_fitと現在のアカウントフェーズの整合チェック。
      Phase 0でlongtailを出そうとしたらblock。
      回転/ロング比率を崩す場合もblock。
    on_failure: block

  - name: enforce_vero_blacklist
    type: PreToolUse
    match:
      tool_name: ["create_inventory_item"]
    run: ./hooks/check_vero_blacklist.py
    description: |
      タイトル・ブランド・カテゴリのいずれかがVeRO blacklist hitならblock。
      動的blacklist（過去のVeRO警告履歴）も照合。
    on_failure: block

  - name: enforce_rate_limit
    type: PreToolUse
    match:
      tool_name: ["search_buyee", "search_rakuten", "search_yahoo_shopping",
                  "fetch_seller_sold_listings", "fetch_marketplace_insights"]
    run: ./hooks/check_rate_limit.py
    description: |
      各APIのrate limit監視。公式上限の80%を本設計の上限とし、
      到達時は当該tool_useをdefer（次サイクルへ繰り延べ）。
    on_failure: defer

  - name: validate_listing_draft_schema
    type: PreToolUse
    match:
      tool_name: ["create_inventory_item", "submit_feed"]
    run: ./hooks/validate_schema.py
    description: |
      入力ListingDraftがlisting_draft.schema.jsonに準拠しているかチェック。
      strict: true でも fly前の最終ガード。
    on_failure: block

  # ============================================
  # PostToolUse Hooks
  # ============================================

  - name: trim_large_ebay_responses
    type: PostToolUse
    match:
      tool_name: ["fetch_seller_sold_listings", "fetch_marketplace_insights",
                  "fetch_seller_active_listings"]
    run: ./hooks/trim_responses.py
    description: |
      eBay APIの巨大レスポンスから必要フィールドだけ抽出。
      description HTML、関連商品、出品者プロフィール等は削除。
      コンテキストを1/10以下に削減。

  - name: update_candidate_status_on_listing_success
    type: PostToolUse
    match:
      tool_name: ["submit_feed"]
      tool_result:
        status: "success"
    run: ./hooks/update_candidate_status.py
    description: |
      Feed APIで出品成功した候補のstatusをDB更新。
      created -> listed への遷移。

  - name: append_to_jsonl_on_digest
    type: PostToolUse
    match:
      tool_name: ["fetch_seller_sold_listings", "fetch_marketplace_insights"]
    run: ./hooks/append_jsonl.py
    description: |
      取得したSoldListingDigestを data/jsonl/digests.jsonl に追記。

  - name: log_structured_error_on_isError
    type: PostToolUse
    match:
      tool_result:
        isError: true
    run: ./hooks/log_structured_error.py
    description: |
      isError=trueのレスポンスを構造化ログに保存。
      type/severity/source/alternative を必須化。

  - name: update_seller_score_after_fetch
    type: PostToolUse
    match:
      tool_name: ["fetch_seller_sold_listings"]
    run: ./hooks/update_seller_score.py
    description: |
      取得結果からセラーのコンバージョン率・売上推定を再計算し、
      sellers.jsonlのpriority_tierを更新。

  - name: vero_warning_auto_blacklist
    type: PostToolUse
    match:
      tool_result:
        contains_vero_warning: true
    run: ./hooks/vero_auto_blacklist.py
    description: |
      eBayレスポンスにVeRO警告が含まれていたら、
      該当パターンを自動でblacklistへ追加し、Discord critical通知。
````

### Hookスクリプト例：`hooks/check_profit_margin.py`

````python
#!/usr/bin/env python3
"""
Hook: enforce_profit_margin_15pct
Triggered: PreToolUse for create_inventory_item / submit_feed

Reads tool_use.input, extracts ListingDraft, checks profit_margin.
Exits 0 to allow, exit 1 to block.
"""
import json
import sys

PROFIT_THRESHOLD = 0.15
BORDERLINE_LOW = 0.13


def main():
    tool_use_input = json.loads(sys.stdin.read())
    draft = tool_use_input.get("listing_draft") or tool_use_input
    
    margin = draft.get("profitability", {}).get("calculated_profit_margin")
    draft_id = draft.get("draft_id", "unknown")
    
    if margin is None:
        print(f"BLOCK: profit_margin missing in draft_id={draft_id}", file=sys.stderr)
        sys.exit(1)
    
    if margin < BORDERLINE_LOW:
        print(f"BLOCK: profit_margin {margin:.4f} < {BORDERLINE_LOW} (rejected)", file=sys.stderr)
        sys.exit(1)
    
    if margin < PROFIT_THRESHOLD:
        print(f"BLOCK: profit_margin {margin:.4f} in borderline [{BORDERLINE_LOW}, {PROFIT_THRESHOLD})", file=sys.stderr)
        # Discord notification triggered separately
        sys.exit(1)
    
    # margin >= 0.15: allow
    print(f"ALLOW: profit_margin {margin:.4f} >= {PROFIT_THRESHOLD}")
    sys.exit(0)


if __name__ == "__main__":
    main()
````

---

## 第7章 Slash Commands完全版

各コマンドは `.claude/commands/<name>.md` に配置。

### 7-1. `/keita-daily-cycle`

````markdown
---
description: ケイタ式の朝の定型サイクル全フェーズ通し実行
allowed-tools:
  - dispatch_subagent
  - log_event
  - escalate_to_human
  - send_discord_critical
context: fork
---

# Keita Daily Cycle

Coordinator agent will execute the Fixed Pipeline:

1. SA-1 seller_watcher: 全アクティブセラーリスト（priority_tier in [S, A, B]）から
   過去24時間のSold Listingを取得
2. SA-2 supply_checker: 各digestについて並列で仕入れ可否確認
3. SA-3 profit_judge: 採算判定・分類
4. profit_verifier: confidence 0.7-0.9 のdraftを独立検証
5. SA-4 ebay_lister: auto_approvedなdraftのみ出品実行

各フェーズの結果は data/jsonl/ に追記される。
最終的にPipelineRunSummaryをDiscord #orders に投稿。

## エスカレーション

以下が発生した場合、即座に human_review キューへ：
- 3回連続API失敗
- VeRO警告検知
- profit_margin が 0.13-0.15 のborderline
- profit_judge.confidence < 0.7
````

### 7-2. `/keita-emergency-halt`

````markdown
---
description: 全パイプラインを緊急停止
allowed-tools:
  - send_discord_critical
require_confirmation: true
---

# Emergency Halt

systemd-managed timersをすべて停止する：
- keita-daily-cycle.timer
- keita-position-check.timer
- keita-inventory-check.timer
- keita-seller-rescore.timer

実行中のagent processをSIGTERMで終了する。

## 確認プロンプト

「本当に全パイプラインを停止しますか？(yes/no)」
yesの場合のみ実行。
````

その他のSlash Commandsは類似形式で配置。

---

## 第8章 MCP server実装スケルトン

各MCP serverは Python + FastMCP（Anthropic公式SDK）で実装。

### 8-1. `mcp/ebay-read-mcp/server.py`（スケルトン）

````python
"""
ebay-read-mcp: 読み取り専用 eBay API サーバ
SA-1 seller_watcher 専用
"""
from fastmcp import FastMCP
from .tools import (
    fetch_seller_sold_listings as _fetch_sold,
    fetch_seller_active_listings as _fetch_active,
    fetch_marketplace_insights as _fetch_mi,
)

mcp = FastMCP("ebay-read-mcp")


@mcp.tool()
def fetch_seller_sold_listings(seller_id: str, days_back: int = 30) -> dict:
    """
    Fetch sold listings for a single eBay seller via Browse API.
    
    Use this when you need recent Sold Listings for competitor analysis.
    Do NOT use for active (unsold) listings — use fetch_seller_active_listings.
    
    Returns SoldListingDigest[] conforming to schemas/sold_listing_digest.schema.json.
    
    Args:
        seller_id: eBay user ID (e.g. 'japan_camera_2024'). 
                   Must match pattern ^[a-zA-Z0-9_-]{1,50}$.
        days_back: Number of days to look back (1-90, default 30).
    
    Returns:
        {"digests": [...], "count": int, "rate_limit_remaining": int}
    
    Raises:
        isError=true if seller_id invalid, API rate limit reached, or auth failure.
    """
    return _fetch_sold(seller_id, days_back)


@mcp.tool()
def fetch_seller_active_listings(seller_id: str) -> dict:
    """Fetch currently-active (unsold) listings. Use for monitoring competitor inventory."""
    return _fetch_active(seller_id)


@mcp.tool()
def fetch_marketplace_insights(seller_id: str, days_back: int = 90) -> dict:
    """
    Fetch sold history via Marketplace Insights API (invitation-only).
    Higher quality than Browse API but requires eBay-issued access.
    """
    return _fetch_mi(seller_id, days_back)


if __name__ == "__main__":
    mcp.run()
````

### 8-2. `mcp/ebay-write-mcp/server.py`（スケルトン）

````python
"""
ebay-write-mcp: 書き込み専用 eBay API サーバ
SA-4 ebay_lister 専用
"""
from fastmcp import FastMCP
from .tools import (
    create_inventory_item as _create,
    submit_feed as _submit,
    update_listing as _update,
    end_listing as _end,
)

mcp = FastMCP("ebay-write-mcp")


@mcp.tool()
def create_inventory_item(
    sku: str,
    product: dict,
    availability: dict,
    condition: str
) -> dict:
    """
    Register a new inventory item via Inventory API.
    
    PRECONDITION: This call will be intercepted by PreToolUse hooks:
    - enforce_profit_margin_15pct
    - enforce_phase_compliance
    - enforce_vero_blacklist
    
    If any hook blocks, this tool will NOT execute.
    """
    return _create(sku, product, availability, condition)


@mcp.tool()
def submit_feed(feed_type: str, file_path: str) -> dict:
    """
    Submit bulk listing feed via Feed API (LMS).
    Max 25MB / 50,000 SKUs per feed.
    """
    return _submit(feed_type, file_path)


@mcp.tool()
def update_listing(listing_id: str, updates: dict) -> dict:
    """Update existing listing (price, quantity, description)."""
    return _update(listing_id, updates)


@mcp.tool()
def end_listing(listing_id: str, reason: str) -> dict:
    """End an active listing (e.g. when supply is no longer available)."""
    return _end(listing_id, reason)


if __name__ == "__main__":
    mcp.run()
````

### 8-3. その他のMCP serverも同様の構造

- `mcp/buyee-mcp/server.py`: search ツール1個
- `mcp/rakuten-mcp/server.py`: search ツール1個
- `mcp/yahoo-shopping-mcp/server.py`: search ツール1個
- `mcp/fx-mcp/server.py`: get_current_rate ツール1個

各serverは独立したPythonパッケージとして配置。`pyproject.toml` で
`fastmcp` と各APIのSDK（`ebaysdk` / `requests` 等）を依存に持つ。

### 8-4. MCP server接続設定

`~/.config/claude/mcp_servers.json`

````json
{
  "mcpServers": {
    "ebay-read": {
      "command": "python",
      "args": ["-m", "ebay_read_mcp.server"],
      "env": {
        "EBAY_APP_ID": "${EBAY_APP_ID}",
        "EBAY_CERT_ID": "${EBAY_CERT_ID}",
        "EBAY_DEV_ID": "${EBAY_DEV_ID}",
        "EBAY_AUTH_TOKEN": "${EBAY_AUTH_TOKEN}"
      }
    },
    "ebay-write": {
      "command": "python",
      "args": ["-m", "ebay_write_mcp.server"],
      "env": {
        "EBAY_APP_ID": "${EBAY_APP_ID}",
        "EBAY_CERT_ID": "${EBAY_CERT_ID}",
        "EBAY_DEV_ID": "${EBAY_DEV_ID}",
        "EBAY_AUTH_TOKEN": "${EBAY_AUTH_TOKEN}"
      }
    },
    "buyee": {
      "command": "python",
      "args": ["-m", "buyee_mcp.server"],
      "env": {"BUYEE_API_KEY": "${BUYEE_API_KEY}"}
    },
    "rakuten": {
      "command": "python",
      "args": ["-m", "rakuten_mcp.server"],
      "env": {"RAKUTEN_APP_ID": "${RAKUTEN_APP_ID}"}
    },
    "yahoo-shopping": {
      "command": "python",
      "args": ["-m", "yahoo_shopping_mcp.server"],
      "env": {"YAHOO_APP_ID": "${YAHOO_APP_ID}"}
    },
    "fx": {
      "command": "python",
      "args": ["-m", "fx_mcp.server"],
      "env": {"OXR_APP_ID": "${OXR_APP_ID}"}
    },
    "deepl": {
      "command": "npx",
      "args": ["-y", "@deepl/mcp-server"],
      "env": {"DEEPL_API_KEY": "${DEEPL_API_KEY}"}
    },
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/mcp-server"],
      "env": {"NOTION_API_KEY": "${NOTION_API_KEY}"}
    },
    "discord": {
      "command": "python",
      "args": ["-m", "discord_mcp.server"],
      "env": {"DISCORD_BOT_TOKEN": "${DISCORD_BOT_TOKEN}"}
    }
  }
}
````

---

## 第9章 systemd unit / timer完全版

### 9-1. `keita-daily-cycle.service`

````ini
[Unit]
Description=Keita-shiki Daily Cycle (Fixed Pipeline ①〜④)
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=keita
WorkingDirectory=/home/keita/keita-vps-auto
EnvironmentFile=/home/keita/keita-vps-auto/config/secrets.env
ExecStart=/usr/local/bin/claude -p "/keita-daily-cycle" \
  --output-format json \
  --permission-mode acceptEdits \
  --max-turns 80
StandardOutput=append:/var/log/keita/daily-cycle.log
StandardError=append:/var/log/keita/daily-cycle.error.log
TimeoutStartSec=3600

[Install]
WantedBy=multi-user.target
````

### 9-2. `keita-daily-cycle.timer`

````ini
[Unit]
Description=Keita-shiki Daily Cycle Timer (07:00 JST)

[Timer]
OnCalendar=*-*-* 07:00:00 Asia/Tokyo
Persistent=true
RandomizedDelaySec=60

[Install]
WantedBy=timers.target
````

### 9-3. `keita-position-check.timer`（昼の3回）

````ini
[Unit]
Description=Keita-shiki Position Check Timer (12, 15, 18 JST)

[Timer]
OnCalendar=*-*-* 12,15,18:00:00 Asia/Tokyo
Persistent=true
RandomizedDelaySec=120

[Install]
WantedBy=timers.target
````

### 9-4. `keita-inventory-check.timer`（深夜02:00）

````ini
[Unit]
Description=Keita-shiki Inventory Check (02:00 JST)

[Timer]
OnCalendar=*-*-* 02:00:00 Asia/Tokyo
Persistent=true

[Install]
WantedBy=timers.target
````

### 9-5. `keita-seller-rescore.timer`（毎週日曜03:00）

````ini
[Unit]
Description=Keita-shiki Weekly Seller Rescore (Sun 03:00 JST)

[Timer]
OnCalendar=Sun *-*-* 03:00:00 Asia/Tokyo
Persistent=true

[Install]
WantedBy=timers.target
````

### 9-6. `keita-order-webhook.service`（常駐）

````ini
[Unit]
Description=Keita-shiki Order Webhook Listener
After=network-online.target

[Service]
Type=simple
User=keita
WorkingDirectory=/home/keita/keita-vps-auto
EnvironmentFile=/home/keita/keita-vps-auto/config/secrets.env
ExecStart=/usr/bin/python3 -m keita.webhook_listener --port 8080
Restart=always
RestartSec=10
StandardOutput=append:/var/log/keita/webhook.log
StandardError=append:/var/log/keita/webhook.error.log

[Install]
WantedBy=multi-user.target
````

### 9-7. 一括有効化コマンド

````bash
# systemdユニットを配置
sudo cp ~/keita-vps-auto/systemd/*.service /etc/systemd/system/
sudo cp ~/keita-vps-auto/systemd/*.timer /etc/systemd/system/

# 再読み込みと有効化
sudo systemctl daemon-reload
sudo systemctl enable --now keita-daily-cycle.timer
sudo systemctl enable --now keita-position-check.timer
sudo systemctl enable --now keita-inventory-check.timer
sudo systemctl enable --now keita-seller-rescore.timer
sudo systemctl enable --now keita-order-webhook.service

# 状態確認
systemctl list-timers --all | grep keita
````

---

## 第10章 ログ・モニタリング設計

### 10-1. ログ階層

````
/var/log/keita/
├── daily-cycle.log              # Coordinator実行ログ
├── daily-cycle.error.log
├── position-check.log
├── inventory-check.log
├── seller-rescore.log
├── webhook.log
├── webhook.error.log
├── hooks/
│   ├── profit_margin_blocks.log
│   ├── phase_compliance_blocks.log
│   ├── vero_blacklist_blocks.log
│   └── rate_limit_defers.log
└── structured/
    └── errors.jsonl              # 構造化エラー全件
````

### 10-2. 構造化ログフォーマット

各エージェント・各フックの出力は以下のJSONL形式で `structured/` に保存。

````json
{
  "timestamp": "2026-05-03T07:23:00+09:00",
  "level": "ERROR",
  "agent": "SA-1",
  "phase": "①",
  "event_type": "api_rate_limit_exceeded",
  "context": {
    "seller_id": "japan_camera_2024",
    "api_source": "ebay_browse",
    "quota_used": "5000/5000"
  },
  "alternative": "Switch to Apify backup",
  "isError": true,
  "escalated_to_human": false
}
````

### 10-3. 監視ダッシュボード（推奨：Grafana + Loki）

````
[Grafana Dashboard]
├── パイプライン実行状況
│   ├── 直近24h成功率
│   ├── 平均実行時間
│   └── Hook block発生率
├── ビジネスKPI
│   ├── 日次出品数（rotation/longtail内訳）
│   ├── auto_approved率
│   ├── human_review_required率
│   └── 平均profit_margin
├── システム健全性
│   ├── API rate limit残量
│   ├── VPS CPU/Memory
│   └── DB容量
└── エスカレーション
    ├── confidence < 0.7 件数
    ├── VeRO警告件数
    └── borderline margin件数
````

VPSスペックを抑えたい場合は、Promtail + Loki + Grafanaのオールインワンを別VPSに分離する。

### 10-4. アラート

| 条件 | チャネル | 重大度 |
|---|---|---|
| Hook block 1時間内10件超 | Discord critical | high |
| VeRO警告検知 | Discord critical | critical |
| API rate limit到達 | Discord notify | medium |
| Coordinator timeout | Discord critical | high |
| daily-cycle 連続2日失敗 | Discord critical + Email | critical |
| profit_margin平均が10日連続で15%下回る | Discord notify | high（ビジネス課題）|

---

## 第11章 障害復旧手順

### 11-1. eBay API停止時

| API | 影響範囲 | 代替 |
|---|---|---|
| Browse API停止 | フェーズ①Sold取得不可 | Marketplace Insights or Apify |
| Marketplace Insights停止 | 公式Sold履歴不可 | Browse API + Apify |
| Inventory API停止 | フェーズ④出品不可 | Trading API（旧API） |
| Feed API停止 | バルク出品不可 | Inventory APIで個別出品（slowdown）|

`hooks/check_rate_limit.py` でcircuit breaker発動 → 代替APIへ自動切替 → Discord notify。

### 11-2. eBay規約変更時

新しいUser Agreement公布を検知したら（手動 or Web監視）：

1. `/keita-emergency-halt` で全パイプライン停止
2. 該当条文を読解
3. 影響範囲を特定（特に自動化禁止条項の追加）
4. Hook・MCP境界を更新
5. 段階的に再開（まずは1日10件で様子見）

### 11-3. メルカリ等仕入れ先のToS変更

1. Discordで外注さんへ即時通知
2. 影響範囲を判断（業者使用禁止強化、API呼び出し回数制限等）
3. 必要なら supply_checker を該当プラットフォームから外す
4. 楽天・Yahoo!ショッピングへの依存度を上げる

### 11-4. VPS障害

1. 別VPSへ即時切替（事前に冷たいスタンバイを準備）
2. systemd unitsとPythonコードを再デプロイ
3. SQLite DBのバックアップから復旧
4. JSONLは追記専用なので、最終バックアップ以降はlost
5. 再起動後、最初の/keita-status でパイプライン健全性チェック

### 11-5. データ破損

````
/var/backups/keita/
├── daily/         # 日次バックアップ（30日保持）
├── weekly/        # 週次（12週保持）
└── monthly/       # 月次（12ヶ月保持）
````

`borgbackup`または`restic`で暗号化して別リージョンへ。

---

## 第12章 環境変数・シークレット管理

### 12-1. `config/secrets.env`（gitignore対象）

````bash
# eBay
EBAY_APP_ID=<YOUR_EBAY_APP_ID>
EBAY_CERT_ID=<YOUR_EBAY_CERT_ID>
EBAY_DEV_ID=<YOUR_EBAY_DEV_ID>
EBAY_AUTH_TOKEN=<YOUR_EBAY_USER_TOKEN>
EBAY_REFRESH_TOKEN=<YOUR_EBAY_REFRESH_TOKEN>

# Buyee
BUYEE_API_KEY=<YOUR_BUYEE_API_KEY>

# 楽天
RAKUTEN_APP_ID=<YOUR_RAKUTEN_APP_ID>

# Yahoo!ショッピング
YAHOO_APP_ID=<YOUR_YAHOO_APP_ID>

# DeepL
DEEPL_API_KEY=<YOUR_DEEPL_API_KEY>

# Notion
NOTION_API_KEY=<YOUR_NOTION_API_KEY>
NOTION_DATABASE_ID=<YOUR_NOTION_DATABASE_ID>

# Discord
DISCORD_BOT_TOKEN=<YOUR_DISCORD_BOT_TOKEN>
DISCORD_WEBHOOK_ORDERS=<YOUR_WEBHOOK_URL_FOR_ORDERS>
DISCORD_WEBHOOK_OUTSOURCE=<YOUR_WEBHOOK_URL_FOR_OUTSOURCE>
DISCORD_WEBHOOK_CRITICAL=<YOUR_WEBHOOK_URL_FOR_CRITICAL>

# Apify (backup scraping)
APIFY_API_TOKEN=<YOUR_APIFY_TOKEN>

# Oxylabs (backup scraping)
OXYLABS_USERNAME=<YOUR_OXYLABS_USERNAME>
OXYLABS_PASSWORD=<YOUR_OXYLABS_PASSWORD>

# OpenExchangeRates (FX)
OXR_APP_ID=<YOUR_OXR_APP_ID>

# Anthropic API (使用するなら)
ANTHROPIC_API_KEY=<YOUR_ANTHROPIC_API_KEY>
````

### 12-2. ファイル権限

````bash
chmod 600 ~/keita-vps-auto/config/secrets.env
chown keita:keita ~/keita-vps-auto/config/secrets.env
````

VPS上では`secrets.env`は600固定。systemd serviceの`EnvironmentFile=`で読み込む。

### 12-3. .gitignore

````
config/secrets.env
data/sqlite/keita.db
data/jsonl/*.jsonl
data/logs/
*.pyc
__pycache__/
.venv/
node_modules/
````

---

## 第13章 config.yaml（受講生が埋める個別設定）

`~/keita-vps-auto/config/config.yaml`

````yaml
# ============================================
# 受講生個別設定（各自の状況に応じて埋める）
# ============================================

ebay_account:
  username: "<YOUR_EBAY_USERNAME>"
  current_feedback_score: 0          # 現在の評価値
  store_subscription: "none"          # none / Starter / Basic / Premium / Anchor / Enterprise
  monthly_listing_limit_amount_usd: 0
  monthly_listing_limit_count: 0

target:
  monthly_profit_jpy: 300000          # 月利目標
  achievement_deadline: "2026-12-31"

vps:
  provider: "sakura"                  # sakura / aws / conoha / vultr
  region: "tokyo"
  os: "ubuntu-24.04"

api_strategy:
  ebay_marketplace_insights:
    invitation_status: "pending"      # pending / approved / denied
  ebay_browse:
    daily_quota: 5000
    usage_cap_pct: 0.80
  buyee:
    plan: "<YOUR_BUYEE_PLAN>"
  scraping_backup:
    enabled: true
    primary: "apify"                  # apify / oxylabs
    fallback: "oxylabs"

profitability:
  minimum_profit_margin: 0.15
  borderline_low: 0.13
  fx_buffer_pct: 0.05
  ebay_fvf: 0.1315
  ebay_intl_fee: 0.0149
  ebay_per_order_usd: 0.30
  tax_refund_rate: 0.07

notifications:
  channels:
    - "discord"
  discord:
    orders_channel_id: "<YOUR_CHANNEL_ID>"
    outsource_channel_id: "<YOUR_CHANNEL_ID>"
    critical_channel_id: "<YOUR_CHANNEL_ID>"
  morning_summary_time: "06:00"
  daily_report_time: "21:00"

outsource:
  contractors:
    - id: "<OUTSOURCER_ID_1>"
      discord_user_id: "<DISCORD_USER_ID_1>"
      timezone: "Asia/Tokyo"
      max_concurrent_tasks: 5
      preferred_platforms: ["mercari", "yahoo_auctions"]

shipping:
  provider: "fedex_ficp"              # fedex_ficp / dhl / cpass
  rate_table_path: "config/shipping_rates.yaml"
````

---

## 第14章 Doc 3への接続

Doc 2で詳細設計を完了した。次のDoc 3では：

1. **ゼロ実装検証手順**：手書きで1件のSoldListingDigestをフェーズ①〜⑤通す
2. **API取得手順**：eBay Developer Program / Marketplace Insights招待申請 / Buyee / 楽天 / Yahoo!ショッピング / DeepL / Notion / Discord
3. **規約遵守チェックリスト**：eBay / メルカリ / 各APIの最新ToS確認
4. **段階的デプロイ手順**：開発環境→stagingアカウント→本番アカウント
5. **運用開始後30日のKPI監視項目**

を扱う。実装着手はDoc 3の「ゼロ実装検証パス後」となる。

---

## 第15章 改訂履歴

| バージョン | 日付 | 変更内容 |
|---|---|---|
| v2.0 (MAX) | 2026年5月3日 | Anthropic公式準拠の中級者向け詳細設計編として新規作成 |

---

## 第16章 出典・参照

- Anthropic公式ドキュメント（platform.claude.com / docs.claude.com）
- Anthropic Cookbook（github.com/anthropics/anthropic-cookbook）
- FastMCP（github.com/jlowin/fastmcp）
- Anthropic Engineering Blog "Building Effective Agents"
- eBay Developer Documentation（developer.ebay.com）
- eBay User Agreement（2026年2月20日改訂版）
- メルカリ利用規約（2025年10月改訂版）
- Buyee API Documentation
- 楽天市場API（webservice.rakuten.co.jp）
- Yahoo!ショッピングAPI（developer.yahoo.co.jp）
- DeepL API（developers.deepl.com）
- Notion API（developers.notion.com）
- Discord Developer Portal（discord.com/developers）
- ケイタ社長の書籍・動画教材（受講生は別途参照）

---

**Doc 2（詳細設計編）終了。Doc 3（実装チェックリスト編）に続きます。**