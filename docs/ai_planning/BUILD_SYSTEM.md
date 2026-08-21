# stock-tool 建置與自動化系統文件 (Build System Documentation)

---

## 1. 系統概覽 (System Overview)

`stock-tool` 採用混合式架構：
1. **前端**：雙軌制
   - **傳統單頁模式 (Legacy SPA)**：基於 `index.html` / `smart-stock-source.html` 的無構建純 HTML+CSS+JS 應用。
   - **現代化構建沙盒 (Vite Migration)**：採用 ES Modules 與 Vite 構建工具，逐步拆分單檔結構。
2. **數據處理管線 (Data Pipeline)**：使用 Node.js (v18+) 與 Python 3 自動抓取數據並輸出快取 JSON。
3. **CI/CD 自動化**：利用 GitHub Actions 自動執行定時掃描與 GitHub Pages 自動部署。

---

## 2. 環境要求 (Prerequisites)

- **Node.js**：v18.0.0 或更高版本
- **npm**：v9.0.0 或更高版本
- **Python**：3.10 或更高版本
- **Ruby**（可選）：運行 `serve.rb` 本地伺服器（需 WEBrick gem）
- **Git**：用於版本控制與 GitHub Actions 同步

---

## 3. 開發環境建置 (Development Setup)

### A. 運行傳統單頁應用 (Legacy Single Page)

您可以選擇以下任一方式啟動本地開發伺服器：

#### 方式 1：使用專案內建 Ruby 伺服器
```bash
ruby serve.rb
# 預設啟動於 http://localhost:8000
```

#### 方式 2：使用 Python 靜態伺服器
```bash
python -m http.server 8000
# 瀏覽 http://localhost:8000
```

#### 方式 3：使用 VSCode Live Server 擴充套件
直接在 VSCode 中右鍵點擊 `index.html` 並選擇 "Open with Live Server"。

---

### B. 運行 Vite 現代化拆分沙盒 (Vite Sandbox)

位於 `vite-migration/` 目錄：

```bash
# 1. 切換至 Vite 沙盒目錄
cd vite-migration

# 2. 安裝前端依賴套件
npm install

# 3. 啟動 Vite 開發伺服器
npm run dev
# 啟動於 http://localhost:5173
```

---

## 4. 數據管線與自動化腳本 (Data Pipeline Execution)

專案包含多個批次處理與掃描腳本，用於更新 `data/` 目錄下的 JSON 快取。

### A. 安裝數據管線依賴

#### Node.js 腳本依賴：
```bash
cd scripts
npm install
```

#### Python 腳本依賴：
```bash
pip install requests pandas numpy beautifulsoup4 html5lib
```

---

### B. 執行數據掃描與更新腳本

在專案根目錄下運行以下指令：

#### 1. 美股/台股股票動量掃描器 (RS Leader Scanner)
```bash
# 掃描美股
node scripts/scan.js us

# 掃描台股
node scripts/scan.js tw

# 掃描全部市場 (npm script)
cd scripts && npm run scan:all
```

#### 2. 台股三大法人買賣超數據抓取
```bash
python scripts/fetch_twse_institutions.py
```

#### 3. 美股 SEC EDGAR 內部人買賣申報掃描
```bash
node scripts/insider_scan.js
```

#### 4. 新聞與市場情緒 AI 分析
```bash
# 抓取新聞資料
python scripts/fetch_news.py

# FinBERT 新聞情緒評分
python scripts/score_news_finbert.py

# Claude 每日市場敘事生成
node scripts/market_narrative.js
```

#### 5. 訊號與策略歷史回測
```bash
# 策略回測
node scripts/backtest.js

# 歷史數據深層回測
python scripts/backtest_historical.py
```

---

## 5. 生產環境構建 (Production Build)

### A. 構建 Vite 專案產物
若要編譯並打包 `vite-migration/` 專案：

```bash
cd vite-migration

# 執行生產環境構建
npm run build

# 預覽構建結果
npm run preview
```
構建產物將輸出至 `vite-migration/dist/` 目錄。

---

## 6. CI/CD 自動化部署流程 (GitHub Actions)

專案在 `.github/workflows/` 下配置了 12 個自動化工作流：

| 工作流檔案 | 觸發時機 / 頻率 | 功能說明 |
|---|---|---|
| `stock-scan.yml` | 每日美股/台股開盤前/後 (Cron) | 執行 `scripts/scan.js` 更新 `us_scan.json` 與 `tw_scan.json` |
| `insider-scan.yml` | 每日美東時間傍晚 | 執行 `insider_scan.js` 向 SEC EDGAR 抓取 Form 4 並生成 `insider_data.json` |
| `news-scan.yml` | 每 4 小時一次 | 抓取最新新聞並透過 FinBERT / Claude 進行情緒評估 |
| `pages.yml` | Push 至 main 檔案變動時 | 自動發布 `index.html` 與 `data/` 至 GitHub Pages |
| `backtest.yml` | 週次定時 | 執行自動化策略回測與統計生成 `signal_stats.json` |
| `smoke-test.yml` | 每次 Pull Request / Commit | 執行語法與腳本冒煙測試 (`smoke_test.py`) |

---

## 7. 環境變數與 Secret 設定

若需執行包含 AI 分析與外部 API 的完整數據管線，需在本地或 GitHub Repository Secrets 中設定以下環境變數：

```bash
# Claude API Key (用於市場敘事與個股 AI 診斷)
ANTHROPIC_API_KEY="sk-ant-..."

# SEC EDGAR 請求標頭 (遵守 SEC 存取規範)
SEC_USER_AGENT="YourName admin@yourdomain.com"
```
