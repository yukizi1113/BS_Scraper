# BS Scraper (`bs_scraper.py`)

日本上場企業の **BS（貸借対照表）指標** を EDINET（優先）/ TDNet（補完）から自動取得し、Excel テンプレートを一括更新するスクレイパー。

---

## 概要

| 項目 | 内容 |
|------|------|
| データソース | EDINET（最優先）→ TDNet（補完） |
| 値の単位 | 百万円（Excelテンプレート準拠） |
| 対応会計基準 | J-GAAP (`jppfs_cor`) / IFRS (`jpigp_cor`, `ifrs-full`) / US-GAAP (`us-gaap`) |
| 上書きポリシー | **空欄のみ**書き込み（既存値は保護） |
| WARN 判定 | 既存値と取得値の差が `warn_tolerance`（デフォルト ±2 百万円）超で橙色マーク |
| 対応銘柄コード | 4 桁数字（例: 9760）および英数混合（例: 142A）の新形式 |
| Current Version | v26_323 |

---

## 取得 BS フィールド

| 内部キー | Excel ヘッダ名 | 内容 |
|----------|---------------|------|
| `assets_total` | 資産合計 | 総資産 |
| `current_assets` | 流動資産 | 流動資産合計 |
| `cash_eq_st_invest` | 現金同等物及び短期性有価証券 | 現金・預金 + 短期性有価証券（コール・ローン等含む） |
| `current_liabilities` | 流動負債 | 流動負債合計 |
| `short_term_borrowings` | 短期借入債務 | 短期借入金 + 1年内返済長期借入金 + 短期社債 + CP + リース債務(CL) |
| `long_term_borrowings` | 長期借入債務 | 長期借入金 + 長期社債 + リース債務(NCL) |
| `shares_outstanding` | 期末発行済株式数 - 普通株 | 普通株式の発行済株式総数（千株単位） |
| `income_taxes_payable` | 未払法人税 | 未払法人税等 |

各フィールドは **prior（前期末）** と **current（当期末）** の 2 時点を取得します。また、有利子負債については **スケジュール補完値**（`_sched_`）も内部的に管理されます。

---

## ソース優先度とスコアリング

```
EDINET (優先度 2) > TDNet (優先度 1)
```

**書類種別スコア（同一ソース内）**:

| 書類種別 | スコア | 内容 |
|---------|--------|------|
| ASR | 3 | 有価証券報告書（Annual Securities Report）|
| QSR | 2 | 四半期報告書（Quarterly Securities Report）|
| SSR | 1 | 半期報告書（Semi-annual Securities Report）|
| OTHER | 0 | その他 |

同一ソース・同一書類種別内では `current`（当期末） > `prior`（前期末）、さらに **完全性スコア**（コアフィールドが埋まっている数）と **提出日の新しさ** で優先度を決定します。

---

## セル着色ルール

| 色 | 意味 |
|----|------|
| **黄色**（`#FFF2CC`） | 今回スクリプトが新規入力した値 |
| **橙色**（`#F4B183`） | 既存値と取得値の差が `warn_tolerance` 超 → WARN |

- `EE` 列（懸念 check 列）: **直近 5 四半期**に 1 件でも橙 WARN があれば `1` を書き込みます。

---

## ワークブック構成（想定）

```
データ取得_BS.xlsx
```

- ヘッダ行に「決算期」「資産合計」「流動資産」... の列ラベルが存在すること。
- スクリプトはヘッダ行を自動検索して列インデックスを構築します（ハードコードなし）。
- `EE` 列は「懸念check」列として自動検出されます。

---

## インストール

```bash
pip install requests beautifulsoup4 lxml openpyxl tqdm
```

Python 3.9 以上を推奨。

---

## 使い方

### 基本実行

```bash
python bs_scraper.py \
  --input  データ取得_BS.xlsx \
  --output データ取得_BS_out.xlsx \
  --edinet-api-key <YOUR_EDINET_API_KEY>
```

### 特定 ticker のみ処理（テスト用）

```bash
python bs_scraper.py \
  --input  データ取得_BS.xlsx \
  --output データ取得_BS_out.xlsx \
  --edinet-api-key <YOUR_EDINET_API_KEY> \
  --only-tickers "4506,6178,9432"
```

### 処理行数を制限（スモークテスト）

```bash
python bs_scraper.py \
  --input  データ取得_BS.xlsx \
  --output データ取得_BS_out.xlsx \
  --edinet-api-key <YOUR_EDINET_API_KEY> \
  --max-rows 50
```

### preflight のみ（ネットワークアクセス確認）

```bash
python bs_scraper.py \
  --input  データ取得_BS.xlsx \
  --output データ取得_BS_out.xlsx \
  --edinet-api-key <YOUR_EDINET_API_KEY> \
  --preflight-only
```

### オフライン回帰テスト（ローカル ZIP を使用）

```bash
python bs_scraper.py \
  --input  データ取得_BS.xlsx \
  --output データ取得_BS_out.xlsx \
  --regression-suite path/to/regression_zips/ \
  --only-tickers "4506,3197"
```

---

## 全 CLI オプション

| オプション | デフォルト | 説明 |
|-----------|-----------|------|
| `--input` | `データ取得_BS.xlsx` | 入力 xlsx |
| `--output` | （自動命名） | 出力 xlsx |
| `--edinet-api-key` | 環境変数 `EDINET_API_KEY` | EDINET API キー |
| `--max-rows` | `500`（`0` で全行） | 処理行数上限 |
| `--only-tickers` | `` | カンマ/空白区切り ticker ホワイトリスト（例: `1301,130A`） |
| `--only-tickers-file` | `` | ticker リストを 1 行 1 ticker で記載したファイル |
| `--days-back-edinet` | `260` | EDINET の遡及日数 |
| `--days-back-tdnet` | `35` | TDNet lookback window |
| `--sleep-edinet` | `0.8` | EDINET リクエスト間隔（秒） |
| `--sleep-tdnet` | `0.8` | TDNet リクエスト間隔（秒） |
| `--progress-mode` | `auto` | 進捗表示モード（`auto` / `plain` / `off`）。Colab や subprocess 実行で見えない場合は `plain` |
| `--preflight-only` | off | preflight チェックのみ実行して終了 |
| `--preflight-skip-network` | off | preflight でネットワーク疎通チェックをスキップ |
| `--regression-suite` | `` | オフライン回帰スイートのパス（dir or zip）。指定時は EDINET/TDNet にアクセスしない |
| `--offline-subprocess` | `auto` | オフラインモードでの subprocess 使用（`auto` / `on` / `off`） |
| `--warn-log` | `` | WARN メッセージを書き込むログファイルパス |
| `--verbose-warnings` | off | WARN の詳細ログを標準エラーに出力 |
| `--suspicious-abs-tol` | `2` | WARN 判定しきい値（百万円） |
| `--suspicious-small-scale` | `2000` | 小規模企業判定しきい値（百万円） |
| `--suspicious-small-abs-tol` | `2` | 小規模企業向け WARN しきい値（百万円） |

---

## アーキテクチャ

```
bs_scraper.py
├── preflight()              ← Excel 読み込み・ticker 一覧取得・API 疎通確認
├── scrape_edinet_bs()       ← EDINET API → XBRL/iXBRL ZIP 解析
├── scrape_tdnet_bs()        ← TDNet LIVE → HTML/XBRL ZIP 解析
├── process_one_zip()        ← ZIP 内の XBRL / iXBRL を解析してフィールド抽出
│   ├── extract_debt_components_xbrl()   ← 有利子負債（ST/LT/リース/社債/CP）
│   ├── extract_cash_components_xbrl()   ← 現金同等物・短期有価証券
│   └── extract_other_financial_liabilities_leases()  ← IFRS 注記補完
├── build_best_store()       ← EDINET + TDNet のレコードを優先度でマージ
└── fill_excel()             ← Excel テンプレートへ書き込み（黄色/橙色着色）
```

### XBRL 解析の概要

1. **概念名（qname）優先**: `jppfs_cor:Assets` 等の完全修飾名で値を取得。
2. **ローカル名 fallback**: プレフィックスが異なる場合はローカル名で照合。
3. **コンテキスト選択**: `FilingDateInstant`、`CurrentYearInstant`、`Prior1YearInstant` 等から `prior` / `current` を判定。
4. **有利子負債の合算**:
   - 短期: `loans_cl + bonds_cl + cp_cl + call_money_cl + lease_cl`
   - 長期: `loans_ncl + bonds_ncl + lease_ncl`
   - 合計が部品の合計と整合しない IBL トータルは使用しない。
5. **HTML 補完**: XBRL で取得できない有利子負債は注記 HTML から抽出（`HTML_ADD_MAX_RATIO` ガード付き）。

---

## 環境変数

`.env` ファイルに以下を設定することで CLI オプションを省略できます。

```dotenv
EDINET_API_KEY=your_edinet_api_key_here
```

---

## 主なバージョン履歴

| バージョン | 主な変更 |
|-----------|---------|
| v26_323 | Rebuild from the v26_194 baseline with narrow debt-only backports. Suppress weak TDNet OTHER prior/current debt comparisons, fix current false WARNs such as 4494/5644/9247/9504/9602, and verify all 3595 tickers in 12 chunks with only held cases (3077/3192/6191) remaining in debt WARNs. |
| v26_182 | Add line-oriented progress output for Colab/subprocess runs. Add `--progress-mode` (`auto` / `plain` / `off`) and keep `--only-tickers` subset execution visible at startup |
| v26_181 | Exclude bank call money from short_term_borrowings. Add TDNet GitHub mirror fallback for XBRL ZIPs from 2025-12-15 onward. Tighten bond short/long rebucket guards to fix 9502 without regressing 8388 |
| v26_165 | Include bank call money (`CallMoneyLiabilitiesBNK`) in short-term borrowings. Exclude sell-back and bond-lending collateral balances. |
| v26_150 | IFRS の `非流動` containing `流動` による誤分類を修正（リース CL/NCL パース改善、4506 対応） |
| v26_117 | リース注記抽出のガード強化（満期分析表・税効果表からの誤 add-back 防止）、オフライン doc_kind 推定の回帰修正 |
| v26_104 | 関係会社流動部分の誤バケット修正（9325）、長期関係会社ローンからの流動部分重複スキャン防止 |

---

## 注意事項

- **Excel テンプレートは上書きされません**。既存値の差が WARN 閾値以内なら変更なし。
- EDINET の遡及日数（`--days-back-edinet`）を大きくすると、より古いデータを取得できますが実行時間も増加します。
- TDNet public retention is short (about 35 days), but this version also backfills historical XBRL ZIPs from the GitHub mirror `yukizi1113/tdnet` for dates from 2025-12-15 onward.
- `--days-back-tdnet` still controls how far the scraper scans TDNet itself.
- Colab から `subprocess.run()` で起動する場合は `--progress-mode plain` を付けると、改行ベースの進捗表示が残ります。
- `--regression-suite` を使ったオフライン検証では、スイート内のローカル ZIP を使用するため EDINET/TDNet へのアクセスは発生しません。

---

## ライセンス

個人利用・社内利用目的。再配布・商用利用は別途確認してください。

