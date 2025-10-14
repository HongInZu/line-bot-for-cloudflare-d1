# Story 08: 排程推播系統

## 描述
實作排程推播功能，允許使用者預約未來的推播時間，支援單次和重複排程，並提供排程管理介面。

## 使用者故事
作為內容管理員，我希望能夠預先安排推播訊息的發送時間，設定重複推播規則，以便在最佳時機觸及目標受眾，無需手動即時操作。

## 驗收標準

### 排程創建
- [ ] 選擇推播內容（模板/Flex Message）
- [ ] 選擇收件人（個人/群組/標籤）
- [ ] 設定發送時間
  - 日期時間選擇器
  - 時區設定
  - 相對時間（X 小時後）
- [ ] 重複規則設定
  - 單次
  - 每日
  - 每週（選擇星期幾）
  - 每月（選擇日期）
  - 自訂 Cron 表達式
- [ ] 排程有效期限
  - 永久重複
  - 重複 N 次
  - 到指定日期為止
- [ ] 排程預覽和確認

### 排程管理
- [ ] 排程列表顯示
  - 待執行排程
  - 執行中排程
  - 已完成排程
  - 失敗排程
- [ ] 狀態篩選
- [ ] 時間範圍篩選
- [ ] 搜尋功能
- [ ] 編輯排程
- [ ] 暫停/恢復排程
- [ ] 取消排程
- [ ] 手動立即執行
- [ ] 複製排程

### 排程詳情
- [ ] 基本資訊
  - 排程名稱
  - 創建者
  - 創建時間
  - 狀態
- [ ] 推播設定
  - 內容預覽
  - 目標收件人數
  - 發送規則
- [ ] 執行歷史
  - 每次執行時間
  - 發送狀態
  - 成功/失敗數量
  - 錯誤訊息
- [ ] 下次執行時間

### 通知提醒
- [ ] 排程即將執行提醒
- [ ] 執行成功通知
- [ ] 執行失敗警告
- [ ] 排程衝突警告

### 時區處理
- [ ] 顯示使用者本地時區
- [ ] 支援多時區設定
- [ ] 時區轉換顯示
- [ ] 夏令時間處理

## API 端點
```
# 排程相關
GET    /api/schedules              # 取得排程列表
POST   /api/schedules              # 創建排程
GET    /api/schedules/:id          # 取得排程詳情
PUT    /api/schedules/:id          # 更新排程
DELETE /api/schedules/:id          # 刪除排程
POST   /api/schedules/:id/pause    # 暫停排程
POST   /api/schedules/:id/resume   # 恢復排程
POST   /api/schedules/:id/execute  # 立即執行
POST   /api/schedules/:id/duplicate # 複製排程

# 執行歷史
GET    /api/schedules/:id/history  # 取得執行歷史
GET    /api/schedules/:id/next     # 取得下次執行時間
```

## 資料模型
```typescript
// 排程
interface Schedule {
  id: string
  name: string
  description?: string
  broadcastConfig: {
    templateId?: string
    flexMessageId?: string
    recipientType: 'all' | 'group' | 'tag' | 'individual'
    recipientIds: string[]
  }
  timing: {
    type: 'once' | 'recurring'
    startTime: string
    timezone: string
    recurrence?: RecurrenceRule
    endCondition?: EndCondition
  }
  status: 'active' | 'paused' | 'completed' | 'cancelled'
  createdBy: string
  createdAt: string
  updatedAt: string
  lastExecutedAt?: string
  nextExecutionAt?: string
}

// 重複規則
interface RecurrenceRule {
  frequency: 'daily' | 'weekly' | 'monthly' | 'custom'
  interval: number // 每隔 N 個單位
  daysOfWeek?: number[] // 0-6 (星期日-星期六)
  dayOfMonth?: number // 1-31
  cronExpression?: string // 自訂 Cron
}

// 結束條件
interface EndCondition {
  type: 'never' | 'count' | 'date'
  count?: number // 執行 N 次後停止
  endDate?: string // 到指定日期停止
}

// 執行歷史
interface ScheduleExecution {
  id: string
  scheduleId: string
  executedAt: string
  status: 'success' | 'failed' | 'partial'
  totalRecipients: number
  successCount: number
  failureCount: number
  errors?: Array<{
    recipientId: string
    error: string
  }>
  duration: number // 執行時長（毫秒）
}
```

## Nuxt.js 頁面結構

### 排程列表頁
```vue
<!-- pages/schedules/index.vue -->
<template>
  <div class="schedules-page">
    <div class="header">
      <h1>排程管理</h1>
      <UButton @click="createSchedule">創建排程</UButton>
    </div>

    <ScheduleFilters v-model="filters" />

    <UTabs v-model="activeTab">
      <UTab name="active" label="待執行" />
      <UTab name="paused" label="已暫停" />
      <UTab name="completed" label="已完成" />
    </UTabs>

    <ScheduleList
      :schedules="schedules"
      @edit="editSchedule"
      @pause="pauseSchedule"
      @delete="deleteSchedule"
    />
  </div>
</template>
```

### 創建/編輯排程頁
```vue
<!-- pages/schedules/create.vue -->
<template>
  <div class="schedule-form">
    <UCard>
      <h2>創建排程</h2>

      <!-- 基本資訊 -->
      <section>
        <UInput v-model="form.name" label="排程名稱" />
        <UTextarea v-model="form.description" label="描述" />
      </section>

      <!-- 推播內容 -->
      <section>
        <h3>推播內容</h3>
        <TemplateSelector v-model="form.templateId" />
        <!-- 或 -->
        <FlexMessageSelector v-model="form.flexMessageId" />
      </section>

      <!-- 收件人選擇 -->
      <section>
        <h3>收件人</h3>
        <RecipientSelector v-model="form.recipients" />
      </section>

      <!-- 時間設定 -->
      <section>
        <h3>發送時間</h3>
        <ScheduleTimingEditor v-model="form.timing" />
      </section>

      <!-- 預覽 -->
      <SchedulePreview :config="form" />

      <div class="actions">
        <UButton @click="save">儲存排程</UButton>
        <UButton variant="outline" @click="cancel">取消</UButton>
      </div>
    </UCard>
  </div>
</template>
```

### 排程詳情頁
```vue
<!-- pages/schedules/[id].vue -->
<template>
  <div class="schedule-detail">
    <ScheduleHeader
      :schedule="schedule"
      @pause="handlePause"
      @resume="handleResume"
      @execute="handleExecute"
    />

    <UCard title="排程設定">
      <ScheduleInfo :schedule="schedule" />
    </UCard>

    <UCard title="執行歷史">
      <ExecutionHistory :schedule-id="schedule.id" />
    </UCard>

    <UCard title="下次執行">
      <NextExecution :schedule="schedule" />
    </UCard>
  </div>
</template>
```

## 核心元件

### 時間設定元件
```vue
<template>
  <div class="timing-editor">
    <!-- 執行類型 -->
    <URadioGroup v-model="timing.type">
      <URadio value="once" label="單次執行" />
      <URadio value="recurring" label="重複執行" />
    </URadioGroup>

    <!-- 開始時間 -->
    <DateTimePicker
      v-model="timing.startTime"
      :timezone="timing.timezone"
    />

    <!-- 重複規則 -->
    <RecurrenceEditor
      v-if="timing.type === 'recurring'"
      v-model="timing.recurrence"
    />

    <!-- 結束條件 -->
    <EndConditionEditor
      v-if="timing.type === 'recurring'"
      v-model="timing.endCondition"
    />
  </div>
</template>
```

### 重複規則編輯器
```vue
<template>
  <div class="recurrence-editor">
    <USelect v-model="recurrence.frequency">
      <option value="daily">每天</option>
      <option value="weekly">每週</option>
      <option value="monthly">每月</option>
      <option value="custom">自訂</option>
    </USelect>

    <!-- 每週 - 星期選擇 -->
    <div v-if="recurrence.frequency === 'weekly'">
      <UCheckboxGroup v-model="recurrence.daysOfWeek">
        <UCheckbox :value="0" label="日" />
        <UCheckbox :value="1" label="一" />
        <!-- ... -->
      </UCheckboxGroup>
    </div>

    <!-- 每月 - 日期選擇 -->
    <div v-if="recurrence.frequency === 'monthly'">
      <UInput
        v-model.number="recurrence.dayOfMonth"
        type="number"
        min="1"
        max="31"
      />
    </div>

    <!-- 自訂 Cron -->
    <div v-if="recurrence.frequency === 'custom'">
      <CronEditor v-model="recurrence.cronExpression" />
    </div>
  </div>
</template>
```

## Cloudflare Workers Cron 設定

### wrangler.toml
```toml
[triggers]
crons = ["*/5 * * * *"]  # 每 5 分鐘檢查一次
```

### 排程執行邏輯
```typescript
export default {
  async scheduled(event: ScheduledEvent, env: Env) {
    // 1. 查詢需要執行的排程
    const now = new Date()
    const schedules = await getSchedulesToExecute(now, env.DB)

    // 2. 執行每個排程
    for (const schedule of schedules) {
      await executeSchedule(schedule, env)
    }

    // 3. 計算下次執行時間
    for (const schedule of schedules) {
      await updateNextExecutionTime(schedule, env.DB)
    }
  }
}
```

## 時區處理

使用函式庫：
- **date-fns-tz** - 時區轉換
- **luxon** - 完整日期時間處理

```typescript
import { zonedTimeToUtc, utcToZonedTime } from 'date-fns-tz'

// 使用者本地時間 -> UTC 儲存
const utcTime = zonedTimeToUtc(localTime, userTimezone)

// UTC -> 使用者本地時間顯示
const localTime = utcToZonedTime(utcTime, userTimezone)
```

## Cron 表達式

支援標準 Cron 格式：
```
┌───────────── 分鐘 (0 - 59)
│ ┌───────────── 小時 (0 - 23)
│ │ ┌───────────── 日期 (1 - 31)
│ │ │ ┌───────────── 月份 (1 - 12)
│ │ │ │ ┌───────────── 星期 (0 - 6)
│ │ │ │ │
* * * * *
```

範例：
- `0 9 * * *` - 每天早上 9:00
- `0 9 * * 1-5` - 每週一到週五早上 9:00
- `0 */2 * * *` - 每 2 小時
- `0 9 1 * *` - 每月 1 號早上 9:00

## 衝突檢測

- [ ] 檢查同時間過多排程
- [ ] 收件人重複推播警告
- [ ] 推播頻率限制（避免騷擾）

## 效能考量

- [ ] 批次執行排程（避免逐筆處理）
- [ ] 失敗重試機制（指數退避）
- [ ] 執行時長監控
- [ ] 大量收件人分批發送
- [ ] 並發控制（避免超過 LINE API 限制）

## 技術需求
- Cloudflare Workers Cron Triggers
- Cron 表達式解析（croner, node-cron）
- 時區處理函式庫（date-fns-tz, luxon）
- 日期時間選擇器（VueDatePicker）
- Cron 編輯器元件（vue-cron-editor）

## 相依性
- Story 01（推播系統）
- Story 02（模板系統）
- Story 07（收件人管理）

## 優先級
中高

## 預估工時
5-6 天
