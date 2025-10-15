# Story 10: 安全性與權限控制

## 描述
建立完善的安全機制，保護系統和使用者的 D1 資料庫存取，包括 API 權限控制、資料庫查詢限制、SQL 注入防護、資料隔離等關鍵安全措施。

## 使用者故事
作為系統架構師，我希望系統具備嚴謹的安全控制，確保使用者只能存取自己的資料，且無法執行危險的 SQL 操作，以保護所有使用者的資料安全。

## 安全威脅分析

### 高風險威脅

1. **SQL 注入攻擊**
   - 使用者可能在變數或查詢中注入惡意 SQL
   - 透過模板變數執行任意 SQL 指令

2. **未授權的資料存取**
   - 使用者 A 可能存取使用者 B 的資料
   - 跨租戶資料洩露

3. **危險的 SQL 操作**
   - DROP TABLE、TRUNCATE、DELETE 等破壞性操作
   - 修改資料結構（ALTER TABLE）
   - 授權管理（GRANT/REVOKE）

4. **資源濫用**
   - 執行大量或複雜查詢導致系統負載過高
   - 無限制的資料查詢（沒有 LIMIT）

5. **敏感資料洩露**
   - 查詢系統表或敏感資料
   - 推播歷史中可能包含個人資料

## 驗收標準

### 1. 身份驗證與授權

#### API 層級權限
- [ ] 所有 API 端點都需要 JWT 驗證
- [ ] JWT token 包含使用者 ID 和角色
- [ ] Token 過期時間設定（例如 24 小時）
- [ ] Refresh token 機制
- [ ] API rate limiting（防止暴力破解）

#### 角色權限矩陣
```typescript
interface Permission {
  resource: string
  actions: ('create' | 'read' | 'update' | 'delete' | 'execute')[]
}

const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  admin: [
    { resource: 'templates', actions: ['create', 'read', 'update', 'delete'] },
    { resource: 'broadcasts', actions: ['create', 'read', 'update', 'delete'] },
    { resource: 'queries', actions: ['create', 'read', 'update', 'delete', 'execute'] },
    { resource: 'recipients', actions: ['create', 'read', 'update', 'delete'] },
    { resource: 'users', actions: ['create', 'read', 'update', 'delete'] },
  ],
  editor: [
    { resource: 'templates', actions: ['create', 'read', 'update'] },
    { resource: 'broadcasts', actions: ['create', 'read'] },
    { resource: 'queries', actions: ['create', 'read', 'execute'] },
    { resource: 'recipients', actions: ['read'] },
  ],
  viewer: [
    { resource: 'templates', actions: ['read'] },
    { resource: 'broadcasts', actions: ['read'] },
    { resource: 'queries', actions: ['read'] },
    { resource: 'recipients', actions: ['read'] },
  ]
}
```

- [ ] 實作角色權限檢查 middleware
- [ ] API 層級的權限驗證
- [ ] 資料層級的權限過濾（Row-Level Security）

### 2. D1 資料庫存取安全

#### 資料隔離
- [ ] 每個組織/租戶使用獨立的 D1 資料庫
  - 或在同一資料庫使用 `tenant_id` 欄位隔離
- [ ] 所有查詢自動加上租戶過濾條件
- [ ] 防止跨租戶資料存取

#### SQL 查詢白名單機制
```typescript
// 允許的 SQL 操作
const ALLOWED_SQL_OPERATIONS = [
  'SELECT'
]

// 禁止的 SQL 關鍵字
const BLOCKED_KEYWORDS = [
  'DROP', 'TRUNCATE', 'DELETE', 'UPDATE', 'INSERT',
  'ALTER', 'CREATE', 'GRANT', 'REVOKE',
  'EXEC', 'EXECUTE', 'PRAGMA',
  '--', '/*', '*/', ';--'
]

// 禁止存取的系統表
const BLOCKED_TABLES = [
  'sqlite_master',
  'sqlite_sequence',
  'sqlite_stat1',
  'information_schema'
]
```

- [ ] 實作 SQL 查詢驗證器
- [ ] 只允許 SELECT 查詢
- [ ] 阻擋危險的 SQL 關鍵字
- [ ] 禁止存取系統表
- [ ] 禁止多語句查詢（防止 SQL 注入）

#### 參數化查詢
```typescript
// ❌ 不安全：字串拼接
const unsafeQuery = `SELECT * FROM users WHERE id = ${userId}`

// ✅ 安全：參數化查詢
const safeQuery = db.prepare('SELECT * FROM users WHERE id = ?').bind(userId)
```

- [ ] 所有查詢必須使用參數化查詢（Prepared Statements）
- [ ] 變數值透過 bind 傳入，不使用字串拼接
- [ ] 驗證變數型別（string, number, boolean）

#### 查詢限制
- [ ] 強制所有 SELECT 查詢包含 LIMIT
  - 預設最大值：1000 筆
  - 可設定的最大值：10000 筆
- [ ] 查詢超時設定（例如 5 秒）
- [ ] 複雜度分析（JOIN 數量、子查詢深度）
- [ ] 每個使用者的查詢次數限制（rate limiting）

#### 唯讀存取
```typescript
// 使用唯讀連線
const readOnlyDb = env.DB.readonly()

// 或在 wrangler.toml 設定
[[d1_databases]]
binding = "DB_READONLY"
database_name = "customer_db"
database_id = "xxx"
read_only = true
```

- [ ] 查詢建構器使用唯讀資料庫連線
- [ ] 防止透過查詢建構器修改資料

### 3. SQL 注入防護

#### 輸入驗證
```typescript
function validateQueryInput(input: any): boolean {
  // 檢查型別
  if (typeof input !== 'string' && typeof input !== 'number' && typeof input !== 'boolean') {
    throw new Error('Invalid input type')
  }

  // 字串長度限制
  if (typeof input === 'string' && input.length > 10000) {
    throw new Error('Input too long')
  }

  // 檢查危險字元
  if (typeof input === 'string') {
    const dangerousPatterns = [
      /;\s*DROP/i,
      /;\s*DELETE/i,
      /;\s*UPDATE/i,
      /UNION\s+SELECT/i,
      /--/,
      /\/\*/,
      /\*\//,
      /xp_/i,
      /exec\s*\(/i
    ]

    for (const pattern of dangerousPatterns) {
      if (pattern.test(input)) {
        throw new Error('Dangerous pattern detected')
      }
    }
  }

  return true
}
```

- [ ] 所有使用者輸入都需驗證
- [ ] 字串長度限制
- [ ] 危險字元和模式偵測
- [ ] 型別驗證（使用 Zod）

#### 模板變數安全
```typescript
// 在渲染前驗證所有變數值
function sanitizeVariableValue(value: any): any {
  if (typeof value === 'string') {
    // HTML 編碼（防止 XSS）
    value = escapeHtml(value)

    // SQL 特殊字元處理
    value = value.replace(/['";\\]/g, '')
  }

  return value
}
```

- [ ] 變數值在渲染前進行清理
- [ ] HTML 編碼（防止 XSS）
- [ ] SQL 特殊字元過濾

### 4. 敏感資料保護

#### 資料加密
- [ ] 傳輸加密（HTTPS only）
- [ ] 敏感欄位加密儲存
  - LINE Channel Access Token
  - API keys
  - 使用者密碼（bcrypt/argon2）
- [ ] 加密金鑰管理（Cloudflare Secrets）

#### 資料遮罩
```typescript
// 在日誌和回應中遮罩敏感資料
function maskSensitiveData(data: any): any {
  const sensitive = ['password', 'token', 'secret', 'lineUserId']

  return JSON.parse(JSON.stringify(data, (key, value) => {
    if (sensitive.includes(key)) {
      return '***REDACTED***'
    }
    return value
  }))
}
```

- [ ] 日誌中不記錄敏感資料
- [ ] API 回應中遮罩敏感欄位
- [ ] 錯誤訊息不洩露系統資訊

#### 個人資料處理（GDPR/PDPA）
- [ ] 使用者同意機制
- [ ] 資料存取記錄（audit log）
- [ ] 資料匯出功能
- [ ] 資料刪除功能（right to be forgotten）

### 5. API 安全

#### CORS 設定
```typescript
app.use('*', cors({
  origin: [
    'https://yourdomain.com',
    'https://admin.yourdomain.com'
  ],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowHeaders: ['Content-Type', 'Authorization'],
  credentials: true
}))
```

- [ ] 嚴格的 CORS 設定
- [ ] 只允許特定的 origin
- [ ] 生產環境不允許 wildcard (*)

#### Rate Limiting
```typescript
// 使用 Cloudflare KV 實作 rate limiting
const rateLimiter = {
  // 每分鐘 60 次請求
  perMinute: 60,
  // 每小時 1000 次請求
  perHour: 1000,
  // 每天 10000 次請求
  perDay: 10000
}
```

- [ ] API rate limiting（基於 IP 或使用者）
- [ ] 不同端點不同的限制
- [ ] 超過限制回傳 429 Too Many Requests
- [ ] 使用 Cloudflare KV 儲存計數

#### 請求驗證
```typescript
// 使用 Zod 驗證請求
const CreateBroadcastSchema = z.object({
  templateId: z.string().uuid(),
  recipientIds: z.array(z.string()).max(10000),
  scheduledAt: z.string().datetime().optional()
})

app.post('/api/broadcasts', async (c) => {
  try {
    const body = await c.req.json()
    const validated = CreateBroadcastSchema.parse(body)
    // ...
  } catch (e) {
    return c.json({ error: 'Invalid request' }, 400)
  }
})
```

- [ ] 所有請求參數使用 Zod 驗證
- [ ] 陣列長度限制
- [ ] 檔案大小限制
- [ ] 內容型別檢查

### 6. 審計與監控

#### 操作記錄
```typescript
interface AuditLog {
  id: string
  userId: string
  action: string // 'query.execute', 'broadcast.create', etc.
  resource: string
  resourceId?: string
  ipAddress: string
  userAgent: string
  status: 'success' | 'failed'
  error?: string
  timestamp: string
  metadata?: Record<string, any>
}
```

- [ ] 記錄所有敏感操作
  - SQL 查詢執行
  - 推播發送
  - 使用者管理
  - 權限變更
- [ ] 記錄包含使用者、時間、IP、動作、結果
- [ ] 審計日誌不可修改

#### 異常偵測
- [ ] 偵測異常查詢模式
  - 短時間大量查詢
  - 失敗率突然增加
  - 查詢複雜度異常
- [ ] 自動警報通知
- [ ] 可疑活動自動封鎖

#### 監控指標
- [ ] 查詢執行時間
- [ ] API 回應時間
- [ ] 錯誤率
- [ ] 認證失敗次數
- [ ] Rate limit 觸發次數

### 7. 錯誤處理

#### 安全的錯誤訊息
```typescript
// ❌ 不安全：洩露系統資訊
throw new Error(`Query failed: SELECT * FROM ${tableName}`)

// ✅ 安全：通用錯誤訊息
if (error) {
  logger.error('Query execution failed', { userId, error })
  return { error: 'Query execution failed. Please check your query.' }
}
```

- [ ] 錯誤訊息不洩露內部資訊
- [ ] 詳細錯誤只記錄在伺服器日誌
- [ ] 使用者只看到通用錯誤訊息
- [ ] 錯誤代碼對應系統

## 技術實作

### 權限檢查 Middleware

```typescript
// middleware/auth.ts
export function requirePermission(resource: string, action: string) {
  return async (c: Context, next: Next) => {
    const user = c.get('user') // 從 JWT 取得

    if (!user) {
      return c.json({ error: 'Unauthorized' }, 401)
    }

    const hasPermission = checkPermission(user.role, resource, action)

    if (!hasPermission) {
      return c.json({ error: 'Forbidden' }, 403)
    }

    await next()
  }
}

// 使用
app.post('/api/broadcasts',
  requireAuth(),
  requirePermission('broadcasts', 'create'),
  async (c) => {
    // ...
  }
)
```

### SQL 查詢驗證器

```typescript
// services/queryValidator.ts
export class QueryValidator {
  validate(sql: string): ValidationResult {
    // 1. 檢查是否只有 SELECT
    if (!sql.trim().toUpperCase().startsWith('SELECT')) {
      return { valid: false, error: 'Only SELECT queries are allowed' }
    }

    // 2. 檢查危險關鍵字
    const upperSql = sql.toUpperCase()
    for (const keyword of BLOCKED_KEYWORDS) {
      if (upperSql.includes(keyword)) {
        return { valid: false, error: `Keyword '${keyword}' is not allowed` }
      }
    }

    // 3. 檢查是否有 LIMIT
    if (!upperSql.includes('LIMIT')) {
      return { valid: false, error: 'LIMIT clause is required' }
    }

    // 4. 解析 LIMIT 值
    const limitMatch = upperSql.match(/LIMIT\s+(\d+)/)
    if (limitMatch) {
      const limit = parseInt(limitMatch[1])
      if (limit > 10000) {
        return { valid: false, error: 'LIMIT cannot exceed 10000' }
      }
    }

    // 5. 檢查表名
    const tables = this.extractTableNames(sql)
    for (const table of tables) {
      if (BLOCKED_TABLES.includes(table.toLowerCase())) {
        return { valid: false, error: `Table '${table}' cannot be accessed` }
      }
    }

    // 6. 檢查多語句
    const statements = sql.split(';').filter(s => s.trim())
    if (statements.length > 1) {
      return { valid: false, error: 'Multiple statements are not allowed' }
    }

    return { valid: true }
  }

  extractTableNames(sql: string): string[] {
    // 簡單的表名提取（生產環境建議使用 SQL parser）
    const fromRegex = /FROM\s+([a-zA-Z0-9_]+)/gi
    const joinRegex = /JOIN\s+([a-zA-Z0-9_]+)/gi

    const tables = []
    let match

    while ((match = fromRegex.exec(sql)) !== null) {
      tables.push(match[1])
    }

    while ((match = joinRegex.exec(sql)) !== null) {
      tables.push(match[1])
    }

    return tables
  }
}
```

### 租戶隔離

```typescript
// middleware/tenantIsolation.ts
export function enforceTenantIsolation() {
  return async (c: Context, next: Next) => {
    const user = c.get('user')
    const tenantId = user.tenantId

    // 將 tenantId 注入到 context
    c.set('tenantId', tenantId)

    // 攔截所有資料庫查詢，自動加上 tenant_id 過濾
    const originalQuery = c.env.DB.prepare.bind(c.env.DB)
    c.env.DB.prepare = function(sql: string) {
      // 自動在 WHERE 子句加上 tenant_id
      const modifiedSql = injectTenantFilter(sql, tenantId)
      return originalQuery(modifiedSql)
    }

    await next()
  }
}

function injectTenantFilter(sql: string, tenantId: string): string {
  // 簡化範例，生產環境需要更完善的 SQL 解析
  if (sql.toUpperCase().includes('WHERE')) {
    return sql.replace(/WHERE/i, `WHERE tenant_id = '${tenantId}' AND`)
  } else {
    return sql.replace(/FROM\s+([a-zA-Z0-9_]+)/i,
      `FROM $1 WHERE tenant_id = '${tenantId}'`)
  }
}
```

### Rate Limiting

```typescript
// middleware/rateLimiter.ts
export function rateLimit(options: RateLimitOptions) {
  return async (c: Context, next: Next) => {
    const identifier = getIdentifier(c) // IP or userId
    const key = `ratelimit:${options.resource}:${identifier}`

    const count = await c.env.KV.get(key)
    const currentCount = count ? parseInt(count) : 0

    if (currentCount >= options.max) {
      return c.json({
        error: 'Rate limit exceeded',
        retryAfter: options.windowMs / 1000
      }, 429)
    }

    await c.env.KV.put(key, String(currentCount + 1), {
      expirationTtl: options.windowMs / 1000
    })

    await next()
  }
}

// 使用
app.post('/api/query/execute',
  requireAuth(),
  rateLimit({ resource: 'query', max: 100, windowMs: 60000 }), // 每分鐘 100 次
  async (c) => {
    // ...
  }
)
```

## 部署配置

### wrangler.toml
```toml
name = "linebot-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"

# D1 資料庫（唯讀）
[[d1_databases]]
binding = "DB_READONLY"
database_name = "customer_db"
database_id = "xxx"

# KV（用於 rate limiting 和 session）
[[kv_namespaces]]
binding = "KV"
id = "xxx"

# Secrets（不要提交到 git）
[vars]
ENVIRONMENT = "production"

# 使用 wrangler secret put 設定
# JWT_SECRET
# LINE_CHANNEL_SECRET
# ENCRYPTION_KEY
```

### 環境變數管理
```bash
# 設定 secrets
wrangler secret put JWT_SECRET
wrangler secret put LINE_CHANNEL_SECRET
wrangler secret put ENCRYPTION_KEY

# 不要在程式碼中硬編碼 secrets
```

## 測試需求

### 安全測試
- [ ] SQL 注入測試
- [ ] 權限繞過測試
- [ ] Rate limiting 測試
- [ ] 跨租戶存取測試
- [ ] XSS 防護測試
- [ ] CSRF 防護測試

### 滲透測試
- [ ] 定期進行滲透測試
- [ ] 使用自動化工具（如 OWASP ZAP）
- [ ] 第三方安全審計

## 相依性
- Story 04（前後端架構）
- Story 05（認證授權）
- Story 09（D1 查詢建構器）

## 優先級
**最高**（所有涉及 D1 存取的功能都依賴此 Story）

## 預估工時
5-7 天

## 合規要求

### GDPR（歐盟）
- [ ] 資料處理同意書
- [ ] 被遺忘權（資料刪除）
- [ ] 資料可攜權（資料匯出）
- [ ] 資料洩露通知機制

### PDPA（台灣個資法）
- [ ] 個人資料蒐集告知
- [ ] 資料使用目的說明
- [ ] 當事人權利保障

## 參考資料
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Cloudflare Workers Security](https://developers.cloudflare.com/workers/platform/security/)
- [D1 Security Best Practices](https://developers.cloudflare.com/d1/platform/security/)
