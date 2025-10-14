# Story 06: 儀表板與數據分析

## 描述
建立一個儀表板頁面，顯示推播系統的關鍵指標和數據分析，幫助使用者了解推播效果和系統使用情況。

## 使用者故事
作為系統使用者，我希望能在儀表板上看到推播的統計數據和趨勢圖表，以便了解推播效果並做出數據驅動的決策。

## 驗收標準

### 儀表板頁面
- [ ] 總覽統計卡片
  - 總推播次數
  - 成功/失敗比例
  - 活躍收件人數
  - 本月推播成長率
- [ ] 推播趨勢圖表
  - 每日推播數量折線圖
  - 成功率趨勢圖
  - 推播類型分布圓餅圖
- [ ] 最近推播列表
  - 顯示最近 10 筆推播
  - 狀態指示器
  - 快速操作按鈕
- [ ] 熱門模板排行
  - 使用次數最多的模板
  - 使用頻率趨勢
- [ ] 時間範圍篩選器
  - 今天/本週/本月/自訂範圍
  - 即時更新圖表

### 數據分析
- [ ] 推播效能分析
  - 平均發送時間
  - 錯誤類型統計
  - 尖峰時段分析
- [ ] 收件人分析
  - 活躍度統計
  - 地區分布（如可取得）
  - 收件人成長趨勢
- [ ] 模板效能分析
  - 各模板使用次數
  - 成功率比較
- [ ] 匯出報表功能
  - CSV 格式
  - PDF 格式（可選）
  - 自訂欄位選擇

### 響應式設計
- [ ] 桌面版最佳化佈局
- [ ] 平板版調整
- [ ] 手機版簡化顯示

## API 端點
```
GET    /api/analytics/overview           # 總覽統計
GET    /api/analytics/broadcasts/trend   # 推播趨勢數據
GET    /api/analytics/broadcasts/types   # 推播類型分布
GET    /api/analytics/templates/popular  # 熱門模板
GET    /api/analytics/recipients/stats   # 收件人統計
GET    /api/analytics/performance        # 效能分析
POST   /api/analytics/export             # 匯出報表
```

## 資料模型
```typescript
// 總覽統計
interface AnalyticsOverview {
  totalBroadcasts: number
  successRate: number
  activeRecipients: number
  monthlyGrowth: number
  period: {
    start: string
    end: string
  }
}

// 趨勢數據
interface TrendData {
  date: string
  count: number
  successCount: number
  failureCount: number
}

// 模板統計
interface TemplateStats {
  templateId: string
  templateName: string
  usageCount: number
  successRate: number
  lastUsed: string
}
```

## Nuxt.js 實作要點

### 1. 儀表板頁面
```vue
<!-- pages/dashboard/index.vue -->
<template>
  <div class="dashboard">
    <StatsCards :data="overview" />
    <div class="charts-grid">
      <TrendChart :data="trendData" />
      <TypeDistribution :data="typeData" />
    </div>
    <RecentBroadcasts :broadcasts="recentBroadcasts" />
    <PopularTemplates :templates="popularTemplates" />
  </div>
</template>

<script setup lang="ts">
const dateRange = ref({ start: '', end: '' })
const { data: overview } = await useFetch('/api/analytics/overview', {
  query: dateRange
})
</script>
```

### 2. 圖表元件
使用圖表函式庫：
- **Chart.js** + vue-chartjs
- 或 **ApexCharts** + vue3-apexcharts
- 或 **ECharts** + vue-echarts

### 3. 數據更新策略
```typescript
// composables/useAnalytics.ts
export const useAnalytics = () => {
  const dateRange = ref({
    start: new Date().toISOString(),
    end: new Date().toISOString()
  })

  const { data, refresh } = useFetch('/api/analytics/overview', {
    query: dateRange,
    // 每 5 分鐘自動更新
    watch: [dateRange],
    // 使用 SWR 策略
    server: false,
    lazy: true
  })

  return { data, refresh, dateRange }
}
```

## UI 元件設計

### 統計卡片
```vue
<StatsCard
  title="總推播次數"
  :value="1234"
  :trend="12.5"
  trend-label="較上月"
  icon="i-heroicons-paper-airplane"
/>
```

### 趨勢圖表
- 折線圖：顯示時間序列數據
- 區域圖：強調數據累積
- 柱狀圖：比較不同時段

### 分布圖表
- 圓餅圖：推播類型分布
- 環形圖：成功/失敗比例
- 條形圖：模板使用排行

## 效能優化
- [ ] 大數據分頁載入
- [ ] 圖表資料快取
- [ ] 懶加載非關鍵圖表
- [ ] 使用虛擬滾動處理長列表
- [ ] SSR 預渲染統計數據

## 技術需求
- 圖表函式庫（Chart.js/ApexCharts/ECharts）
- 日期選擇器（VueDatePicker）
- 資料導出函式庫（json2csv, jsPDF）
- 後端數據聚合邏輯
- Cloudflare D1 分析查詢

## 相依性
- Story 01（推播系統）
- Story 02（模板系統）
- Story 05（認證授權）

## 優先級
中

## 預估工時
4-5 天
