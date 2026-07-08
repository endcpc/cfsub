<h1 align="center">
  <img src="./public/icons/auto.svg" alt="CloudflareSub Logo" height="40" align="absmiddle" /> CloudflareSub
</h1>

<p align="center"><em>一個輕量化的優選IP訂閱器</em></p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-2ea44f" alt="License MIT" />
  <img src="https://img.shields.io/badge/platform-Windows-0078D6" alt="Windows" />
  <img src="https://img.shields.io/badge/platform-macOS-111111" alt="macOS" />
  <img src="https://img.shields.io/badge/platform-Linux-FCC624?logo=linux&logoColor=black" alt="Linux" />
  <img src="https://img.shields.io/badge/runtime-Cloudflare%20Workers-F38020?logo=cloudflare&logoColor=white" alt="Cloudflare Workers" />
  <img src="https://img.shields.io/badge/status-active-00C853" alt="Status Active" />
</p>

## 功能特性

- 支持 `vmess`、`vless`、`trojan` 節點解析
- 支持 Base64 訂閱文本自動展開
- 支持 `host[:port][#remark]` 格式的優選地址
- 結果寫入 Workers KV，生成 `/sub/:id` 短鏈
- 相同輸入自動去重（7 天 TTL）
- 支持 `SUB_ACCESS_TOKEN` 訪問令牌保護
- 支持導出：Raw（Base64）/ Clash（YAML）/ Surge（文本）

## 項目結構

```text
cloudflaresub/
├─ src/
│  ├─ worker.js      # Worker 入口（API + 訂閱輸出）
│  └─ core.js        # 解析/渲染核心函數（測試使用）
├─ public/           # 前端靜態資源
├─ tests/smoke.mjs   # Smoke test
├─ wrangler.toml
└─ package.json
```

## 快速開始（Cloudflare 網頁端）
```text
視頻部署流程：https://youtu.be/E5PI0LsQ43M
```
下面按 Cloudflare Dashboard 流程操作，儘量不依賴命令行。

### 1) 準備代碼

- 把本項目代碼放到本地（你現在已經有）
- 確認 `wrangler.toml` 中 `name`、`main`、`assets` 路徑與項目一致

### 2) 在 Dashboard 創建 Worker

- 打開 Cloudflare Dashboard
- 進入 `Workers & Pages`
- 點擊 `Create application` -> `Create Worker`
- 先創建一個 Worker（用於初始化項目）

### 3) 綁定到 GitHub 倉庫（推薦）

- 在 `Workers & Pages` 點擊 `Create` -> `Import a repository`
- 授權 GitHub，並選擇倉庫 `InfiCheesy/cloudflaresub`
- 構建設置建議：
  - Framework preset: `None`
  - Build command: 留空
  - Build output directory: 留空
- 保存並開始部署

說明：這個項目是 Worker 項目，入口在 `src/worker.js`，靜態資源在 `public/`。

### 4) 創建 KV Namespace

- 進入 `Storage & Databases` -> `KV`
- 點擊 `Create namespace`
- 名稱建議：`SUB_STORE`

### 5) 給 Worker 綁定 KV

- 回到 Worker 項目頁面
- 進入 `Settings` -> `Bindings`
- 點擊 `Add binding`，類型選擇 `KV namespace`
- Variable name 填：`SUB_STORE`
- Namespace 選擇上一步創建的 KV
- 保存並重新部署

### 6) 配置訪問令牌 Secret

- 在 Worker 項目中進入 `Settings` -> `Variables`
- 在 `Secrets` 區域添加：
  - Key: `SUB_ACCESS_TOKEN`
  - Value: 你自定義的一串令牌
- 保存後重新部署

說明：
- 設置後，請求 `/sub/:id` 必須帶 `?token=...`
- 不設置也可運行，但訂閱鏈接沒有二次訪問保護

### 7) 驗證線上服務

- 打開 Worker 域名（如 `https://<name>.<subdomain>.workers.dev`）
- 訪問首頁 `/`，應看到前端表單
- 在頁面輸入節點和優選地址，點擊生成
- 拿到 `/sub/:id` 後測試：
  - `?target=raw&token=...`
  - `?target=clash&token=...`
  - `?target=surge&token=...`

### 8) 後續更新代碼

- 如果你使用 GitHub 自動部署：直接 push 到對應分支，Cloudflare 會自動重新部署
- 如果你不用 GitHub 自動部署：可在 Dashboard 在線編輯器中修改後手動部署

## API 說明

### `POST /api/generate`

輸入原始節點與優選地址，返回短鏈訂閱。

請求體示例：

```json
{
  "nodeLinks": "vmess://...\nvless://...",
  "preferredIps": "104.16.1.2#HK\n104.17.2.3:2053#US",
  "namePrefix": "CF",
  "keepOriginalHost": true
}
```

字段說明：
- `nodeLinks`: 多行節點鏈接
- `preferredIps`: 多行優選地址，格式 `host[:port][#remark]`
- `namePrefix`: 節點名附加前綴
- `keepOriginalHost`: 是否保留原始 Host/SNI（默認 `true`）

返回示例（節選）：

```json
{
  "ok": true,
  "shortId": "AbC123xYz9",
  "urls": {
    "auto": "https://<worker>/sub/AbC123xYz9?token=...",
    "raw": "https://<worker>/sub/AbC123xYz9?target=raw&token=...",
    "clash": "https://<worker>/sub/AbC123xYz9?target=clash&token=...",
    "surge": "https://<worker>/sub/AbC123xYz9?target=surge&token=..."
  }
}
```

### `GET /sub/:id`

按 `target` 返回訂閱內容：
- `target=raw`（默認）
- `target=clash`
- `target=surge`

示例：

```bash
curl "https://<worker>/sub/<id>?target=clash&token=<SUB_ACCESS_TOKEN>"
```

## 前端頁面

根路徑 `/` 提供網頁表單（來自 `public/`）：
- 粘貼節點鏈接
- 粘貼優選 IP / 域名
- 生成並展示各客戶端訂閱鏈接
- 一鍵複製 / 生成二維碼


## 注意事項

- `src/worker.js` 當前是 KV 短鏈方案，不依賴 `SUB_LINK_SECRET`
- 每條訂閱記錄默認保存 7 天（TTL）
- Surge 導出當前僅包含 `vmess` / `trojan`

## License

MIT
