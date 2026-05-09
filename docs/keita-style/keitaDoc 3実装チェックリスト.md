# ケイタ式 VPS完全自動化システム 実装チェックリスト編

**ドキュメント番号：** Doc 3 / 3
**バージョン：** v2.0 (MAX)
**発行日：** 2026年5月3日
**対象読者：** Doc 1（設計原則編）・Doc 2（詳細設計編）を読了し、実装に着手する受講生
**前提：** Doc 1のチェックリストとDoc 2のリポジトリ構造を理解していること

---

## 0. このドキュメントの位置づけ

本書は、Doc 1・Doc 2で確立した設計を実装・デプロイ・運用するためのチェックリスト集である。Anthropic公式の7ステップ設計のうち**Step 6（ゼロ実装検証）→ Step 7（実装着手）→ デプロイ → 運用30日**を扱う。

| 章 | 内容 |
|---|---|
| 第1章 | API取得チェックリスト（10種） |
| 第2章 | 規約遵守チェックリスト |
| 第3章 | ゼロ実装検証手順（Step 6） |
| 第4章 | 環境構築チェックリスト |
| 第5章 | 段階的デプロイ（dev→staging→prod） |
| 第6章 | 運用開始前の最終チェック |
| 第7章 | 運用30日のKPI監視 |
| 第8章 | トラブルシューティング |
| 第9章 | スケール時の判断基準 |

**Step 6（ゼロ実装検証）が最重要。** ここを通さずにコードを書くのは、Anthropic公式の鉄則違反である。本書は徹底的にStep 6に紙幅を割いている。

---

## 第1章 API取得チェックリスト

実装に着手する前に、以下のAPI取得を完了させること。Marketplace Insightsは数週間かかる可能性があるので、最優先で申請する。

### 1-1. eBay Developer Program（必須・最優先）

#### 取得手順

- [ ] developer.ebay.com にアクセスし、eBayアカウントでログイン
- [ ] "Application Keys" ページで Production Keyset を作成
- [ ] App ID（Client ID）・Cert ID（Client Secret）・Dev IDを取得
- [ ] OAuth 2.0 User Tokenを取得（Inventory/Feed APIの書き込みに必須）
- [ ] Refresh Tokenを保存（User Tokenは2時間で期限切れ）
- [ ] secrets.envに記載：`EBAY_APP_ID` `EBAY_CERT_ID` `EBAY_DEV_ID` `EBAY_AUTH_TOKEN` `EBAY_REFRESH_TOKEN`

#### 利用可能なAPI

- [ ] Browse API（Active Listingsの取得）
- [ ] Trading API（旧API・互換性用）
- [ ] Inventory API（出品作成）
- [ ] Feed API（バルク出品）
- [ ] Account API（ポリシー管理）

#### 注意事項

- Sandbox環境でまずテスト → Productionへ
- eBayアカウントが Below Standard だとAPI使用に制限がかかる場合あり
- 登録から実利用まで通常1〜3日

### 1-2. eBay Marketplace Insights API（招待制・最優先で申請）

#### 取得手順

- [ ] developer.ebay.com の "Marketplace Insights API" ページで申請
- [ ] 申請理由（自社の出品最適化のため）を明記
- [ ] 月間想定コール数を記載
- [ ] 承認待ち（数週間〜数ヶ月）
- [ ] 承認後、Production Keysetに権限が追加される

#### 招待降下までの暫定運用

- [ ] Browse API + Apify or Oxylabsスクレイピングで補完
- [ ] 招待降下後はスクレイピング停止 or 補完用途のみに限定
- [ ] config.yaml の `ebay_marketplace_insights.invitation_status` を `approved` に更新

### 1-3. Buyee API

#### 取得手順

- [ ] buyee.jp の Developer Portal でアカウント作成
- [ ] APIプラン選択（受講生は商用プラン推奨）
- [ ] APIキー発行を依頼
- [ ] secrets.envに記載：`BUYEE_API_KEY`

#### 利用範囲

- [ ] 検索：メルカリ・ヤフオク統合
- [ ] 商品詳細取得
- [ ] 在庫確認
- [ ] 価格取得

**注意：** Buyee APIは「仕入れ可否確認・価格取得」専用に使う。実際の購入は外注さんが手動で行う。

### 1-4. 楽天市場API

#### 取得手順

- [ ] webservice.rakuten.co.jp でアプリ登録（無料）
- [ ] アプリIDを取得
- [ ] secrets.envに記載：`RAKUTEN_APP_ID`

#### 利用上限（無料枠）

- 1秒あたり1リクエスト
- 月間アクセス数の上限なし（公平利用ガイドライン遵守）

### 1-5. Yahoo!ショッピングAPI

#### 取得手順

- [ ] developer.yahoo.co.jp でアプリ登録
- [ ] Client ID（App ID）を取得
- [ ] secrets.envに記載：`YAHOO_APP_ID`

#### 注意

- Yahoo!オークション公式APIは**廃止済み**。今回の用途では使わない
- Yahoo!ショッピングAPIで一部のヤフオク商品もカバー可

### 1-6. DeepL API

#### 取得手順

- [ ] developers.deepl.com でAPI Free（月50万文字無料）またはAPI Pro登録
- [ ] APIキーを取得
- [ ] secrets.envに記載：`DEEPL_API_KEY`

#### 推奨プラン

- 月間翻訳量が50万文字以下：Free
- それ以上：Pro（月750円〜）

### 1-7. Notion API

#### 取得手順

- [ ] notion.so の Settings → Integrations で内部統合を作成
- [ ] Internal Integration Tokenを取得
- [ ] 外注タスク用データベースを作成（Database ID取得）
- [ ] データベースをIntegrationに共有
- [ ] secrets.envに記載：`NOTION_API_KEY` `NOTION_DATABASE_ID`

#### 必須プロパティ

- task_id（Title）
- ebay_order_id（Text）
- 締切時刻（Date）
- 仕入れ先プラットフォーム（Select）
- 仕入れ先URL（URL）
- 仕入れ先タイトル（Text）
- 仕入れ価格（Number）
- 商品画像（Files）
- 状態（Select：created / notified / accepted / purchased / shipped / completed / failed）
- 担当外注ID（Select）
- 想定利益（Number）

### 1-8. Discord（Bot Token + Webhook）

#### 取得手順

- [ ] discord.com/developers でApplicationを作成
- [ ] Botを作成し、Bot Tokenを取得
- [ ] サーバーにBotを招待（権限：Send Messages, Embed Links, Attach Files）
- [ ] 3つのチャンネルを作成：`#orders` `#outsource` `#critical`
- [ ] 各チャンネルでWebhook URLを発行
- [ ] secrets.envに記載：`DISCORD_BOT_TOKEN` `DISCORD_WEBHOOK_ORDERS` `DISCORD_WEBHOOK_OUTSOURCE` `DISCORD_WEBHOOK_CRITICAL`

### 1-9. OpenExchangeRates API（為替レート）

#### 取得手順

- [ ] openexchangerates.org で無料アカウント作成（月1000コール無料）
- [ ] App IDを取得
- [ ] secrets.envに記載：`OXR_APP_ID`

#### 代替

- 月1000コール超える場合：Pro plan（月12 USD）
- 完全無料代替：exchangerate-api.com（月1500コール無料）

### 1-10. Apify or Oxylabs（スクレイピング・暫定）

#### Apifyの場合

- [ ] apify.com でアカウント作成
- [ ] APIトークンを取得
- [ ] eBay Scraperアクター（既製品）の使用権を購入
- [ ] secrets.envに記載：`APIFY_API_TOKEN`

#### Oxylabsの場合

- [ ] oxylabs.io で営業に問い合わせ（個人プランあり）
- [ ] eBay Scraper APIプランを選択（$49〜）
- [ ] Username / Passwordを取得
- [ ] secrets.envに記載：`OXYLABS_USERNAME` `OXYLABS_PASSWORD`

**注意：** Marketplace Insights APIが招待降下したらスクレイピングは停止 or 補完用途のみに限定。

---

## 第2章 規約遵守チェックリスト

実装前・実装中・運用中の各段階で必ず確認すること。

### 2-1. eBay規約（最重要）

- [ ] eBay User Agreement（2026年2月20日改訂版）を**全文読了**
- [ ] 「人間の確認なしのend-to-endフロー」禁止条項を理解した
- [ ] 自動購入ツールはシステムに搭載しない設計になっている
- [ ] SA-5 outsource_dispatcher の `allowed_tools` に購入実行ツールがない
- [ ] eBay公式API（Browse / Marketplace Insights / Inventory / Feed）を主軸にしている
- [ ] スクレイピングは Marketplace Insights 招待降下までの暫定運用と理解している
- [ ] 各APIのrate limitを80%以下に設定している
- [ ] VeRO blacklistを持ち、Hookで物理的に強制している
- [ ] eBay規約変更を月1回監視するプロセスがある

### 2-2. メルカリ利用規約

- [ ] メルカリ利用規約（2025年10月改訂版）を**全文読了**
- [ ] 業者使用禁止条項を理解した
- [ ] メルカリ直接スクレイピングはしない設計になっている
- [ ] Buyee APIを介して仕入れ可否確認している（Buyeeは合法ルート）
- [ ] 実際の購入は外注さん（人間）が個人アカウントで行う体制になっている
- [ ] 外注さんへの指示書に「購入は個人として行う・業者としては行わない」を明記

### 2-3. ヤフオク・楽天・Yahoo!ショッピング

- [ ] 楽天ウェブサービス利用規約を読了
- [ ] Yahoo!デベロッパーネットワーク利用規約を読了
- [ ] APIごとのrate limitを遵守する設計になっている
- [ ] スクレイピングではなく公式APIのみ使用している

### 2-4. その他のAPI規約

- [ ] DeepL API利用規約を読了（翻訳結果の保存・再利用範囲）
- [ ] Notion API利用規約を読了
- [ ] Discord Developer Terms of Serviceを読了
- [ ] OpenExchangeRates利用規約を読了

### 2-5. 法令・税務関連

- [ ] 古物商免許の取得状況を確認（中古品取扱なら必要）
- [ ] 個人事業主か法人かの選択を済ませている
- [ ] 消費税還付（仕入れ消費税7%）の認識が正しい（インボイス制度未登録仕入れ先の場合）
- [ ] 越境EC関連の輸出入規制を理解している
- [ ] 顧客個人情報の取り扱い（PIPA・GDPR準拠）を理解している

---

## 第3章 ゼロ実装検証手順（Step 6・最重要）

**コードを一行も書かずに、設計が本当に動くかを手書きで検証する**作業。Anthropic公式の鉄則であり、ここを飛ばすと実装で大量の手戻りが発生する。

### 3-1. ゼロ実装検証の目的

- スキーマに必須フィールドが足りないことを発見する
- Hooksの判定順序ミスを発見する
- MCP境界の切り方が実用的でないことを発見する
- データコントラクトの型ミスマッチを発見する
- 利益率計算式の数値ミスを発見する

### 3-2. 検証用テストケース3つ

実装着手前に、以下3つのテストケースを手書きで全フェーズ通す。

#### テストケースA：rotation・auto_approved（成功パス）

```yaml
シナリオ:
  ライバルセラーJapanCameraStoreが、Vintage Canon AE-1 Cameraを$45で売った
  メルカリで2,800円の同等品が見つかる
  利益率は20%以上、phase_0で適合、VeRO該当なし
  → 自動承認 → eBay自動出品 → 成功
```

#### テストケースB：unclear・human_review_required（境界線パス）

```yaml
シナリオ:
  ライバルセラーが$120のヴィンテージ時計を売った
  楽天で8,000円の同等品が見つかる
  利益率14%（borderline）
  → human_review_required → Discord通知 → 人間判断
```

#### テストケースC：excluded・rejected（拒否パス）

```yaml
シナリオ:
  ライバルセラーがSupreme Box Logo Teeを売った
  Buyeeで類似品が見つかる
  → VeRO blacklist hit → rejected → 出品されない
```

### 3-3. テストケースAの完全トレース

#### Phase ① — SA-1 seller_watcher

入力（Coordinatorが渡す）：

```json
{
  "seller_ids": ["JapanCameraStore"],
  "days_back": 30
}
```

SA-1がfetch_seller_sold_listings呼び出し→PostToolUse Hook（trim_responses）でレスポンス刈り込み→以下のSoldListingDigestを出力：

```json
{
  "digest_id": "550e8400-e29b-41d4-a716-446655440001",
  "generated_at": "2026-05-03T07:01:23+09:00",
  "seller_id": "JapanCameraStore",
  "listing_id": "175432109876",
  "title_en": "Vintage Canon AE-1 Camera 50mm Lens Tested Working Excellent+++",
  "category_id": 31388,
  "category_path": "Cameras & Photo > Vintage Cameras",
  "condition_id": 3000,
  "sold_price_usd": 45.00,
  "sold_date": "2026-05-02T14:23:00Z",
  "shipping_cost_usd": 0,
  "primary_image_url": "https://i.ebayimg.com/images/g/example/s-l1600.jpg",
  "image_urls": ["https://i.ebayimg.com/images/g/example/s-l1600.jpg"],
  "item_specifics": {
    "brand": "Canon",
    "model": "AE-1",
    "mpn": null,
    "year": "1976"
  },
  "supply_search_keys": {
    "primary_jp_keyword": "Canon AE-1 中古",
    "alternative_jp_keywords": ["キヤノン AE-1", "Canon AE-1 動作確認", "AE-1 ヴィンテージ"],
    "brand_jp": "キヤノン",
    "model_jp": "AE-1"
  },
  "data_source": "browse_api"
}
```

**検証ポイント：**
- [ ] SoldListingDigestスキーマに準拠しているか（必須フィールド全て埋まっているか）
- [ ] supply_search_keys が日本語化できているか
- [ ] data_source がenumに含まれる値か

#### Phase ② — SA-2 supply_checker

入力（Coordinatorが①の出力を渡す）：

```json
{"digest": <上記digest>}
```

SA-2が並列実行：
- search_buyee("Canon AE-1 中古")
- search_rakuten("Canon AE-1 中古")
- search_yahoo_shopping("Canon AE-1 中古")

それぞれの結果からmatch_imageでスコアリング→best_matchを選択→以下のSupplyMatchを出力：

```json
{
  "match_id": "660e8400-e29b-41d4-a716-446655440002",
  "digest_id": "550e8400-e29b-41d4-a716-446655440001",
  "checked_at": "2026-05-03T07:02:45+09:00",
  "supply_available": true,
  "best_match": {
    "platform": "mercari",
    "item_id": "m12345678901",
    "url": "https://buyee.jp/item/mercari/m12345678901",
    "title_jp": "Canon AE-1 動作品 50mm レンズ付き 美品",
    "price_jpy": 2800,
    "domestic_shipping_jpy": 800,
    "total_cost_jpy": 3600,
    "image_urls": ["https://static.mercdn.net/item/example.jpg"],
    "image_match_score": 0.92,
    "seller_rating": "良"
  },
  "alternative_matches": [
    {
      "platform": "yahoo_auctions",
      "item_id": "y987654321",
      "url": "https://buyee.jp/item/yahoo/y987654321",
      "price_jpy": 3200,
      "total_cost_jpy": 4000,
      "image_match_score": 0.88
    }
  ],
  "no_match_reason": null,
  "confidence": 0.92
}
```

**検証ポイント：**
- [ ] best_matchのimage_match_score >= 0.85（高信頼）
- [ ] total_cost_jpyが商品価格+国内送料の合計
- [ ] alternative_matchesが best_match より低スコア

#### Phase ③ — SA-3 profit_judge

入力（Coordinatorが①②の出力を渡す）：

```json
{
  "digest": <Phase①のdigest>,
  "match": <Phase②のmatch>,
  "current_phase": "phase_0",
  "current_feedback_score": 12
}
```

SA-3が以下を実行：

1. get_current_fx_rate → `{"rate": 150.50, "fetched_at": "2026-05-03T05:00:00Z"}`

2. calculate_profitability：

```
revenue_jpy = 45.00 × 150.50 × (1 - 0.05) = 6,433
ebay_fees_usd = 45 × 0.1315 + 45 × 0.0149 + 0.30 + 45 × 0.06 = 9.29
ebay_fees_jpy = 9.29 × 150.50 = 1,398
total_cost_jpy = 2800 + 800 + 1500 + 1398 + 50 = 6,548
tax_refund_jpy = 2800 × 0.07 = 196
net_profit_jpy = 6,433 - 6,548 + 196 = 81
profit_margin = 81 / 6,433 = 0.0126
```

**ここで問題発覚！** 利益率が1.26%しかない。テストケースAは「利益率20%以上」を想定していたが、計算してみると低い。

これがゼロ実装検証の真価である。**テストケースA（rotation・auto_approved成功パス）で、実際は利益率不足で rejected になるべき**ことが分かった。

#### Phase ③の修正版テストケース

利益率15%以上にするには、いずれかが必要：
- sold_price_usd を引き上げる（$60に変更）
- supply_cost_jpy を下げる（2,000円に変更）
- 国際送料を下げる（FedEx FICP使用で1,200円に下げる）

仮にsupply_cost_jpyを2,000円に変更：

```
revenue_jpy = 6,433
total_cost_jpy = 2000 + 800 + 1500 + 1398 + 50 = 5,748
tax_refund_jpy = 2000 × 0.07 = 140
net_profit_jpy = 6,433 - 5,748 + 140 = 825
profit_margin = 825 / 6,433 = 0.128
```

まだ borderline（13-15%）。さらに sold_price_usd を$60に上げる：

```
revenue_jpy = 60 × 150.50 × 0.95 = 8,579
ebay_fees_jpy = (60 × 0.1315 + 60 × 0.0149 + 0.30 + 60 × 0.06) × 150.50 = 1,830
total_cost_jpy = 2000 + 800 + 1500 + 1830 + 50 = 6,180
tax_refund_jpy = 140
net_profit_jpy = 8,579 - 6,180 + 140 = 2,539
profit_margin = 2,539 / 8,579 = 0.296
```

これで29.6%、auto_approved。

**Phase③の出力（修正版）：**

```json
{
  "draft_id": "770e8400-e29b-41d4-a716-446655440003",
  "digest_id": "550e8400-e29b-41d4-a716-446655440001",
  "match_id": "660e8400-e29b-41d4-a716-446655440002",
  "decided_at": "2026-05-03T07:03:12+09:00",
  "profitability": {
    "passes_15pct_threshold": true,
    "calculated_profit_margin": 0.296,
    "expected_profit_jpy": 2539,
    "fx_rate_used": 150.50,
    "fees_breakdown": {
      "ebay_fvf_usd": 7.89,
      "ebay_intl_fee_usd": 0.89,
      "ebay_per_order_usd": 0.30,
      "promoted_listings_fee_usd": 3.60,
      "fx_conversion_fee_usd": 0.10,
      "tax_refund_jpy": 140
    }
  },
  "classification": {
    "type": "rotation",
    "phase_fit": "phase_0",
    "confidence": 0.94
  },
  "ebay_listing_payload": {
    "title_en": "Vintage Canon AE-1 Camera 50mm Lens Tested Working Excellent+",
    "category_id": 31388,
    "condition_id": 3000,
    "price_usd": 60.00,
    "quantity": 1,
    "shipping_policy_id": "<YOUR_SHIPPING_POLICY_ID>",
    "return_policy_id": "<YOUR_RETURN_POLICY_ID>",
    "payment_policy_id": "<YOUR_PAYMENT_POLICY_ID>",
    "promoted_listings_rate": 0.06,
    "item_specifics": {
      "Brand": "Canon",
      "Model": "AE-1",
      "Year": "1976"
    },
    "image_urls": ["https://buyee-img.example.com/example.jpg"]
  },
  "approval_status": "auto_approved",
  "rejection_reason": null
}
```

**検証ポイント：**
- [ ] profit_margin >= 0.15（15%以上）
- [ ] phase_fitが現在フェーズに合致
- [ ] confidence >= 0.9（auto_approved条件）
- [ ] approval_status が "auto_approved"
- [ ] ebay_listing_payload に必須フィールド全て埋まっている
- [ ] price_usd が修正された値（$60）になっている

#### Phase ④ — SA-4 ebay_lister

入力（Coordinatorが auto_approved な ListingDraftを渡す）：

```json
{"draft": <Phase③のdraft>}
```

SA-4の処理フロー：

1. translate_deepl 呼び出し → タイトル翻訳確認

2. create_inventory_item 呼び出し前に**3つのPreToolUse Hook**が走る：

   a. `enforce_profit_margin_15pct`：
      - profit_margin = 0.296 >= 0.15 → ALLOW

   b. `enforce_phase_compliance`：
      - phase_fit = "phase_0", current_phase = "phase_0" → ALLOW
      - 現在の出品ポートフォリオで rotation/longtail 比率を確認 → 維持される → ALLOW

   c. `enforce_vero_blacklist`：
      - title に "Supreme" "LV" "Disney" 等は含まれない
      - brand "Canon" は blacklist になし → ALLOW

3. 全Hook通過後、create_inventory_item実行：

```json
{
  "sku": "KEITA-CANON-AE1-001",
  "product": {
    "title": "Vintage Canon AE-1 Camera 50mm Lens Tested Working Excellent+",
    "description": "<HTML description>",
    "imageUrls": [...]
  },
  "availability": {"shipToLocationAvailability": {"quantity": 1}},
  "condition": "USED_EXCELLENT"
}
```

4. submit_feed で公開：

```json
{"status": "success", "listing_id": "234567890123"}
```

5. PostToolUse Hook `update_candidate_status_on_listing_success` 発火 → SQLiteのcandidatesテーブル更新（status: created → listed）

**検証ポイント：**
- [ ] PreToolUse Hookすべて通過
- [ ] DeepL翻訳結果が80文字以内
- [ ] SKUが命名規則に従う
- [ ] eBay listingが作成された（listing_id取得）
- [ ] PostToolUse Hookで status 更新

#### Phase ⑤ — SA-5 outsource_dispatcher

入力（eBay Webhookが注文受領を通知）：

```json
{
  "ebay_order_id": "12-34567-89012",
  "ebay_buyer_username": "buyer_us_test",
  "listing_id": "234567890123",
  "sold_price_usd": 60.00,
  "shipping_address": {
    "name": "John Doe",
    "country": "US",
    "state": "CA",
    "city": "San Francisco",
    "postal_code": "94102",
    "address_line_1": "123 Market St"
  }
}
```

SA-5が以下を順次実行：

1. listing_id から ListingDraft を引く（draft_id取得）
2. supply_target を抽出（mercari URL・価格・画像）
3. deadline計算（受注+24時間）
4. create_notion_task でNotionにカード作成
5. send_discord_notification で `#outsource` チャンネルに外注さん向け通知
6. send_discord_notification で `#orders` チャンネルに運営者向け通知
7. data/jsonl/orders.jsonl に OrderEvent 追記

**検証ポイント：**
- [ ] Notionカードが作成された（page_id取得）
- [ ] Discord通知が外注・運営両方に届いた
- [ ] OrderEvent JSONL追記成功
- [ ] **SA-5には購入実行ツールがないので、ここで処理完了**

### 3-4. テストケースB・Cの簡易トレース

#### テストケースB（unclear・human_review）

- Phase ①〜②：rotation判定の境界
- Phase ③：profit_margin = 0.14（borderline 0.13-0.15）
- 出力：approval_status = "human_review_required"、Discord通知のみ
- Phase ④以降：実行されない
- **検証ポイント：** Hookで block されず、自動的にhuman_review_requiredステータスに遷移

#### テストケースC（excluded・rejected）

- Phase ①：title="Supreme Box Logo Tee"
- Phase ②：Buyeeで類似品見つかる
- Phase ③：check_blacklist でVeRO hit → rejected
- Phase ④：実行されない
- **検証ポイント：** Phase③で rejected され、ebay_lister へ進まない

### 3-5. ゼロ実装検証完了チェックリスト

3つのテストケースすべてを手書きで通した後、以下を確認：

- [ ] テストケースA（rotation・auto_approved）：5フェーズ全て通った
- [ ] テストケースB（unclear・human_review）：Phase③でhuman_review_requiredになった
- [ ] テストケースC（excluded・rejected）：Phase③でrejectedになった
- [ ] スキーマで型エラーが起きなかった
- [ ] Hookの判定順序に矛盾がなかった
- [ ] MCP境界（read/write分離）が実用的だった
- [ ] 利益率計算式に数値ミスがなかった
- [ ] confidenceとescalation_requiredが適切に設定された
- [ ] エラーケースが構造化エラー形式に従った

**3つすべて完了して初めて、Step 7（実装着手）に進む。**

---

## 第4章 環境構築チェックリスト

ゼロ実装検証完了後、VPS環境を構築する。

### 4-1. VPS基本設定

- [ ] さくらVPS（または他プロバイダ）契約済み
- [ ] Ubuntu 24.04 LTSインストール済み
- [ ] SSH鍵認証設定済み（パスワード認証無効）
- [ ] ufw（ファイアウォール）有効化、必要ポートのみ開放
- [ ] fail2banインストール済み
- [ ] 自動セキュリティアップデート有効化

### 4-2. ユーザー・権限

- [ ] 専用ユーザー `keita` 作成
- [ ] sudo権限付与
- [ ] keitaのHomeディレクトリに作業フォルダ配置

### 4-3. ランタイム

- [ ] Python 3.11+ インストール
- [ ] uv または pip + venv で仮想環境作成
- [ ] Node.js 20+ インストール（Notion-MCP・DeepL-MCP用）
- [ ] npm or pnpm インストール
- [ ] Claude Code CLI インストール（公式手順）

### 4-4. Claude Code設定

- [ ] `~/.claude/CLAUDE.md` 配置（Doc 2 第2-1節）
- [ ] `~/.config/claude/mcp_servers.json` 配置（Doc 2 第8-4節）
- [ ] `claude --version` で正常動作確認

### 4-5. リポジトリクローン

- [ ] `~/keita-vps-auto/` にDoc 2 第1章のディレクトリ構造を再現
- [ ] git initし、自分のプライベートGitHubリポジトリへpush
- [ ] .gitignoreで `secrets.env` `data/sqlite/*.db` を除外

### 4-6. シークレット管理

- [ ] `config/secrets.env` 作成、`chmod 600`
- [ ] 第1章で取得したAPIキー全て記載
- [ ] systemd serviceの `EnvironmentFile=` で読み込み確認

### 4-7. データベース

- [ ] SQLiteインストール（Ubuntu標準で含まれる）
- [ ] `data/sqlite/keita.db` 初期化
- [ ] schema作成スクリプト実行
- [ ] `data/jsonl/` ディレクトリ作成、書き込み権限確認

### 4-8. ログ

- [ ] `/var/log/keita/` 作成
- [ ] keitaユーザーに書き込み権限付与
- [ ] logrotate設定（30日保持）

### 4-9. MCP servers

- [ ] `mcp/ebay-read-mcp/` 実装・テスト起動
- [ ] `mcp/ebay-write-mcp/` 実装・テスト起動
- [ ] `mcp/buyee-mcp/` 実装・テスト起動
- [ ] `mcp/rakuten-mcp/` 実装・テスト起動
- [ ] `mcp/yahoo-shopping-mcp/` 実装・テスト起動
- [ ] `mcp/fx-mcp/` 実装・テスト起動
- [ ] DeepL/Notion/DiscordはOSS or 公式MCPを設置

### 4-10. Hooks

- [ ] `hooks/check_profit_margin.py` 実装、`chmod +x`
- [ ] `hooks/check_phase_compliance.py` 実装
- [ ] `hooks/check_vero_blacklist.py` 実装
- [ ] `hooks/check_rate_limit.py` 実装
- [ ] `hooks/trim_responses.py` 実装
- [ ] `hooks/update_candidate_status.py` 実装
- [ ] `hooks/log_structured_error.py` 実装
- [ ] `.claude/hooks.yaml` 配置

### 4-11. systemd

- [ ] `systemd/*.service` `*.timer` を `/etc/systemd/system/` にコピー
- [ ] `sudo systemctl daemon-reload`
- [ ] 各 timer/service を `enable --now`
- [ ] `systemctl list-timers --all | grep keita` で起動確認

---

## 第5章 段階的デプロイ（dev → staging → prod）

### 5-1. Phase Dev：開発環境（本番アカウント未接続）

**期間：** 1週間〜2週間
**目標：** すべてのフェーズが動作することを確認

- [ ] eBay Sandbox環境のAPIキーを使用
- [ ] テストセラー（自分が架空作成）で5件のSold Listingを作成
- [ ] フェーズ①〜⑤を全て手動trigger（`/keita-daily-cycle`）
- [ ] エラーが出たら修正→再実行
- [ ] 5件すべてが auto_approved → listed まで通ることを確認
- [ ] テストケースB・Cも実行し、想定通りの分岐になることを確認

### 5-2. Phase Staging：本番アカウント接続・出品停止モード

**期間：** 2週間
**目標：** 本番データで動くが、出品実行はskipして観察

- [ ] eBay Production APIキーに切替
- [ ] 既存ライバルセラーリストから10人をテスト用に選抜
- [ ] config.yaml の `dry_run: true` でDigest生成・採算判定までは実行、出品はskip
- [ ] 14日間、毎日 daily-cycle を実行
- [ ] 出力されるListingDraftを目視レビュー
- [ ] 採算判定の妥当性、profit_marginの正確性、phase_fitの整合を検証
- [ ] human_review_requiredが出る頻度を確認（多すぎなら閾値見直し）

### 5-3. Phase Prod 開始：出品開始・小規模

**期間：** 30日
**目標：** 1日1〜3件ずつ出品して安定運用を確認

- [ ] config.yaml の `dry_run: false`
- [ ] 1日の出品上限を `max_listings_per_day: 3` に制限
- [ ] 監視対象セラーは10人のまま
- [ ] 毎朝 Discord #orders で daily-summary を確認
- [ ] エスカレーション通知に即対応
- [ ] 1週間ごとに `borderline率` `rejected率` `human_review率` をレビュー

### 5-4. Phase Prod スケール：徐々に拡大

**期間：** 60日〜
**目標：** 月利目標達成までスケール

- [ ] 監視対象セラーを段階的に拡大（10→50→100→528人）
- [ ] 出品上限を段階的に緩和（3→10→30→無制限）
- [ ] Marketplace Insights API招待降下後、データソースをmarketplace_insights主軸に切替
- [ ] スクレイピング依存を段階的に下げる
- [ ] 評価フェーズが上がれば config.yaml の `current_feedback_score` を更新

---

## 第6章 運用開始前の最終チェック

Phase Prod 開始の朝に必ず確認すること。

### 6-1. システム整合

- [ ] CLAUDE.md（User / Project / Directory）3階層が配置されている
- [ ] .claude/rules/ の7ファイル全てが配置され、alwaysApply設定が正しい
- [ ] Hooks YAML が `.claude/hooks.yaml` に配置されている
- [ ] Hookスクリプト全てが実行可能（`chmod +x`）
- [ ] MCP serverが全て起動可能（手動で1個ずつ`python -m`して確認）
- [ ] systemd timer が全て enabled かつ active

### 6-2. シークレット

- [ ] secrets.env の全項目が `<YOUR_*>` のままになっていない
- [ ] secrets.env のパーミッションが600
- [ ] secrets.env がgitに含まれていない（`git status`で確認）

### 6-3. データ

- [ ] SQLite DBにテーブルが作成されている
- [ ] 監視対象セラーリスト（10人）が sellers.jsonl に登録済み
- [ ] phase_strategy.yaml に各フェーズの値が正しい
- [ ] vero-blacklist.md にあなたが取引しないブランドがリスト化済み
- [ ] shipping_rates.yaml にあなたの送料テーブルが記載済み

### 6-4. 通知

- [ ] Discord 3チャンネル（#orders #outsource #critical）が存在
- [ ] 各 Webhook URLが secrets.envに正しく記載
- [ ] テスト送信で各チャンネルに通知が届くことを確認
- [ ] 外注さんがDiscordサーバーに参加し、#outsourceの通知を受信できる状態
- [ ] 外注さんが Notion データベースを閲覧できる状態

### 6-5. 規約

- [ ] eBay User Agreement最新版を再確認（運用開始日に変更がないか）
- [ ] メルカリ利用規約最新版を再確認
- [ ] 各APIプロバイダの最新規約を再確認

---

## 第7章 運用30日のKPI監視

### 7-1. 毎朝確認すべき指標（Discord #orders に自動投稿）

```yaml
daily_summary:
  - 前日の daily-cycle 実行ステータス（成功 / 部分成功 / 失敗）
  - 前日の出品数（auto_approved → listed）
  - 前日のhuman_review_required件数
  - 前日のrejected件数 + 主な理由
  - 前日のescalation件数
  - 前日のVeRO警告件数
  - 累計監視対象セラー数 / 優良セラー数
  - 平均profit_margin（過去7日）
  - 累計listing数（rotation / longtail内訳）
```

### 7-2. 毎週レビュー

- [ ] auto_approved率（全digest中の割合）：目標30〜50%
- [ ] human_review_required率：目標10〜20%
- [ ] rejected率：目標30〜60%
- [ ] 受注率（出品×コンバージョン）：目標2〜5%
- [ ] 受注後 → 外注 accept までの平均時間：目標6時間以内
- [ ] 受注後 → 発送までの平均時間：目標48時間以内

### 7-3. 毎月レビュー

- [ ] 月商・月利・利益率（実績 vs 計画）
- [ ] フェーズ進行状況（評価値推移）
- [ ] セラーリスト再スコアリング結果
- [ ] VeRO blacklist の追加項目
- [ ] APIコスト合計（eBay / Anthropic / Apify / DeepL等）
- [ ] VPSコスト

### 7-4. KPI悪化時の対応

| 症状 | 想定原因 | 対応 |
|---|---|---|
| auto_approved率<20% | 利益率閾値が厳しすぎる or 仕入れ可否率が低い | supply_checker のマッチング閾値を見直し |
| rejected率>70% | セラーの質が低い or VeRO blacklistが過剰 | priority_tier C・EXCLUDED のセラーを除外 |
| human_review率>30% | profit_judgeのconfidenceが低い | Few-shot examplesを追加・モデルをOpusに昇格 |
| 受注率<1% | 価格・タイトル・カテゴリが最適化不足 | promoted_rate見直し・タイトル翻訳改善 |
| Hook block多発 | フェーズ進行とポートフォリオの不整合 | phase_strategy.yaml調整 |

---

## 第8章 トラブルシューティング

### 8-1. systemd timer起動しない

```bash
# 確認
sudo systemctl status keita-daily-cycle.timer
sudo systemctl status keita-daily-cycle.service
sudo journalctl -u keita-daily-cycle.service -n 50

# よくある原因
- ExecStartのパスミス
- EnvironmentFileの権限不足
- WorkingDirectoryが存在しない
```

### 8-2. Claude Codeがheadlessで止まる

```bash
# 必須フラグ忘れ
claude -p "/keita-daily-cycle" \
  --output-format json \
  --permission-mode acceptEdits \
  --max-turns 80
```

`--permission-mode`なしだとファイル編集権限の確認待ちで止まる。

### 8-3. Hookが実行されない

- `.claude/hooks.yaml` の配置場所を確認
- スクリプトに `chmod +x`
- shebang（`#!/usr/bin/env python3`）を冒頭に

### 8-4. MCP serverに接続できない

```bash
# 手動起動でエラー確認
cd ~/keita-vps-auto/mcp/ebay-read-mcp
python -m ebay_read_mcp.server
```

`mcp_servers.json` のパス・envが正しいか確認。

### 8-5. eBay APIエラー

| エラーコード | 原因 | 対応 |
|---|---|---|
| 1004 | 認証失敗 | Refresh Tokenで再取得 |
| 12031 | レート制限 | rate limit hookでdefer |
| 21916883 | VeRO違反 | blacklistへ追加 |
| 21919301 | カテゴリ不正 | category mapping見直し |

### 8-6. プロンプトcaching効かない

`cache_control: {type: "ephemeral"}` を各リクエストの該当ブロックに付与。
5分で期限切れ → 頻繁にアクセスする場面でのみ有効。

---

## 第9章 スケール時の判断基準

運用が安定し、月利30万円達成後の拡張判断。

### 9-1. PostgreSQL移行の判断

以下のいずれかを満たしたら検討：

- [ ] 監視対象セラー数 > 1,000人
- [ ] 月間出品数 > 1,000件
- [ ] SQLite DBサイズ > 1GB
- [ ] 同時書き込み競合が頻発

### 9-2. workflow engine（Airflow / Prefect）導入の判断

- [ ] 連携ジョブ数 > 20
- [ ] 依存関係が複雑化（DAGが必要）
- [ ] systemd timer管理が辛い

### 9-3. Redis/RabbitMQ導入の判断

- [ ] 注文Webhook受信数 > 秒間10件
- [ ] 並列処理がボトルネック
- [ ] 失敗時のretry queueが必要

### 9-4. VPS強化 or 複数VPS化の判断

- [ ] CPU使用率 > 70% が連続3日
- [ ] メモリスワップ発生
- [ ] 1ジョブの実行時間が60分超過

### 9-5. 他プラットフォーム展開

eBayで月利100万円達成後：

- [ ] eBay UK / DE / AU 展開（Marketplace Insights同様の手法）
- [ ] Etsy展開（別パイプライン構築）
- [ ] Mercari Global展開
- [ ] Shopify独自店舗展開

---

## 第10章 Doc 1〜3を統合した最終チェックリスト

実装着手前の最終確認。**全項目チェックが入って初めて運用開始**。

### 設計理解

- [ ] Doc 1（設計原則編）を全章読了
- [ ] Doc 2（詳細設計編）を全章読了
- [ ] Doc 3（実装チェックリスト編・本書）を全章読了
- [ ] ケイタ式核心思想3つを自分の言葉で説明可能
- [ ] Anthropic公式5ドメインを自分の言葉で説明可能
- [ ] eBay規約遵守の絶対境界線を理解

### 設計検証

- [ ] テストケースA・B・Cすべてゼロ実装検証パス
- [ ] スキーマに問題発見なし
- [ ] Hooks判定順序問題発見なし
- [ ] MCP境界に問題発見なし

### API取得

- [ ] eBay Developer Program 取得
- [ ] Marketplace Insights API 申請（または取得）
- [ ] Buyee API 取得
- [ ] 楽天市場API 取得
- [ ] Yahoo!ショッピングAPI 取得
- [ ] DeepL API 取得
- [ ] Notion API 取得
- [ ] Discord Bot Token + Webhook 取得
- [ ] OpenExchangeRates API 取得
- [ ] Apify or Oxylabs 登録（暫定スクレイピング用）

### 規約遵守

- [ ] eBay User Agreement 最新版読了
- [ ] メルカリ利用規約 最新版読了
- [ ] 楽天・Yahoo・各API規約 確認済み
- [ ] 古物商免許 確認
- [ ] 税務体制 確認

### 環境構築

- [ ] VPS契約・基本設定完了
- [ ] Python・Node.js・Claude Code インストール
- [ ] リポジトリ配置完了
- [ ] secrets.env 配置完了（権限600）
- [ ] SQLite DB 初期化
- [ ] MCP server 全起動確認
- [ ] Hooks 全動作確認
- [ ] systemd timer 全有効化

### 通知体制

- [ ] Discord 3チャンネル設置
- [ ] 外注さん Discord参加
- [ ] 外注さん Notion閲覧権限付与
- [ ] テスト通知 全チャンネル成功

### 段階的デプロイ準備

- [ ] Phase Dev用テストセラー作成
- [ ] config.yaml の dry_run: true 設定
- [ ] 1日の出品上限 max_listings_per_day: 3 設定

### 監視準備

- [ ] daily-summary 自動投稿スクリプト動作確認
- [ ] 構造化ログ /var/log/keita/structured/errors.jsonl 書き込み確認
- [ ] エスカレーション通知 テスト実行

---

## 第11章 改訂履歴

| バージョン | 日付 | 変更内容 |
|---|---|---|
| v2.0 (MAX) | 2026年5月3日 | Anthropic公式準拠の中級者向け実装チェックリスト編として新規作成 |

---

## 第12章 出典・参照

- Anthropic公式ドキュメント（platform.claude.com / docs.claude.com）
- Anthropic Cookbook（github.com/anthropics/anthropic-cookbook）
- Claude Certified Architect認定（CCA-F）試験対策資料
- ケイタ社長の書籍・動画教材（受講生は別途参照）
- eBay Developer Documentation（developer.ebay.com）
- eBay User Agreement（2026年2月20日改訂版）
- メルカリ利用規約（2025年10月改訂版）
- Buyee API Documentation
- 楽天市場API（webservice.rakuten.co.jp）
- Yahoo!ショッピングAPI（developer.yahoo.co.jp）
- DeepL API（developers.deepl.com）
- Notion API（developers.notion.com）
- Discord Developer Portal（discord.com/developers）

---

## 結語

3部作（Doc 1：設計原則編、Doc 2：詳細設計編、Doc 3：実装チェックリスト編）が完成した。

ケイタ式の核心思想を、Anthropic公式エージェント設計5ドメインで体系化し、VPS上で24時間365日自動稼働するシステムとして実装可能な仕様まで落とし込んだ。

特に重要なのは以下3点：

1. **規約遵守の絶対境界線**：人間の確認なしのend-to-endフローをシステムレベルで物理的に防いでいる（SA-5に購入実行ツールを与えない）
2. **ケイタ式の核心原則をHookで決定論的に強制**：利益率15%・評価フェーズ整合・VeRO blacklistはプロンプトでの依頼ではなくHookでblockする
3. **ゼロ実装検証の重視**：Step 6を飛ばさず、3つのテストケースを手書きで通してから実装着手する

これらを守れば、月利30万円・100万円・1000万円のスケールに、運営者の作業時間をほぼ増やさずに到達できる設計となっている。

各受講生は、自身のフェーズ（評価値・出品数・現状売上）に応じてconfig.yaml・phase_strategy.yamlを調整し、本書の段階的デプロイ手順に従って実装すること。

健闘を祈る。

---

**Doc 3（実装チェックリスト編）終了。3部作完成。**