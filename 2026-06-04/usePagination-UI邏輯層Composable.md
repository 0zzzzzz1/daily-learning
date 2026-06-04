# usePagination — UI 邏輯層 Composable（M2-W7 Day 3）

> **日期**：2026-06-04  
> **週次**：M2-W7 Day 3（截止 2026-06-08，剩 4 天）  
> **前置**：useAsyncData（Day 1）× useLocalStorage（Day 2）→ 今日：usePagination

---

## 目標技術

- **核心**：Composable UI 邏輯層設計 — usePagination
- **整合**：與 useAsyncData 的協同（頁碼變更 → 重新 fetch）
- **TypeScript**：泛型計算屬性、computed 回傳型別明確標注

---

## 核心知識點

1. usePagination 的設計邊界（只管「哪頁」，不管「取什麼資料」）
2. `computed` 驅動的純計算屬性（totalPages、pageRange、hasPrev/hasNext）
3. 與 useAsyncData 整合的兩種模式（watch vs `execute` 主動呼叫）
4. PageInfo 介面設計（使用端視角優先）

---

## 為什麼學這個

```
Week 3  useTasks          → 「任務清單」業務邏輯 × computed/watch
Week 7 Day 1  useAsyncData    → 「非同步層」× AbortController × 三態管理
Week 7 Day 2  useLocalStorage → 「持久化層」× parseJSON<T> × Ref<T> 邊界
Week 7 Day 3  usePagination   → 「UI 邏輯層」× pure computed × 與非同步層整合
```

前兩天建立了「非同步資料取得」和「本地持久化」兩層 Composable。  
今天建立第三層：**UI 邏輯層**——分頁計算邏輯。

這一層的特徵：
- **無副作用**：只做數學計算，不碰網路、不碰 LocalStorage
- **純 computed 驅動**：所有衍生狀態都是 `currentPage` + `total` + `pageSize` 的函式
- **與其他層整合**：`watch(currentPage)` 觸發 `useAsyncData.execute()`

---

## 知識說明

### 1. usePagination 的設計邊界

**核心問題**：分頁邏輯應該抽取「什麼」？

```
❌ 錯誤邊界（混入業務邏輯）
usePagination({
  fetchFn: () => fetchProperties(currentPage.value),  // ← 不應該在這裡
  pageSize: 10,
  total
})

✅ 正確邊界（只管頁碼計算）
const { currentPage, totalPages, goToPage, nextPage, prevPage, hasPrev, hasNext } 
  = usePagination({ total, pageSize: 10 })
```

**設計原則**：usePagination 只知道「現在第幾頁」和「共幾頁」，**不知道資料如何取得**。資料取得是 useAsyncData 的責任。

---

### 2. 完整實作

```typescript
// usePagination.ts
import { ref, computed, type Ref, type ComputedRef } from 'vue'

export interface PaginationOptions {
  total: Ref<number>    // 資料總筆數（響應式，因為 fetch 後才知道）
  pageSize?: number     // 每頁幾筆，預設 10
  initialPage?: number  // 初始頁碼，預設 1
}

export interface PaginationResult {
  currentPage: Ref<number>
  totalPages: ComputedRef<number>
  pageSize: number
  hasPrev: ComputedRef<boolean>
  hasNext: ComputedRef<boolean>
  pageRange: ComputedRef<number[]>  // 顯示的頁碼列表（帶省略號邏輯）
  goToPage: (page: number) => void
  nextPage: () => void
  prevPage: () => void
  reset: () => void
}

export function usePagination(options: PaginationOptions): PaginationResult {
  const { total, pageSize = 10, initialPage = 1 } = options

  const currentPage = ref(initialPage)

  // ── computed（純計算，無副作用）──────────────────────────────────
  
  const totalPages = computed(() => Math.max(1, Math.ceil(total.value / pageSize)))

  const hasPrev = computed(() => currentPage.value > 1)
  const hasNext = computed(() => currentPage.value < totalPages.value)

  // 頁碼範圍：最多顯示 5 個，中間顯示當前頁附近，超出顯示 0 代表「...」
  const pageRange = computed<number[]>(() => {
    const total = totalPages.value
    const current = currentPage.value
    
    if (total <= 7) {
      // 總頁數少，全部顯示
      return Array.from({ length: total }, (_, i) => i + 1)
    }
    
    // 超過 7 頁：顯示首頁、尾頁，中間最多 3 頁，用 0 代表省略號
    const pages: number[] = [1]
    
    const start = Math.max(2, current - 1)
    const end = Math.min(total - 1, current + 1)
    
    if (start > 2) pages.push(0)  // 省略號
    for (let i = start; i <= end; i++) pages.push(i)
    if (end < total - 1) pages.push(0)  // 省略號
    pages.push(total)
    
    return pages
  })

  // ── 操作函式 ─────────────────────────────────────────────────────
  
  const goToPage = (page: number): void => {
    // 邊界防禦：不允許超出範圍
    const clamped = Math.max(1, Math.min(page, totalPages.value))
    currentPage.value = clamped
  }

  const nextPage = (): void => {
    if (hasNext.value) currentPage.value++
  }

  const prevPage = (): void => {
    if (hasPrev.value) currentPage.value--
  }

  const reset = (): void => {
    currentPage.value = initialPage
  }

  return {
    currentPage,
    totalPages,
    pageSize,
    hasPrev,
    hasNext,
    pageRange,
    goToPage,
    nextPage,
    prevPage,
    reset,
  }
}
```

---

### 3. 與 useAsyncData 整合

兩種整合模式的選擇：

#### 模式 A：元件層用 watch 銜接（鬆耦合）

```typescript
// PropertyList.vue <script setup>
const total = ref(0)
const pagination = usePagination({ total, pageSize: 10 })

const { data, loading, error, execute } = useAsyncData(async () => {
  const res = await fetchProperties({
    page: pagination.currentPage.value,
    size: pagination.pageSize
  })
  total.value = res.total  // ← 更新 total，pagination.totalPages 自動重算
  return res.items
})

// 頁碼變更時重新 fetch
watch(pagination.currentPage, () => execute())

// 初始載入
onMounted(() => execute())
```

#### 模式 B：在 Composable 中封裝（緊耦合，但介面更乾淨）

```typescript
// usePropertyList.ts（業務 Composable，整合多層）
export function usePropertyList() {
  const total = ref(0)
  const pagination = usePagination({ total, pageSize: 10 })
  
  const { data, loading, error, execute } = useAsyncData(async () => {
    const res = await fetchProperties({
      page: pagination.currentPage.value,
      size: pagination.pageSize,
    })
    total.value = res.total
    return res.items
  })

  watch(pagination.currentPage, () => execute())
  
  return { data, loading, error, pagination, refresh: execute }
}

// PropertyList.vue — 使用端更乾淨
const { data, loading, pagination } = usePropertyList()
```

**選用決策**：
- 邏輯只用一次 → 模式 A（不用額外抽 Composable）
- 多個元件共用同一套「資料 + 分頁」邏輯 → 模式 B

---

### 4. 常見陷阱

#### 陷阱 1：total 不是響應式

```typescript
// ❌ 錯誤：number 不是響應式，totalPages 不會更新
const pagination = usePagination({ total: 100, pageSize: 10 })

// ✅ 正確：傳入 Ref<number>
const total = ref(0)
const pagination = usePagination({ total, pageSize: 10 })
// fetch 後：total.value = res.total → totalPages 自動重算 ✓
```

#### 陷阱 2：頁碼超出範圍時未做邊界保護

```typescript
// ❌ 若 total 從 100 → 30（過濾條件變嚴），currentPage 可能是 10，
//    但 totalPages 只剩 3，會停在無效頁
// ✅ 在 total 改變時，若 currentPage > totalPages，重設到最後一頁

watch(totalPages, (newTotal) => {
  if (currentPage.value > newTotal) {
    currentPage.value = newTotal
  }
})
```

#### 陷阱 3：`pageRange` 中的 0 忘記處理

```vue
<!-- ❌ 直接 render，0 會顯示成「0」 -->
<button v-for="p in pageRange" :key="p">{{ p }}</button>

<!-- ✅ 判斷 0 顯示省略號 -->
<template v-for="p in pageRange" :key="p">
  <span v-if="p === 0">…</span>
  <button v-else @click="goToPage(p)" :class="{ active: p === currentPage }">{{ p }}</button>
</template>
```

---

## 作業說明

### 情境背景

你正在開發 ihouseBMS（房屋管理系統）的「房源列表頁」。  
後端 API `/api/properties?page=1&size=10` 回傳分頁資料。

### 功能需求

1. **`usePagination.ts`**：根據上方完整實作完成
   - 加入 `total` 變化時的邊界保護（watch totalPages）
   - 確保所有 computed 回傳型別明確標注（`ComputedRef<number>` 等）

2. **`PropertyList.vue`**：整合 `usePagination` + `useAsyncData`
   - 使用**模式 A**（元件層 watch 銜接）
   - 顯示房源列表（卡片 or 列表皆可）
   - 顯示分頁控制列（上/下頁按鈕 + 頁碼，含省略號）
   - Loading 狀態時顯示 Skeleton（複用 W2 的概念）
   - Error 狀態時顯示錯誤訊息 + 重試按鈕

3. **`useUserPreferences.ts`**（選做 bonus）：整合 `useLocalStorage`
   - 儲存使用者偏好的 pageSize（10 / 20 / 50）
   - 讓 PropertyList 使用 `useUserPreferences` 取得 pageSize

### 技術規範

```typescript
// 最低型別要求
export function usePagination(options: PaginationOptions): PaginationResult
// PaginationOptions.total 必須是 Ref<number>（不接受 number 直接傳入）
// PaginationResult 每個欄位都有明確型別標注
```

### 評分標準

| 項目 | 配分 | 說明 |
|------|------|------|
| usePagination 邊界保護完整 | 25 分 | 包含 totalPages 變化時的 currentPage 修正 |
| PropertyList.vue 整合正確 | 30 分 | watch 銜接 + 三態（loading/error/data）處理 |
| TypeScript 零錯誤（vue-tsc）| 20 分 | 所有型別明確，無 any |
| 分頁 UI（省略號邏輯）| 15 分 | 省略號顯示合理（非 ≤7 頁時才出現）|
| useUserPreferences bonus | 10 分 | 正確封裝 pageSize 偏好、型別安全 |

---

## 自我檢核問題

1. **為什麼 `usePagination` 的 `total` 參數型別是 `Ref<number>` 而不是 `number`？**  
   （提示：fetch 後才知道總筆數，需要響應式更新 totalPages）

2. **`pageRange` 用 `computed` 而不是 `method`，有什麼實際差異？**  
   （提示：computed 有快取，pageRange 計算成本不低，currentPage 沒變就不重算）

3. **模式 A 和模式 B 在 `currentPage` 變更時，`execute()` 的呼叫路徑分別是什麼？**  
   （提示：模式 A 在元件 watch，模式 B 封裝在 usePropertyList 內 watch）

4. **`goToPage` 裡的 `Math.max(1, Math.min(page, totalPages.value))` 是什麼防禦？**  
   （提示：防止傳入 0、負數、或超出 totalPages 的頁碼）

5. **若過濾條件從「全部房源」改成「台北市房源」，total 從 500 → 30，currentPage 可能停在第 20 頁，如何修正？**  
   （提示：watch totalPages，若 currentPage > newTotal 就 reset）

---

## 明日預告（Day 4）

**useEventListener — 基礎設施層 Composable**

```
今日：usePagination（UI 邏輯層，純計算）
明日：useEventListener（基礎設施層，addEventListener/removeEventListener 封裝）
     + PropertyList.vue 實作（整合 usePagination × useAsyncData × useEventListener）
```

useEventListener 是所有 Composable 中最「基礎設施」的一個：
- 包裝 `addEventListener` / `removeEventListener`，自動在 `onUnmounted` 清理
- 使用端無需手動清理事件監聽
- 可用在：keyboard 快捷鍵（翻頁）、scroll 無限加載、resize 響應

```typescript
// 預習：使用端介面長什麼樣？
useEventListener(window, 'keydown', (e: KeyboardEvent) => {
  if (e.key === 'ArrowRight') nextPage()
  if (e.key === 'ArrowLeft') prevPage()
})
// onUnmounted 時自動 removeEventListener，使用端完全不需要管理
```
