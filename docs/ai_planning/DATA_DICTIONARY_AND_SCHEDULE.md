# stock-tool 資料來源、抓取頻率與欄位規格說明文件 (Data Dictionary & Schedule)

---

## 1. 專案數據架構總覽 (Overview)

`stock-tool` 採用分散式數據管線，所有抓取到的資料會經由後端腳本清洗、計算、整合後，統一輸出為結構化的 **JSON 檔案** 存放於 [data/](file:///d:/MyLab/stock-tool/data) 目錄中。

自動化運作依靠 GitHub Actions 定時觸發，前端透過 AJAX (`fetch`) 讀取快取 JSON 檔進行渲染。

---

## 2. 資料來源與抓取頻率對照表 (Data Sources & Frequency Matrix)

| 數據種類 (Data Category) | 資料來源 (Source & Endpoint) | 執行腳本 (Script) | 產出檔案 (Target File) | 抓取頻率 (Schedule) | 觸發機制 (Trigger) |
|---|---|---|---|---|---|
| **美股/台股動量掃描** | Yahoo Finance API (`query2.finance.yahoo.com`) | `scripts/scan.js`<br>`scripts/enrich_quotes.py` | `data/us_scan.json`<br>`data/tw_scan.json` | 美股交易日每日 3 次<br>(09:17, 12:47, 15:38 ET) | GitHub Actions (`stock-scan.yml`) |
| **台股三大法人買賣超** | 臺灣證券交易所 (TWSE T86)<br>證券櫃檯買賣中心 (TPEx) | `scripts/fetch_twse_institutions.py` | `data/tw_institutions.json`<br>`data/tw_institutions_history.json` | 台股交易日每日收盤後<br>(15:30 CST) | GitHub Actions (`stock-scan.yml`) |
| **美股內部人交易 (Form 4)** | 美國證券交易委員會<br>(SEC EDGAR EFTS API) | `scripts/insider_scan.js` | `data/insider_data.json`<br>`data/cik_map.json` | 每日美東時間 01:00 AM<br>(06:00 UTC) | GitHub Actions (`insider-scan.yml`) |
| **新聞情緒打分** | Yahoo News API<br>FinBERT NLP 模型 | `scripts/fetch_news.py`<br>`scripts/score_news_finbert.py` | `data/us_news_sentiment.json`<br>`data/tw_news_sentiment.json` | 每次股票掃描完成後<br>(每日約 3 次) | GitHub Actions (`news-scan.yml`) |
| **即時新聞 Feed** | 財經新聞 RSS / Feed | `scripts/fetch_news_feed.py` | `data/news_feed.json` | 每 2 小時觸發一次 | GitHub Actions (`news-feed.yml`) |
| **今日市場敘事** | Anthropic Claude API<br>大盤 ETF & VIX & 殖利率 | `scripts/market_narrative.js` | `data/us_market_narrative.json`<br>`data/tw_market_narrative.json` | 每日美股/台股收盤後 | GitHub Actions (`stock-scan.yml`) |
| **全市場股票基本面** | Yahoo Finance `quoteSummary` | `scripts/fetch_fundamentals.py` | `data/fundamentals.json`<br>`data/quote_cache.json` | 每週日 00:00 UTC | GitHub Actions (`fundamentals-universe.yml`) |
| **歷史策略回測數據** | 歷史 OHLCV 價格數據<br>+ 掃描訊號觸發紀錄 | `scripts/backtest.js`<br>`scripts/backtest_historical.py` | `data/signal_history.json`<br>`data/signal_stats.json` | 每週日定時重新重算 | GitHub Actions (`backtest.yml`) |
| **政治人物交易與散戶情緒** | Capitol Trades API<br>StockTwits / Reddit | `scripts/fetch_politician_trades.py`<br>`scripts/fetch_retail_sentiment.py` | `data/politician_trades.json`<br>`data/earnings_whisper.json` | 每日一次 | GitHub Actions (`stock-scan.yml`) |

---

## 3. 詳細 JSON 資料欄位規格書 (Data Field Specifications)

### 📌 A. `us_scan.json` / `tw_scan.json` (股票動量掃描結果)

| 欄位名稱 (Field) | 資料型態 (Type) | 說明與備註 (Description) |
|---|---|---|
| `generatedAt` | String (ISO) | 掃描生成時間戳記 |
| `market` | String | 市場標識 (`us`美股 / `tw`台股) |
| `universeSize` | Number | 當次掃描涵蓋之總股票池檔數 (如 1850) |
| `leaders` | Array[Object] | 符合動量篩選之領頭股列表 |
| └ `symbol` | String | 股票代號 (例如 `NVDA` 或 `2330.TW`) |
| └ `name` | String | 公司名稱 |
| └ `price` | Number | 當前最新成交價 |
| └ `changePct` | Number | 當日漲跌幅 (%) |
| └ `rsRank` | Number | 相對強勢排名 (1-99，數值越大代表強於越多年同業) |
| └ `rsRating` | Number | RS 分級分數 |
| └ `sector` | String | 所屬產業板塊 (如 `Technology`, `Healthcare`) |
| └ `sectorETF` | String | 對應之板塊 ETF 代號 (如 `XLK`, `SMH`) |
| └ `vcpScore` | Number | VCP (波動收窄型態) 評分 (0 至 4 分) |
| └ `vcpBase` | String | 底型階段標示 (`Base 1`, `Base 2`, `Base 3`, `Base 4+`) |
| └ `pivotPrice` | Number | 樞紐關鍵突破買點價格 |
| └ `volumeRatio` | Number | 成交量放大倍數 (當前量 / 20日均量) |
| └ `shortPctOfFloat` | Number | 空單佔流通股比率 (%) |
| └ `daysToEarnings` | Number | 距離下一次財報發布天數 (天) |
| └ `earningsDate` | Number (Unix) | 財報發布 Unix Timestamp 時間戳記 |
| └ `recentUpgradesCount` | Number | 過去 30 天分析師調升評等次數 |
| └ `recentDowngradesCount`| Number | 過去 30 天分析師調降評等次數 |

---

### 📌 B. `tw_institutions.json` (台股三大法人買賣超數據)

| 欄位名稱 (Field) | 資料型態 (Type) | 說明與備註 (Description) |
|---|---|---|
| `generatedAt` | String (ISO) | 數據更新時間 |
| `tradeDate` | String | 交易日期 (`YYYY-MM-DD`) |
| `bySymbol` | Object | 以股票代號為 Key 的資料字典 |
| └ `[symbol].foreign` | Number | 外資及陸資買賣超股數 (正數為買超，負數為賣超) |
| └ `[symbol].foreignAmt` | Number | 外資買賣超金額估計 (NT$) |
| └ `[symbol].investment_trust` | Number | 投信買賣超股數 |
| └ `[symbol].dealer` | Number | 自營商買賣超股數 |
| └ `[symbol].total` | Number | 三大法人買賣超合計股數 |
| └ `[symbol].conviction` | String | 三大法人綜合評價 (`strong_buy` \| `buy` \| `neutral` \| `sell` \| `strong_sell`) |

---

### 📌 C. `insider_data.json` (美股內部人交易 Form 4 數據)

| 欄位名稱 (Field) | 資料型態 (Type) | 說明與備註 (Description) |
|---|---|---|
| `generatedAt` | String (ISO) | 數據更新時間 |
| `bySymbol` | Object | 以股票代號為 Key |
| └ `[symbol].filings` | Array[Object] | 過去 30 天內 Form 4 交易明細 |
| &nbsp;&nbsp;&nbsp;&nbsp;└ `insider` | String | 內部人姓名 |
| &nbsp;&nbsp;&nbsp;&nbsp;└ `title` | String | 內部人職務 (如 `CFO`, `CEO`, `Director`) |
| &nbsp;&nbsp;&nbsp;&nbsp;└ `date` | String | 申報交易日期 |
| &nbsp;&nbsp;&nbsp;&nbsp;└ `shares` | Number | 買賣股數 |
| &nbsp;&nbsp;&nbsp;&nbsp;└ `price` | Number | 交易單價 (USD) |
| &nbsp;&nbsp;&nbsp;&nbsp;└ `value` | Number | 交易總金額 (USD) |
| └ `[symbol].totalValue30d` | Number | 過去 30 天內部人淨買入總金額 |
| └ `[symbol].buyerCount30d` | Number | 過去 30 天內買入之內部人總人數 |
| └ `[symbol].clusterBuy` | Boolean | 是否觸發「叢集買入」條件 (≥2位內部人買入或總額>$100K) |

---

### 📌 D. `us_news_sentiment.json` / `tw_news_sentiment.json` (新聞情緒評分)

| 欄位名稱 (Field) | 資料型態 (Type) | 說明與備註 (Description) |
|---|---|---|
| `generatedAt` | String (ISO) | 更新時間 |
| `bySymbol` | Object | 以個股代號為 Key |
| └ `[symbol].sentiment` | String | 評級 (`bullish`看多 \| `bearish`看空 \| `mixed`分歧 \| `neutral`中性) |
| └ `[symbol].score` | Number | 數值化情緒得分 (-1.0 極度看空 至 +1.0 極度看多) |
| └ `[symbol].summary` | String | 25 字內 AI 綜合新聞摘要 |
| └ `[symbol].keyHeadline` | String | 當期最具影響力之新聞標題 |
| └ `[symbol].headlineCount` | Number | 納入分析的新聞總篇數 |

---

## 4. 股票池構建機制：掃所有股票還是依照主清單？ (Universe Building Mechanism)

本專案採用 **「混合式動態擴充 (Hybrid Dynamic Universe)」** 架構，美股與台股有不同的掃描策略：

```
                              ┌──────────────────────────────────────────────┐
                              │ 美股掃描池 (~1,800–2,500 檔動態組合)          │
                              ├──────────────────────────────────────────────┤
                              │ 1. 基礎主清單 (S&P 500 + Growth + Russell)    │
                              │ 2. Yahoo 7 大熱門篩選器 (實時熱門/飆股)        │
                              │ 3. 30 天發掘記憶庫 (近30天出現過即保留)      │
                              └──────────────────────────────────────────────┘

                              ┌──────────────────────────────────────────────┐
                              │ 台股掃描池 (~200–300 檔核心標的)              │
                              ├──────────────────────────────────────────────┤
                              │ 1. 核心權值股 + 熱門中小型股 (TW_POOL 清單)    │
                              └──────────────────────────────────────────────┘
```

### 🇺🇸 美股 (US Scan)：混合動態擴充模式 (Hybrid Dynamic)
美股並不是純粹只掃死板的固定清單，而是整合了 **「靜態主清單 + 實時動態篩選 + 30天記憶庫」**：

1. **靜態主清單 (Base Master Lists)**：
   - **S&P 500**：標普 500 指數成分股 (~500 檔)。
   - **GROWTH_EXTENDED**：延伸成長股清單 (~80 檔)。
   - **DISCOVERY_POOL**：潛力飆股種子池 (~150 檔)。
   - **RUSSELL_EXTENDED**：羅素 2000 小型股擴充清單 (~1500 檔)。
2. **實時動態篩選器 (Yahoo Finance Live Screeners)**：
   - 每次掃描時，會實時調用 Yahoo 7 大篩選器補進當日暴漲或熱門個股：
     - `day_gainers` (當日漲幅榜)
     - `small_cap_gainers` (小型暴漲股)
     - `growth_technology_stocks` (科技成長股)
     - `most_actives` (熱門成交榜)
     - `undervalued_growth_stocks` (低估成長股)
     - `aggressive_small_caps` (積極型小型股)
     - `undervalued_large_caps` (低估大型股)
3. **30 天發掘記憶庫 (Discovery Memory - `universe_memory.json`)**：
   - 任何個股只要曾經出現在上述熱門篩選器中（即便只出現一天），系統會在快取記憶庫中保留 **30 天**。
   - **優點**：確保能追蹤到「剛爆發起漲、隨後暫時脫離熱門榜但仍處於整理階段」的潛力飆股。

---

## 5. 如何擴充精選主清單？ (How to Expand the Master Ticker List)

若您想增加關注的股票（不論台股或美股），只需修改 `scripts/scan.js` 中的代碼陣列即可：

### 🛠️ 擴充步驟說明：

#### 步驟 1：修改 `scripts/scan.js` 檔案
開啟 [scripts/scan.js](file:///d:/MyLab/stock-tool/scripts/scan.js)：

* **新增台股標的**：
  找到 `const TW_POOL = [...]` 陣列（大約第 822 行），在陣列中加入新的台股代號：
  - **上市股票**：格式為 `代號.TW`（例如 `'3231.TW'` 緯創、`'2376.TW'` 技嘉）
  - **上櫃股票**：格式為 `代號.TWO`（例如 `'8069.TWO'` 元太、`'3529.TWO'` 力旺）
  ```javascript
  const TW_POOL = [
    '2330.TW', '2317.TW', '2454.TW',
    '3231.TW', // 👈 新增緯創
    '8069.TWO',// 👈 新增元太
    // ...
  ];
  ```

* **新增美股標的**：
  找到 `DISCOVERY_POOL` 或 `GROWTH_EXTENDED` 陣列（大約第 650～815 行），直接加入美股代號（例如 `'PLTR'`, `'ARM'`, `'SMCI'`）：
  ```javascript
  const DISCOVERY_POOL = [
    'PLTR', 'ARM', 'SMCI', // 👈 新增目標美股代號
    // ...
  ];
  ```

#### 步驟 2：同步更新 `data/universe_static_tw.json` (可選)
在 [data/universe_static_tw.json](file:///d:/MyLab/stock-tool/data/universe_static_tw.json) 的 `"tickers"` 陣列中同步填入新代號，確保靜態備份檔案一致。

#### 步驟 3：提交變更至 Git
在 Shell 終端機執行 Commit 與 Push：
```bash
git add scripts/scan.js data/universe_static_tw.json
git commit -m "feat: 擴充台股/美股掃描主清單"
git push origin main
```
在下一次 GitHub Actions 自動觸發（或您手動點擊 Run workflow）時，系統就會自動將新股票納入每日掃描、VCP 型態計算與 AI 情緒評分中！


