# stock-tool 資料抓取機制與雲端部署指南 (Data Fetching & Cloud Deployment Guide)

---

##  PART 1：資料抓取機制詳細說明 (How Data is Fetched)

### 1. 數據架構設計理念

`stock-tool` 採用 **「後端批次預處理 + 前端靜態載入 (Batch Processing + Static Fetch)」** 的數據管線架構。

- **為什麼不直接由瀏覽器即時爬資料？**
  1. **跨網域限制 (CORS)**：瀏覽器直接請求 Yahoo Finance 或 TWSE 會被 CORS 機制拒絕。
  2. **API 頻率限制 (Rate Limits)**：若幾千名使用者同時載入網頁並對 Yahoo/TWSE 發起數萬次請求，IP 會被永久封鎖。
  3. **載入速度與效能**：預先整理為輕量 `.json` 檔案後，前端只需一次 HTTP 請求即可呈現數千檔股票數據。

---

### 2. 四大資料來源與抓取實作

```
┌────────────────────────┐      ┌─────────────────────────┐
│     資料來源 (Sources)  │      │   抓取腳本 (Scripts)    │
├────────────────────────┤      ├─────────────────────────┤
│ Yahoo Finance API      │ ───► │ scan.js / fetch_*.py    │ ──┐
│ 台灣證券交易所 (TWSE)  │ ───► │ fetch_twse_institutions │ ──┼─► 寫入 data/*.json
│ 美國 SEC EDGAR API     │ ───► │ insider_scan.js         │ ──┤   (GitHub Repository)
│ 新聞 & AI (Claude/BERT)│ ───► │ score_news_finbert.py   │ ──┘
└────────────────────────┘      └─────────────────────────┘
```

#### A. 美股/台股行情與基本面 (Yahoo Finance API)
- **核心腳本**：`scripts/scan.js`、`scripts/fetch_ohlcv.py`
- **抓取內容**：日 K線 (OHLCV)、成交量、本益比 (PE)、股利殖利率、財報發布日期與分析師評等。
- **機制與防封鎖措施**：
  - **Proxy 代理輪替**：配置 `PROXIES` 代理伺服器陣列（`allorigins`, `corsproxy` 等）進行流量輪替。
  - **快取機制**：設有 24 小時 TTL 的 `data/quote_cache.json` 與 1 年 TTL 的 `data/sector_cache.json`，避免重複請求不常變動的資料。

#### B. 台股三大法人買賣超 (TWSE / TPEx)
- **核心腳本**：`scripts/fetch_twse_institutions.py`
- **抓取內容**：外資及陸資、投信、自營商每日買賣超張數與金額。
- **目標 API**：
  - TWSE (證交所)：`https://www.twse.com.tw/fund/T86`
  - TPEx (櫃買中心)：`https://www.tpex.org.tw/web/stock/3insti/daily_trade/`
- **數據加工**：腳本自動將買賣超金額換算為 `strong_buy` / `buy` / `neutral` / `sell` / `strong_sell` 五級信心指標。

#### C. 美股內部人交易 (SEC EDGAR)
- **核心腳本**：`scripts/insider_scan.js`
- **抓取內容**：美國上市公司高管 (CFO, CEO) 與董事會成員的 Form 4 申報紀錄。
- **目標 API**：`https://efts.sec.gov/LATEST/search-index`
- **數據加工**：計算過去 30 天內是否有 2 位以上內部人買入或金額超過 $100K，標記為叢集買入 (`clusterBuy`)。

#### D. 新聞情緒與 AI 市場敘事 (FinBERT + Claude API)
- **核心腳本**：`scripts/fetch_news.py`、`scripts/score_news_finbert.py`、`scripts/market_narrative.js`
- **抓取與處理**：
  1. 抓取最新財經新聞標題。
  2. 透過 `FinBERT` 自然語言模型進行看多/看空情緒打分。
  3. 將大盤、VIX、公債殖利率與新聞標題傳送至 Anthropic Claude API，生成 100 字內的「今日市場敘事」摘要。

---

## PART 2：未來雲端託管與部署指南 (Cloud Deployment Guide)

當您想把這個網頁部署到雲端伺服器或雲端平台上時，可以選擇以下幾種常見的現代化託管方案：

---

### 方案 1：Vercel 託管 (最推薦 ⭐⭐⭐⭐⭐)

Vercel 是目前前端界最流行的雲端平台，具備全球 CDN、免費 SSL 憑證與 GitHub 自動連動。

#### 部署步驟：
1. 前往 [Vercel 官網](https://vercel.com/) 註冊帳號並綁定您的 GitHub 帳號。
2. 點擊 **"Add New"** $\rightarrow$ **"Project"**，選擇 `stock-tool` 倉庫。
3. 設定 Framework Preset 與打包指令：
   - **若部署傳統單頁 (`index.html`)**：
     - Framework Preset 選擇 `Other`
     - Root Directory 填寫 `./`
   - **若部署 Vite 拆分版本 (`vite-migration`)**：
     - Root Directory 選擇 `vite-migration`
     - Build Command 填寫 `npm run build`
     - Output Directory 填寫 `dist`
4. 點擊 **"Deploy"**。此後每次您推動程式碼至 GitHub `main` 分支時，Vercel 將自動在幾秒內升級雲端網頁。

---

### 方案 2：Cloudflare Pages 託管 (全球極速 + 免費流量無限 ⭐⭐⭐⭐⭐)

Cloudflare 擁有全球最多的 Edge 網路節點，非常適合靜態網站託管。

#### 部署步驟：
1. 登入 [Cloudflare Dashboard](https://dash.cloudflare.com/)，點擊左側 **"Workers & Pages"** $\rightarrow$ **"Create application"** $\rightarrow$ **"Pages"**。
2. 連接您的 GitHub 帳號並選擇 `stock-tool` 倉庫。
3. 設定構建指令（Build settings）：
   - Build command: `cd vite-migration && npm run build`（或維持留空若僅發布靜態根目錄）
   - Build output directory: `vite-migration/dist` 或 `/`
4. 點擊 **"Save and Deploy"** 即可獲得免費 `.pages.dev` 網址，且可綁定自訂網域。

---

### 方案 3：AWS S3 + CloudFront 託管 (企業級擴充方案 ⭐⭐⭐⭐)

適合企業級自建雲端架構。

#### 部署步驟：
1. 在 AWS Console 建立一個 **Amazon S3 Bucket**，並開啟「Static website hosting」。
2. 使用 AWS CLI 將打包好的前端檔案同步至 S3：
   ```bash
   aws s3 sync ./vite-migration/dist s3://your-bucket-name --delete
   ```
3. 建立 **AWS CloudFront Distribution**，設定 Origin 為該 S3 Bucket，啟用全網 CDN 加速與 HTTPS 加密。
4. 使用 **Route 53** 將您的網域對應至 CloudFront 網址。

---

### 關鍵提醒：雲端部署後的「數據更新」維護

網頁前端部署到 Vercel 雲端後，**數據更新管線有以下 3 種實作方案**：

#### 方案 A：GitHub Actions + Direct Fetch / Auto Deploy (最推薦 ⭐⭐⭐⭐⭐)
- **運作模式**：
  1. 網頁在 Vercel 上運作（負責 UI 展現）。
  2. 後端爬蟲持續由現有的 **GitHub Actions** (`.github/workflows/*.yml`) 定時執行（每日/每4小時）。
  3. GitHub Actions 爬完數據後 `git commit` 更新 `data/*.json`。
- **資料自動同步方式**（二選一）：
  - **作法 1（直接讀取 GitHub Raw，無需重新 Deploy）**：
    前端 `fetch()` 請求改拉取 `https://raw.githubusercontent.com/<使用者名稱>/<倉庫名稱>/main/data/us_scan.json`。只要 GitHub Actions 更新檔案，使用者重新整理網頁就能立刻看到最新數據！
  - **作法 2（GitHub Commit 自動觸發 Vercel 重構）**：
    Vercel 預設會監聽 GitHub `main` 分支。當 GitHub Actions 將最新 `data/*.json` 推送至 `main` 分支時，Vercel 會在 10 秒內自動重新打包發布網頁。

#### 方案 B：使用 Vercel Deploy Hooks 觸發器 (主動通知打包 ⭐⭐⭐⭐)
1. 在 Vercel 控制台進入 **Project Settings** $\rightarrow$ **Git** $\rightarrow$ **Deploy Hooks**，建立一個 Hook URL（例如 `https://api.vercel.com/v1/integrations/deploy/prj_xxx`）。
2. 在 GitHub Actions 工作流（如 `stock-scan.yml`）最後一步新增通知：
   ```yaml
   - name: Trigger Vercel Re-deploy
     run: curl -X POST https://api.vercel.com/v1/integrations/deploy/prj_xxx
   ```
3. 爬蟲更新完資料後，發出 HTTP POST 請求，主動命令 Vercel 欄位發佈最新網頁。

#### 方案 C：使用 Vercel Cron Jobs + Serverless Functions (純 Vercel 全雲端架構 ⭐⭐⭐)
- 將爬蟲腳本寫成 Vercel Serverless Function (例如 `api/cron-scan.js`)。
- 在 `vercel.json` 中配置 Cron 定時器：
  ```json
  {
    "crons": [
      {
        "path": "/api/cron-scan",
        "schedule": "0 0 * * *"
      }
    ]
  }
  ```
- 當定時器觸發時，Vercel Function 自動爬資料並寫入雲端資料庫（如 Supabase / Firebase / Vercel KV）。

