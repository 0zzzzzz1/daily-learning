## 目標技術

`JavaScript ES6+` `TypeScript`

## 核心知識

Promise、async/await 非同步程式設計 與 ESM 模組系統（ES Modules）

## 知識說明

### 為什麼今天學這個？（與前幾日的連結）

W1 四天我們打穩了 ES6+ 語法基礎：

- **Day 1（4/23）**：Vue 3 響應式原理，`ref` / `reactive` / `computed` / `watch`
- **Day 2（4/24）**：解構賦值、Spread 展開、Rest 其餘運算子
- **Day 3（4/27）**：箭頭函式 `this` 靜態綁定、Template Literal 與標籤函式

今天進入 **W2 的第一天**，主題正式轉向 **非同步程式設計**：

> **Promise**：從「callback hell」解脫出來的非同步抽象層，是所有現代非同步 API 的基礎
> **async/await**：建構在 Promise 之上的語法糖，讓非同步程式碼讀起來像同步，卻不阻塞執行緒
> **ESM 模組**：JavaScript 官方的模組系統，讓程式碼可以被拆分、重用、靜態分析

在 Vue 3 / Vite 專案中，這三個概念無所不在——API 呼叫、資料載入、元件懶加載，全都依賴非同步與模組系統。

---

### 一、Promise

**定義**：代表一個「尚未完成但最終會有結果」的非同步操作，有三種狀態：`pending`（等待中）→ `fulfilled`（已完成）或 `rejected`（已拒絕），狀態一旦改變便不可逆。

---

#### 1. 建立 Promise

```ts
// Promise<T> 泛型中 T 是 resolve 時傳入的型別
const fetchData = new Promise<string>((resolve, reject) => {
  setTimeout(() => {
    const success = Math.random() > 0.3
    if (success) {
      resolve('資料載入成功')   // fulfilled
    } else {
      reject(new Error('網路錯誤')) // rejected
    }
  }, 1000)
})
```

---

#### 2. .then / .catch / .finally

```ts
fetchData
  .then((data: string) => {
    console.log('成功：', data)
    return data.toUpperCase()      // 可以繼續回傳值，形成鏈式調用
  })
  .then((upperData: string) => {
    console.log('處理後：', upperData)
  })
  .catch((error: Error) => {
    console.error('失敗：', error.message)
  })
  .finally(() => {
    console.log('無論成功或失敗都會執行')
  })
```

**鏈式調用的注意事項：**

```ts
// ✅ 正確：每個 .then 回傳新的 Promise，形成鏈
fetchUser(1)
  .then(user => fetchOrders(user.id))   // 回傳 Promise
  .then(orders => console.log(orders))
  .catch(err => console.error(err))

// ❌ 錯誤：Promise 嵌套（Promise Hell），可讀性差
fetchUser(1).then(user => {
  fetchOrders(user.id).then(orders => {  // 嵌套！
    console.log(orders)
  })
})
```

---

#### 3. Promise 並發控制

```ts
// Promise.all：全部成功才繼續，任一失敗立即 reject
const [user, orders, config] = await Promise.all([
  fetchUser(1),
  fetchOrders(1),
  fetchConfig()
])

// Promise.allSettled：等全部結束，無論成功或失敗，不拋出錯誤
const results = await Promise.allSettled([
  fetchUser(1),
  fetchOrders(1),
  fetchConfig()
])
results.forEach(result => {
  if (result.status === 'fulfilled') {
    console.log('成功：', result.value)
  } else {
    console.error('失敗：', result.reason)
  }
})

// Promise.race：取第一個完成（或失敗）的結果
const result = await Promise.race([
  fetchData(),
  timeout(5000)   // 5 秒超時
])

// Promise.any：取第一個成功的結果（全部失敗才 reject）
const fastest = await Promise.any([
  fetchFromServer1(),
  fetchFromServer2(),
  fetchFromServer3()
])
```

---

#### 4. Promise 靜態方法

```ts
// Promise.resolve：立即建立已完成的 Promise
const resolved = Promise.resolve<number>(42)
// 等價於 new Promise<number>(resolve => resolve(42))

// Promise.reject：立即建立已拒絕的 Promise
const rejected = Promise.reject<never>(new Error('直接失敗'))

// 常見用途：標準化回傳值
async function getCache(key: string): Promise<string> {
  const cached = localStorage.getItem(key)
  if (cached) return Promise.resolve(cached)  // 快速回傳快取
  return fetchFromServer(key)                 // 從伺服器取得
}
```

---

### 二、async/await

**定義**：ES2017 引入的語法糖，讓 Promise 的使用更直覺。`async` 函式總是回傳 Promise，`await` 暫停函式執行直到 Promise 解決。

---

#### 1. 基本語法

```ts
// async 函式的回傳型別總是 Promise<T>
async function fetchUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`)
  // await 暫停此函式（但不阻塞主執行緒）
  // 等到 fetch 完成後繼續
  if (!response.ok) {
    throw new Error(`HTTP Error: ${response.status}`)
  }
  const user: User = await response.json()
  return user  // 自動被包裝成 Promise.resolve(user)
}

// 箭頭函式 async 版本
const fetchUser = async (id: number): Promise<User> => {
  const response = await fetch(`/api/users/${id}`)
  if (!response.ok) throw new Error(`HTTP Error: ${response.status}`)
  return response.json() as Promise<User>
}
```

---

#### 2. 錯誤處理

```ts
// 方法一：try/catch（推薦，語意清晰）
async function loadUserData(id: number): Promise<void> {
  try {
    const user = await fetchUser(id)
    const orders = await fetchOrders(id)
    console.log('使用者：', user)
    console.log('訂單：', orders)
  } catch (error) {
    if (error instanceof Error) {
      console.error('載入失敗：', error.message)
    }
  } finally {
    console.log('載入流程結束')
  }
}

// 方法二：Promise .catch（適合單行處理）
const user = await fetchUser(1).catch(() => null)
if (!user) return  // 提前返回

// 方法三：封裝成 Result 型別（進階，避免 try/catch 散落各處）
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E }

const tryCatch = async <T>(
  promise: Promise<T>
): Promise<Result<T>> => {
  try {
    const value = await promise
    return { ok: true, value }
  } catch (error) {
    return { ok: false, error: error as Error }
  }
}

// 使用
const result = await tryCatch(fetchUser(1))
if (!result.ok) {
  console.error(result.error.message)
  return
}
console.log(result.value.name)
```

---

#### 3. 常見陷阱與最佳實踐

```ts
// ❌ 陷阱一：在迴圈中串行 await（效能差）
const userIds = [1, 2, 3, 4, 5]
for (const id of userIds) {
  const user = await fetchUser(id)  // 等前一個完成才開始下一個
  console.log(user)
}

// ✅ 改善：使用 Promise.all 並行請求
const users = await Promise.all(
  userIds.map(id => fetchUser(id))
)

// ❌ 陷阱二：forEach 中的 await 不如預期
const results: User[] = []
userIds.forEach(async (id) => {
  const user = await fetchUser(id)  // forEach 不等待 async callback！
  results.push(user)
})
console.log(results) // [] 空陣列！forEach 已跑完但 async 還沒結束

// ✅ 改善：使用 for...of 或 Promise.all + map
for (const id of userIds) {
  const user = await fetchUser(id)
  results.push(user)
}

// ❌ 陷阱三：忘記 await，拿到 Promise 物件而非值
const user = fetchUser(1)  // 回傳 Promise<User>，不是 User
console.log(user.name)     // undefined！user 是 Promise 物件

// ✅ 正確
const user = await fetchUser(1)  // 等待 Promise 解決
console.log(user.name)           // 正確的值
```

---

#### 4. 在 Vue 3 中的應用

```ts
import { ref, onMounted } from 'vue'

interface Post {
  id: number
  title: string
  body: string
}

const usePosts = () => {
  const posts = ref<Post[]>([])
  const isLoading = ref<boolean>(false)
  const error = ref<string | null>(null)

  const fetchPosts = async (): Promise<void> => {
    isLoading.value = true
    error.value = null
    try {
      const response = await fetch('https://jsonplaceholder.typicode.com/posts')
      if (!response.ok) throw new Error('Failed to fetch')
      posts.value = await response.json() as Post[]
    } catch (e) {
      error.value = e instanceof Error ? e.message : '未知錯誤'
    } finally {
      isLoading.value = false
    }
  }

  onMounted(fetchPosts)

  return { posts, isLoading, error, fetchPosts }
}
```

---

### 三、ESM 模組系統（ES Modules）

**定義**：JavaScript 官方的模組規範（ES2015+），使用 `import` / `export` 語法，支援靜態分析、Tree Shaking，是現代前端開發（Vite、Vue 3）的基礎。

---

#### 1. 具名匯出（Named Export）

```ts
// utils/math.ts
export const PI = 3.14159

export const add = (a: number, b: number): number => a + b

export const multiply = (a: number, b: number): number => a * b

export interface MathConfig {
  precision: number
  roundingMode: 'floor' | 'ceil' | 'round'
}

// 也可以在底部集中匯出
const subtract = (a: number, b: number): number => a - b
const divide = (a: number, b: number): number => {
  if (b === 0) throw new Error('除以零')
  return a / b
}
export { subtract, divide }
```

```ts
// 引入具名匯出
import { add, multiply, PI } from './utils/math'
import { subtract, divide } from './utils/math'

// 重命名匯入（避免命名衝突）
import { add as mathAdd } from './utils/math'

// 全部匯入為命名空間
import * as MathUtils from './utils/math'
MathUtils.add(1, 2)
MathUtils.PI
```

---

#### 2. 預設匯出（Default Export）

```ts
// services/apiClient.ts
class ApiClient {
  private baseUrl: string

  constructor(baseUrl: string) {
    this.baseUrl = baseUrl
  }

  async get<T>(path: string): Promise<T> {
    const response = await fetch(`${this.baseUrl}${path}`)
    if (!response.ok) throw new Error(`API Error: ${response.status}`)
    return response.json() as Promise<T>
  }

  async post<T, B = unknown>(path: string, body: B): Promise<T> {
    const response = await fetch(`${this.baseUrl}${path}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body)
    })
    if (!response.ok) throw new Error(`API Error: ${response.status}`)
    return response.json() as Promise<T>
  }
}

export default new ApiClient('https://api.example.com')
```

```ts
// 引入預設匯出（可自訂名稱）
import apiClient from './services/apiClient'
import myApi from './services/apiClient'  // 同一個物件，可改名

// 同時引入預設匯出與具名匯出
import ApiClient, { type ApiConfig } from './services/apiClient'
```

---

#### 3. Re-export（轉發匯出）

```ts
// utils/index.ts — 集中匯出，建立統一入口
export { add, multiply, PI } from './math'
export { formatDate, parseDate } from './date'
export { formatCurrency } from './currency'
export type { MathConfig } from './math'

// 重命名後再匯出
export { subtract as mathSubtract } from './math'

// 預設匯出的轉發
export { default as apiClient } from '../services/apiClient'
```

```ts
// 使用者只需從統一入口引入
import { add, formatDate, apiClient } from '../utils'
// 不需要知道每個函式分別在哪個檔案
```

---

#### 4. 動態匯入（Dynamic Import）

```ts
// 靜態匯入：在模組頂部，會在解析時載入
import { heavyUtil } from './utils/heavy'

// 動態匯入：回傳 Promise，按需載入（程式碼分割）
const loadHeavyUtil = async () => {
  const { heavyUtil } = await import('./utils/heavy')
  return heavyUtil
}

// Vue 3 中的路由懶加載（最常見的應用）
const routes = [
  {
    path: '/dashboard',
    component: () => import('./views/DashboardView.vue')  // 動態匯入
  },
  {
    path: '/settings',
    component: () => import('./views/SettingsView.vue')
  }
]

// 條件式載入（根據功能需求）
async function initAnalytics(): Promise<void> {
  if (process.env.NODE_ENV === 'production') {
    const { Analytics } = await import('./services/analytics')
    Analytics.init()
  }
}
```

---

#### 5. ESM vs CommonJS（Node.js 差異）

| 特性 | ESM (`import/export`) | CommonJS (`require/module.exports`) |
|------|----------------------|-------------------------------------|
| 語法 | `import { x } from 'y'` | `const x = require('y').x` |
| 執行時機 | **靜態**：編譯期解析 | **動態**：執行期解析 |
| Tree Shaking | ✅ 支援（打包工具可移除未使用程式碼） | ❌ 不支援 |
| 頂層 `await` | ✅ 支援（ESM 模組中） | ❌ 不支援 |
| 循環依賴 | 有限支援（Live Binding） | 支援（但可能拿到不完整物件） |
| 使用場景 | 瀏覽器、現代 Node.js、Vite | 傳統 Node.js 專案、舊版套件 |

---

#### 6. Vue 3 / Vite 專案中的 ESM 應用

```ts
// composables/useApi.ts — 使用具名匯出的 Composable
export interface ApiState<T> {
  data: T | null
  isLoading: boolean
  error: string | null
}

export const useApi = <T>(url: string) => {
  const state = reactive<ApiState<T>>({
    data: null,
    isLoading: false,
    error: null
  })

  const execute = async (): Promise<void> => {
    state.isLoading = true
    state.error = null
    try {
      const response = await fetch(url)
      state.data = await response.json() as T
    } catch (e) {
      state.error = e instanceof Error ? e.message : '載入失敗'
    } finally {
      state.isLoading = false
    }
  }

  return { ...toRefs(state), execute }
}
```

```ts
// 在元件中引入使用
import { useApi } from '@/composables/useApi'
import type { Post } from '@/types/post'

const { data: posts, isLoading, error, execute } = useApi<Post[]>('/api/posts')
```

---

### 四、三者整合：Promise + async/await + ESM

在真實 Vue 3 專案中，三者密不可分：

```ts
// services/propertyService.ts — 封裝 API 呼叫
import type { Property, PropertyFilters } from '@/types/property'
import { apiClient } from '@/utils/http'

export const getProperties = async (
  filters?: PropertyFilters
): Promise<Property[]> => {
  const params = new URLSearchParams(filters as Record<string, string>)
  return apiClient.get<Property[]>(`/properties?${params}`)
}

export const getPropertyById = async (id: number): Promise<Property> => {
  return apiClient.get<Property>(`/properties/${id}`)
}

export const createProperty = async (
  data: Omit<Property, 'id' | 'createdAt'>
): Promise<Property> => {
  return apiClient.post<Property>('/properties', data)
}
```

```ts
// composables/useProperties.ts — 組合使用
import { ref } from 'vue'
import {
  getProperties,
  getPropertyById,
  createProperty
} from '@/services/propertyService'
import type { Property, PropertyFilters } from '@/types/property'

export const useProperties = () => {
  const properties = ref<Property[]>([])
  const currentProperty = ref<Property | null>(null)
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  const fetchAll = async (filters?: PropertyFilters): Promise<void> => {
    isLoading.value = true
    error.value = null
    try {
      properties.value = await getProperties(filters)
    } catch (e) {
      error.value = e instanceof Error ? e.message : '載入失敗'
    } finally {
      isLoading.value = false
    }
  }

  const fetchOne = async (id: number): Promise<void> => {
    isLoading.value = true
    try {
      currentProperty.value = await getPropertyById(id)
    } catch (e) {
      error.value = e instanceof Error ? e.message : '載入失敗'
    } finally {
      isLoading.value = false
    }
  }

  return {
    properties,
    currentProperty,
    isLoading,
    error,
    fetchAll,
    fetchOne
  }
}
```

---

## 作業說明

### 作業主題：實作「非同步資料流封裝模組」

使用 **Promise + async/await + ESM 模組系統**（加上 TypeScript 型別定義），實作一組可在 Vue 3 專案中直接使用的非同步資料流封裝工具。

---

### 情境背景

ihouseBMS 前端需要與後端 API 溝通，目前的問題是：
1. API 呼叫散落在各元件中，沒有統一封裝
2. 錯誤處理不一致，有些地方沒有 catch
3. 載入狀態難以管理，使用者體驗差
4. 重複請求沒有防護機制

你的任務是設計一套模組化的非同步資料流封裝，解決上述問題。

```ts
// 基礎型別定義
interface Property {
  id: number
  title: string
  address: string
  price: number
  area: number
  type: 'apartment' | 'house' | 'studio'
  isAvailable: boolean
}

interface ApiResponse<T> {
  data: T
  message: string
  statusCode: number
}

interface PaginatedResponse<T> {
  items: T[]
  total: number
  page: number
  pageSize: number
}
```

---

### 功能需求

#### 1. `createHttpClient(baseUrl)` — 建立 HTTP 客戶端模組

要求：
- 使用**具名匯出**
- 封裝 `get`、`post`、`put`、`delete` 方法，每個方法皆使用 `async/await`
- 自動加入 `Content-Type: application/json` header
- 請求失敗時（非 2xx 狀態碼）拋出包含狀態碼與訊息的錯誤

```ts
const client = createHttpClient('https://api.ihouse.com')

// get 方法
const property = await client.get<Property>('/properties/1')

// post 方法
const newProperty = await client.post<Property>('/properties', {
  title: '台北信義三房',
  price: 25000000
})
```

---

#### 2. `withRetry(fn, options?)` — 請求重試裝飾器

要求：
- 使用**具名匯出**的高階函式
- 接受任何回傳 Promise 的函式
- 支援設定最大重試次數（預設 3 次）與重試間隔（預設 1000ms）
- 使用**遞迴或迴圈配合 Promise**實作

```ts
const fetchWithRetry = withRetry(
  () => client.get<Property>('/properties/1'),
  { maxRetries: 3, delay: 1000 }
)

const property = await fetchWithRetry()
```

---

#### 3. `withCache(fn, ttl?)` — 請求快取裝飾器

要求：
- 使用**具名匯出**
- 對相同參數的呼叫快取結果，TTL（存活時間）預設 5 分鐘
- 快取使用 `Map<string, { data: T; expiredAt: number }>` 儲存
- 使用 `JSON.stringify(args)` 作為快取 key

```ts
const cachedFetch = withCache(
  (id: number) => client.get<Property>(`/properties/${id}`),
  5 * 60 * 1000  // 5 分鐘
)

const p1 = await cachedFetch(1)  // 發出請求
const p2 = await cachedFetch(1)  // 從快取回傳，不發請求
const p3 = await cachedFetch(2)  // 不同參數，發出新請求
```

---

#### 4. `fetchConcurrent(requests, limit?)` — 並行請求控制器

要求：
- 使用**具名匯出**
- 同時執行多個請求，但控制最大並行數量（預設 3）
- 保持結果順序與輸入順序一致
- 任一請求失敗時整體 reject

```ts
const propertyIds = [1, 2, 3, 4, 5, 6, 7, 8]
const properties = await fetchConcurrent(
  propertyIds.map(id => () => client.get<Property>(`/properties/${id}`)),
  3  // 同時最多 3 個請求
)
// properties 陣列順序對應 propertyIds 順序
```

---

#### 5. `useAsyncState(fn)` — 非同步狀態管理（Vue 3 Composable）

要求：
- 使用**具名匯出**
- 包裝任何非同步函式，自動管理 `data`、`isLoading`、`error` 狀態
- 使用 Vue 3 `ref` 管理響應式狀態
- 回傳 `execute` 函式可手動觸發

```ts
// 使用範例
const { data, isLoading, error, execute } = useAsyncState(
  () => client.get<Property[]>('/properties')
)

// 在 onMounted 中執行
onMounted(execute)

// 模板中使用
// v-if="isLoading" → 顯示 loading
// v-if="error" → 顯示錯誤
// v-for="item in data" → 顯示資料
```

---

#### 6. 模組入口（`index.ts`）

要求：
- 建立 `index.ts` 作為統一出口
- 將以上所有工具函式集中 re-export
- 讓使用者只需從一個路徑引入所有工具

```ts
// 使用者可以這樣引入
import { createHttpClient, withRetry, withCache, fetchConcurrent, useAsyncState } from './async-utils'
```

---

### 技術規範

| 項目 | 要求 |
|------|------|
| 語法 | 全部使用 `async/await`，禁止直接操作 `.then().catch()` 鏈（除非有充分理由） |
| ESM | 所有模組使用具名匯出（`export const`），統一透過 `index.ts` 集中匯出 |
| TypeScript | 所有函式需完整泛型與型別標注，**禁止使用 `any`** |
| 錯誤處理 | 每個非同步函式必須有完整的 `try/catch` 或透過上層統一處理 |
| 純函式 | `withRetry` 和 `withCache` 不得修改傳入的函式 |
| 測試 | 附上對每個模組的手動測試案例（可使用 mock fetch），每個功能至少 2 個案例 |

---

### 評分標準

| 項目 | 分數 |
|------|------|
| `createHttpClient` 完整封裝 4 種方法，錯誤正確拋出 | 20 分 |
| `withRetry` 重試邏輯正確，間隔符合預期 | 15 分 |
| `withCache` 快取命中/失效邏輯正確 | 15 分 |
| `fetchConcurrent` 並行限制與順序保持正確 | 20 分 |
| `useAsyncState` 三個狀態正確響應式更新 | 15 分 |
| `index.ts` 集中匯出結構正確 | 5 分 |
| TypeScript 型別完整，無 `any` | 5 分 |
| 測試案例覆蓋正常流程與錯誤流程 | 5 分 |
| **總計** | **100 分** |

---

### 延伸思考題（選做，不計分）

1. `withCache` 在多個元件同時發出相同請求時，如何避免「快取穿透」（多個請求同時 miss cache 導致重複請求）？提示：研究「Request Deduplication」
2. `fetchConcurrent` 的並行控制若需要支援「失敗繼續」（不因單一失敗中止整批），應如何修改？
3. Vue 3 的 `watchEffect` 和 `watch` 中能直接 `await` 嗎？如果不行，應該怎麼處理非同步監聽？
4. ESM 的「頂層 await」（Top-Level Await）是什麼？在什麼情境下有用？有什麼風險？

---

### 提交方式

在 `daily-learning` 專案中，於 `2026-04-28/` 目錄下建立以下結構，並發 PR 請求審核：

```
2026-04-28/
├── async-utils/
│   ├── httpClient.ts      # createHttpClient
│   ├── retry.ts           # withRetry
│   ├── cache.ts           # withCache
│   ├── concurrent.ts      # fetchConcurrent
│   ├── useAsyncState.ts   # Vue 3 Composable
│   └── index.ts           # 統一匯出
└── 作業實作/
    └── test.ts            # 手動測試案例
```
