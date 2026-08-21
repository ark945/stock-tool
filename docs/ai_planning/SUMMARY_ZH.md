# stock-tool 專案簡單摘要文件 (Summary)

---

## 1. 專案概覽 (Project Overview)

`stock-tool` 是一個針對**美股與台股**交易者打造的綜合型股票分析與預測平台。系統整合了自動化數據管線 (Data Pipeline)、技術分析引擎、AI 新聞情緒與敘事生成、短線掃描器，以及個人交易日誌與風控管理功能。

---

## 2. 專案核心架構 (Core Architecture)

| 模組區塊 | 主要檔案/目錄 | 說明與功能 |
|---|---|---|
| **前端應用 (Frontend)** | `index.html`<br>`smart-stock-source.html` | 單頁式 Web 應用程式 (SPA)，包含 K 線圖 (Lightweight Charts)、動量掃描器、投資組合風控與交易日誌。 |
| **數據管線 (Scripts)** | `scripts/scan.js`<br>`scripts/fetch_*.py`<br>`scripts/ai_analysis.js` | 透過 Node.js 與 Python 腳本抓取 Yahoo Finance、SEC EDGAR、TWSE 三大法人及新聞資料，進行特徵計算與 AI 處理。 |
| **資料快取 (Data)** | `data/*.json` | 由 GitHub Actions 每日/定期自動更新之 JSON 資料集，提供前端即時讀取與呈現。 |
| **現代化拆分 (Sandbox)** | `vite-migration/` | 為了解決單一 HTML 檔案過龐大問題，進行模組化 (ES Modules) 與 Vite 構建的轉型測試區域。 |
| **實戰教學 (Education)** | `trade-guide/` | 國際貿易與實戰談判教學導覽 Markdown 文件集。 |

---

## 3. 三大核心規劃文件摘要 (Key Planning Documents)

### 📌 A. 系統升級計劃 (`IMPROVEMENT_PLAN_ZH.md`)
* **核心目標**：將系統從「被動追蹤漲幅的技術分析工具」轉型為「事前捕捉暴漲條件的 α 預測系統」。
* **重點升級項目**：
  1. **股票池擴充**：從 150 檔擴大至 1500–2500 檔（引入 52 週新高突破、動態記憶庫）。
  2. **催化劑層 (Catalysts)**：加入 SEC EDGAR 內部人買入 (Form 4) 偵測、新聞情緒 AI 評分、分析師升降評追蹤。
  3. **三重共振雷達 (Triple-Resonance Radar)**：同時符合「財報前 14 天 + VCP 型態 + 內部人叢集買入」的黃金組合發掘。
  4. **回測可信度**：每種技術訊號均附帶過去 252 天滾動勝率、平均報酬率與 Sharpe 比率。
  5. **投資組合智慧**：自動計算凱利公式 (Auto-Kelly)、持倉相關性矩陣 (防止過度集中)、最大回撤追蹤。

### 📌 B. 內容變現規劃 (`MONETIZATION_PLAN.md`)
* **核心目標**：將 repo 現有技術資產轉化為具備現金流與訂閱愛好的產品。
* **策略路線**：
  * **主線 A**：台股/美股交易日誌模板 (Notion / Google Sheets) 探針驗證 → 進階推出「台股交易日誌 SaaS」（主打結合三大法人數據與中文介面，解決 TradeZella 等競品無台股數據之痛點）。
  * **副線 B**：利用現有 Python 數據管線，自動生成「台股籌碼週報」（於方格子/Substack 免費發行，作為主線產品之獲客漏斗）。

### 📌 C. 短線功能深化指令 (`short-term-enhancement-instructions.md`)
* **核心目標**：在前端直接擴充 7 大短線實戰模組。
* **功能包含**：短線掃描器（動量突破、超賣反彈、放量異動）、風控中心（R:R 計算、止損防護）、績效分析儀表板及短線實戰教學。

---

## 4. 關鍵執行路線與下一步 (Execution Roadmap)

1. **第一波（基礎奠基）**：完成 `scan.js` 股票池擴充、Yahoo 板塊精準映射與財報日期/空單比率收集。
2. **第二波（催化劑與回測）**：建立 `insider_scan.js` 與 `news_scan.js`，上架首頁「三重共振雷達」。
3. **第三波（架構優化）**：配合 `vite-migration` 將大檔解構為獨立模組，提升開發效率與可維護性。
