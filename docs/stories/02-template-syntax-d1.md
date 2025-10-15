# Story 02: 模板引擎與動態資料渲染

## 描述
實作模板引擎系統，支援變數替換、條件判斷和迴圈語法，讓系統能在推播時動態填入從 D1 資料庫查詢的資料或收件人資訊。

## 使用者故事
作為系統開發者，我希望有一個強大的模板引擎，能在推播執行時動態填入資料，支援條件邏輯和迴圈處理，以便實現各種複雜的訊息場景。

## 核心功能

這個 Story 專注於**後端模板引擎的實作**，與 Story 03（訊息預覽與變數綁定）搭配：
- **Story 03**：前端介面，使用者設定變數和資料來源
- **Story 02**：後端引擎，在推播時執行資料填入

## 驗收標準

### 模板語法支援
- [ ] 變數替換：`{{variable}}`
- [ ] 巢狀物件：`{{user.profile.name}}`
- [ ] 陣列存取：`{{items[0].name}}`
- [ ] 條件判斷：
  ```handlebars
  {{#if condition}}
    內容
  {{else}}
    其他內容
  {{/if}}
  ```
- [ ] 迴圈處理：
  ```handlebars
  {{#each items}}
    - {{this.name}}: {{this.price}}
  {{/each}}
  ```
- [ ] 比較運算子：`==`, `!=`, `>`, `<`, `>=`, `<=`
- [ ] 邏輯運算子：`&&`, `||`, `!`
- [ ] 輔助函數：
  - 格式化：`{{format date 'YYYY-MM-DD'}}`
  - 數字：`{{formatNumber amount}}`
  - 字串：`{{uppercase name}}`

### 資料來源整合
- [ ] 從 D1 查詢結果填入資料
- [ ] 從收件人資料填入
- [ ] 支援多資料來源合併
- [ ] 處理資料不存在的情況（預設值、錯誤處理）

### 渲染引擎
- [ ] 解析模板 JSON（包含 Flex Message）
- [ ] 遞迴處理巢狀結構
- [ ] 填入變數值
- [ ] 執行條件邏輯
- [ ] 執行迴圈展開
- [ ] 輸出最終 JSON

### 錯誤處理
- [ ] 變數不存在時的處理策略
  - 保留原始變數語法
  - 使用預設值
  - 拋出錯誤
- [ ] 循環引用檢測
- [ ] 語法錯誤提示
- [ ] 型別不匹配處理

### 效能優化
- [ ] 模板快取（避免重複解析）
- [ ] 變數查找優化
- [ ] 批次渲染支援（多個收件人）

## 模板語法範例

### 範例 1：基本變數替換
```json
{
  "type": "text",
  "text": "您好 {{user.name}}，您的會員等級是 {{user.memberLevel}}"
}
```

填入資料：
```json
{
  "user": {
    "name": "張三",
    "memberLevel": "VIP"
  }
}
```

結果：
```json
{
  "type": "text",
  "text": "您好 張三，您的會員等級是 VIP"
}
```

### 範例 2：條件判斷
```json
{
  "type": "flex",
  "contents": {
    "type": "bubble",
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "訂單狀態：{{order.status}}"
        },
        "{{#if order.status == 'shipped'}}",
        {
          "type": "text",
          "text": "追蹤碼：{{order.trackingNumber}}",
          "color": "#00AA00"
        },
        "{{/if}}"
      ]
    }
  }
}
```

### 範例 3：迴圈處理
```json
{
  "type": "flex",
  "contents": {
    "type": "bubble",
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "購物清單",
          "weight": "bold"
        },
        "{{#each items}}",
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "{{this.name}}"
            },
            {
              "type": "text",
              "text": "NT$ {{this.price}}"
            }
          ]
        },
        "{{/each}}",
        {
          "type": "text",
          "text": "總計：NT$ {{total}}"
        }
      ]
    }
  }
}
```

## API 端點
```
POST   /api/templates/render     # 渲染模板（主要 API）
POST   /api/templates/validate   # 驗證模板語法
POST   /api/templates/test       # 測試渲染（開發用）
```

### 渲染 API 請求範例
```typescript
POST /api/templates/render
{
  "templateJson": { /* 模板 JSON */ },
  "data": {
    "user": { "name": "張三" },
    "order": { "id": "12345" }
  },
  "options": {
    "strict": false,  // 嚴格模式（變數不存在時拋錯）
    "removeEmpty": true  // 移除空值元素
  }
}
```

### 渲染 API 回應範例
```typescript
{
  "success": true,
  "rendered": { /* 渲染後的 JSON */ },
  "warnings": [
    "變數 'user.email' 未定義"
  ],
  "stats": {
    "variablesUsed": 5,
    "renderTime": 12  // ms
  }
}
```

## 技術實作

### 後端渲染引擎
```typescript
// backend/src/services/templateEngine.ts
export class TemplateEngine {
  // 主要渲染方法
  render(template: any, data: Record<string, any>, options?: RenderOptions): any {
    const context = new RenderContext(data, options)
    return this.renderNode(template, context)
  }

  private renderNode(node: any, context: RenderContext): any {
    if (typeof node === 'string') {
      return this.renderString(node, context)
    }

    if (Array.isArray(node)) {
      return this.renderArray(node, context)
    }

    if (node && typeof node === 'object') {
      return this.renderObject(node, context)
    }

    return node
  }

  private renderString(str: string, context: RenderContext): string {
    // 處理變數替換
    return str.replace(/\{\{([^}]+)\}\}/g, (match, expression) => {
      const value = context.evaluate(expression.trim())
      return value !== undefined ? String(value) : match
    })
  }

  private renderArray(arr: any[], context: RenderContext): any[] {
    const result: any[] = []

    for (let i = 0; i < arr.length; i++) {
      const item = arr[i]

      // 檢查是否為控制語法
      if (typeof item === 'string') {
        // {{#if}} 語法
        if (item.match(/^\{\{#if\s+(.+)\}\}$/)) {
          const condition = RegExp.$1
          const trueBlock: any[] = []
          const falseBlock: any[] = []
          let currentBlock = trueBlock
          let depth = 1

          // 找到對應的 {{/if}}
          for (i++; i < arr.length && depth > 0; i++) {
            const next = arr[i]
            if (typeof next === 'string') {
              if (next.match(/^\{\{#if/)) depth++
              else if (next === '{{/if}}') {
                depth--
                if (depth === 0) break
              } else if (next === '{{else}}' && depth === 1) {
                currentBlock = falseBlock
                continue
              }
            }
            currentBlock.push(next)
          }

          // 評估條件
          if (context.evaluate(condition)) {
            result.push(...this.renderArray(trueBlock, context))
          } else {
            result.push(...this.renderArray(falseBlock, context))
          }
          continue
        }

        // {{#each}} 語法
        if (item.match(/^\{\{#each\s+(.+)\}\}$/)) {
          const arrayPath = RegExp.$1
          const loopBody: any[] = []
          let depth = 1

          // 找到對應的 {{/each}}
          for (i++; i < arr.length && depth > 0; i++) {
            const next = arr[i]
            if (typeof next === 'string') {
              if (next.match(/^\{\{#each/)) depth++
              else if (next === '{{/each}}') {
                depth--
                if (depth === 0) break
              }
            }
            loopBody.push(next)
          }

          // 取得陣列資料
          const arrayData = context.get(arrayPath)
          if (Array.isArray(arrayData)) {
            arrayData.forEach((itemData, index) => {
              const itemContext = context.createChild({
                this: itemData,
                '@index': index,
                '@first': index === 0,
                '@last': index === arrayData.length - 1
              })
              result.push(...this.renderArray(loopBody, itemContext))
            })
          }
          continue
        }
      }

      // 一般元素
      result.push(this.renderNode(item, context))
    }

    return result
  }

  private renderObject(obj: any, context: RenderContext): any {
    const result: any = {}
    for (const [key, value] of Object.entries(obj)) {
      result[key] = this.renderNode(value, context)
    }
    return result
  }
}

// 渲染上下文
class RenderContext {
  constructor(
    private data: Record<string, any>,
    private options?: RenderOptions,
    private parent?: RenderContext
  ) {}

  get(path: string): any {
    // 從當前上下文或父上下文取值
    const value = this.getFromPath(this.data, path)
    if (value !== undefined) return value
    if (this.parent) return this.parent.get(path)
    return undefined
  }

  evaluate(expression: string): any {
    // 簡單的表達式評估
    // 支援比較運算子、邏輯運算子等
    // 可以使用 eval (需注意安全性) 或自行實作解析器
  }

  createChild(data: Record<string, any>): RenderContext {
    return new RenderContext(data, this.options, this)
  }
}
```

### 輔助函數
```typescript
// 註冊自訂輔助函數
templateEngine.registerHelper('formatDate', (date: string, format: string) => {
  // 日期格式化邏輯
})

templateEngine.registerHelper('formatNumber', (num: number) => {
  return new Intl.NumberFormat('zh-TW').format(num)
})

// 在模板中使用
"{{formatDate order.createdAt 'YYYY/MM/DD'}}"
"{{formatNumber order.total}}"
```

## 技術需求
- 模板解析引擎（可考慮使用 Handlebars.js 或自行實作）
- 表達式評估器
- JSON 深度遍歷
- 效能優化（快取、批次處理）

## 相依性
- Story 03（訊息預覽與變數綁定 - 提供模板和資料來源設定）
- Story 09（D1 查詢建構器 - SQL 資料來源）

## 優先級
高

## 預估工時
4-5 天

## 安全考量
- 避免在模板中執行任意程式碼
- 限制模板複雜度（防止 DoS）
- 變數注入驗證
