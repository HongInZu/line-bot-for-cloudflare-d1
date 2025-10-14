# Story 09: Cloudflare D1 查詢建構器

## 描述
建立一個視覺化的 D1 資料庫查詢建構器介面，讓使用者能夠透過圖形化介面建立查詢，無需手寫 SQL，並將查詢結果與模板語法整合。

## 使用者故事
作為內容編輯者，我希望能夠透過簡單的圖形化介面從 D1 資料庫查詢資料，並將查詢結果綁定到推播模板中，而不需要了解 SQL 語法。

## 驗收標準

### 資料庫瀏覽
- [ ] 顯示所有資料表列表
- [ ] 查看資料表結構（欄位、型別、索引）
- [ ] 瀏覽資料表資料（分頁）
- [ ] 資料表關聯視覺化
- [ ] 搜尋資料表和欄位

### 查詢建構器
- [ ] 選擇資料表
- [ ] 選擇欄位（SELECT）
  - 單選/多選欄位
  - 欄位別名
  - 聚合函數（COUNT, SUM, AVG, MAX, MIN）
- [ ] 條件設定（WHERE）
  - 多條件組合
  - AND/OR 邏輯
  - 比較運算子（=, !=, >, <, >=, <=, LIKE, IN, IS NULL）
  - 參數化查詢（變數綁定）
- [ ] 排序（ORDER BY）
  - 多欄位排序
  - ASC/DESC
- [ ] 分組（GROUP BY）
- [ ] 限制筆數（LIMIT/OFFSET）
- [ ] JOIN 操作
  - INNER JOIN
  - LEFT JOIN
  - RIGHT JOIN
  - 選擇關聯欄位

### 查詢執行
- [ ] 即時 SQL 預覽
- [ ] 執行查詢
- [ ] 顯示查詢結果
  - 表格檢視
  - JSON 檢視
- [ ] 查詢效能資訊
  - 執行時間
  - 回傳筆數
- [ ] 錯誤提示和建議
- [ ] 查詢歷史記錄

### 查詢儲存
- [ ] 儲存查詢為範本
- [ ] 命名和描述查詢
- [ ] 查詢分類/標籤
- [ ] 查詢版本控制
- [ ] 匯入/匯出查詢

### 參數化查詢
- [ ] 定義查詢參數
  - 參數名稱
  - 參數型別（string, number, boolean, date）
  - 預設值
  - 必填/選填
- [ ] 參數輸入介面
- [ ] 參數驗證
- [ ] 測試參數值

### 與模板整合
- [ ] 將查詢綁定到模板
- [ ] 查詢結果映射到模板變數
- [ ] 支援巢狀資料結構
- [ ] 陣列資料處理（迴圈）
- [ ] 條件顯示（依查詢結果）

## API 端點
```
# 資料庫管理
GET    /api/d1/tables                    # 取得資料表列表
GET    /api/d1/tables/:name              # 取得資料表詳情
GET    /api/d1/tables/:name/schema       # 取得資料表結構
GET    /api/d1/tables/:name/data         # 瀏覽資料
GET    /api/d1/tables/:name/relationships # 取得關聯資訊

# 查詢相關
POST   /api/d1/query/execute             # 執行查詢
POST   /api/d1/query/validate            # 驗證查詢
POST   /api/d1/query/explain             # 查詢計畫分析

# 儲存的查詢
GET    /api/queries                      # 取得查詢列表
POST   /api/queries                      # 儲存查詢
GET    /api/queries/:id                  # 取得查詢詳情
PUT    /api/queries/:id                  # 更新查詢
DELETE /api/queries/:id                  # 刪除查詢
POST   /api/queries/:id/execute          # 執行儲存的查詢
```

## 資料模型
```typescript
// 儲存的查詢
interface SavedQuery {
  id: string
  name: string
  description?: string
  config: QueryConfig
  sql: string // 生成的 SQL
  parameters: QueryParameter[]
  category?: string
  tags: string[]
  createdBy: string
  createdAt: string
  updatedAt: string
  usageCount: number
}

// 查詢配置
interface QueryConfig {
  table: string
  select: SelectClause[]
  where: WhereClause[]
  orderBy: OrderByClause[]
  groupBy: string[]
  limit?: number
  offset?: number
  joins: JoinClause[]
}

interface SelectClause {
  field: string
  alias?: string
  aggregate?: 'COUNT' | 'SUM' | 'AVG' | 'MAX' | 'MIN'
}

interface WhereClause {
  field: string
  operator: '=' | '!=' | '>' | '<' | '>=' | '<=' | 'LIKE' | 'IN' | 'IS NULL'
  value: any
  parameter?: string // 參數名稱（如為參數化查詢）
  logic: 'AND' | 'OR'
}

interface OrderByClause {
  field: string
  direction: 'ASC' | 'DESC'
}

interface JoinClause {
  type: 'INNER' | 'LEFT' | 'RIGHT'
  table: string
  on: {
    left: string
    right: string
  }
}

// 查詢參數
interface QueryParameter {
  name: string
  type: 'string' | 'number' | 'boolean' | 'date'
  defaultValue?: any
  required: boolean
  description?: string
}

// 查詢結果
interface QueryResult {
  success: boolean
  data: any[]
  rowCount: number
  executionTime: number
  sql: string
  error?: string
}
```

## Nuxt.js 頁面結構

### 查詢建構器主頁
```vue
<!-- pages/query-builder/index.vue -->
<template>
  <div class="query-builder-page">
    <div class="layout">
      <!-- 左側：資料庫瀏覽器 -->
      <aside class="database-explorer">
        <DatabaseTreeView
          :tables="tables"
          @select="selectTable"
        />
      </aside>

      <!-- 中間：查詢建構器 -->
      <main class="query-builder">
        <QueryBuilderForm v-model="queryConfig" />
        <SqlPreview :config="queryConfig" />
        <QueryActions
          @execute="executeQuery"
          @save="saveQuery"
        />
      </main>

      <!-- 右側：結果顯示 -->
      <aside class="query-results">
        <QueryResultsTable :results="results" />
      </aside>
    </div>
  </div>
</template>
```

### 儲存的查詢列表
```vue
<!-- pages/queries/index.vue -->
<template>
  <div class="saved-queries">
    <QueryFilters v-model="filters" />
    <QueryList
      :queries="queries"
      @edit="editQuery"
      @execute="executeQuery"
      @delete="deleteQuery"
    />
  </div>
</template>
```

## 核心元件

### 資料表選擇器
```vue
<template>
  <div class="table-selector">
    <USelect
      v-model="selectedTable"
      :options="tables"
      placeholder="選擇資料表"
    >
      <template #option="{ option }">
        <div class="table-option">
          <Icon name="i-heroicons-table-cells" />
          <span>{{ option.name }}</span>
          <span class="row-count">{{ option.rowCount }} rows</span>
        </div>
      </template>
    </USelect>
  </div>
</template>
```

### 欄位選擇器
```vue
<template>
  <div class="field-selector">
    <h3>選擇欄位</h3>
    <UCheckbox v-model="selectAll" label="全選" />

    <div v-for="field in fields" :key="field.name">
      <UCheckbox
        v-model="selectedFields"
        :value="field.name"
        :label="field.name"
      />
      <span class="field-type">{{ field.type }}</span>

      <!-- 聚合函數 -->
      <USelect
        v-if="selectedFields.includes(field.name)"
        v-model="field.aggregate"
        :options="aggregateFunctions"
        placeholder="聚合函數"
      />

      <!-- 別名 -->
      <UInput
        v-if="selectedFields.includes(field.name)"
        v-model="field.alias"
        placeholder="別名"
        size="sm"
      />
    </div>
  </div>
</template>
```

### WHERE 條件編輯器
```vue
<template>
  <div class="where-editor">
    <h3>篩選條件</h3>

    <div v-for="(condition, idx) in conditions" :key="idx" class="condition">
      <!-- 邏輯運算子 -->
      <USelect
        v-if="idx > 0"
        v-model="condition.logic"
        :options="['AND', 'OR']"
      />

      <!-- 欄位 -->
      <USelect
        v-model="condition.field"
        :options="fields"
        placeholder="選擇欄位"
      />

      <!-- 運算子 -->
      <USelect
        v-model="condition.operator"
        :options="operators"
      />

      <!-- 值 -->
      <UInput
        v-model="condition.value"
        placeholder="值"
      />

      <!-- 或使用參數 -->
      <UCheckbox
        v-model="condition.useParameter"
        label="使用參數"
      />
      <UInput
        v-if="condition.useParameter"
        v-model="condition.parameter"
        placeholder="參數名稱"
      />

      <!-- 刪除 -->
      <UButton
        icon="i-heroicons-trash"
        @click="removeCondition(idx)"
      />
    </div>

    <UButton @click="addCondition">新增條件</UButton>
  </div>
</template>
```

### SQL 預覽
```vue
<template>
  <div class="sql-preview">
    <h3>SQL 預覽</h3>
    <pre><code class="language-sql">{{ generatedSql }}</code></pre>
    <UButton
      icon="i-heroicons-clipboard"
      @click="copySql"
    >
      複製 SQL
    </UButton>
  </div>
</template>

<script setup lang="ts">
const generatedSql = computed(() => {
  return buildSql(props.config)
})

function buildSql(config: QueryConfig): string {
  // SQL 生成邏輯
  let sql = `SELECT ${buildSelectClause(config.select)}`
  sql += ` FROM ${config.table}`

  if (config.joins.length > 0) {
    sql += ` ${buildJoinClause(config.joins)}`
  }

  if (config.where.length > 0) {
    sql += ` WHERE ${buildWhereClause(config.where)}`
  }

  if (config.groupBy.length > 0) {
    sql += ` GROUP BY ${config.groupBy.join(', ')}`
  }

  if (config.orderBy.length > 0) {
    sql += ` ORDER BY ${buildOrderByClause(config.orderBy)}`
  }

  if (config.limit) {
    sql += ` LIMIT ${config.limit}`
  }

  if (config.offset) {
    sql += ` OFFSET ${config.offset}`
  }

  return sql
}
</script>
```

### 查詢結果表格
```vue
<template>
  <div class="query-results">
    <div class="results-header">
      <span>{{ results.rowCount }} 筆結果</span>
      <span>執行時間: {{ results.executionTime }}ms</span>

      <UButton icon="i-heroicons-arrow-down-tray" @click="exportResults">
        匯出
      </UButton>
    </div>

    <!-- 表格檢視 -->
    <UTable
      v-if="viewMode === 'table'"
      :rows="results.data"
      :columns="columns"
    />

    <!-- JSON 檢視 -->
    <pre v-else><code>{{ JSON.stringify(results.data, null, 2) }}</code></pre>

    <!-- 檢視模式切換 -->
    <UTabs v-model="viewMode">
      <UTab name="table" label="表格" />
      <UTab name="json" label="JSON" />
    </UTabs>
  </div>
</template>
```

### 參數編輯器
```vue
<template>
  <div class="parameter-editor">
    <h3>查詢參數</h3>

    <div v-for="param in parameters" :key="param.name">
      <UInput
        v-model="param.name"
        label="參數名稱"
        placeholder="例如: userId"
      />

      <USelect
        v-model="param.type"
        label="型別"
        :options="['string', 'number', 'boolean', 'date']"
      />

      <UInput
        v-model="param.defaultValue"
        label="預設值"
      />

      <UCheckbox
        v-model="param.required"
        label="必填"
      />

      <UTextarea
        v-model="param.description"
        label="說明"
      />
    </div>

    <UButton @click="addParameter">新增參數</UButton>
  </div>
</template>
```

## SQL 注入防護

**重要安全措施：**
- [ ] 使用參數化查詢（Prepared Statements）
- [ ] 輸入驗證和清理
- [ ] 禁止執行危險 SQL（DROP, TRUNCATE, DELETE without WHERE 等）
- [ ] 限制查詢權限（只讀帳號）
- [ ] 查詢超時設定
- [ ] 結果筆數限制

```typescript
// 後端驗證範例
function validateQuery(sql: string): boolean {
  const dangerousKeywords = [
    'DROP', 'TRUNCATE', 'DELETE', 'UPDATE', 'INSERT',
    'ALTER', 'CREATE', 'GRANT', 'REVOKE'
  ]

  const upperSql = sql.toUpperCase()
  return !dangerousKeywords.some(keyword => upperSql.includes(keyword))
}
```

## 與模板系統整合

### 範例：綁定查詢到模板
```typescript
// 模板定義
const template = {
  queryId: 'latest-orders',
  queryParams: {
    userId: '{{user.lineUserId}}'
  },
  content: `
    您好 {{user.name}}，

    您最近的訂單：
    {{#each orders}}
    - 訂單編號：{{this.id}}
      金額：{{this.amount}}
      狀態：{{this.status}}
    {{/each}}
  `
}

// 執行時
// 1. 執行查詢
const orders = await executeQuery('latest-orders', {
  userId: user.lineUserId
})

// 2. 將結果注入模板
const message = renderTemplate(template.content, {
  user,
  orders
})
```

## 效能優化

- [ ] 查詢結果快取
- [ ] 慢查詢警告
- [ ] 索引建議
- [ ] EXPLAIN 分析
- [ ] 分頁查詢（避免一次載入大量資料）
- [ ] 連線池管理

## 技術需求
- SQL 解析器（node-sql-parser）
- 語法高亮（Prism.js, Shiki）
- 虛擬滾動（大量結果）
- Cloudflare D1 API
- 參數驗證（Zod）

## 相依性
- Story 02（模板系統 - 查詢綁定）
- Story 04（前後端架構）

## 優先級
高（核心功能之一）

## 預估工時
6-7 天
