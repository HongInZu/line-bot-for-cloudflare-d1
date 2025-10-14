# Story 05: 使用者認證與授權

## 描述
實作使用者認證與授權系統，確保只有經過認證的使用者才能訪問系統，並根據角色控制不同的權限。

## 使用者故事
作為系統管理員，我希望有一個安全的登入系統，以便控制誰可以訪問系統，並根據不同角色授予不同的權限。

## 驗收標準

### 前端（Nuxt.js）
- [ ] 登入頁面
- [ ] 註冊頁面（如需要）
- [ ] 登出功能
- [ ] Auth middleware 保護需要認證的路由
- [ ] 儲存和管理認證 token（cookie/localStorage）
- [ ] 自動刷新 token 機制
- [ ] 登入狀態持久化
- [ ] 未認證自動導向登入頁
- [ ] 登入成功後導向原目標頁面

### 後端（Cloudflare Workers）
- [ ] 使用者註冊 API
- [ ] 登入 API（回傳 JWT token）
- [ ] Token 驗證 middleware
- [ ] Refresh token 機制
- [ ] 密碼加密儲存（bcrypt/argon2）
- [ ] 角色權限檢查 middleware
- [ ] 登出/撤銷 token 機制
- [ ] 密碼重設功能（可選）

### 安全性
- [ ] HTTPS only
- [ ] CSRF 保護
- [ ] Rate limiting（防暴力破解）
- [ ] 密碼強度驗證
- [ ] XSS 防護
- [ ] 敏感操作需重新驗證

## API 端點
```
POST   /api/auth/register      # 註冊新使用者
POST   /api/auth/login         # 登入
POST   /api/auth/logout        # 登出
POST   /api/auth/refresh       # 刷新 token
GET    /api/auth/me            # 取得當前使用者資訊
POST   /api/auth/forgot-password  # 忘記密碼（可選）
POST   /api/auth/reset-password   # 重設密碼（可選）
```

## 資料模型
```typescript
// User
interface User {
  id: string
  email: string
  password: string // hashed
  name: string
  role: 'admin' | 'editor' | 'viewer'
  createdAt: string
  updatedAt: string
}

// Session/Token
interface Session {
  userId: string
  token: string
  refreshToken: string
  expiresAt: string
  createdAt: string
}
```

## 角色權限設計
| 功能 | Admin | Editor | Viewer |
|------|-------|--------|--------|
| 查看推播列表 | ✓ | ✓ | ✓ |
| 創建推播 | ✓ | ✓ | ✗ |
| 刪除推播 | ✓ | ✓ | ✗ |
| 編輯模板 | ✓ | ✓ | ✗ |
| 查看分析數據 | ✓ | ✓ | ✓ |
| 使用者管理 | ✓ | ✗ | ✗ |
| 系統設定 | ✓ | ✗ | ✗ |

## Nuxt.js 實作要點

### 1. Auth Composable
```typescript
// composables/useAuth.ts
export const useAuth = () => {
  const user = useState<User | null>('user', () => null)
  const token = useCookie('auth_token')

  const login = async (email: string, password: string) => {
    // 登入邏輯
  }

  const logout = async () => {
    // 登出邏輯
  }

  const checkAuth = async () => {
    // 檢查認證狀態
  }

  return { user, login, logout, checkAuth }
}
```

### 2. Auth Middleware
```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const { user } = useAuth()

  if (!user.value) {
    return navigateTo('/login')
  }
})
```

### 3. Pinia Store (可選)
```typescript
// stores/auth.ts
export const useAuthStore = defineStore('auth', {
  state: () => ({
    user: null as User | null,
    token: null as string | null,
  }),
  actions: {
    async login(credentials) { },
    async logout() { },
  }
})
```

## 技術需求
- JWT 生成與驗證
- 密碼加密函式庫
- Cloudflare D1 儲存使用者資料
- Nuxt Auth 整合（或自建）
- Cookie/Session 管理

## 相依性
- Story 04（前後端分離架構）

## 優先級
高（需在其他功能前完成）

## 預估工時
3-4 天
