# useLocalStorage<T> — 可複用持久化 Composable

> **日期**：2026-06-03（M2-W7 Day 2）
> **週次主題**：Composable 設計（抽取邏輯、可複用 Hook）
> **截止日**：2026-06-08（W7）

---

## 目標技術與核心知識點

| 技術 | 核心知識點 |
|------|-----------|
| Vue 3 Composable 設計 | 泛型 Composable API 設計、watch 寫入回路、onMounted 讀取 |
| TypeScript 泛型 | `useLocalStorage<T>`、型別守衛（`parseJSON<T>`）、泛型預設值設計 |
| LocalStorage 安全讀寫 | JSON.parse 三個風險點、SSR 安全性、型別保障策略 |

---

## 為什麼學這個（與前幾日的連結）

```
W3：useTasks.ts 中的持久化邏輯
    └── watch(tasks, () => localStorage.setItem(...))
    └── onMounted(() => { JSON.parse(...) })
    → 這段邏輯「寫死在 useTasks 裡面」，只能給任務清單用

W7 Day 1：useAsyncData<T>
    → 把「三態管理 + AbortController」從每個元件中抽出，成為可複用 Hook

W7 Day 2（今日）：useLocalStorage<T>
    → 把「持久化讀寫回路」從 useTasks 中抽出，成為任何資料都能用的通用 Composable
```

W3 的 `useTasks` 已經包含 LocalStorage 持久化，但它是特定業務邏輯（任務）的一部分。  
今日目標：**把「持久化」這件事獨立出來**，設計成泛型 `useLocalStorage<T>`，讓任何需要持久化的資料都能用。

---

## 知識說明

### 一、W3 持久化的問題——「邏輯鎖在業務 Composable 裡」

W3 的 `useTasks.ts` 持久化寫法：

```ts
// useTasks.ts（W3 版本）
const tasks = ref<Task[]>([])

onMounted(() => {
  const saved = localStorage.getItem('tasks')
  if (saved) {
    try {
      tasks.value = JSON.parse(saved) as Task[]  // ← as T 是承諾非保證！
    } catch {
      tasks.value = []
    }
  }
})

watch(tasks, (val) => {
  localStorage.setItem('tasks', JSON.stringify(val))
}, { deep: true })
```

**問題**：
1. 這段邏輯在 `useUserPreferences`、`useCart`、`useFormDraft` 中幾乎一模一樣
2. `as Task[]` 是不安全的型別斷言（JSON.parse 回傳 unknown，不是 Task[]）
3. `onMounted` 在 Composable 內部的 SSR 安全性未考慮

---

### 二、useLocalStorage<T> API 設計（使用端視角優先）

設計前先問：**使用端想怎麼用？**

```ts
// 理想的使用端體驗
const tasks = useLocalStorage<Task[]>('tasks', [])
const theme = useLocalStorage<'light' | 'dark'>('theme', 'light')
const userProfile = useLocalStorage<UserProfile | null>('user', null)

// tasks 就是一個普通的 Ref，可以直接讀寫
tasks.value.push({ id: 1, text: 'Learn Vue 3', done: false })
// 改完之後，LocalStorage 自動同步——使用端零感知
```

**API 設計**：

```ts
function useLocalStorage<T>(
  key: string,
  defaultValue: T,
  options?: {
    serializer?: {
      read: (raw: string) => T
      write: (val: T) => string
    }
  }
): Ref<T>
```

回傳的是**可寫入的 `Ref<T>`**（注意：不是 readonly！使用端需要寫入）。

---

### 三、JSON 安全讀寫——三個風險點

W3 提過「安全讀取三個風險點」，今日深化到實作層：

```ts
function parseJSON<T>(value: string | null): T | null {
  // 風險 1：value 可能是 null（localStorage 取不到）
  if (value === null) return null

  // 風險 2：JSON.parse 可能拋出 SyntaxError（被人工篡改的 localStorage）
  try {
    const parsed: unknown = JSON.parse(value)

    // 風險 3：parse 成功但型別不符（JSON 合法，但不是我們期待的 T）
    // → 這裡只能相信型別（as T），或傳入型別守衛做驗證
    return parsed as T
  } catch {
    return null
  }
}
```

**三個風險對照**：

| 風險 | 原因 | 處理方式 |
|------|------|---------|
| `null` | key 不存在，`getItem` 回 null | 直接判斷 null，回傳預設值 |
| `SyntaxError` | JSON 格式損壞（手動編輯、版本遷移）| try/catch，回傳預設值 |
| 型別不符 | JSON 合法但資料結構變了（版本遷移）| `as T` 承諾（進階：傳入型別守衛）|

---

### 四、完整實作

```ts
// useLocalStorage.ts
import { ref, watch, onMounted, type Ref } from 'vue'

function parseJSON<T>(raw: string | null): T | null {
  if (raw === null) return null
  try {
    return JSON.parse(raw) as T
  } catch {
    return null
  }
}

export function useLocalStorage<T>(key: string, defaultValue: T): Ref<T> {
  // SSR 安全：伺服器端沒有 window，直接用預設值
  const isBrowser = typeof window !== 'undefined'

  // 初始化：有已儲存的值就用，否則用預設值
  const stored = isBrowser ? parseJSON<T>(localStorage.getItem(key)) : null
  const data = ref<T>(stored !== null ? stored : defaultValue) as Ref<T>

  // 寫入回路：data 改變時，同步到 localStorage
  watch(
    data,
    (val) => {
      if (isBrowser) {
        localStorage.setItem(key, JSON.stringify(val))
      }
    },
    { deep: true }  // 若 T 是物件/陣列，需要 deep watch
  )

  return data
}
```

**與 W3 實作的差異對照**：

| 面向 | W3 useTasks 版本 | 今日 useLocalStorage<T> |
|------|-----------------|------------------------|
| 適用範圍 | 只給 tasks 用 | 任何 key + 任意型別 T |
| onMounted 讀取 | 在 Composable 內部 | 移到 `ref()` 初始化時（更早，無需等掛載）|
| SSR 安全 | 未處理 | `typeof window !== 'undefined'` |
| 型別守衛 | `as Task[]`（不安全）| `parseJSON<T>` 隔離 try/catch |
| watch deep | 有 | 有（通用）|

> **為什麼把讀取從 `onMounted` 移到初始化？**
> `ref()` 在 Composable 被呼叫時就執行，比 `onMounted` 更早。  
> 如果用 `onMounted` 讀取，模板在掛載前渲染時會先看到 `defaultValue`，然後觸發一次 diff 更新——這是不必要的閃爍。  
> 直接在初始化時讀取 LocalStorage，`ref` 的初始值就是正確的持久化值，模板第一次渲染就對。

---

### 五、用 useLocalStorage 升級 useTasks

```ts
// useTasks.v3.ts
import { computed, watch } from 'vue'
import { useLocalStorage } from './useLocalStorage'
import type { Task } from './types'

export function useTasks() {
  // ← 原本 20 行的持久化邏輯，現在是 1 行
  const tasks = useLocalStorage<Task[]>('tasks', [])

  const addTask = (text: string) => {
    tasks.value.push({ id: Date.now(), text, done: false })
  }

  const toggleTask = (id: number) => {
    const task = tasks.value.find(t => t.id === id)
    if (task) task.done = !task.done
  }

  const clearDone = () => {
    tasks.value = tasks.value.filter(t => !t.done)
  }

  const completedCount = computed(() => tasks.value.filter(t => t.done).length)
  const totalCount = computed(() => tasks.value.length)

  return { tasks, addTask, toggleTask, clearDone, completedCount, totalCount }
}
```

**收益**：`useTasks` 的職責清晰了——只負責業務邏輯，持久化是 `useLocalStorage` 的責任。

---

### 六、常見陷阱

**陷阱 1：watch deep 的效能考量**

```ts
// 若 T 是原始值（string、number、boolean），不需要 deep
const theme = useLocalStorage<'light' | 'dark'>('theme', 'light')
// watch deep: true 對原始值無害，但若 T 是大型陣列，watch deep 會遞迴追蹤每個元素
```

進階版本可依 T 是否為物件決定 `deep` 選項：

```ts
watch(data, (val) => {
  localStorage.setItem(key, JSON.stringify(val))
}, { deep: typeof defaultValue === 'object' && defaultValue !== null })
```

**陷阱 2：多個 tab 的 localStorage 同步**

瀏覽器的 `storage` 事件可以監聽其他 tab 對 localStorage 的修改：

```ts
// 選擇性進階：跨 tab 同步
if (isBrowser) {
  window.addEventListener('storage', (event) => {
    if (event.key === key && event.newValue !== null) {
      const updated = parseJSON<T>(event.newValue)
      if (updated !== null) data.value = updated
    }
  })
}
```

本次作業不要求實作，但面試常考。

---

## 作業說明

### 情境背景

你正在開發 `ihouseBMS` 的「使用者偏好設定」功能：
- 使用者可以選擇介面語言（`zh-TW` / `en`）
- 使用者可以切換深淺主題（`light` / `dark`）
- 這些設定在重新整理後應該保留

同時，將 W7 Day 1 的 `useAsyncData<T>` 作業延伸：
- 用 `useLocalStorage` 快取最後一次查詢的房源搜尋條件

### 功能需求

**Part 1：實作 `useLocalStorage<T>`**

依照本文知識說明，實作 `useLocalStorage.ts`，需滿足：
- 泛型 `<T>`，接受任意型別
- `parseJSON<T>` 獨立函式（隔離 try/catch）
- SSR 安全（`typeof window !== 'undefined'` 判斷）
- `watch({ deep: true })` 自動同步

**Part 2：實作 `useUserPreferences.ts`**

```ts
export function useUserPreferences() {
  const language = useLocalStorage<'zh-TW' | 'en'>('lang', 'zh-TW')
  const theme = useLocalStorage<'light' | 'dark'>('theme', 'light')

  // 需要：切換函式
  const toggleTheme = () => { ... }
  const setLanguage = (lang: 'zh-TW' | 'en') => { ... }

  return { language, theme, toggleTheme, setLanguage }
}
```

**Part 3：整合到 `PropertyList.vue`**（W7 Day 1 作業延伸）

將搜尋條件（城市、房型）用 `useLocalStorage` 持久化，讓使用者重新整理後搜尋條件保留。

### 技術規範

- `useLocalStorage.ts`：純 TypeScript，無 Vue 模板依賴
- `useUserPreferences.ts`：使用 `useLocalStorage`，不重新實作 localStorage 邏輯
- `PropertyList.vue`：整合 `useAsyncData` + `useLocalStorage`（兩個 Composable 共存）
- `vue-tsc --noEmit` 零錯誤

### 評分標準

| 項目 | 分數 | 說明 |
|------|------|------|
| `useLocalStorage<T>` 核心實作正確 | 30 | parseJSON 安全、watch deep、SSR 判斷 |
| `useUserPreferences` 設計清晰 | 20 | 職責單一、回傳介面乾淨 |
| 持久化實際有效（重新整理保留值）| 20 | 瀏覽器驗證 |
| TypeScript 型別完整，無 any | 20 | vue-tsc 零錯誤 |
| 程式碼可讀性 & 命名一致性 | 10 | 符合 Conventional 規範 |

---

## 自我檢核問題

1. **為什麼 `parseJSON<T>` 要獨立出來，而不是直接在 `useLocalStorage` 裡面 try/catch？**  
   （提示：單一職責原則、測試性）

2. **`useLocalStorage` 回傳 `Ref<T>` 而不是 `Readonly<Ref<T>>`——這和 `useAsyncData` 的設計決策有什麼不同？為什麼？**  
   （提示：使用端是否需要寫入？封裝邊界在哪裡？）

3. **如果同一個 `key` 被兩個不同的 `useLocalStorage('user', ...)` 呼叫，會發生什麼事？**  
   （提示：watch 各自獨立 vs 共享同一個 ref）

4. **`watch({ deep: true })` 在什麼情況下會有效能問題？你會怎麼設計更好的版本？**

---

## 明日預告（W7 Day 3）

明日主題：**`usePagination` × `useEventListener` — 基礎設施型 Composable 設計**

```
useLocalStorage → 持久化層 Composable（資料保存）
useAsyncData   → 非同步層 Composable（網路請求）
usePagination  → UI 邏輯層 Composable（分頁計算）
useEventListener → 基礎設施層 Composable（事件管理）
```

四種 Composable 各自服務不同的關注點，合在一起就是一個完整的前端 Composable 架構分層。
