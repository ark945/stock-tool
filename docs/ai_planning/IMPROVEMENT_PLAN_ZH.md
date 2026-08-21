# 美股/台股分析網站 — 完整改進計劃（繁體中文版）

> 原作者：Opus 4.7 深度分析  
> 執行者：Sonnet（開始前請仔細閱讀本計劃）  
> 建立日期：2026-05-15  
> 目標：將網站從「被動技術分析檢視器」轉型為「具備 Alpah 預測能力的預測系統」

---

## 背景

使用者（資深交易者）基於基本面信念在財報發布前買入 ONDS。股票因財報超預期的 20% 漲幅大跳空。當時網站：
- 未在任何排行榜中列出 ONDS
- 財報前顯示「不宜買入」
- 在股價已大漲 20% 後才顯示「買入」（後見之明偏誤）

**根本原因**：網站結構上是一個「追蹤現有動量」的工具，而非「在動量發生前發掘機會」的工具。為了打敗市場，我們必須加入**催化劑發掘（Catalyst Discovery）、回測可信度（Backtest Credibility）與預測訊號疊加（Predictive Signal Stacking）**。

---

## 架構總覽

| 層級 | 目前狀態 | 計劃完成後 |
|-------|--------------|------------|
| 股票池 (Universe) | ~150 檔硬編碼股票 | 羅素 2000 + 動態發掘池 (~2500 檔) |
| 訊號 (Signals) | 純被動技術分析 (TA) | TA + 催化劑 + 情緒分析 + 內部人交易 |
| 驗證 (Validation) | 無 | 每種訊號具備滾動 252 天回測數據 |
| AI 層 | 綜合現有技術分析 | 多來源敘事 + 催化劑整合 |
| 使用者體驗 (UX) | 資訊埋藏過深 | 以催化劑雷達 (Catalyst Radar) 作為首頁 |

---

## 階段 1 — 基礎工程 (必須優先完成，阻擋後續所有階段)

### 1.1 擴充掃描股票池

**檔案：** `scripts/scan.js`

**現狀：** 硬編碼 `SP500` (~500) + `GROWTH_EXTENDED` (~80) + `DISCOVERY_POOL` (~150) 及 3 個篩選器呼叫。

**變更：** 透過新增 Yahoo 篩選器實現動態股票池擴充：
```javascript
const ADDITIONAL_SCREENERS = [
  'most_actives',           // 前 50 大熱門成交
  'undervalued_growth',     // 低估成長股組合
  'aggressive_small_caps',  // 積極型小型成長股
  'undervalued_large_caps', // 低估大型股
  'high_returns_value',     // 高報酬價值股
  '52_week_high_breakouts', // 關鍵 — 捕捉突破 52 週新高的股票
];
```

此外，新增**持久化「發掘記憶庫 (Discovery Memory)」**——過去 30 天內出現在任何篩選器中的標的皆保留在股票池中。儲存於 `data/universe_memory.json`：
```json
{
  "tickers": {
    "ONDS": { "firstSeen": "2026-04-15", "lastSeen": "2026-05-14", "sources": ["small_cap_gainers"] }
  }
}
```

每次掃描時：載入記憶庫，加入當前篩選器的新標的，剔除超過 30 天未出現的標的。目標股票池規模：**1500–2500 檔不重複標的**。

**驗收標準：** 掃描啟動時記錄的股票池規模應 ≥ 1500 檔。

---

### 1.2 使用 Yahoo `assetProfile` 修復板塊對應

**檔案：** `scripts/scan.js`

**現狀：** 靜態 `SECTOR_MAP` 約 120 檔股票 → 脆弱且易錯（例如：加密貨幣礦業者被歸類至 XLF 金融版圖）。

**變更：** 透過 Yahoo `quoteSummary` 模組建立板塊快取：
```javascript
async function getSectorInfo(symbol) {
  const url = `https://query2.finance.yahoo.com/v10/finance/quoteSummary/${symbol}?modules=assetProfile,summaryProfile`;
  const data = await yfFetch(url);
  const profile = data?.quoteSummary?.result?.[0]?.assetProfile;
  return {
    sector:   profile?.sector,
    industry: profile?.industry,
  };
}

// 映射 Yahoo 板塊 → ETF
const SECTOR_TO_ETF = {
  'Technology':             'XLK',
  'Communication Services': 'XLC',
  'Consumer Cyclical':      'XLY',
  'Consumer Defensive':     'XLP',
  'Financial Services':     'XLF',
  'Healthcare':             'XLV',
  'Industrials':            'XLI',
  'Energy':                 'XLE',
  'Basic Materials':        'XLB',
  'Real Estate':            'XLRE',
  'Utilities':              'XLU',
};

const INDUSTRY_TO_ETF = {  // 更高精準度
  'Semiconductors':           'SMH',
  'Software—Application':     'IGV',
  'Software—Infrastructure':  'IGV',
  'Biotechnology':            'IBB',
  'Aerospace & Defense':      'XAR',
  'Gold':                     'GDX',
  'Silver':                   'GDX',
  'Solar':                    'TAN',
};
```

將板塊查詢結果快取於 `data/sector_cache.json`（設定 1 年 TTL — 因極少變動）。掃描時，僅對無快取紀錄的標的發出請求。

**驗收標準：** 領頭股 (Leaders) 的 `sectorRS` 非空值率應 ≥ 90%（目前僅約 24%）。

---

### 1.3 掃描輸出增加 `opens` 與更豐富的 OHLC 數據

**檔案：** `scripts/scan.js`

`getOHLCV` 應同步返回 `opens`（開盤價，Yahoo 回應中已包含）。可用於後續跳空分析。

---

### 1.4 掃描加入財報日期與空單比率 (Short Interest)

**檔案：** `scripts/scan.js`

對通過領頭股/發掘篩選的候選標的，調用 `quoteSummary` 模組：
```javascript
modules: 'calendarEvents,defaultKeyStatistics,upgradeDowngradeHistory,recommendationTrend,earningsHistory'
```

新增至輸出紀錄：
```javascript
{
  ...existing,
  earningsDate:     unix timestamp or null,
  daysToEarnings:   number or null,
  shortPctOfFloat:  number or null,         // 空單佔流通股比率
  shortRatio:       number or null,         // 借券補回天數 (Days to cover)
  recentUpgrades:   [{ firm, action, date }, ...] 最多 3 筆,
  surpriseHistory:  [{ qtr, actual, estimate, surprisePct }, ...] 過去 4 季,
}
```

**效能說明：** 每檔領頭股/發掘標的額外增加約 2 次 API 呼叫。保存在 `data/quote_cache.json` 中（24小時 TTL）。

**驗收標準：** 領頭股的 `daysToEarnings` 填補率 ≥ 80%。

---

## 階段 2 — 催化劑層 (核心主打功能)

### 2.1 內部人買入偵測 (SEC EDGAR)

**新檔案：** `scripts/insider_scan.js`

SEC EDGAR 提供機器可讀格式的 Form 4（內部人交易）數據：
```
https://efts.sec.gov/LATEST/search-index?q=%22Form%204%22&forms=4&dateRange=custom&startdt=2026-04-15&enddt=2026-05-15&ciks=<CIK>
```

**作法：** 對於掃描池中的每檔股票，查詢其 CIK（從 SEC `company_tickers.json` 一次性獲取），並檢查過去 30 天內的 Form 4 申報。

**叢集買入 (Cluster Buying)** 定義 = 過去 30 天內 ≥ 2 位內部人買入，或單一內部人買入金額 > $100K。

輸出格式：
```json
{
  "TICKER": {
    "filings": [
      { "insider": "John Doe", "title": "CFO", "date": "2026-05-10", "shares": 5000, "price": 12.50, "value": 62500 }
    ],
    "totalValue30d": 250000,
    "buyerCount30d": 3,
    "clusterBuy": true
  }
}
```

儲存於 `data/insider_data.json`，每日透過新的 GitHub Action `insider-scan.yml` 重新整理一次。

---

### 2.2 個股新聞情緒分析

**新檔案：** `scripts/news_scan.js`

使用無需驗證的 Yahoo Finance 新聞 API：
```
https://query2.finance.yahoo.com/v1/finance/search?q=<TICKER>&newsCount=10
```

對每檔領頭股/發掘標的獲取最新新聞，將標題傳送至 Claude Haiku 進行情緒評分：
```javascript
prompt = `將這些新聞標題評級為：bullish, bearish, mixed, neutral 之一。
股票代號：${symbol}
新聞標題（過去 7 天）：
1. ${h1}
2. ${h2}
...

返回 JSON: {"sentiment": "bullish|bearish|mixed|neutral", "summary": "<25字內摘要>", "keyHeadline": "..."}`;
```

輸出至 `data/news_sentiment.json`。在同一個工作流中於主掃描完成後觸發。

---

### 2.3 分析師評等追蹤器

**現有來源：** 已可透過 Yahoo `quoteSummary` 獲取，僅需在掃描紀錄中展現最新變更（參考 1.4）。顯示規則：
- 過去 30 天內 ≥ 2 次升評 = 看多催化劑
- 過去 30 天內 ≥ 2 次降評 = 看空警示標記

---

### 2.4 三重共振雷達 (Triple-Resonance Radar，首頁亮點功能)

**檔案：** `index.html`

新增顯眼儀表板卡片（置於首頁頂部，高於現有 RS Leader 卡片）：

```
🎯 三重共振雷達 — 財報 + VCP + 內部人買入

下列股票同時符合三大領先指標：
✓ 未來 14 天內發布財報
✓ VCP 得分 ≥ 2  
✓ 過去 30 天內部人叢集買入 OR 分析師升評
✓ RS Rating ≥ 80

[STOCK]  財報倒數 7 天 · VCP 4/4 · 內部人買入 $850K · ★★★★★
[STOCK]  財報倒數 12 天 · VCP 3/4 · 2 位分析師升評 · ★★★★

這些股票具備「事前可見的爆發條件組合」。歷史回測：勝率 67%、平均報酬 +14.2%、平均持有 18 天。
```

三重共振候選者在掃描時計算並儲存於 `data/triple_resonance.json`。

**驗收標準：** 當雷達顯示候選標的時，點擊直接跳轉至已預載資料的個股頁面。若當天無符合標的，顯示「今日無完全符合條件之標的 — 是否放寬篩選標準？」。

---

### 2.5 財報季強勢股佈局 (Earnings Season Power Play)

**檔案：** `index.html`，新增區塊

展示日曆視圖：「未來 14 天內發布財報的 RS Leader」——篩選出即將發布財報的強勢標的。排序依據：（VCP 得分）+（財報倒數天數升冪）+（RS 排名）。

對每檔標的顯示過去 4 季的財報驚喜歷史（Earnings Surprise History）——持續超越預期的股票傾向於繼續超越預期。

---

## 階段 3 — 回測基礎設施 (建立可信度)

### 3.1 訊號績效追蹤器

**新檔案：** `scripts/backtest.js`

**步驟 1：** 維護 `data/signal_history.json` 紀錄每筆訊號發出時的進場價、停損價、目標價與開關狀態（open / hit_target / hit_stop / timed_out）。

**步驟 2：** 每次掃描執行時，更新並檢查未平倉訊號是否觸及目標價或停損價，超過 60 天則平倉。

**步驟 3：** 計算各訊號類型的滾動統計數據（過去 252 天勝率、平均盈虧比、Sharpe 比率等）。

**步驟 4：** 前端在掃描器介面中顯示訊號績效。

---

### 3.2 策略比較頁面

**`index.html` 新增分頁：** `策略表現`

直方圖展示各策略之勝率、平均報酬率、Sharpe 比率。使用者可明確比較各策略之歷史信賴度。

---

## 階段 4 — 型態品質升級

### 4.1 依據 Minervini 底型計數的 VCP

**檔案：** `scripts/scan.js`，重構 `calcVCP`

真正的 VCP 核心在於**底型結構 (Base Structure)**，而不僅是波幅收窄：
- 識別底型 (Base 1 至 Base 4+)
- Base 1（第一階段底型）：最高品質（得分 4）
- Base 2：優良（得分 3）
- Base 3：普通但風險較高（得分 2）
- Base 4+：末段底型（得分 1，警示紅燈）
- 若當前拉回小於先前拉回，加 1 分（標準收窄）
- 若當前底型成交量小於先前底型，加 1 分

---

### 4.2 樞紐買點偵測 (Pivot Point Detection)

**檔案：** `index.html`，`calcTA` 函式

標示出盤整區最高點作為關鍵樞紐買點（Pivot），並標註成交量需放大 1.5 倍以上作為確認條件。

---

### 4.3 GMMA (顧比複合移動平均線)

**檔案：** `index.html`，`calcTA`

計算 6 條短天期 EMA (3, 5, 8, 10, 12, 15) 與 6 條長天期 EMA (30, 35, 40, 45, 50, 60)，判斷強勢多頭、盤整或空頭趨勢。

---

### 4.4 成交量輪廓 (Volume Profile / Point of Control)

**檔案：** `index.html`，`calcTA`

計算過去 60 天成交量分佈，標示 POC (成交量最大集中區) 與 VAH/VAL (70% 成交量分佈區間)，展示機構資金支撐位置。

---

## 階段 5 — 投資組合智慧

### 5.1 依歷史紀錄自動計算凱利公式 (Auto-Kelly)

從使用者交易日誌 (`getJ()`) 的歷史勝率與盈虧比，自動帶入凱利公式算式，提供個人化倉位建議。

---

### 5.2 持倉相關性矩陣 (Holdings Correlation Matrix)

計算目前持倉之間的 60 天價格相關性，對相關係數 > 0.7 的持倉組合提出集中度風險警示。

---

### 5.3 板塊集中度警示

以圓餅圖展示持倉板塊分佈，若單一板塊佔投資組合比率 > 40% 則發出警示。

---

### 5.4 投資組合回撤追蹤 (Portfolio Drawdown Tracker)

針對交易日誌計算權益曲線、最大回撤 (Max Drawdown) 及復原因子 (Recovery Factor)。

---

### 5.5 相對弱勢「建議減碼」警示

比對持倉當前 RS 排名與進場時之差距，若 RS 排名顯著下滑或跌出前 100 名，提示減碼。

---

## 階段 6 — AI 增強 (實質預測層)

### 6.1 每日市場敘事 (Daily Market Narrative)

**新檔案：** `scripts/market_narrative.js`

掃描後將大盤、板塊 ETF、VIX、十年期美債殖利率與熱門新聞輸入 Claude，生成 100 字內之「今日市場敘事」展示於首頁。

---

### 6.2 AI 投資組合診斷

按鈕：「🤖 讓 Claude 檢視我的持倉」，評估持倉集中度風險、弱勢標的減碼建議與對沖策略。

---

### 6.3 結合催化劑之個股分析

增強 `scripts/ai_analysis.js`，將新聞情緒、內部人買賣與分析師評等變動一併輸入 AI Prompt，提供綜合多方來源的個股洞察。

---

## 階段 7 — UX 與資訊層級優化

1. **資訊優先級重構**：催化劑警示與 AI 預測前置於個股頁頂部。
2. **條件式警報系統**：支援價格突破、RSI、回撤與 VIX 警報。
3. **行動端響應式全面審計**：確保手機瀏覽體驗流暢無誤。
4. **教學至操作之連結**：策略教學內容直接附帶當前符合條件股票之捷徑。

---

## 階段 8 — 選配進階功能

1. **即時數據源評估**：評估引進 Polygon.io 或 Alpaca 即時行情。
2. **券商 API 整合**：整合 Alpaca 模擬交易 API 進行自動日誌記錄與訊號驗證。

---

## 實作順序與 Wave 分工

- **Wave 1 (基礎工程 - 優先)**: 1.1 股票池擴充, 1.2 板塊對應修復, 1.3 OHLC 增加 Opens, 1.4 財報與空單數據
- **Wave 2 (平行開發)**: 2.1 內部人掃描, 2.2 新聞情緒, 2.3 分析師評等, 2.4 三重共振雷達, 2.5 財報專區, 3.1 回測基礎設施
- **Wave 3 (型態品質)**: 4.1 Minervini VCP, 4.2 樞紐買點, 4.3 GMMA, 4.4 Volume Profile
- **Wave 4 (UX 與風控)**: 5.1 Auto-Kelly, 5.2 相關性矩陣, 5.3 板塊警示, 5.4 回撤追蹤, 5.5 弱勢警報, 7.1 UX 重構, 7.3 行動端審計, 7.4 教學連結
- **Wave 5 (AI 綜合整合)**: 6.1 市場敘事, 6.2 AI 組合診斷, 6.3 催化劑個股分析
- **Wave 6 (回測展示)**: 3.2 策略比較頁面
- **Wave 7 (警報系統)**: 7.2 條件式警報

---

## 關鍵限制與風險

- **Yahoo API 頻率限制**：必須建立嚴格快取機制（`quote_cache.json` 24h TTL）。
- **SEC EDGAR 限制**：遵守最高 10 req/sec 及指定 User-Agent。
- **Claude API 成本**：預估每月 API 費用不超過 $5 美元。
- **GitHub Actions 配額**：預估每月消耗 ~1800 分鐘，在免費額度（2000分鐘/月）以內。

---

## 執行細則

1. 每次修改後進行語法檢核與資料輸出測試 (`node scripts/scan.js us`)。
2. 保持小步快跑提交 Git commit。
3. 始終維護現有功能之穩定性。
