# Story 03: 訊息預覽與變數綁定系統

## 描述
建立一個訊息編輯與預覽系統，使用者可以貼上 LINE 訊息的 JSON（包含 Flex Message），系統會自動解析變數，讓使用者設定資料來源（手動輸入或 SQL 查詢），並即時預覽實際推播的樣子。

## 使用者故事
作為內容編輯者，我希望能夠貼上 LINE 訊息的 JSON，系統自動識別其中的變數，讓我設定資料來源，並即時預覽填入資料後的訊息外觀，以便快速建立動態推播內容。

## 核心功能流程

### 1. JSON 輸入階段
使用者貼上 LINE 訊息 JSON（可能包含 Flex Message 或純文字訊息）
```json
{
  "type": "flex",
  "altText": "訂單通知",
  "contents": {
    "type": "bubble",
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "您好，{{user.name}}！",
          "weight": "bold"
        },
        {
          "type": "text",
          "text": "訂單編號：{{order.id}}"
        },
        {
          "type": "text",
          "text": "金額：NT$ {{order.amount}}"
        }
      ]
    }
  }
}
```

### 2. 變數解析階段
系統自動掃描 JSON，識別所有變數（例如 `{{user.name}}`, `{{order.id}}`）

### 3. 資料來源設定階段
使用者為每個變數設定資料來源：
- **選項 A：手動輸入範例資料**（用於測試預覽）
- **選項 B：SQL 查詢結果**（從 D1 資料庫查詢）
- **選項 C：收件人屬性**（LINE 使用者資料）

### 4. 即時預覽階段
右側顯示：
- **變數列表**：所有偵測到的變數及其資料來源
- **Flex Message 預覽**：模擬 LINE 聊天室中的實際外觀
- **JSON 預覽**：填入資料後的完整 JSON

## 驗收標準

### JSON 編輯器
- [ ] 提供 JSON 輸入區域（語法高亮）
- [ ] JSON 格式驗證
- [ ] 支援 Flex Message 和純文字訊息
- [ ] JSON 格式化/壓縮功能
- [ ] 範本庫（常用 Flex Message 範本）
- [ ] 從 LINE Flex Message Simulator 匯入

### 變數解析
- [ ] 自動偵測變數語法（`{{variable}}`, `{{object.property}}`）
- [ ] 支援巢狀物件（`{{user.profile.name}}`）
- [ ] 支援陣列存取（`{{items[0].name}}`）
- [ ] 條件語法偵測（`{{#if}}`, `{{#each}}`）
- [ ] 變數去重和分類
- [ ] 顯示變數使用次數

### 資料來源設定

#### 手動輸入模式
- [ ] 為每個變數提供輸入欄位
- [ ] 自動型別推斷（string, number, boolean）
- [ ] 範例資料建議
- [ ] 批次填入測試資料

#### SQL 查詢模式
- [ ] 整合 D1 查詢建構器（Story 09）
- [ ] 欄位映射介面（SQL 欄位 → 變數）
- [ ] 查詢結果預覽
- [ ] 參數化查詢支援
- [ ] 處理單筆/多筆結果
  - 單筆：直接映射
  - 多筆：結合 `{{#each}}` 迴圈

#### 收件人屬性模式
- [ ] 從收件人資料映射
  - `{{recipient.lineUserId}}`
  - `{{recipient.displayName}}`
  - `{{recipient.groups}}`
  - `{{recipient.tags}}`
- [ ] 自訂收件人欄位支援

### 即時預覽

#### 變數面板
- [ ] 列出所有偵測到的變數
- [ ] 顯示資料來源類型（手動/SQL/收件人）
- [ ] 顯示當前值
- [ ] 快速編輯功能
- [ ] 變數狀態指示器（已設定/未設定/錯誤）

#### Flex Message 預覽
- [ ] 模擬 LINE 聊天室介面
- [ ] 手機尺寸模擬（iPhone/Android）
- [ ] 深色/淺色主題切換
- [ ] 填入實際資料渲染
- [ ] 錯誤訊息顯示（缺少變數、格式錯誤）
- [ ] 可捲動預覽區域

#### JSON 預覽
- [ ] 顯示填入資料後的完整 JSON
- [ ] 語法高亮
- [ ] 複製到剪貼簿
- [ ] 下載 JSON 檔案

### 模板儲存
- [ ] 儲存訊息模板（含變數設定）
- [ ] 模板名稱和描述
- [ ] 模板分類/標籤
- [ ] 複製模板
- [ ] 版本控制（可選）

## UI/UX 設計

### 三欄式佈局
```
┌─────────────────────────────────────────────────────────────┐
│  📝 訊息模板編輯                                              │
├──────────────────┬────────────────┬─────────────────────────┤
│                  │                │                         │
│  JSON 編輯器      │  變數設定       │   即時預覽              │
│                  │                │                         │
│  [JSON 輸入區]    │  變數列表：     │   [LINE 聊天室模擬]     │
│                  │                │                         │
│  - 語法高亮       │  user.name     │   [Flex Message 渲染]   │
│  - 自動完成       │  └ 資料來源: 手動│                         │
│  - 格式化         │     值: 張三    │   [變數值顯示]          │
│                  │                │                         │
│  [範本庫按鈕]     │  order.id      │   [JSON 輸出]           │
│  [驗證按鈕]       │  └ 資料來源: SQL│                         │
│                  │     查詢: ...   │   [複製/下載]           │
│                  │                │                         │
│                  │  [測試推播按鈕] │   [錯誤提示]            │
│                  │                │                         │
└──────────────────┴────────────────┴─────────────────────────┘
```

## API 端點
```
# 模板相關
POST   /api/message-templates          # 創建訊息模板
GET    /api/message-templates          # 取得模板列表
GET    /api/message-templates/:id      # 取得模板詳情
PUT    /api/message-templates/:id      # 更新模板
DELETE /api/message-templates/:id      # 刪除模板

# 變數解析
POST   /api/message-templates/parse    # 解析 JSON 中的變數
POST   /api/message-templates/preview  # 預覽（填入資料）
POST   /api/message-templates/validate # 驗證 JSON 格式

# 測試推播
POST   /api/message-templates/:id/test # 發送測試訊息到指定 LINE ID
```

## 資料模型
```typescript
// 訊息模板
interface MessageTemplate {
  id: string
  name: string
  description?: string
  type: 'text' | 'flex' | 'image' | 'video'

  // 原始 JSON（含變數）
  messageJson: any

  // 變數設定
  variables: VariableConfig[]

  // 分類
  category?: string
  tags: string[]

  createdBy: string
  createdAt: string
  updatedAt: string
  usageCount: number
}

// 變數配置
interface VariableConfig {
  name: string // 變數名稱，例如 "user.name"
  path: string[] // 解析路徑 ["user", "name"]
  type: 'string' | 'number' | 'boolean' | 'array' | 'object'

  // 資料來源
  source: VariableSource

  // 範例值（用於預覽）
  exampleValue?: any

  required: boolean
  description?: string
}

// 資料來源
type VariableSource =
  | { type: 'manual', value: any }
  | { type: 'sql', queryId: string, fieldMapping: string }
  | { type: 'recipient', field: string }
  | { type: 'function', expression: string } // 進階：支援表達式

// 預覽請求
interface PreviewRequest {
  messageJson: any
  variables: Record<string, any> // 變數名稱 -> 值
  recipientData?: any // 模擬收件人資料
}

// 預覽回應
interface PreviewResponse {
  success: boolean
  renderedMessage: any // 填入資料後的 JSON
  preview: {
    type: 'flex' | 'text'
    content: any
  }
  errors?: Array<{
    variable: string
    message: string
  }>
  missingVariables: string[]
}
```

## Nuxt.js 頁面結構

### 訊息模板編輯頁面
```vue
<!-- pages/templates/editor.vue -->
<template>
  <div class="template-editor">
    <div class="editor-header">
      <UInput v-model="template.name" placeholder="模板名稱" />
      <UButton @click="saveTemplate">儲存模板</UButton>
    </div>

    <div class="editor-layout">
      <!-- 左側：JSON 編輯器 -->
      <section class="json-editor-panel">
        <h3>訊息 JSON</h3>
        <JsonEditor
          v-model="template.messageJson"
          @change="parseVariables"
        />

        <div class="actions">
          <UButton @click="formatJson">格式化</UButton>
          <UButton @click="validateJson">驗證</UButton>
          <UButton @click="openTemplateLibrary">範本庫</UButton>
        </div>
      </section>

      <!-- 中間：變數設定 -->
      <section class="variables-panel">
        <h3>變數設定 ({{ variables.length }})</h3>

        <div v-for="variable in variables" :key="variable.name" class="variable-item">
          <div class="variable-header">
            <code>{{ variable.name }}</code>
            <UBadge :color="getSourceColor(variable.source.type)">
              {{ variable.source.type }}
            </UBadge>
          </div>

          <!-- 資料來源選擇 -->
          <USelect
            v-model="variable.source.type"
            :options="['manual', 'sql', 'recipient']"
            @change="onSourceTypeChange(variable)"
          />

          <!-- 手動輸入 -->
          <UInput
            v-if="variable.source.type === 'manual'"
            v-model="variable.source.value"
            :placeholder="variable.name"
            @input="updatePreview"
          />

          <!-- SQL 查詢 -->
          <div v-if="variable.source.type === 'sql'">
            <QuerySelector
              v-model="variable.source.queryId"
              @change="updatePreview"
            />
            <FieldMapping
              v-model="variable.source.fieldMapping"
              :query-id="variable.source.queryId"
            />
          </div>

          <!-- 收件人屬性 -->
          <USelect
            v-if="variable.source.type === 'recipient'"
            v-model="variable.source.field"
            :options="recipientFields"
            @change="updatePreview"
          />
        </div>

        <UButton
          color="primary"
          block
          @click="testBroadcast"
        >
          發送測試訊息
        </UButton>
      </section>

      <!-- 右側：預覽 -->
      <section class="preview-panel">
        <UTabs v-model="previewTab">
          <UTab name="visual" label="視覺預覽" />
          <UTab name="json" label="JSON" />
        </UTabs>

        <!-- 視覺預覽 -->
        <div v-if="previewTab === 'visual'" class="visual-preview">
          <LineMessagePreview
            :message="previewMessage"
            :theme="previewTheme"
          />

          <div class="preview-controls">
            <UButton @click="toggleTheme">
              切換主題
            </UButton>
          </div>

          <!-- 錯誤提示 -->
          <UAlert
            v-if="previewErrors.length > 0"
            color="red"
            title="變數錯誤"
          >
            <ul>
              <li v-for="error in previewErrors" :key="error.variable">
                {{ error.variable }}: {{ error.message }}
              </li>
            </ul>
          </UAlert>

          <!-- 缺少變數警告 -->
          <UAlert
            v-if="missingVariables.length > 0"
            color="yellow"
            title="未設定的變數"
          >
            {{ missingVariables.join(', ') }}
          </UAlert>
        </div>

        <!-- JSON 預覽 -->
        <div v-else class="json-preview">
          <JsonViewer :data="previewMessage" />
          <UButton @click="copyJson">複製 JSON</UButton>
          <UButton @click="downloadJson">下載 JSON</UButton>
        </div>
      </section>
    </div>
  </div>
</template>

<script setup lang="ts">
const template = ref<MessageTemplate>({
  name: '',
  messageJson: {},
  variables: [],
  tags: []
})

const variables = ref<VariableConfig[]>([])
const previewMessage = ref<any>(null)
const previewErrors = ref<any[]>([])
const missingVariables = ref<string[]>([])
const previewTab = ref('visual')
const previewTheme = ref<'light' | 'dark'>('light')

// 解析變數
async function parseVariables() {
  const { data } = await $fetch('/api/message-templates/parse', {
    method: 'POST',
    body: { messageJson: template.value.messageJson }
  })

  variables.value = data.variables
}

// 更新預覽
async function updatePreview() {
  const variableValues = {}
  variables.value.forEach(v => {
    variableValues[v.name] = getVariableValue(v)
  })

  const { data } = await $fetch('/api/message-templates/preview', {
    method: 'POST',
    body: {
      messageJson: template.value.messageJson,
      variables: variableValues
    }
  })

  previewMessage.value = data.renderedMessage
  previewErrors.value = data.errors || []
  missingVariables.value = data.missingVariables || []
}

// 測試推播
async function testBroadcast() {
  // 彈出對話框讓使用者輸入測試用的 LINE User ID
  const userId = await promptForUserId()

  await $fetch(`/api/message-templates/${template.value.id}/test`, {
    method: 'POST',
    body: { userId }
  })

  // 顯示成功訊息
}
</script>
```

## 核心元件

### JSON 編輯器
```vue
<!-- components/JsonEditor.vue -->
<template>
  <div class="json-editor">
    <codemirror
      v-model="code"
      :extensions="extensions"
      @change="handleChange"
    />
  </div>
</template>

<script setup lang="ts">
import { Codemirror } from 'vue-codemirror'
import { json } from '@codemirror/lang-json'
import { oneDark } from '@codemirror/theme-one-dark'

const extensions = [json(), oneDark]
</script>
```

### LINE 訊息預覽元件
```vue
<!-- components/LineMessagePreview.vue -->
<template>
  <div class="line-preview" :class="theme">
    <div class="phone-frame">
      <div class="line-header">
        <div class="back-button">←</div>
        <div class="chat-name">預覽</div>
      </div>

      <div class="chat-messages">
        <!-- Flex Message 渲染 -->
        <div v-if="message.type === 'flex'" class="message-bubble">
          <FlexMessageRenderer :contents="message.contents" />
        </div>

        <!-- 純文字訊息 -->
        <div v-else-if="message.type === 'text'" class="message-bubble text">
          {{ message.text }}
        </div>

        <!-- 其他類型... -->
      </div>
    </div>
  </div>
</template>
```

### Flex Message 渲染器
```vue
<!-- components/FlexMessageRenderer.vue -->
<template>
  <div class="flex-message" :class="contents.type">
    <!-- 遞迴渲染 Flex Message 結構 -->
    <component
      :is="getFlexComponent(contents.type)"
      :data="contents"
    />
  </div>
</template>

<script setup lang="ts">
// 根據 Flex Message 的 type 渲染對應元件
// bubble, carousel, box, text, image, button, etc.
</script>
```

### 變數解析邏輯（後端）
```typescript
// backend/src/services/variableParser.ts
export function parseVariables(json: any): string[] {
  const variables = new Set<string>()
  const variableRegex = /\{\{([^}]+)\}\}/g

  function traverse(obj: any) {
    if (typeof obj === 'string') {
      const matches = obj.matchAll(variableRegex)
      for (const match of matches) {
        variables.add(match[1].trim())
      }
    } else if (Array.isArray(obj)) {
      obj.forEach(traverse)
    } else if (obj && typeof obj === 'object') {
      Object.values(obj).forEach(traverse)
    }
  }

  traverse(json)
  return Array.from(variables)
}

// 填入變數
export function renderTemplate(json: any, variables: Record<string, any>): any {
  const rendered = JSON.parse(JSON.stringify(json))

  function traverse(obj: any) {
    if (typeof obj === 'string') {
      return obj.replace(/\{\{([^}]+)\}\}/g, (match, varName) => {
        const value = getNestedValue(variables, varName.trim())
        return value !== undefined ? String(value) : match
      })
    } else if (Array.isArray(obj)) {
      return obj.map(traverse)
    } else if (obj && typeof obj === 'object') {
      const result = {}
      for (const [key, value] of Object.entries(obj)) {
        result[key] = traverse(value)
      }
      return result
    }
    return obj
  }

  return traverse(rendered)
}

function getNestedValue(obj: any, path: string): any {
  return path.split('.').reduce((curr, key) => curr?.[key], obj)
}
```

## 技術需求
- JSON 編輯器（CodeMirror 或 Monaco Editor）
- JSON Schema 驗證
- 模板引擎（Handlebars / Mustache / 自訂）
- Flex Message 渲染引擎
- LINE Messaging API SDK
- 正規表達式解析

## 相依性
- Story 01（推播系統 - 測試推播）
- Story 07（收件人管理 - 收件人屬性）
- Story 09（D1 查詢建構器 - SQL 資料來源）

## 優先級
高（核心功能）

## 預估工時
5-6 天
