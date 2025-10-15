# Stories

This directory contains user stories and feature documentation for the linebot project.

## Template

Each story should include:
- **Title**: Clear, concise story name
- **Description**: What the story is about
- **Acceptance Criteria**: Conditions that must be met
- **Technical Notes**: Implementation details (optional)

## Stories List

### 核心功能
1. [LINE 推播系統](01-line-push-notification.md) - 建立 LINE 推播核心功能
2. [模板引擎與動態資料渲染](02-template-syntax-d1.md) - 後端模板引擎實作
3. [訊息模板配置與預覽](03-message-preview-variable-binding.md) - JSON 輸入、變數綁定與即時預覽
4. [前後端分離架構](04-frontend-backend-separation.md) - 系統架構設計與實作（Vue 3 + Nuxt.js 3）

### 系統功能
5. [使用者認證與授權](05-authentication-authorization.md) - 登入系統與權限控制
6. [儀表板與數據分析](06-dashboard-analytics.md) - 統計數據與視覺化圖表
7. [收件人管理系統](07-recipient-management.md) - 訂閱者管理、群組、標籤
8. [排程推播系統](08-scheduled-broadcasts.md) - 預約推播與重複排程
9. [Cloudflare D1 查詢建構器](09-d1-query-builder.md) - 視覺化資料庫查詢工具

### 安全性（最優先）
10. [**安全性與權限控制**](10-security-permissions.md) - **D1 存取安全、SQL 注入防護、租戶隔離** ⚠️

## ⚠️ 重大安全警告

**本系統會存取客戶的 Cloudflare D1 資料庫，安全性是最高優先級！**

### 關鍵安全議題：
1. **使用者可以透過查詢建構器執行 SQL 查詢**
   - 必須防止 SQL 注入攻擊
   - 必須限制只能執行 SELECT 查詢
   - 必須禁止存取系統表

2. **多租戶資料隔離**
   - 使用者 A 不能存取使用者 B 的資料
   - 所有查詢必須自動加上租戶過濾條件

3. **權限控制**
   - 不同角色有不同的操作權限
   - API 層級和資料層級的雙重驗證

4. **資源保護**
   - 防止惡意查詢導致系統負載過高
   - Query timeout 和 rate limiting

**請務必詳閱 [Story 10: 安全性與權限控制](10-security-permissions.md)**

---

## 開發階段規劃

### 第零階段：安全基礎（預估 5-7 天）⚠️ 必須最優先
0. **Story 10** - **安全性與權限控制**
   - SQL 查詢驗證器（阻擋危險 SQL）
   - 租戶隔離機制
   - 唯讀資料庫連線
   - Rate limiting
   - 審計日誌

   **重要：必須在開發 D1 查詢功能之前完成！**

### 第一階段：基礎架構（預估 3-4 天）
1. **Story 04** - 前後端分離架構
   - 建立 Nuxt.js 前端專案
   - 建立 Cloudflare Workers 後端
   - 設定 D1 資料庫
   - API 設計與文檔

### 第二階段：認證與核心功能（預估 5-7 天）
2. **Story 05** - 使用者認證與授權
3. **Story 01** - LINE 推播系統
4. **Story 07** - 收件人管理系統

### 第三階段：訊息模板系統（預估 10-12 天）
5. **Story 09** - D1 查詢建構器（依賴 Story 10）
6. **Story 03** - 訊息模板配置與預覽
7. **Story 02** - 模板引擎與動態資料渲染

### 第四階段：擴展功能（預估 6-8 天）
8. **Story 08** - 排程推播系統
9. **Story 06** - 儀表板與數據分析

## 總預估工時
- **總計**: 約 29-38 天（含安全性）
- **建議團隊規模**: 2-3 人
- **預計完成時間**: 2-2.5 個月（含測試與調整）
- **安全審計**: 額外 3-5 天

## 核心工作流程

### 訊息模板建立流程
```
使用者操作（Story 03）
    ↓
1. 貼上 LINE 訊息 JSON（含 Flex Message）
    ↓
2. 系統自動解析變數 (user.name, order.id 等)
    ↓
3. 設定每個變數的資料來源：
   - 手動輸入測試資料
   - SQL 查詢（Story 09）
   - 收件人屬性
    ↓
4. 右側即時預覽：
   - 變數列表
   - LINE 聊天室模擬
   - 填入資料後的 JSON
    ↓
5. 儲存為訊息模板
    ↓
推播執行時（Story 02）
    ↓
6. 後端模板引擎處理：
   - 執行 SQL 查詢取得資料
   - 填入收件人資訊
   - 處理條件判斷和迴圈
   - 渲染最終 JSON
    ↓
7. 發送到 LINE (Story 01)
```

## 技術棧總覽

### 前端
- **框架**: Vue 3 + Nuxt.js 3
- **UI**: Tailwind CSS + Nuxt UI
- **狀態管理**: Pinia
- **圖表**: Chart.js / ApexCharts
- **表單驗證**: VeeValidate + Zod
- **JSON 編輯器**: CodeMirror / Monaco Editor

### 後端
- **運行環境**: Cloudflare Workers
- **框架**: Hono
- **資料庫**: Cloudflare D1
- **認證**: JWT
- **排程**: Cloudflare Cron Triggers
- **模板引擎**: Handlebars.js / 自訂實作

### 整合
- **LINE Messaging API**
- **RESTful API**
- **Webhook 處理**
