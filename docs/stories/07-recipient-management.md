# Story 07: 收件人管理系統

## 描述
建立收件人（LINE 使用者）管理系統，能夠管理訂閱者、建立群組、標籤分類，並支援批次匯入和匯出。

## 使用者故事
作為系統管理員，我希望能夠管理所有 LINE 訂閱者的資訊，將他們分組和標記，以便針對不同受眾發送客製化的推播訊息。

## 驗收標準

### 收件人列表
- [ ] 顯示所有收件人列表（分頁）
- [ ] 搜尋功能（姓名、LINE ID、標籤）
- [ ] 篩選功能
  - 依群組篩選
  - 依標籤篩選
  - 依訂閱狀態篩選
  - 依加入時間篩選
- [ ] 排序功能（加入時間、最後活躍、姓名）
- [ ] 批次選擇操作
- [ ] 收件人詳細資料檢視

### 收件人詳情
- [ ] 基本資訊
  - LINE 使用者 ID
  - 顯示名稱
  - 頭像
  - 加入時間
  - 最後活躍時間
- [ ] 訂閱狀態管理
- [ ] 群組歸屬
- [ ] 標籤管理
- [ ] 推播歷史記錄
- [ ] 互動記錄（如有）
- [ ] 備註欄位

### 群組管理
- [ ] 創建群組
- [ ] 編輯群組資訊
- [ ] 刪除群組
- [ ] 將收件人加入群組
- [ ] 從群組移除收件人
- [ ] 查看群組成員列表
- [ ] 群組統計資訊

### 標籤系統
- [ ] 創建標籤
- [ ] 編輯標籤（名稱、顏色）
- [ ] 刪除標籤
- [ ] 為收件人添加標籤
- [ ] 移除收件人標籤
- [ ] 標籤顏色分類
- [ ] 標籤使用統計

### 批次操作
- [ ] 批次匯入收件人
  - CSV 格式
  - Excel 格式
  - 欄位映射設定
  - 匯入預覽
  - 重複資料處理
- [ ] 批次匯出收件人
  - 選擇匯出欄位
  - CSV/Excel 格式
- [ ] 批次添加標籤
- [ ] 批次加入群組
- [ ] 批次刪除/封鎖

### LINE Bot 整合
- [ ] 自動同步 LINE 好友
- [ ] 處理加入/封鎖事件
- [ ] 自動更新使用者資訊
- [ ] Webhook 接收 LINE 事件

## API 端點
```
# 收件人相關
GET    /api/recipients              # 取得收件人列表
GET    /api/recipients/:id          # 取得收件人詳情
POST   /api/recipients              # 新增收件人
PUT    /api/recipients/:id          # 更新收件人
DELETE /api/recipients/:id          # 刪除收件人
POST   /api/recipients/import       # 批次匯入
POST   /api/recipients/export       # 批次匯出
POST   /api/recipients/batch        # 批次操作

# 群組相關
GET    /api/groups                  # 取得群組列表
POST   /api/groups                  # 創建群組
GET    /api/groups/:id              # 取得群組詳情
PUT    /api/groups/:id              # 更新群組
DELETE /api/groups/:id              # 刪除群組
GET    /api/groups/:id/recipients   # 取得群組成員
POST   /api/groups/:id/recipients   # 添加成員到群組
DELETE /api/groups/:id/recipients/:recipientId  # 移除成員

# 標籤相關
GET    /api/tags                    # 取得標籤列表
POST   /api/tags                    # 創建標籤
PUT    /api/tags/:id                # 更新標籤
DELETE /api/tags/:id                # 刪除標籤

# LINE Webhook
POST   /api/webhook/line            # LINE Bot Webhook
```

## 資料模型
```typescript
// 收件人
interface Recipient {
  id: string
  lineUserId: string
  displayName: string
  pictureUrl?: string
  statusMessage?: string
  subscribed: boolean
  blocked: boolean
  groupIds: string[]
  tagIds: string[]
  notes?: string
  metadata?: Record<string, any>
  createdAt: string
  updatedAt: string
  lastActiveAt?: string
}

// 群組
interface Group {
  id: string
  name: string
  description?: string
  recipientCount: number
  createdAt: string
  updatedAt: string
}

// 標籤
interface Tag {
  id: string
  name: string
  color: string
  usageCount: number
  createdAt: string
}

// 推播歷史
interface BroadcastHistory {
  recipientId: string
  broadcastId: string
  status: 'success' | 'failed'
  sentAt: string
  error?: string
}
```

## Nuxt.js 頁面結構

### 收件人列表頁
```vue
<!-- pages/recipients/index.vue -->
<template>
  <div class="recipients-page">
    <RecipientFilters v-model="filters" />
    <RecipientTable
      :recipients="recipients"
      :selected="selectedIds"
      @select="handleSelect"
    />
    <BatchActions :selected-count="selectedIds.length" />
    <Pagination v-model="page" :total="total" />
  </div>
</template>
```

### 收件人詳情頁
```vue
<!-- pages/recipients/[id].vue -->
<template>
  <div class="recipient-detail">
    <RecipientProfile :recipient="recipient" />
    <RecipientGroups v-model="recipient.groupIds" />
    <RecipientTags v-model="recipient.tagIds" />
    <BroadcastHistory :recipient-id="recipient.id" />
  </div>
</template>
```

### 群組管理頁
```vue
<!-- pages/groups/index.vue -->
<template>
  <div class="groups-page">
    <GroupList :groups="groups" />
    <GroupDialog v-model="showDialog" />
  </div>
</template>
```

## UI 元件設計

### 收件人表格
- 虛擬滾動支援大量資料
- 多選 checkbox
- 即時搜尋
- 拖拉排序欄位

### 篩選器
```vue
<RecipientFilters>
  <SearchInput v-model="search" />
  <GroupSelect v-model="selectedGroups" multiple />
  <TagSelect v-model="selectedTags" multiple />
  <DateRangePicker v-model="dateRange" />
  <StatusFilter v-model="status" />
</RecipientFilters>
```

### 批次操作面板
- 固定在底部（選擇項目時顯示）
- 顯示已選數量
- 快速操作按鈕

### 標籤元件
```vue
<Tag
  :label="tag.name"
  :color="tag.color"
  removable
  @remove="removeTag"
/>
```

## 匯入/匯出格式

### CSV 匯入範例
```csv
lineUserId,displayName,groups,tags,notes
U1234567890,張三,"VIP,活動A","重要客戶,高價值",這是備註
U0987654321,李四,"活動B","一般客戶",
```

### 欄位映射
- 自動偵測 CSV 標題
- 拖拉對應欄位
- 預覽前 5 筆資料
- 處理衝突規則設定

## LINE Bot Webhook 處理

### 好友事件
```typescript
// Follow event - 使用者加入好友
{
  type: 'follow',
  userId: 'U1234567890'
}
// 自動創建收件人記錄

// Unfollow event - 使用者封鎖
{
  type: 'unfollow',
  userId: 'U1234567890'
}
// 標記為 blocked
```

### 自動同步
- 定期同步 LINE 好友列表
- 更新使用者顯示名稱和頭像
- 清理已封鎖的使用者

## 效能優化
- [ ] 收件人列表分頁（每頁 50 筆）
- [ ] 搜尋結果快取
- [ ] 虛擬滾動長列表
- [ ] 批次操作進度提示
- [ ] 背景處理大量匯入

## 技術需求
- CSV/Excel 解析函式庫（papaparse, xlsx）
- 虛擬滾動函式庫（vue-virtual-scroller）
- 檔案上傳處理
- LINE Messaging API
- Cloudflare D1 資料庫
- Worker Cron（定期同步）

## 相依性
- Story 01（推播系統 - 查看推播歷史）
- Story 05（認證授權）

## 優先級
高

## 預估工時
5-6 天
