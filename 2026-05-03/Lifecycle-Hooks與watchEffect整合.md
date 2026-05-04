# Lifecycle Hooks 深入 × watchEffect 整合模式

> **日期**：2026-05-03（M1-W2 Day 6，W2 截止前最後一天）
> **類型**：W3 預習深化日（Lifecycle Hooks 完整梳理 + watchEffect 整合）
> **目標技術**：Vue 3 Lifecycle Hooks、watchEffect、onCleanup 模式
> **截止日提醒**：⚠️ W2 截止 **明日（5/4）**，確認 4 份 PR 狀態

---

## 目標技術與核心知識點

| 技術 | 核心知識點 |
|------|-----------|
| Vue 3 Lifecycle Hooks | 完整生命週期順序、`<script setup>` 寫法、SSR 安全鉤子 |
| `watchEffect` | 自動依賴追蹤、與 `onMounted` 的互動、`onCleanup` |
| 副作用清理模式 | `AbortController` + `onCleanup`、停止監聽函式 |
| `watch` + `immediate` | 取代 `onMounted + watch` 雙段邏輯 |

---

## 為什麼學這個？（與前幾日的連結）

昨日（5/2）的 W3 預習中，我們深入了 `ref` 陣列解包陷阱、`watch` 四種監聽形式、`computed` 快取精確定義。但有一個關鍵主題只點到但沒展開：

> **「`immediate: true` 在 `onMounted` 之前執行，對 SSR 有重要意義」**

這句話背後是一整套 **Lifecycle Hooks 的運作邏輯**，而且 `watchEffect` 的行為也和生命週期深度耦合。理解生命週期 → 才能真正理解為什麼 Vue 的響應式系統這樣設計。

另一方面，在 `async-utils` 的 `useAsyncState` Composable 中（4/30 實作），我們用了 `onMounted` 來觸發 API 呼叫。今天的學習能讓你重新審視那段程式碼，並理解更好的寫法。

---

## 知識說明

### 一、Vue 3 完整生命週期

#### 1. 生命週期順序圖

```
元件建立
  │
  ▼
┌─────────────────────────────────────────────┐
│  setup() / <script setup>                   │  ← 響應式資料建立、props 可用
│  ├── onBeforeMount()                         │  ← DOM 尚未建立，$el 不可用
│  ├── [Vue 渲染 vDOM，建立真實 DOM]            │
│  └── onMounted()                             │  ← DOM 已建立，可操作 DOM
│                                             │
│  ── 響應式資料變化 → 觸發更新 ──              │
│  ├── onBeforeUpdate()                        │  ← DOM 即將更新
│  ├── [Vue 重新渲染，patch DOM]               │
│  └── onUpdated()                             │  ← DOM 已更新（⚠️ 別在這改響應式資料！）
│                                             │
│  ── 元件卸載（路由切換、v-if 移除）─────      │
│  ├── onBeforeUnmount()                       │  ← 元件仍可見，適合清理副作用
│  └── onUnmounted()                          │  ← DOM 已移除，清理計時器/事件監聽
└─────────────────────────────────────────────┘
```

#### 2. `<script setup>` 的寫法

Vue 3 Options API → Composition API 的生命週期對應：

| Options API | Composition API (`<script setup>`) |
|-------------|-------------------------------------|
| `created` | 不需要（`setup()` 本身就是 created 階段）|
| `mounted` | `onMounted()` |
| `beforeMount` | `onBeforeMount()` |
| `updated` | `onUpdated()` |
| `beforeUpdate` | `onBeforeUpdate()` |
| `beforeUnmount` | `onBeforeUnmount()` |
| `unmounted` | `onUnmounted()` |
| `errorCaptured` | `onErrorCaptured()` |

```vue
<script setup lang="ts">
import {
  ref,
  onMounted,
  onBeforeMount,
  onUpdated,
  onBeforeUnmount,
  onUnmounted
} from 'vue'

const el = ref<HTMLDivElement | null>(null)
const count = ref(0)

// setup 本身 = created 階段（最早執行）
console.log('1. setup 執行（= created）')

onBeforeMount(() => {
  // DOM 尚未建立
  console.log('2. onBeforeMount — DOM 未建立')
  console.log(el.value)  // null
})

onMounted(() => {
  // DOM 已建立，可以安全操作 DOM
  console.log('3. onMounted — DOM 就緒')
  console.log(el.value)  // <div>...</div>
  el.value?.focus()
})

onBeforeUpdate(() => {
  console.log('4. onBeforeUpdate — 即將更新，count =', count.value)
})

onUpdated(() => {
  console.log('5. onUpdated — 更新完成')
  // ⚠️ 不要在這裡修改響應式資料，會造成無限更新循環！
})

onBeforeUnmount(() => {
  // 最後清理副作用的時機（元件還存在）
  console.log('6. onBeforeUnmount — 即將卸載')
})

onUnmounted(() => {
  // DOM 已移除
  console.log('7. onUnmounted — 卸載完成')
})
</script>

<template>
  <div ref="el">{{ count }}</div>
  <button @click="count++">+</button>
</template>
```

---

### 二、常見陷阱：哪些鉤子在 SSR 中不安全？

Vue 3 支援 SSR（伺服器端渲染）。在 SSR 環境中，**沒有 DOM**，因此：

| 鉤子 | SSR 是否執行 | 說明 |
|------|:-----------:|------|
| `setup()` | ✅ 執行 | 響應式資料建立在 SSR 中也需要 |
| `onBeforeMount` | ❌ 不執行 | 伺服器沒有「掛載」概念 |
| **`onMounted`** | ❌ **不執行** | **伺服器端沒有 DOM** |
| `onUpdated` | ❌ 不執行 | 無 DOM 更新 |
| `onBeforeUnmount` | ❌ 不執行 | 無卸載概念 |
| `onUnmounted` | ❌ 不執行 | 無卸載概念 |

**實際意義**：

```typescript
// ❌ 如果用 Nuxt / SSR，這段 API 呼叫只在客戶端執行
// 伺服器渲染時資料是空的，SEO 不友好
onMounted(async () => {
  const data = await fetchUser(userId.value)
  user.value = data
})

// ✅ watch + immediate 的方式
// 在 SSR 的 setup 階段也會同步執行（不過非同步 fetch 還是需要 useFetch 等工具）
watch(
  () => props.userId,
  async (userId) => {
    user.value = await fetchUser(userId)
  },
  { immediate: true }
)
// immediate: true 在 setup 階段同步觸發，早於 onMounted
```

---

### 三、watchEffect 的完整理解

#### 1. 執行時機：第一次在 `onMounted` 之前

```vue
<script setup lang="ts">
import { ref, watchEffect, onMounted } from 'vue'

const count = ref(0)

// watchEffect 的第一次執行在 setup 階段（早於 onMounted）
watchEffect(() => {
  console.log('watchEffect 執行，count =', count.value)
})

onMounted(() => {
  console.log('onMounted 執行')
})
</script>
```

**執行順序輸出**：
```
watchEffect 執行，count = 0    ← 先執行（setup 階段）
onMounted 執行                  ← 後執行
```

若你要用 `watchEffect` 操作 DOM（讀取元素尺寸等），需用 `flush: 'post'`：

```typescript
// 在 DOM 更新後執行（等同 watchPostEffect）
watchEffect(() => {
  console.log(listEl.value?.scrollHeight)
}, { flush: 'post' })

// 語法糖（Vue 3.2+）
import { watchPostEffect } from 'vue'
watchPostEffect(() => {
  console.log(listEl.value?.scrollHeight)
})
```

---

#### 2. 自動依賴追蹤 vs 明確指定

```typescript
import { ref, watch, watchEffect } from 'vue'

const userId = ref(1)
const filters = ref({ active: true })
const user = ref(null)

// watch：必須明確指定監聽來源
watch(userId, async (id) => {
  user.value = await fetchUser(id)
})
// filters 改變不會觸發（沒有監聽它）

// watchEffect：自動追蹤所有在 callback 中讀到的響應式資料
watchEffect(async () => {
  // Vue 會追蹤以下兩個 ref 的讀取
  const data = await fetchUsers(userId.value, filters.value)
  //                            ^^^^^^^^^^^  ^^^^^^^^^^^^^
  //                            自動建立依賴   自動建立依賴
  user.value = data
})
// userId 或 filters 任一改變，都會重新執行
```

**注意**：非同步 `watchEffect` 的依賴追蹤只在第一個 `await` 之前的同步程式碼有效：

```typescript
watchEffect(async () => {
  // ✅ 此行在 await 之前，依賴被追蹤
  const id = userId.value

  const data = await fetchUser(id)  // await 分界線

  // ⚠️ 此行在 await 之後，依賴「不會」被追蹤
  console.log(otherRef.value)  // 改變 otherRef 不會重新觸發
})
```

---

#### 3. onCleanup：副作用清理（最重要的模式之一）

這是昨日 `async-utils` 中 `AbortController` 模式在 Vue 中的正式應用：

```typescript
import { ref, watchEffect, watch } from 'vue'

const userId = ref(1)
const userData = ref(null)

// 模式 1：watchEffect + onCleanup
watchEffect((onCleanup) => {
  const controller = new AbortController()

  // 在 cleanup 時取消上一次的請求
  onCleanup(() => {
    controller.abort()
    console.log('上一次請求已取消')
  })

  fetch(`/api/user/${userId.value}`, { signal: controller.signal })
    .then(r => r.json())
    .then(data => { userData.value = data })
    .catch(err => {
      if (err.name !== 'AbortError') throw err
    })
})

// 模式 2：watch + onCleanup（需要 oldVal 的場景）
watch(userId, (newId, oldId, onCleanup) => {
  const controller = new AbortController()

  onCleanup(() => controller.abort())

  fetch(`/api/user/${newId}`, { signal: controller.signal })
    .then(r => r.json())
    .then(data => { userData.value = data })
    .catch(err => {
      if (err.name !== 'AbortError') throw err
    })
})
```

**為什麼需要 onCleanup？**

想像快速切換 userId：
1. userId 從 1 → 2 → 3（快速切換）
2. 若沒有 cleanup，請求 1、2、3 都在飛，哪個先回來就用哪個
3. 可能出現「請求 3 先回來，但請求 2 後到，最終顯示 userId=2 的資料」這種 Race Condition
4. `onCleanup` + `AbortController` 確保「新 userId 觸發時，舊請求立刻取消」

---

#### 4. 停止 watchEffect（手動控制）

```typescript
// watchEffect 回傳停止函式
const stop = watchEffect(() => {
  console.log('watching:', count.value)
})

// 某條件達成時停止
function handleComplete() {
  stop()  // 之後 count 變化不再觸發
}

// Vue 元件 unmount 時會自動停止，但在元件外（如全域 store）需手動停止
```

---

### 四、整合模式：Lifecycle Hooks × watch/watchEffect

#### 模式 1：資料獲取（傳統 vs 推薦）

```vue
<script setup lang="ts">
import { ref, watch, onMounted } from 'vue'

const props = defineProps<{ userId: number }>()
const user = ref(null)
const loading = ref(false)

// ❌ 舊習慣：onMounted + watch 兩段重複邏輯
onMounted(async () => {
  loading.value = true
  user.value = await fetchUser(props.userId)
  loading.value = false
})
watch(
  () => props.userId,
  async (newId) => {
    loading.value = true
    user.value = await fetchUser(newId)
    loading.value = false
  }
)

// ✅ 推薦：watch + immediate（一段邏輯，自動處理初始載入）
watch(
  () => props.userId,
  async (newId, _, onCleanup) => {
    const controller = new AbortController()
    onCleanup(() => controller.abort())

    loading.value = true
    try {
      user.value = await fetchUser(newId, controller.signal)
    } finally {
      loading.value = false
    }
  },
  { immediate: true }
)
</script>
```

#### 模式 2：DOM 操作（必須用 onMounted）

```vue
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const chartEl = ref<HTMLDivElement | null>(null)
let chartInstance: ChartInstance | null = null

// 必須等 DOM 就緒才能初始化圖表
onMounted(() => {
  if (chartEl.value) {
    chartInstance = createChart(chartEl.value)
  }
})

// 元件卸載時清理圖表實例（防止記憶體洩漏）
onUnmounted(() => {
  chartInstance?.destroy()
  chartInstance = null
})
</script>

<template>
  <div ref="chartEl" class="chart-container" />
</template>
```

#### 模式 3：事件監聽的清理

```vue
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'

function handleResize() {
  console.log('window resized:', window.innerWidth)
}

// ✅ 對稱的 mount/unmount 模式
onMounted(() => {
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>
```

#### 模式 4：Composable 中封裝生命週期（W3 預習）

```typescript
// useWindowSize.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useWindowSize() {
  const width = ref(window.innerWidth)
  const height = ref(window.innerHeight)

  function update() {
    width.value = window.innerWidth
    height.value = window.innerHeight
  }

  // 生命週期鉤子可以在 Composable 中呼叫！
  // Vue 會把它們綁定到呼叫這個 Composable 的元件上
  onMounted(() => window.addEventListener('resize', update))
  onUnmounted(() => window.removeEventListener('resize', update))

  return { width, height }
}

// 使用
// <script setup>
// const { width, height } = useWindowSize()
```

---

### 五、常見陷阱彙整

| 陷阱 | 問題 | 正確做法 |
|------|------|---------|
| `onUpdated` 中修改響應式資料 | 無限更新循環 | 改用 `watch` |
| `onMounted` 存取不存在的 DOM ref | `ref.value` 為 null | 確保 template 已渲染，或使用 `?.` |
| 全域 `watchEffect` 忘記停止 | 記憶體洩漏 | 儲存停止函式並在適當時機呼叫 |
| `watchEffect` 在 `await` 後讀取 ref | 依賴未被追蹤 | 在 `await` 前讀取需要追蹤的值 |
| 事件監聽未在 `onUnmounted` 移除 | 記憶體洩漏 / 殭屍監聽 | 對稱的 mount/unmount |

---

## 作業說明（W3 銜接作業，5/5 前完成）

> 本作業為「預習驗證型」——用不查文件的方式完成，檢測真實理解深度。

### 情境背景

你在開發 `ihouseBMS` 的房源列表頁面。該頁面需要：
1. 根據 `filters`（篩選條件）即時呼叫 API 取得房源資料
2. 使用者切換篩選條件時，**取消上一次未完成的請求**（避免 Race Condition）
3. 頁面卸載時，確保所有副作用都被清理

### 功能需求

實作一個 `usePropertyList.ts` Composable，包含：

```typescript
// 預期介面
interface UsePropertyListReturn {
  properties: Ref<Property[]>
  loading: Ref<boolean>
  error: Ref<Error | null>
}

function usePropertyList(
  filters: Ref<PropertyFilters>
): UsePropertyListReturn
```

#### 技術規範

- **監聽**：使用 `watch` 或 `watchEffect` 監聽 `filters` 的變化
- **初始載入**：頁面進入時立即載入（不需等待 filters 改變）
- **Race Condition 防護**：用 `AbortController` + `onCleanup` 取消舊請求
- **載入狀態**：`loading` 在請求開始時為 `true`，完成（成功或失敗）後為 `false`
- **錯誤處理**：非 `AbortError` 的錯誤應存入 `error`
- **生命週期**：使用正確的鉤子（Composable 中也可呼叫）

#### 評分標準

| 項目 | 說明 | 分數 |
|------|------|------|
| 監聽邏輯正確 | 能響應 filters 變化並重新載入 | /25 |
| Race Condition 防護 | `AbortController` 正確使用 | /25 |
| 載入/錯誤狀態管理 | loading / error 行為符合規範 | /25 |
| TypeScript 型別完整 | 無 any，介面清晰 | /25 |

---

## 自我檢核問題（W3 預習驗證）

**1. `onBeforeMount` 和 `onMounted` 的差異是什麼？什麼情況下你需要用 `onBeforeMount`？**

> 提示：大多數情況下你只需要 `onMounted`，`onBeforeMount` 很少使用

**2. 為什麼說「`watchEffect` 在 await 後的依賴不被追蹤」？請用程式碼示例說明。**

> 提示：Vue 的依賴追蹤是同步收集的

**3. 設計一個 `useInterval(callback, ms)` Composable，要求：元件 unmount 時自動清除計時器。**

> 提示：`setInterval` + `onUnmounted` + `clearInterval`

**4. 下面哪種方式在 SSR 環境中能正確觸發初始資料載入？為什麼？**

```typescript
// 方式 A
onMounted(async () => {
  data.value = await fetchData()
})

// 方式 B
watch(source, async (val) => {
  data.value = await fetchData(val)
}, { immediate: true })
```

**5. `watchEffect` 的 `flush: 'post'` 解決了什麼問題？請給出一個需要它的真實場景。**

---

## 明日預告（5/4，W2 截止日）

**⚠️ 明日為 W2 截止日，主要任務是作業確認，非學習日**：

- 確認 4 份 PR 的最終狀態（W1 × 3、W2 × 1）
- 若有 Code Review 回饋，整理回覆
- 可以開始思考 W3 正式作業（任務清單元件，第二版）的設計

**5/5（週一）起正式進入 W3**：
- 主題：`ref` / `reactive` / `computed` / `watch` 完整深入教材
- 作業：任務清單元件（TaskList.vue 第二版），納入本週所有預習知識
- 重點：Lifecycle Hooks + watchEffect 整合（即本日內容的實作應用）
