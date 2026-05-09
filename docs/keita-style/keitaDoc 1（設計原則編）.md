markdown# ケイタ式 VPS完全自動化システム 設計原則編

**ドキュメント番号：** Doc 1 / 3
**バージョン：** v2.0 (MAX)
**発行日：** 2026年5月3日
**対象読者：** Anthropic Claude Code（CLAUDE.md / Hooks / Slash Commands / MCP / Headless Mode）の運用経験があり、本番環境にエージェントシステムを投入した経験のあるケイタ式受講生
**前提知識：**
- ケイタ式の基本フロー（ライバルセラー逆引き・回転商品/ロングテール・評価フェーズ別戦略・利益率15%基準）
- Anthropic公式CCA-Fの5ドメイン
- eBay Developer Program（Browse / Marketplace Insights / Inventory / Feed / Trading の各API仕様）
- JSON Schema Draft 2020-12 / OAuth 2.0 / Webhookハンドリング

---

## 0. このドキュメントの位置づけ

本書は、ケイタ式eBay輸出ビジネスをVPS上でAnthropic公式エージェント設計5ドメインに準拠した形で完全自動化するための、設計原則編である。3部作の1冊目に当たる。

| ドキュメント | 内容 | 紙幅 |
|---|---|---|
| **Doc 1：設計原則編**（本書） | アーキテクチャ意思決定・5ドメインのケイタ式適用・設計トレードオフ | 設計判断の根拠 |
| Doc 2：詳細設計編 | 5エージェント完全定義・全JSON Schema・全Hooks YAML・MCP実装 | 実装可能な仕様書 |
| Doc 3：実装チェックリスト編 | ゼロ実装検証手順・API取得手順・規約遵守チェック・運用開始手順 | 実装からデプロイまで |

Doc 1の役割は、Doc 2の詳細設計に踏み込む前に「**なぜこの設計を選び、別の選択肢を捨てたのか**」を明確にすることである。実装段階で別の方針に揺らがないよう、ここで全ての設計トレードオフを言語化する。

---

## 1. ケイタ式の構造と自動化要件の翻訳

### 1-1. ケイタ式の構造を技術要件に翻訳する

ケイタ式は本質的に以下のサイクルで構成される。

```
ライバルセラー監視（Sold Listing取得）
  → 仕入れ可否確認（国内マーケットプレイス検索）
  → 利益率計算（為替・送料・手数料・還付込み）
  → 回転/ロングテール分類 × 評価フェーズ整合チェック
  → eBay出品（タイトル翻訳・カテゴリ・広告率・価格）
  → 注文受領（Webhook）→ 仕入れ実行（人間） → 発送代行
```

これを技術要件に翻訳すると以下になる。

| ケイタ式の概念 | 技術要件 |
|---|---|
| 需要逆算型 | データソースの優先順位は「Sold Listing > Active Listing」 |
| ライバルセラー1000人収集 | 大量並列スクレイピング＋スケジューリング |
| Ship from Japan・コンバージョン5%以上のフィルタ | セラースコアリング層が必要 |
| 回転商品の毎日ポジションチェック | カテゴリ別の差分監視ジョブ |
| 評価フェーズ別の出品比率制御 | 出品実行前の決定論的Hook |
| 利益率15%絶対基準 | 出品実行前の決定論的Hook |
| 雑務地獄からの脱出 | 人間介入ポイントを「規約上必要な箇所」のみに限定 |

設計の指針は明確：**ケイタ式の各原則を、プロンプトでの依頼ではなくHook・JSON Schema・MCP境界で物理的に強制する**。これがAnthropic公式の「Deterministic Guarantee」の実践である。

### 1-2. 規約遵守という絶対制約

2026年2月20日施行のeBay新User Agreementは、AIエージェントによる自動購入を明示的に禁じた。

> "use any robot, spider, scraper, data mining tools, data gathering and extraction tools, or other automated means (including, without limitation buy-for-me agents, LLM-driven bots, or any end-to-end flow that attempts to place orders without human review) to access our Services for any purpose, except with the prior express permission of eBay"

この条文は本設計の境界線を決定する。**eBay側では出品の自動化は合法だが、購入の自動化は違反**。仕入れ側（メルカリ等）も同様にToSが厳格化している（メルカリは2025年10月に業者使用禁止を明文化）。

ゆえに本設計では、自動化境界を以下に固定する。

```
[100%自動化]
  ┌─ ライバルセラーSold Listing監視
  ├─ 仕入れ可否確認（API経由）
  ├─ 利益率計算・採算判定
  ├─ eBay出品データ生成
  └─ eBayへの自動出品（Inventory/Feed API）

[人間介在（規約遵守の戦略的配置）]
  ┌─ 注文受領 → 外注タスク自動生成
  ├─ メルカリ等での仕入れ実行（既存信頼外注さんが人間として購入）
  └─ 発送代行（既存運用継続）
```

「人間の確認なしのend-to-endフローではない」という構造を、**システム上で物理的に保証**するため、SA-5（outsource_dispatcher）の出力には「Notion APIでカード生成」までしか実装しない。**システムは購入を実行するツールを持たない**。これがTool Designの最重要設計判断である。

---

## 2. アーキテクチャ意思決定（Domain 1：27%）

### 2-1. Hub-and-Spoke × Fixed Pipelineの採用根拠

| 検討候補 | 採用 | 理由 |
|---|---|---|
| **Hub-and-Spoke + Fixed Pipeline** | ✅ | フェーズ順序が確定、再現性最重視、デバッグ容易 |
| Hub-and-Spoke + Dynamic Adaptive Decomposition | ❌ | 探索的タスクではないため過剰設計 |
| Mesh / Peer-to-Peer | ❌ | Anthropic公式が推奨せず、責任所在が曖昧化 |
| Single-Agent Monolith | ❌ | Tool Scopingが効かず、Principle of Least Privilege違反 |

ケイタ式は「監視→可否→採算→出品→注文」の順序が確定しており、各フェーズの実行主体も固定可能。Dynamic Adaptive Decompositionの動的判断ループを入れると、Hooksでの決定論的強制が困難になる。

### 2-2. エージェント数の決定（5体）

Anthropic公式推奨は3〜5体。本設計はその上限の5体を採用する。

| Agent | モデル | 役割 | フェーズ |
|---|---|---|---|
| **Coordinator** | Sonnet 4.5 | Fixed Pipeline統括・stop_reason判定・エスカレーション管理・週次再スコアリング | 全体 |
| **SA-1: seller_watcher** | Haiku 4.5 | Sold/Active Listing取得・セラーフィルタリング | ① |
| **SA-2: supply_checker** | Haiku 4.5 | Buyee/楽天/Yahoo API並列検索・画像マッチング・最安値抽出 | ② |
| **SA-3: profit_judge** | Sonnet 4.5 | 利益率計算・回転/ロング分類・評価フェーズ整合判定 | ③ |
| **SA-4: ebay_lister** | Sonnet 4.5 | DeepL翻訳・Inventory API登録・Feed APIバルク投入・広告率最適化 | ④ |
| **SA-5: outsource_dispatcher** | Haiku 4.5 | 注文Webhook受信・Notionタスク生成・Discord通知 | ⑤ |

#### モデル振り分けの根拠

| エージェント | モデル選定理由 |
|---|---|
| Coordinator: Sonnet 4.5 | 全体の文脈を保持し、stop_reason判定とエスカレーション基準の解釈が必要 |
| SA-1, SA-2, SA-5: Haiku 4.5 | 構造化APIコール＋データ整形のみ。判断要素が少ない。Haikuで十分かつ10倍以上コスト削減 |
| SA-3: Sonnet 4.5 | 利益率計算・分類判定・フェーズ整合の複合判断。Haikuだと精度低下の懸念 |
| SA-4: Sonnet 4.5 | DeepL翻訳結果の品質チェック・カテゴリ最適化判断・広告率調整。Sonnet以上が必要 |

Opus 4.7はエスカレーション時の人間判断補助のみで使用し、定常パイプラインからは除外する。これにより、24時間運用時のコストをSonnet/Haikuベースで最適化する。

### 2-3. Subagent間の通信ルール

```
[OK]
  Coordinator → SA-1 (Tool Use)
  SA-1 → Coordinator (構造化出力)
  Coordinator → SA-2 (SA-1の出力を必要部分のみ抽出して渡す)

[NG]
  SA-1 → SA-2 (直接通信)
  SA-2 → SA-1 (親文脈の継承)
  SA-3 → SA-1 (バックチャネル)
```

Subagentは親Coordinatorの文脈を継承しない。各Subagentは独立したセッションとして起動し、Coordinatorが必要なデータを抽出して明示的に渡す。これによりLost in the Middle対策とコンテキスト最小化が両立する。

### 2-4. Agentic Loopの停止条件

```python
# 全Subagent共通の停止判定（Coordinatorが実装）
def is_subagent_done(response):
    # NG: テキスト解析
    # if "完了" in response.text: return True
    
    # NG: 反復回数による打ち切り
    # if iteration_count >= 10: return True
    
    # OK: stop_reasonでの判定
    return response.stop_reason == "end_turn"
```

`stop_reason`が`max_tokens`や`tool_use`で返ってきた場合は、Coordinatorが再度Subagentを呼び直す。`end_turn`のみが完了サインである。

### 2-5. エスカレーショントリガーの明示化

```yaml
escalation_triggers:
  - trigger: confidence_below_threshold
    condition: SA-3.output.confidence < 0.7
    action: Discord notify to human, queue to manual_review
  
  - trigger: consecutive_api_failures
    condition: same_api_error_count >= 3
    action: Discord critical alert, halt pipeline
  
  - trigger: profit_margin_borderline
    condition: 0.13 <= profit_margin < 0.15
    action: Discord notify with recommendation, do_not_auto_list
  
  - trigger: vero_warning_detected
    condition: ebay_response.contains("VeRO")
    action: Discord critical alert, blacklist_listing_pattern
```

感情分析や自己評価信頼度ではなく、明示的・数値的トリガーで判定する。

---

## 3. ツール設計（Domain 2：18%）

### 3-1. Principle of Least Privilege の適用

各エージェントの`allowed_tools`を最小化する。

| Agent | allowed_tools | 理由 |
|---|---|---|
| Coordinator | dispatch_subagent, log_event, escalate_to_human, schedule_next_run, send_discord_critical | 統制と通知のみ |
| SA-1 | fetch_seller_sold_listings, fetch_seller_active_listings, score_seller | eBay読み取りのみ |
| SA-2 | search_buyee, search_rakuten, search_yahoo_shopping, match_image | 仕入れ検索のみ |
| SA-3 | calculate_profitability, classify_rotation_or_longtail, check_phase_fit, check_blacklist | 計算・判定のみ |
| SA-4 | translate_deepl, create_inventory_item, submit_feed, set_promoted_rate | eBay書き込み（出品のみ） |
| SA-5 | create_notion_task, send_discord_notification | 通知・タスク生成のみ |

**SA-5には購入実行ツールを与えない。** 「仕入れ実行ツールが存在しないので物理的に自動購入できない」状態を作ることで、規約遵守をシステムレベルで保証する。

### 3-2. Tool Definitionの記述例（SA-3 calculate_profitability）

```json
{
  "name": "calculate_profitability",
  "description": "Calculate profit margin for a single listing candidate using ケイタ式 formula (FX rate with 5% buffer, eBay fees, Promoted Listings ad, supply cost, domestic+international shipping, 7% tax refund). Input requires sold_price_usd, supply_cost_jpy, item category, and shipping policy ID. Output is structured profitability decision with confidence. Use this AFTER supply_check confirms availability. Do NOT use for items where supply_check returned no_match. If FX rate is older than 6 hours, this tool will return isError=true and require fresh fx_fetch first.",
  "input_schema": {
    "type": "object",
    "properties": {
      "candidate_id": {"type": "string", "format": "uuid"},
      "sold_price_usd": {"type": "number", "minimum": 0.01},
      "supply_cost_jpy": {"type": "integer", "minimum": 1},
      "category_id": {"type": "integer"},
      "shipping_policy_id": {"type": "string"},
      "promoted_rate_override": {"type": ["number", "null"], "minimum": 0, "maximum": 0.20},
      "fx_rate_usd_jpy": {"type": "number", "minimum": 100, "maximum": 200},
      "fx_fetched_at": {"type": "string", "format": "date-time"}
    },
    "required": ["candidate_id", "sold_price_usd", "supply_cost_jpy", "category_id", "shipping_policy_id", "fx_rate_usd_jpy", "fx_fetched_at"],
    "additionalProperties": false
  },
  "strict": true
}
```

descriptionには「いつ使うか・いつ使わないか・エラー条件」まで記述する。Anthropic公式の鉄則：**ツール選択判断はnameではなくdescriptionで行われる**。

### 3-3. MCP Server構成

| MCP Server | 用途 | 接続Subagent | 実装方針 |
|---|---|---|---|
| ebay-mcp | Browse / Marketplace Insights / Inventory / Feed / Trading | SA-1, SA-4 | 自作（Python + FastMCP） |
| buyee-mcp | Buyee検索API | SA-2 | 自作 |
| rakuten-mcp | 楽天市場API（公式・無料枠） | SA-2 | 自作 |
| yahoo-shopping-mcp | Yahoo!ショッピングAPI（公式） | SA-2 | 自作 |
| deepl-mcp | DeepL翻訳API | SA-4 | 既存OSS |
| notion-mcp | Notion API（タスク自動生成） | SA-5 | 公式 |
| discord-mcp | Discord Webhook | Coordinator, SA-5 | 既存OSS |
| fx-mcp | 為替レート取得（OpenExchangeRates） | SA-3 | 自作 |

#### MCP境界の設計判断

ebay-mcpとして1つにまとめるか、ebay-read-mcp / ebay-write-mcp に分割するかは検討の余地がある。本設計ではTool Scopingを優先して**ebay-read-mcp / ebay-write-mcp**に分割する。

```
ebay-read-mcp:
  - fetch_seller_sold_listings
  - fetch_seller_active_listings
  - fetch_marketplace_insights
  → SA-1のみが使用可能

ebay-write-mcp:
  - create_inventory_item
  - submit_feed
  - update_listing
  - end_listing
  → SA-4のみが使用可能
```

これにより、Capability Confusion（読み取り役が誤って書き込みする）を物理的に防ぐ。

### 3-4. Rate Limiting設計

| API | 公式制限 | 本設計での運用 |
|---|---|---|
| eBay Browse API | 5,000 calls/day | 4,000 calls/day（バッファ20%） |
| Marketplace Insights | 招待制（個別交渉） | 申請後に確定 |
| Trading API GetSellerList | 300 calls/15秒 | 200 calls/15秒 |
| Feed API LMS | 400 requests/day | 320 requests/day |
| Buyee API | プラン依存 | プラン上限の80% |
| 楽天市場API | 1 request/秒 | 1 request/1.5秒 |
| Yahoo!ショッピングAPI | 50,000 requests/day | 40,000 requests/day |

各MCP serverに`exponential_backoff`と`circuit_breaker`を実装する。3回連続失敗で当該API系統を一時停止し、Discord通知。

---

## 4. 実行環境設計（Domain 3：20%）

### 4-1. CLAUDE.md階層構造

```
~/.claude/CLAUDE.md
  ├─ Anthropic公式10の鉄則
  ├─ 共通NG語・文体ルール
  └─ Hooks強制ルールの参照

~/Desktop/keita-vps-auto/CLAUDE.md
  ├─ ケイタ式の核心原則3つ（評価フェーズ・利益率15%・回転/ロング比率）
  ├─ 規約遵守の絶対境界線
  ├─ エスカレーション基準
  └─ @.claude/rules/ への参照

~/Desktop/keita-vps-auto/agents/seller_watcher/CLAUDE.md
  ├─ SA-1のシステムプロンプト
  ├─ allowed_tools
  └─ Few-shot examples

~/Desktop/keita-vps-auto/agents/profit_judge/CLAUDE.md
~/Desktop/keita-vps-auto/agents/ebay_lister/CLAUDE.md
（以下、各エージェントごと）
```

### 4-2. .claude/rules/ のトピック別分割

```
.claude/rules/
├─ ebay-tos-compliance.md          # alwaysApply: true
├─ ketai-shiki-philosophy.md       # alwaysApply: true
├─ phase-strategy.md               # globs: agents/profit_judge/**
├─ outsource-protocol.md           # globs: agents/outsource_dispatcher/**
├─ profit-margin-15pct-strict.md   # alwaysApply: true
├─ rotation-longtail-criteria.md   # globs: agents/profit_judge/**
└─ vero-blacklist.md               # alwaysApply: true
```

`alwaysApply: true`のルールは全セッションで強制適用される。ケイタ式の絶対原則と規約遵守はここで物理的に守る。

### 4-3. Hooks設計（業務ルールの決定論的強制）

#### PreToolUse Hooks

```yaml
# 出品実行前：利益率15%チェック
- name: enforce_profit_margin_15pct
  match: tool_use.name == "create_inventory_item" || tool_use.name == "submit_feed"
  run: ./hooks/check_profit_margin.py
  description: |
    利益率が15%未満なら出品をブロック。borderline (13-15%)は警告通知。
  on_failure: block
  on_block_message: |
    PROFIT_MARGIN_VIOLATION: candidate_id={candidate_id} margin={margin} threshold=0.15

# 出品実行前：評価フェーズ整合チェック
- name: enforce_phase_compliance
  match: tool_use.name == "create_inventory_item"
  run: ./hooks/check_phase_compliance.py
  description: |
    現在のアカウント評価フェーズと、回転/ロング比率の整合をチェック。
    Phase 0 (eval 0-30) で longtail を出そうとしたら block。
  on_failure: block

# 出品実行前：VeRO（権利侵害）チェック
- name: enforce_vero_blacklist
  match: tool_use.name == "create_inventory_item"
  run: ./hooks/check_vero_blacklist.py
  description: |
    商品タイトル・ブランド名が VeRO blacklist に該当するなら block。
  on_failure: block

# 仕入れ可否チェック前：API rate limit確認
- name: enforce_rate_limit
  match: tool_use.name in ["search_buyee", "search_rakuten", "search_yahoo_shopping"]
  run: ./hooks/check_rate_limit.py
  on_failure: defer  # 制限到達なら次のサイクルへ繰り延べ
```

#### PostToolUse Hooks

```yaml
# eBay APIレスポンスの刈り込み（コンテキスト軽量化）
- name: trim_large_ebay_responses
  match: tool_use.name in ["fetch_seller_sold_listings", "fetch_marketplace_insights"]
  run: ./hooks/trim_responses.py
  description: |
    必要フィールド（listing_id, title, sold_price_usd, sold_date, image_urls, item_specifics）
    だけを残し、それ以外は削除してコンテキストを軽量化。

# 出品成功後：候補DBステータス更新
- name: update_candidate_status_on_listing_success
  match: tool_use.name == "submit_feed" && tool_result.status == "success"
  run: ./hooks/update_candidate_status.py

# エラー検知時：構造化エラーログ
- name: structured_error_logging
  match: tool_result.isError == true
  run: ./hooks/log_structured_error.py
```

`enforce_profit_margin_15pct`と`enforce_phase_compliance`がケイタ式の核心原則をシステムレベルで保証する。プロンプトは確率的（90%）、Hookは決定論的（100%）。

### 4-4. Slash Commands設計

```
/keita-daily-cycle           # 朝07:00定型実行（全フェーズ通し）
/keita-supply-check          # 仕入れ可否チェック単発
/keita-listing-now           # 出品候補の即時実行
/keita-position-check        # ポジションチェック（12:00, 15:00, 18:00）
/keita-status                # システム状態レポート
/keita-emergency-halt        # 全パイプライン緊急停止
/keita-fx-refresh            # 為替レート手動更新
/keita-blacklist-add         # 赤字パターン手動追加
```

各Slash CommandはYAML frontmatterで`allowed-tools`を制限する。`/keita-emergency-halt`は破壊的操作なので確認プロンプトを必須化する。

### 4-5. Headless Mode実行（VPS cron）

```bash
# /etc/systemd/system/keita-daily-cycle.service
[Unit]
Description=Keita-shiki Daily Cycle
After=network-online.target

[Service]
Type=oneshot
User=keita
WorkingDirectory=/home/keita/keita-vps-auto
ExecStart=/usr/local/bin/claude -p "/keita-daily-cycle" \
  --output-format json \
  --permission-mode acceptEdits \
  --max-turns 50
StandardOutput=append:/var/log/keita/daily-cycle.log
StandardError=append:/var/log/keita/daily-cycle.error.log

[Install]
WantedBy=multi-user.target
```

```bash
# /etc/systemd/system/keita-daily-cycle.timer
[Unit]
Description=Keita-shiki Daily Cycle Timer

[Timer]
OnCalendar=*-*-* 07:00:00 Asia/Tokyo
Persistent=true

[Install]
WantedBy=timers.target
```

`-p`フラグ・`--output-format json`・`--max-turns`は必須。`--permission-mode acceptEdits`はファイル書き込み権限を制限する。

### 4-6. ジョブスケジュール全体

| 時刻（JST） | ジョブ | 担当 | 想定実行時間 |
|---|---|---|---|
| 02:00 | inventory_check（在庫切れ自動取り下げ） | Coordinator → SA-2, SA-4 | 15分 |
| 06:00 | morning_summary（前日売上集計） | Coordinator | 5分 |
| 07:00 | daily_cycle（全フェーズ通し） | Coordinator → SA-1〜SA-4 | 30〜60分 |
| 09:00 / 12:00 / 15:00 / 18:00 | position_check | SA-1単独 | 10分 |
| 21:00 | daily_report（日次レポート） | Coordinator | 5分 |
| Webhook | order_received_handler | SA-5 | 即時 |
| 毎週日曜03:00 | seller_rescore（セラーリスト再スコアリング） | Coordinator → SA-1 | 60分 |

### 4-7. ディレクトリ構造

```
/home/keita/keita-vps-auto/
├── CLAUDE.md
├── .claude/
│   ├── rules/
│   │   ├── ebay-tos-compliance.md
│   │   ├── ketai-shiki-philosophy.md
│   │   ├── phase-strategy.md
│   │   ├── outsource-protocol.md
│   │   ├── profit-margin-15pct-strict.md
│   │   ├── rotation-longtail-criteria.md
│   │   └── vero-blacklist.md
│   └── commands/
│       ├── keita-daily-cycle.md
│       ├── keita-supply-check.md
│       ├── keita-listing-now.md
│       ├── keita-position-check.md
│       ├── keita-status.md
│       ├── keita-emergency-halt.md
│       ├── keita-fx-refresh.md
│       └── keita-blacklist-add.md
├── agents/
│   ├── coordinator/
│   │   ├── CLAUDE.md
│   │   └── system_prompt.md
│   ├── seller_watcher/
│   ├── supply_checker/
│   ├── profit_judge/
│   ├── ebay_lister/
│   └── outsource_dispatcher/
├── mcp/
│   ├── ebay-read-mcp/
│   ├── ebay-write-mcp/
│   ├── buyee-mcp/
│   ├── rakuten-mcp/
│   ├── yahoo-shopping-mcp/
│   ├── fx-mcp/
│   └── (deepl, notion, discord は外部接続)
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
│   ├── supply_check.schema.json
│   ├── profitability.schema.json
│   ├── listing_candidate.schema.json
│   └── outsource_task.schema.json
├── data/
│   ├── jsonl/
│   ├── sqlite/
│   └── logs/
├── config/
│   ├── config.yaml
│   ├── phase_strategy.yaml
│   └── secrets.env
└── tests/
```

---

## 5. JSON駆動設計（Domain 4：20%）

### 5-1. JSON駆動設計7ステップの本案件への適用

| Step | 内容 | 本案件の答え |
|---|---|---|
| Step 0 | 自動化すべきか | YES（28〜31項目の手順、毎日繰り返し、判断はルールベース） |
| Step 1 | 最終アウトプット1文 | 「24時間365日、ケイタ式に準拠した利益率15%以上の出品候補をeBayへ自動出品し、注文受領時に外注メルカリ購入タスクをNotionへ自動生成する」 |
| Step 2 | フェーズ分解 | ①Sold監視 ②仕入れ可否 ③採算/分類 ④出品 ⑤注文ハンドリング |
| Step 3 | JSON Schema設計 | 各フェーズの`tool_use` `input_schema`定義（後述） |
| Step 4 | データコントラクト | SoldListingDigest / SupplyMatch / ListingDraft / OrderEvent |
| Step 5 | 各フェーズの実行主体 | SA-1 / SA-2 / SA-3 / SA-4 / SA-5 |
| Step 6 | ゼロ実装検証 | Doc 3で扱う（手書き1件をフェーズ①〜⑤通す） |
| Step 7 | 実装着手 | Step 6合格後 |

### 5-2. データコントラクト4つの先行定義

#### SoldListingDigest（フェーズ① → フェーズ②）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "SoldListingDigest",
  "type": "object",
  "properties": {
    "digest_id": {"type": "string", "format": "uuid"},
    "generated_at": {"type": "string", "format": "date-time"},
    "seller_id": {"type": "string"},
    "listing_id": {"type": "string"},
    "title_en": {"type": "string"},
    "category_id": {"type": "integer"},
    "category_path": {"type": "string"},
    "condition_id": {"type": "integer"},
    "sold_price_usd": {"type": "number"},
    "sold_date": {"type": "string", "format": "date-time"},
    "shipping_cost_usd": {"type": "number"},
    "primary_image_url": {"type": "string", "format": "uri"},
    "image_urls": {"type": "array", "items": {"type": "string", "format": "uri"}},
    "item_specifics": {
      "type": "object",
      "properties": {
        "brand": {"type": ["string", "null"]},
        "model": {"type": ["string", "null"]},
        "mpn": {"type": ["string", "null"]},
        "year": {"type": ["string", "null"]}
      }
    },
    "supply_search_keys": {
      "type": "object",
      "properties": {
        "primary_jp_keyword": {"type": "string"},
        "alternative_jp_keywords": {"type": "array", "items": {"type": "string"}},
        "brand_jp": {"type": ["string", "null"]},
        "model_jp": {"type": ["string", "null"]}
      },
      "required": ["primary_jp_keyword", "alternative_jp_keywords"]
    },
    "data_source": {"type": "string", "enum": ["browse_api", "marketplace_insights", "apify", "oxylabs"]}
  },
  "required": ["digest_id", "generated_at", "seller_id", "listing_id", "title_en", "category_id", "sold_price_usd", "sold_date", "primary_image_url", "supply_search_keys", "data_source"],
  "additionalProperties": false
}
```

#### SupplyMatch（フェーズ② → フェーズ③）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "SupplyMatch",
  "type": "object",
  "properties": {
    "match_id": {"type": "string", "format": "uuid"},
    "digest_id": {"type": "string", "format": "uuid"},
    "checked_at": {"type": "string", "format": "date-time"},
    "supply_available": {"type": "boolean"},
    "best_match": {
      "type": ["object", "null"],
      "properties": {
        "platform": {"type": "string", "enum": ["mercari", "yahoo_auctions", "rakuten", "yahoo_shopping"]},
        "item_id": {"type": "string"},
        "url": {"type": "string", "format": "uri"},
        "title_jp": {"type": "string"},
        "price_jpy": {"type": "integer"},
        "domestic_shipping_jpy": {"type": "integer"},
        "total_cost_jpy": {"type": "integer"},
        "image_urls": {"type": "array", "items": {"type": "string", "format": "uri"}},
        "image_match_score": {"type": "number", "minimum": 0, "maximum": 1},
        "seller_rating": {"type": "string"}
      },
      "required": ["platform", "item_id", "url", "price_jpy", "total_cost_jpy", "image_match_score"]
    },
    "alternative_matches": {
      "type": "array",
      "items": {"$ref": "#/properties/best_match"}
    },
    "no_match_reason": {"type": ["string", "null"]},
    "confidence": {"type": "number", "minimum": 0, "maximum": 1}
  },
  "required": ["match_id", "digest_id", "checked_at", "supply_available", "confidence"],
  "additionalProperties": false
}
```

#### ListingDraft（フェーズ③ → フェーズ④）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "ListingDraft",
  "type": "object",
  "properties": {
    "draft_id": {"type": "string", "format": "uuid"},
    "digest_id": {"type": "string", "format": "uuid"},
    "match_id": {"type": "string", "format": "uuid"},
    "decided_at": {"type": "string", "format": "date-time"},
    "profitability": {
      "type": "object",
      "properties": {
        "passes_15pct_threshold": {"type": "boolean"},
        "calculated_profit_margin": {"type": "number"},
        "expected_profit_jpy": {"type": "integer"},
        "fx_rate_used": {"type": "number"},
        "fees_breakdown": {
          "type": "object",
          "properties": {
            "ebay_fvf_usd": {"type": "number"},
            "ebay_intl_fee_usd": {"type": "number"},
            "ebay_per_order_usd": {"type": "number"},
            "promoted_listings_fee_usd": {"type": "number"},
            "fx_conversion_fee_usd": {"type": "number"},
            "tax_refund_jpy": {"type": "integer"}
          }
        }
      },
      "required": ["passes_15pct_threshold", "calculated_profit_margin"]
    },
    "classification": {
      "type": "object",
      "properties": {
        "type": {"type": "string", "enum": ["rotation", "longtail", "unclear", "excluded"]},
        "phase_fit": {"type": "string", "enum": ["phase_0", "phase_1", "phase_2", "phase_3", "phase_4", "phase_5", "none"]},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1}
      },
      "required": ["type", "phase_fit", "confidence"]
    },
    "ebay_listing_payload": {
      "type": "object",
      "properties": {
        "title_en": {"type": "string", "maxLength": 80},
        "category_id": {"type": "integer"},
        "condition_id": {"type": "integer"},
        "price_usd": {"type": "number"},
        "quantity": {"type": "integer", "const": 1},
        "shipping_policy_id": {"type": "string"},
        "return_policy_id": {"type": "string"},
        "payment_policy_id": {"type": "string"},
        "promoted_listings_rate": {"type": "number"},
        "item_specifics": {"type": "object"},
        "description_html": {"type": "string"},
        "image_urls": {"type": "array", "items": {"type": "string", "format": "uri"}}
      },
      "required": ["title_en", "category_id", "condition_id", "price_usd", "shipping_policy_id"]
    },
    "approval_status": {"type": "string", "enum": ["auto_approved", "human_review_required", "rejected"]},
    "rejection_reason": {"type": ["string", "null"]}
  },
  "required": ["draft_id", "digest_id", "match_id", "decided_at", "profitability", "classification", "approval_status"],
  "additionalProperties": false
}
```

#### OrderEvent（フェーズ④ → フェーズ⑤）

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "OrderEvent",
  "type": "object",
  "properties": {
    "order_event_id": {"type": "string", "format": "uuid"},
    "received_at": {"type": "string", "format": "date-time"},
    "ebay_order_id": {"type": "string"},
    "ebay_buyer_username": {"type": "string"},
    "draft_id": {"type": ["string", "null"], "format": "uuid"},
    "supply_target": {
      "type": "object",
      "properties": {
        "platform": {"type": "string"},
        "url": {"type": "string", "format": "uri"},
        "title_jp": {"type": "string"},
        "price_jpy": {"type": "integer"},
        "image_urls": {"type": "array", "items": {"type": "string", "format": "uri"}}
      },
      "required": ["platform", "url", "price_jpy"]
    },
    "shipping_address": {
      "type": "object",
      "properties": {
        "name": {"type": "string"},
        "country": {"type": "string"},
        "state": {"type": ["string", "null"]},
        "city": {"type": "string"},
        "postal_code": {"type": "string"},
        "address_line_1": {"type": "string"},
        "address_line_2": {"type": ["string", "null"]}
      },
      "required": ["name", "country", "city", "postal_code", "address_line_1"]
    },
    "deadline_for_supply_purchase": {"type": "string", "format": "date-time"},
    "expected_profit_jpy": {"type": "integer"}
  },
  "required": ["order_event_id", "received_at", "ebay_order_id", "supply_target", "shipping_address", "deadline_for_supply_purchase"],
  "additionalProperties": false
}
```

### 5-3. confidence・enum逃げ道・nullable設計の徹底

すべての判定エージェントの出力に以下を含める：

| 項目 | 設計意図 |
|---|---|
| `confidence: number` (0-1) | 自己確信度。0.7未満で人間エスカレーション |
| `escalation_required: boolean` | 明示的トリガー |
| `escalation_reason: string \| null` | エスカレーション理由 |
| `enum`に`"unclear"` / `"none"` / `"excluded"` | 逃げ道。enumを真偽値的に作らない |
| `nullable`フィールド | API取れない可能性のあるフィールドは`["string", "null"]` |
| `additionalProperties: false` | 余計なフィールドの混入を防ぐ |
| `strict: true` | tool_use生成時に文法レベルで強制 |

### 5-4. Few-shot Promptingの徹底

各Subagentのシステムプロンプトに3〜5個の具体例を埋め込む。

```
SA-3 profit_judge のFew-shot例：

例1（rotation・自動承認）:
  入力: sold_price_usd=$45, supply_cost_jpy=2800, category="Cameras > Vintage", sold_30d_count=4
  出力: {"type": "rotation", "phase_fit": "phase_0", "passes_15pct_threshold": true, "confidence": 0.95}

例2（longtail・自動承認）:
  入力: sold_price_usd=$320, supply_cost_jpy=18000, category="Audio > Vintage", sold_30d_count=0, sold_total=2
  出力: {"type": "longtail", "phase_fit": "phase_3", "passes_15pct_threshold": true, "confidence": 0.88}

例3（excluded・低単価で赤字）:
  入力: sold_price_usd=$15, supply_cost_jpy=800, intl_shipping_jpy=2400
  出力: {"type": "excluded", "phase_fit": "none", "passes_15pct_threshold": false, "confidence": 0.99}

例4（unclear・エスカレーション）:
  入力: sold_price_usd=$120, supply_cost_jpy=8000, category="Watches", sold_30d_count=1
  出力: {"type": "unclear", "phase_fit": "phase_2", "confidence": 0.65, "escalation_required": true,
         "escalation_reason": "Borderline rotation/longtail and confidence < 0.7"}

例5（excluded・blacklisted brand）:
  入力: title="Supreme Box Logo Tee", category="Clothing"
  出力: {"type": "excluded", "phase_fit": "none", "passes_15pct_threshold": false, "confidence": 1.0,
         "escalation_reason": "VeRO risk: Supreme is on blacklist"}
```

---

## 6. コンテキスト管理・信頼性（Domain 5：15%）

### 6-1. Lost in the Middle対策

各Subagentのプロンプト構造：

```
[SECTION A: 冒頭 - 絶対ルール]
  ケイタ式の絶対ルール3つ
    1. 利益率15%未満は出品しない
    2. 評価フェーズに整合しない出品はしない
    3. VeRO blacklistの商品は除外
  規約遵守の絶対境界線
  confidence < 0.7 はescalation必須

[SECTION B: 中段 - 入力データ]
  当該フェーズの入力JSON（最小限に刈り込み済み）

[SECTION C: 末尾 - 判定指示]
  今回判定すべき具体的な質問
  出力スキーマの再掲（最終確認）
  Few-shot例3つ
```

中段が長くても、AとCで挟むことでLost in the Middleが起きにくい。

### 6-2. Generator-Verifier Pattern

| 役割 | 担当 | モデル | 起動方式 |
|---|---|---|---|
| Generator | SA-3 profit_judge | Sonnet 4.5 | Coordinator経由で呼び出し |
| Independent Verifier | profit_verifier | Sonnet 4.5 | 別セッション・親文脈なし |

VerifierにはGeneratorのpromptを渡さない。**ListingDraft（生成物）と判定基準（CLAUDE.md / .claude/rules/）だけ**を渡し、独立判断させる。GeneratorとVerifierが一致した場合のみauto_approved、不一致なら人間エスカレーション。

これは全候補に対して実行するとコスト過多なので、本設計では：
- `confidence >= 0.9` → Verifier省略（Generatorのみ）
- `0.7 <= confidence < 0.9` → Verifier呼び出し
- `confidence < 0.7` → Verifierスキップして直接人間エスカレーション

### 6-3. PostToolUseでの結果刈り込み

eBay APIレスポンスは1リスティング当たり数千トークン。500人のセラー × 各30件取得すると、刈り込みなしではコンテキストが破綻する。

```python
# hooks/trim_responses.py の概要
def trim_ebay_response(raw_response):
    return {
        "listings": [
            {
                "listing_id": item["itemId"],
                "title": item["title"][:120],
                "sold_price_usd": item["price"]["value"],
                "sold_date": item["soldDate"],
                "primary_image_url": item["image"]["imageUrl"],
                "category_id": item["categoryId"],
                "item_specifics": extract_essential_specifics(item),
            }
            for item in raw_response["itemSummaries"]
            if passes_basic_filter(item)
        ]
    }
```

`description`の冗長な部分・出品者プロフィール文・関連商品リンクは全て削除。これでトークン消費を1/10以下に削減。

### 6-4. Provenance（来歴）の保持

要約に頼らず、別ブロックで以下を保持する。

| データ | 保管先 | 形式 |
|---|---|---|
| 利益率15%・回転70%/ロング30%等の閾値 | `config/phase_strategy.yaml` | YAML |
| 評価フェーズ境界値（30, 100, 300, 500, 700） | `config/phase_strategy.yaml` | YAML |
| 各セラーのID・販売実績数値 | `data/sqlite/keita.db sellers` | SQLite |
| 為替レート（取得日時付き） | `data/jsonl/fx_rates.jsonl` | JSONL |
| 送料テーブル（FedEx/DHL/CPaSS） | `config/shipping_rates.yaml` | YAML |
| 過去赤字商品ブラックリスト | `data/sqlite/keita.db blacklist` | SQLite |
| API取得元・取得日時（全レコードに付与） | 各Schemaの`data_source`・`generated_at` | JSON |

要約のたびに数値が劣化するProgressive Summarizationを物理的に防ぐ。

### 6-5. 構造化エラーハンドリング

```json
{
  "isError": true,
  "error": {
    "type": "rate_limit_exceeded",
    "severity": "high",
    "source": "ebay_browse_api",
    "occurred_at": "2026-05-03T07:23:00+09:00",
    "occurred_during": "fetch_seller_sold_listings",
    "input_snapshot": {"seller_id": "japan_camera_2024", "days_back": 30},
    "cause": "Daily API quota reached (5000/5000)",
    "alternative": "Switch to Apify scraper backup",
    "auto_retry_at": "2026-05-04T00:00:00+09:00",
    "escalated_to_human": false
  }
}
```

「失敗しました」だけの汎用エラーは禁止。`type`・`severity`・`source`・`alternative`を必ず含める。

### 6-6. コスト最適化

#### Prompt Caching

| キャッシュ対象 | 想定削減率 |
|---|---|
| 各Subagentのシステムプロンプト | 90% |
| ケイタ式ルール（CLAUDE.md `alwaysApply`） | 90% |
| Few-shot example群 | 90% |
| JSON Schema定義 | 90% |

`cache_control: {type: "ephemeral"}` を各Subagentの起動時に指定。

#### Batch API活用

| ジョブ | 同期/非同期 | 理由 |
|---|---|---|
| daily_cycle（朝07:00） | 同期 | リアルタイム性が必要 |
| seller_rescore（毎週日曜03:00） | **Batch API** | 24時間以内に結果が出ればOK・50%割引 |
| inventory_check（毎日02:00） | **Batch API** | 同上 |
| position_check（昼の3回） | 同期 | 当日の判断に使う |

#### モデル振り分け再掲

| 用途 | モデル | コスト目安 |
|---|---|---|
| API取得・データ整形 | Haiku 4.5 | 最安 |
| 判定・分類 | Sonnet 4.5 | Haikuの3-4倍 |
| 翻訳・タイトル最適化 | Sonnet 4.5 | 同上 |
| エスカレーション補助 | Opus 4.7（手動）| Sonnetの5倍 |

定常運用の主力はHaikuとSonnet。Opusは人間エスカレーション時のみ。

---

## 7. 5フェーズの責務マッピング（全体俯瞰）

```
[フェーズ① ライバルセラーSold監視]
  Coordinator
    └→ SA-1 seller_watcher (Haiku 4.5)
         ├─ MCP: ebay-read-mcp.fetch_seller_sold_listings
         ├─ MCP: ebay-read-mcp.fetch_marketplace_insights
         ├─ Hook (PostToolUse): trim_large_ebay_responses
         └─ 出力: SoldListingDigest[]

[フェーズ② 仕入れ可否確認]
  Coordinator (SoldListingDigest[]を受領)
    └→ SA-2 supply_checker (Haiku 4.5) [並列実行可]
         ├─ MCP: buyee-mcp.search
         ├─ MCP: rakuten-mcp.search
         ├─ MCP: yahoo-shopping-mcp.search
         ├─ Tool: match_image (画像マッチング)
         ├─ Hook (PreToolUse): enforce_rate_limit
         └─ 出力: SupplyMatch[]

[フェーズ③ 採算判定・分類]
  Coordinator (SoldListingDigest + SupplyMatchを受領)
    ├→ SA-3 profit_judge (Sonnet 4.5)
    │   ├─ MCP: fx-mcp.get_current_rate
    │   ├─ Tool: calculate_profitability
    │   ├─ Tool: classify_rotation_or_longtail
    │   ├─ Tool: check_phase_fit
    │   ├─ Tool: check_blacklist
    │   └─ 出力: ListingDraft (status: auto_approved | human_review_required | rejected)
    │
    └→ profit_verifier (条件付き起動・Sonnet 4.5・別セッション)
         └─ 入力: ListingDraft + .claude/rules/ のみ
         └─ 出力: 一致/不一致

[フェーズ④ eBay自動出品]
  Coordinator (auto_approved な ListingDraftのみ)
    └→ SA-4 ebay_lister (Sonnet 4.5)
         ├─ MCP: deepl-mcp.translate
         ├─ MCP: ebay-write-mcp.create_inventory_item
         ├─ MCP: ebay-write-mcp.submit_feed
         ├─ Hook (PreToolUse): enforce_profit_margin_15pct
         ├─ Hook (PreToolUse): enforce_phase_compliance
         ├─ Hook (PreToolUse): enforce_vero_blacklist
         ├─ Hook (PostToolUse): update_candidate_status_on_listing_success
         └─ 出力: 出品完了通知

[フェーズ⑤ 注文ハンドリング]
  eBay Order Webhook (随時)
    └→ Coordinator
         └→ SA-5 outsource_dispatcher (Haiku 4.5)
              ├─ MCP: notion-mcp.create_task_card
              ├─ MCP: discord-mcp.send_notification
              └─ 出力: OrderEvent (Notionカード生成・Discord通知完了)
              
              ※ SA-5には購入実行ツールを与えない
              ※ メルカリでの購入は外注さん（人間）が実行
```

---

## 8. 設計トレードオフの言語化

実装段階で「やっぱり別の方針が良いのでは」と揺らぐのを防ぐため、検討した代替案と却下理由を明記する。

### 8-1. なぜRAGを使わないのか

| 候補 | 採用 |
|---|---|
| RAG（Retrieval-Augmented Generation）でセラーリスト検索 | ❌ |
| 構造化DB + Tool Useで検索 | ✅ |

理由：ケイタ式の判断は「セラーIDで完全一致検索 + 数値閾値判定」。曖昧検索ではない。RAGは過剰設計でコストも余計にかかる。

### 8-2. なぜLangChain / LlamaIndexを使わないのか

| 候補 | 採用 |
|---|---|
| LangChain / LlamaIndex | ❌ |
| Anthropic公式 + MCP直接接続 | ✅ |

理由：Anthropic公式の機能（Hooks・Slash Commands・MCP・CLAUDE.md）でケイタ式の要件は完全に満たせる。LangChain層を挟むと、Anthropicの新機能追従が遅れ、抽象化リーク（leaky abstraction）に苦しむ。

### 8-3. なぜOrchestrationにworkflow engineを使わないのか

| 候補 | 採用 |
|---|---|
| Airflow / Prefect / Temporal | ❌ |
| Coordinator Agent + systemd timer | ✅ |

理由：本案件のスケジュールは7パターン程度で、依存関係も単純。workflow engineを入れると運用コストが跳ね上がる。`systemd timer`で十分。500社規模に成長したら再検討。

### 8-4. なぜPostgreSQLを使わないのか

| 候補 | 採用 |
|---|---|
| PostgreSQL | ❌（初期は） |
| SQLite + JSONL | ✅ |

理由：初期データ規模は数千〜数万レコード。SQLiteで十分かつ運用コストゼロ。月利30万円達成後に必要なら移行する。**早すぎる最適化を避ける**。

### 8-5. なぜキューイングシステム（Redis/RabbitMQ）を使わないのか

| 候補 | 採用 |
|---|---|
| Redis / RabbitMQ | ❌（初期は） |
| Coordinator Agent + JSONL queue + cron | ✅ |

理由：同上。初期は単純構成で運用負荷を下げる。秒間10件超のWebhookが来るようになったら導入検討。

### 8-6. なぜスクレイピングを完全に避けないのか

| 候補 | 採用 |
|---|---|
| 公式APIのみ（Marketplace Insights招待待ち） | ❌（保留） |
| 招待降下までApify/Oxylabsで補完 | ✅（暫定） |

理由：Marketplace Insights API招待は1〜2週間〜数ヶ月かかる。その間にビジネスを止めるのは機会損失。スクレイピングはeBay公式ToSのグレーゾーンだが、リクエスト間隔を遵守し、招待降下後は段階的に依存度を下げる。

### 8-7. なぜメルカリ仕入れを外注に出すのか

| 候補 | 採用 |
|---|---|
| Buyee API経由で完全自動購入 | ❌ |
| 業者の仕入れ代行サービス | ❌ |
| 既存信頼外注さんが人間として購入 | ✅ |

理由：eBay新ToSの「人間の確認なしのend-to-endフロー」禁止条項を回避するため、人間介入ポイントを必ず1箇所作る。Buyee API経由でも自動購入にすると、end-to-endフローと判定されるリスクがある。**規約遵守 > 自動化率**。

---

## 9. 受講生が事前に整理すべき変数

Doc 2の詳細設計に入る前に、以下を整理してください。**これらが埋まっていないと、Doc 2のエージェント定義・Hooks設定・JSON Schemaを自分の文脈に当てはめられません。**

### 9-1. eBayアカウント情報

| 項目 | 自分の値 |
|---|---|
| 現在の評価値 | __________ |
| セラーレベル | (Above Standard / Top Rated / Below Standard) |
| 月間出品リミット（金額） | $__________ |
| 月間出品リミット（品数） | __________ |
| eBay Storeの契約 | (なし / Starter / Basic / Premium / Anchor / Enterprise) |
| Top-rated seller認定 | (有 / 無) |

### 9-2. 現状の経営数値

| 項目 | 自分の値 |
|---|---|
| 現在のアクティブ出品数 | __________ |
| 直近1ヶ月の売上 | $__________ |
| 直近1ヶ月の販売個数 | __________ |
| 直近1ヶ月の実質利益 | __________円 |
| 全体のコンバージョン率 | __________% |
| 平均広告費率 | __________% |
| 既存ライバルセラーリスト規模 | __________人 |

### 9-3. 目標設定

| 項目 | 自分の値 |
|---|---|
| 月利目標金額 | __________円 |
| 達成期限 | ______年______月 |
| 必要月商（逆算） | $__________ |
| 必要出品数（逆算） | __________品 |
| 必要セラーリスト規模（逆算） | __________人 |

### 9-4. インフラ・体制

| 項目 | 自分の値 |
|---|---|
| VPSプロバイダ | (さくら / AWS / ConoHa / Vultr / その他：______) |
| OSバージョン | __________ |
| 通知チャネル | (Discord / Slack / LINE / Email / 複数：______) |
| 外注さん（仕入れ実行） | (確保済 / 新規募集 / 業者代行 / 未定) |
| 発送代行 | (利用中 / 検討中 / 自己発送) |

### 9-5. API取得状況

| API | 取得状況 |
|---|---|
| eBay Developer Program | (取得済 / 申請中 / 未) |
| Marketplace Insights API招待 | (取得済 / 申請中 / 未) |
| Buyee API | (取得済 / 申請中 / 未) |
| 楽天市場API | (取得済 / 申請中 / 未) |
| Yahoo!ショッピングAPI | (取得済 / 申請中 / 未) |
| DeepL API | (取得済 / 未) |
| Notion API | (取得済 / 未) |
| Discord Webhook | (設定済 / 未) |
| OpenExchangeRates API | (取得済 / 未) |
| Apify or Oxylabs | (登録済 / 未) |

これらを埋めた状態でDoc 2に進む。

---

## 10. 設計原則チェックリスト（Doc 2に進む前の最終確認）

### Anthropic公式準拠

- [ ] Hub-and-Spoke + Fixed Pipelineを採用する理由を自分の言葉で説明できる
- [ ] エージェント数を5体に抑え、Coordinatorを1体だけ立てる
- [ ] Subagent間は直接通信させず、すべてCoordinator経由
- [ ] 各Subagentは親文脈を継承せず、必要文脈だけ明示的に受け取る
- [ ] `stop_reason == "end_turn"`での完了判定
- [ ] Principle of Least Privilegeで`allowed_tools`を最小化
- [ ] 1エージェントのツール数は10個以下
- [ ] descriptionに使い分け基準・エラー条件を含める
- [ ] CLAUDE.mdをUser/Project/Directoryの3階層で配置
- [ ] `.claude/rules/`でトピック別分割
- [ ] `alwaysApply: true`で核心原則を全セッション強制
- [ ] Slash Commandsで定型作業を呼び出し可能化
- [ ] Headless Mode（`-p`フラグ）でCI/CD実行
- [ ] 業務ルールはHookで決定論的強制（プロンプトでの依頼ではない）
- [ ] 構造化出力は`tool_use` + JSON Schema + `strict: true`
- [ ] `confidence`フィールドで確信度併記
- [ ] enumに`unclear`/`none`/`excluded`の逃げ道
- [ ] 取れない可能性のあるフィールドはnullable
- [ ] 重要情報は冒頭か末尾（Lost in the Middle対策）
- [ ] Generator-Verifier Patternで自己レビュー禁止
- [ ] PostToolUseで巨大レスポンスを刈り込み
- [ ] エラーは構造化（type/severity/source/alternative/isError）
- [ ] エスカレーションは明示的トリガー（数値・条件）
- [ ] Prompt Caching活用
- [ ] Batch APIで非リアルタイムジョブを50%割引
- [ ] モデル振り分け（Haiku/Sonnet/Opus）

### ケイタ式準拠

- [ ] 評価フェーズ別の出品比率制御をHookで強制
- [ ] 利益率15%基準をHookで強制
- [ ] VeRO blacklistをHookで強制
- [ ] 回転/ロングテール分類のenum定義
- [ ] Ship from Japan・コンバージョン5%以上のセラーフィルタ
- [ ] 仕入れ可否確認は公式API経由（Buyee/楽天/Yahoo）
- [ ] メルカリ等での仕入れ実行は人間（外注さん）が担当
- [ ] 注文Webhook → Notionカード自動生成 → Discord通知の構造
- [ ] SA-5には購入実行ツールを与えない（規約遵守の物理保証）

### eBay規約準拠

- [ ] eBay公式API利用が前提（Browse / Marketplace Insights / Inventory / Feed）
- [ ] 招待制API（Marketplace Insights）は申請済みまたは申請予定
- [ ] スクレイピングは招待降下までの暫定運用
- [ ] リクエスト間隔・rate limitを遵守
- [ ] 「人間の確認なしのend-to-endフロー」を構造的に回避

すべてチェックが入ったら、Doc 2へ進む。

---

## 11. Doc 2への接続

Doc 1で確立した設計原則に基づいて、Doc 2では以下を完全定義する。

1. 5体のエージェント定義（システムプロンプト全文・allowed_tools・Few-shot examples）
2. 6種のJSON Schema完全版（Seller / Listing / SoldListingDigest / SupplyMatch / ListingDraft / OrderEvent / OutsourceTask）
3. CLAUDE.md全文（User / Project / Directory各階層）
4. .claude/rules/ 全ファイル
5. Hooks YAML完全版（PreToolUse / PostToolUse 全エントリ）
6. Slash Commands YAML完全版
7. MCP server実装スケルトン（ebay-read / ebay-write / buyee / rakuten / yahoo-shopping / fx）
8. systemd unit / timer完全版
9. ログ・モニタリング設計（CloudWatch相当 or 自前）
10. 障害復旧手順（API停止・eBay規約変更・VPS障害）

---

## 12. 改訂履歴

| バージョン | 日付 | 変更内容 |
|---|---|---|
| v2.0 (MAX) | 2026年5月3日 | Anthropic公式エージェント設計5ドメイン準拠の中級者向け版として新規作成 |

---

## 13. 出典・参照

- Anthropic公式ドキュメント（platform.claude.com / docs.claude.com）
- Anthropic Cookbook（github.com/anthropics/anthropic-cookbook）
- Claude Certified Architect認定（CCA-F）試験対策資料
- Anthropic Engineering Blog "Building Effective Agents"
- ケイタ社長の書籍・動画教材（受講生は別途参照）
- eBay Developer Documentation（developer.ebay.com）
- eBay User Agreement（2026年2月20日改訂版）
- メルカリ利用規約（2025年10月改訂版）
- Buyee API Documentation
- 楽天市場API（webservice.rakuten.co.jp）
- Yahoo!ショッピングAPI（developer.yahoo.co.jp）

---

**Doc 1（設計原則編）終了。Doc 2（詳細設計編）に続きます。**