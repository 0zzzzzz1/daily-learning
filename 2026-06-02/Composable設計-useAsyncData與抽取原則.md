# Composable 設計：useAsyncData 與抽取邏輯原則

> **日期**：2026-06-02（M2-W7 Day 1）
> **週次主題**：Composable 設計（抽取邏輯、可複用 Hook）
> **截止日**：2026-06-08（W7）

---

## 目標技術與核心知識點

| 技術 | 核心知識點 |
|------|-----------|
| Vue 3 Composable 設計 | 抽取時機三訊號、回傳介面設計、Composable vs Pinia 邊界 |
| `useAsyncData<T>` | 三態管理（loading/data/error）、AbortController 防 Race Condition、型別安全 |
| TypeScript 泛型應用 | 泛型 Composable 設計、型別守衛、回傳介面型別化 |

---

## 為什麼學這個（與前幾日的連結）

W6 建立了「跨層通訊（provide/inject）× 彈性插槽（Slots）× 可維護性（Composable 提取）」三主軸。
其中 `usePanelTheme` 讓你體驗了「從重複樣板識別提取時機」的第一個真實場景。

W7 的任務是**系統化掌握 Composable 設計能力**：
- W6 的 Composable 提取是「被動識別重複」
- W7 要訓練「主動設計可複用 Hook」——在需求階段就知道應該設計成 Composable

`useAsyncData` 是最經典的 Composable 設計案例，幾乎每個真實專案都需要它：
```
元件 A：await fetchUser()
元件 B：await fetchOrders()
元件 C：await fetchProperties()
```
三個元件都有相同的三態管理邏輯 → 這就是 Composable 的典型場景。

---

## 知識說明

### 一、Composable 抽取時機三訊號（系統化）

> W5/W6 已接觸，今日正式系統化

**訊號 1：重複樣板（超過 2 個元件）**
```ts
// 元件 A
const loading = ref(false)
const data = ref<User | null>(null)
const error = ref<string | null>(null)

onMounted(async () => {
  loading.value = true
  try {
    data.value = await fetchUser(id)
  } catch (e) {
    error.value = String(e)
  } finally {
    loading.value = false
  }
})
```
看到這個樣板第二次出現 → 抽取訊號亮起。

**訊號 2：邏輯與 UI 混合**
```ts
// setup() 超過 50 行，非同步邏輯與 UI 狀態混在一起
// → 邏輯應該移出到 Composable
```

**訊號 3：跨元件複用需求**
```ts
// 多個頁面元件需要相同的 localStorage 讀寫邏輯
// → 設計成 useLocalStorage<T>
```

**反訊號（不應提取的情況）**
- 邏輯只出現一次、簡單到 3 行以內
- 提取後依賴過多元件內部細節（高耦合）
- 強行抽取讓介面比原本更複雜

---

### 二、useAsyncData\<T\> 完整實作

#### 需求分析（API 設計草稿）

```ts
// 使用端期望的 API
const { data, loading, error, refresh } = useAsyncData(() => fetchUser(userId.value))

// 支援響應式參數（userId 改變時自動重新 fetch）
// 支援手動 refresh
// 防止 Race Condition（舊請求回應不覆蓋新請求）
```

#### 回傳介面設計

```ts
interface AsyncDataResult<T> {
  data: Readonly<Ref<T | null>>
  loading: Readonly<Ref<boolean>>
  error: Readonly<Ref<string | null>>
  refresh: () => void
}
```

> **設計決策：為什麼 `data` 是 `Ref<T | null>` 而非 `Ref<T | undefined>`？**
> - `null` 語義：「我知道這裡沒有值」（明確的空狀態）
> - `undefined` 語義：「我不知道這裡有沒有值」（未定義狀態）
> - 三態系統中，初始化前和載入中都是「尚未獲得值」，`null` 更語義精確。

#### 三態管理設計

```
loading: true  → data: null   → error: null    ← 載入中
loading: false → data: T      → error: null    ← 成功
loading: false → data: null   → error: string  ← 失敗
```

**重要設計：成功後保留舊 data**
```ts
// 常見錯誤做法：
data.value = null  // ← 重新 fetch 時先清空，畫面閃爍

// 正確做法（refresh 時保留舊值，直到新資料回來）：
// loading 變 true，但 data 維持上次的值
// 讓使用端決定是否顯示舊值 + loading indicator
```

#### AbortController 防 Race Condition

```ts
// Race Condition 場景：
// t=0ms: 請求 A 發出（userId=1）
// t=100ms: userId 改為 2，請求 B 發出
// t=200ms: 請求 B 回應（快，回傳 user2）
// t=300ms: 請求 A 回應（慢，回傳 user1）← 此時 data 被覆蓋成 user1！

// AbortController 解法：
// 每次發新請求時，abort 上一個請求的 signal
```

#### 完整實作

```ts
// composables/useAsyncData.ts
import { ref, watch, onUnmounted, readonly } from 'vue'
import type { Ref } from 'vue'

interface AsyncDataResult<T> {
  data: Readonly<Ref<T | null>>
  loading: Readonly<Ref<boolean>>
  error: Readonly<Ref<string | null>>
  refresh: () => void
}

export function useAsyncData<T>(
  fetcher: (signal: AbortSignal) => Promise<T>
): AsyncDataResult<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const loading = ref(false)
  const error = ref<string | null>(null)

  let abortController: AbortController | null = null

  async function execute() {
    // 中止上一個請求（防 Race Condition）
    if (abortController) {
      abortController.abort()
    }
    abortController = new AbortController()
    const signal = abortController.signal

    loading.value = true
    error.value = null
    // 注意：不清空 data.value，保留舊值直到新資料到來

    try {
      const result = await fetcher(signal)
      // 確認請求未被中止
      if (!signal.aborted) {
        data.value = result
      }
    } catch (e) {
      if (!signal.aborted) {
        // AbortError 不是真正的錯誤，忽略
        if (e instanceof DOMException && e.name === 'AbortError') return
        error.value = e instanceof Error ? e.message : String(e)
      }
    } finally {
      if (!signal.aborted) {
        loading.value = false
      }
    }
  }

  // 元件卸載時中止進行中的請求
  onUnmounted(() => {
    abortController?.abort()
  })

  // 立即執行一次
  execute()

  return {
    data: readonly(data),
    loading: readonly(loading),
    error: readonly(error),
    refresh: execute,
  }
}
```

#### 使用端範例

```vue
<script setup lang="ts">
import { useAsyncData } from '@/composables/useAsyncData'
import { fetchUser } from '@/api/users'

const props = defineProps<{ userId: number }>()

const { data: user, loading, error, refresh } = useAsyncData(
  (signal) => fetchUser(props.userId, signal)
)
</script>

<template>
  <div>
    <div v-if="loading">載入中...</div>
    <div v-else-if="error">錯誤：{{ error }}</div>
    <div v-else-if="user">{{ user.name }}</div>
  </div>
</template>
```

---

### 三、常見陷阱

#### 陷阱 1：fetcher 依賴響應式資料但未觸發重新執行

```ts
// ❌ 問題：userId 改變時，useAsyncData 不會重新 fetch
const { data } = useAsyncData(() => fetchUser(props.userId))

// ✅ 解法 A：watch 外部觸發 refresh
const { data, refresh } = useAsyncData(() => fetchUser(props.userId))
watch(() => props.userId, refresh)

// ✅ 解法 B：設計 watchSource 參數（進階）
// 讓 useAsyncData 內部 watch 指定的響應式資料
```

#### 陷阱 2：忘記在卸載時清理 AbortController

```ts
// 若使用者快速切換頁面，元件卸載時請求仍在進行
// 回應回來後嘗試更新 data.value → 可能產生警告或錯誤
// → 這是為什麼 onUnmounted 中需要 abort()
```

#### 陷阱 3：AbortError 沒有被過濾

```ts
// fetch API 被 abort 時，會 throw DOMException { name: 'AbortError' }
// 這不是真正的錯誤，不應顯示給使用者
// → 需要明確過濾 AbortError
if (e instanceof DOMException && e.name === 'AbortError') return
```

---

## 作業說明

### 情境背景

你在開發 ihouseBMS 的「房源列表頁」，頁面需要呼叫 API 取得房源資料，並顯示載入中、錯誤、資料三種狀態。同樣的邏輯也會在「出租記錄頁」和「維修工單頁」出現。

### 功能需求

實作 `useAsyncData<T>` Composable，並建立一個 `PropertyList.vue` 展示元件：

1. **useAsyncData\<T\>**
   - 泛型支援任意資料型別
   - 三態管理（`loading` / `data` / `error`）
   - AbortController 防 Race Condition
   - 元件卸載時自動中止請求
   - `refresh()` 函式支援手動重新載入
   - 回傳值使用 `readonly` 保護，防止外部直接修改

2. **PropertyList.vue**
   - 使用 `useAsyncData` 取得模擬房源資料（可用 setTimeout 模擬非同步）
   - 顯示三種狀態：載入中 spinner、錯誤訊息、房源卡片列表
   - 提供「重新載入」按鈕呼叫 `refresh()`
   - TypeScript 嚴格型別（定義 `Property` interface）

### 技術規範

- 使用 `<script setup lang="ts">` + 泛型
- `useAsyncData.ts` 中不應有任何 UI 相關邏輯
- AbortController 必須實作（即使模擬 API 無法真正中止）
- `vue-tsc --noEmit` 零錯誤

### 評分標準

| 項目 | 分數 |
|------|------|
| 三態管理正確（loading/data/error 無語義混淆）| 20 分 |
| AbortController 實作（含 AbortError 過濾）| 20 分 |
| 泛型設計（`useAsyncData<T>` 正確推導）| 20 分 |
| `readonly` 保護回傳值 | 10 分 |
| PropertyList.vue 三狀態 UI 完整 | 20 分 |
| TypeScript 零錯誤 + 回傳介面定義 | 10 分 |

---

## 自我檢核問題

1. **Race Condition 是什麼場景下會發生？** 如果不使用 AbortController，實作中有其他方法可以避免嗎？（提示：思考「最後一個請求勝出」的標記法）

2. **為什麼 `data` 的初始值是 `null` 而不是 `undefined`？** 在三態系統中，這個語義差異對使用端的 `v-if` 判斷有什麼影響？

3. **如果 `useAsyncData` 的 `fetcher` 本身依賴一個響應式值（如 `props.userId`），這個 Composable 需要負責 watch 這個依賴嗎？** 還是應該讓使用端自己 watch 並呼叫 `refresh()`？兩種設計的 trade-off 是什麼？

4. **`readonly()` 的作用是什麼？** 如果移除它，使用端能做到什麼原本不應該做的事？

5. **`onUnmounted` 中的 `abort()` 和 `execute()` 內部的 `abort()` 有什麼不同的職責？**

---

## 明日預告（W7 Day 2）

明日主題：**`useLocalStorage<T>` 完整實作**

核心知識點：
- JSON 序列化 / 反序列化（安全讀寫 + try/catch）
- 型別守衛（在讀取時驗證資料結構，而非直接 `as T`）
- SSR 相容性（`typeof window !== 'undefined'` 守衛）
- 跨分頁同步（`storage` 事件監聽）
- `watch` 寫入 + `onMounted` 讀取 的持久化回路（W3 學過！今日複習並升級）

> W3 的 `useTasks.ts` 中已實作過 LocalStorage 持久化，明日我們把它提取成可複用的 `useLocalStorage<T>`。
