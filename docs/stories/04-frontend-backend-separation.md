# Story 04: 前後端分離架構

## 描述
設計並實作前後端分離的架構，前端負責使用者介面和互動，後端負責業務邏輯、資料處理和 LINE API 整合。

## 使用者故事
作為開發團隊，我們希望採用前後端分離的架構，以便前後端可以獨立開發、測試和部署，提升開發效率和系統可維護性。

## 驗收標準

### 前端
- [ ] 選擇並建立前端框架專案
- [ ] 實作 API 呼叫層（axios/fetch wrapper）
- [ ] 統一的錯誤處理機制
- [ ] 路由管理
- [ ] 狀態管理（如需要）
- [ ] 環境變數設定（開發/生產環境）
- [ ] 建置和部署流程

### 後端
- [ ] 設計 RESTful API 架構
- [ ] API 文檔（OpenAPI/Swagger）
- [ ] 認證與授權機制（JWT 或其他）
- [ ] CORS 設定
- [ ] 錯誤處理與統一回應格式
- [ ] 日誌記錄
- [ ] 環境變數管理
- [ ] 建置和部署流程

### API 端點規劃
```
# 推播相關
POST   /api/broadcasts          # 創建推播
GET    /api/broadcasts          # 取得推播列表
GET    /api/broadcasts/:id      # 取得推播詳情
DELETE /api/broadcasts/:id      # 刪除推播

# 模板相關
POST   /api/templates           # 創建模板
GET    /api/templates           # 取得模板列表
GET    /api/templates/:id       # 取得模板詳情
PUT    /api/templates/:id       # 更新模板
DELETE /api/templates/:id       # 刪除模板
POST   /api/templates/:id/preview # 預覽模板

# 資料查詢（D1）
POST   /api/query               # 執行資料查詢
GET    /api/tables              # 取得資料表列表
GET    /api/tables/:name/schema # 取得資料表結構
```

## 技術架構

### 前端技術棧
- **框架**: Vue 3 + Nuxt.js 3
- **建置工具**: Vite (內建於 Nuxt 3)
- **UI 框架**: Tailwind CSS + Nuxt UI
- **狀態管理**: Pinia
- **HTTP 客戶端**: $fetch (Nuxt 內建)
- **表單驗證**: VeeValidate + Zod
- **圖示**: Iconify (nuxt-icon)
- **部署**: Cloudflare Pages

### 後端技術棧
- **運行環境**: Cloudflare Workers
- **框架**: Hono / itty-router
- **資料庫**: Cloudflare D1
- **認證**: JWT
- **驗證**: Zod
- **部署**: Cloudflare Workers

### 開發工具
- **API 文檔**: OpenAPI 3.0
- **型別定義**: TypeScript
- **程式碼規範**: ESLint + Prettier
- **版本控制**: Git

## 專案結構建議
```
linebot/
├── frontend/          # Nuxt.js 前端專案
│   ├── app/
│   │   ├── components/        # Vue 元件
│   │   │   ├── broadcast/    # 推播相關元件
│   │   │   ├── template/     # 模板相關元件
│   │   │   ├── query/        # 查詢建構器元件
│   │   │   └── common/       # 共用元件
│   │   ├── pages/            # 頁面路由
│   │   ├── layouts/          # 佈局
│   │   ├── composables/      # Composition API
│   │   ├── stores/           # Pinia stores
│   │   ├── types/            # TypeScript 型別
│   │   └── utils/            # 工具函數
│   ├── server/               # Nuxt Server API (可選)
│   ├── public/               # 靜態資源
│   ├── nuxt.config.ts
│   └── package.json
│
├── backend/           # Cloudflare Workers 後端專案
│   ├── src/
│   │   ├── routes/           # API 路由
│   │   ├── services/         # 業務邏輯
│   │   ├── models/           # 資料模型
│   │   ├── middleware/       # 中介層
│   │   ├── utils/            # 工具函數
│   │   └── types/            # TypeScript 型別
│   ├── package.json
│   └── wrangler.toml
│
└── docs/             # 文檔
    └── stories/
```

## 相依性
- 無（基礎架構故事）

## 優先級
最高（需先完成才能開發其他功能）

## 預估工時
3-4 天
