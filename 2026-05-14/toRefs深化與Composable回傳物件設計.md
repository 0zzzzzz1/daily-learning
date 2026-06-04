# toRefs 深化與 Composable 回傳物件設計

> **日期**：2026-05-14（W4 Day 3）
> **週次**：W4（5/12–5/18）— Vue 3: Lifecycle Hooks × watchEffect × toRefs
> **前置知識**：W3 toRefs 基本概念（Composable 標準回傳模式）、W4 Day 1 生命週期完整時間軸、W4 Day 2 watchEffect 深化 × onCleanup × Race Condition 防禦
> **帶著這個問題進入今天**：你已經在 W3 學過了 `toRefs` 的基本概念（Composable 標準回傳模式）。但在 W4 的情境下，有一個更進階的問題：當 Composable 的回傳值包含「函式」和「響應式資料」時，是否需要對整個回傳物件做 `toRefs`？有沒有什麼情況下，`toRefs` 反而是**多此一舉**甚至會造成問題的？

---

## 目標技術

| 技術 | 今日重點 |
|------|---------|
| `toRefs` 深化 | 完整決策樹：什麼時候需要、什麼時候不需要 |
| Composable 回傳物件設計 | 函式 vs 響應式資料的不同處理策略 |
| `toRef`（單一屬性）| 與 `toRefs` 的適用場景區分 |
| `storeToRefs` 概念 | Pinia 的先驅思維（M2/M6 預熱）|
| LifecycleLogger.vue 實作啟動 | 完整元件骨架 × useLifecycleLogger.ts 設計 |

---

## 為什麼學這個

W3 學過了 `toRefs` 的第一個用法：**Composable 標準回傳模式**。

```ts
// W3 學到的標準模式
export function useTasks() {
  const state = reactive({ tasks: [], filter: 'all' })
  // ...
  return { ...toRefs(state), addTask, removeTask }
}
```

但 W3 的實作中，其實沒有全面用 `reactive` 包狀態——而是用了獨立的 `ref`：

```ts
// 你實際在 W3 中這樣寫
const tasks = ref<Task[]>([])
const filter = ref<FilterType>('all')
// ...
return { tasks, filter, addTask, removeTask }  // ← 沒有 toRefs，也完全正確！
```

這個差異引出今天的核心問題：**`toRefs` 在什麼情況下是必要的？在什麼情況下反而是多此一舉？**

今天要建立精確的決策樹，讓你在往後任何 Composable 的設計中，都能立刻判斷：這裡需要 `toRefs` 嗎？

另外，今天也是 **LifecycleLogger.vue 實作期啟動**——W4 的作業。昨天（Day 2）設計了 `useWatchEffectLogger.ts` 的介面，今天要開始完整的元件骨架設計，讓剩餘 4 天（截止 5/18）有清晰的實作路線。

---

## 核心知識點

### 1. toRefs 的本質：讓響應式物件的屬性「可分離」

**前提回顧**：`reactive()` 物件的問題是無法解構——解構後會失去響應性。

```ts
const state = reactive({ count: 0, name: 'Alice' })

// ❌ 解構後失去響應性：count 和 name 變成普通變數
const { count, name } = state
count++  // 不會觸發視圖更新

// ✅ toRefs 讓每個屬性都成為 ref，指回原物件
const { count, name } = toRefs(state)
count.value++  // 觸發更新，且 state.count 同步改變 ✅
```

**`toRefs` 做的事**：把 `reactive` 物件的每個屬性轉成一個「連動 ref」——本質上是 `toRef(state, key)` 的批次版本。

---

### 2. toRefs 決策樹：何時需要、何時不需要

```
Composable 回傳物件設計
│
├── 內部狀態是 reactive() 物件？
│   ├── YES → 需要 toRefs 才能讓使用端解構
│   │         return { ...toRefs(state), method1, method2 }
│   └── NO（用了個別的 ref()）
│       └── 直接回傳 ref 即可，不需要 toRefs
│           return { count, name, method1, method2 }
│
└── 只需要暴露「部分」屬性？
    └── 用 toRef(state, 'key') 單獨轉換特定屬性
        return { count: toRef(state, 'count'), method1 }
```

**關鍵原則**：
- `ref` 本身已經是 ref → 不需要 `toRefs`
- `reactive` 物件的屬性不是 ref → 需要 `toRefs` 才能解構後保留響應性
- 函式（methods）既不是 `ref` 也不是 `reactive` → 直接放在回傳物件中，完全不需要 `toRef` 處理

---

### 3. 什麼時候 toRefs 是「多此一舉」甚至造成問題？

#### 情境 A：對獨立 ref 再做 toRefs（多此一舉）

```ts
// ❌ 這樣是無意義的
const count = ref(0)
const name = ref('Alice')

// toRefs 只能用於 reactive() 物件，對普通物件使用沒有效果
const wrapped = toRefs({ count, name })
// wrapped.count 是 RefImpl，wrapped.count.value.value 才是 0！（雙層包裝！）

// ✅ 直接回傳就好
return { count, name }
```

**陷阱**：`toRefs({ count, name })` 中的 `{ count, name }` 是普通物件，不是 `reactive` 物件。`toRefs` 對它做的「轉換」會把 `count`（本身就是 ref）再包一層，造成 `ref.value` 是另一個 ref 的情況。

#### 情境 B：對函式做 toRef（造成問題）

```ts
const state = reactive({
  count: 0,
  increment() { this.count++ }  // ← 方法也是屬性
})

// ❌ 不應該這樣
const { count, increment } = toRefs(state)
// count.value → 0 ✅
// increment.value → 函式 ← 呼叫時需要 increment.value()，語意混亂！

// ✅ 正確做法：響應式資料用 toRefs，方法直接解構
const { count } = toRefs(state)
const { increment } = state  // 直接取方法
```

**結論**：`toRefs` 只應該用於「響應式資料屬性」，函式（方法）應該直接從 `state` 取出或獨立定義後回傳。

#### 情境 C：普通物件（非 reactive）使用 toRefs（無響應性）

```ts
// ❌ 這樣的 toRefs 完全沒意義
const plainObj = { count: 0, name: 'test' }
const refs = toRefs(plainObj)
// refs.count.value === 0 ✅（能讀）
// refs.count.value = 1 → plainObj.count === 1 ✅（能寫）
// 但不會觸發 Vue 更新！因為 plainObj 不是響應式的
```

---

### 4. Composable 回傳物件的最佳設計模式

根據上述分析，Composable 的標準設計有**兩種風格**，各有適用場景：

#### 風格 A：個別 ref 風格（W3 useTasks 的做法）

```ts
export function useTasks() {
  const tasks = ref<Task[]>([])
  const filter = ref<FilterType>('all')
  const newTaskText = ref('')

  // 衍生資料
  const filteredTasks = computed(() => { /* ... */ })
  const taskCount = computed(() => tasks.value.length)

  // 方法
  function addTask() { /* ... */ }
  function removeTask(id: string) { /* ... */ }

  // ✅ 直接回傳 ref，不需要 toRefs
  return {
    tasks,
    filter,
    newTaskText,
    filteredTasks,
    taskCount,
    addTask,
    removeTask
  }
}
```

**優點**：簡單直接；每個 ref 可以獨立命名和組合。
**適用於**：狀態互相獨立、命名已清晰的 Composable。

#### 風格 B：reactive 狀態集中風格（適合複雜 Composable）

```ts
export function useSearchState() {
  const state = reactive({
    query: '',
    results: [] as SearchResult[],
    loading: false,
    error: null as string | null
  })

  async function search() { /* ... */ }
  function clearResults() { state.results = []; state.error = null }

  // ✅ toRefs 讓使用端可以解構
  return {
    ...toRefs(state),  // 這裡只轉 reactive 資料屬性
    search,            // 方法直接放進去
    clearResults
  }
}

// 使用端可以解構，且響應性保持
const { query, results, loading, search } = useSearchState()
query.value = 'Vue 3'  // 響應式更新 ✅
```

**優點**：狀態集中管理，語意清晰（所有狀態屬於同一個「搜尋狀態群」）。
**適用於**：多個狀態高度相關、需要作為整體傳遞的場景。

---

### 5. storeToRefs 概念（Pinia 的先驅思維）

Pinia（Vue 3 官方狀態管理）中有一個 `storeToRefs` 函式，功能和 `toRefs` 非常相似：

```ts
// Pinia 的使用（M6 主題預熱，今日只需了解概念）
import { useCounterStore } from '@/stores/counter'
import { storeToRefs } from 'pinia'

const counterStore = useCounterStore()

// ❌ 直接解構 Pinia Store 失去響應性（Store 是 reactive 物件）
const { count, name } = counterStore

// ✅ storeToRefs 讓解構後保持響應性（同時只轉 state 和 getter，不轉 action）
const { count, name } = storeToRefs(counterStore)
// 方法（actions）直接從 store 取
const { increment, reset } = counterStore
```

**`storeToRefs` vs `toRefs` 的差異**：
- `toRefs` 會轉換物件的**所有屬性**（包含函式，會造成問題）
- `storeToRefs` 只轉換 **state 和 getter**，自動跳過 actions

這正是今天「函式不應該用 toRef 轉換」的最佳佐證：Pinia 的作者也認為函式不需要 toRef，所以設計了 `storeToRefs` 只處理資料屬性。

---

### 6. toRef vs toRefs：單一屬性 vs 全屬性

| | `toRef` | `toRefs` |
|---|---------|---------|
| 轉換範圍 | 單一屬性 | 所有屬性 |
| 使用場景 | 只暴露部分屬性、或建立跨元件的「指標」| Composable 回傳整個 reactive 物件時 |
| 語法 | `toRef(state, 'count')` | `toRefs(state)` |

```ts
// 只暴露 count，不暴露 internalConfig
const state = reactive({ count: 0, internalConfig: {} })
return {
  count: toRef(state, 'count'),  // 只暴露一個
  increment: () => state.count++
}
```

---

## 作業說明：LifecycleLogger.vue 實作期啟動

### 今日目標

W4 的作業是 **LifecycleLogger.vue**——一個整合生命週期觀察與 watchEffect 追蹤的工具元件（截止 5/18，今日起剩 **4 天**）。

**Day 2（昨日）**已完成：`useWatchEffectLogger.ts` 的設計規範。
**Day 3（今日）**目標：完成完整的元件骨架設計與 `useLifecycleLogger.ts` 的責任邊界規劃。

---

### 元件架構設計

```
LifecycleLogger.vue
├── useLifecycleLogger.ts        ← 生命週期追蹤 Composable
│   ├── logs: ref<LifecycleLog[]>
│   ├── hookCounts: reactive<Record<HookName, number>>
│   └── clearLogs(): void
│
└── useWatchEffectLogger.ts      ← watchEffect 追蹤 Composable（Day 2 設計）
    ├── logs: ref<WatchEffectLog[]>
    ├── cleanupCount: ref<number>
    ├── isAsync: ref<boolean>
    └── flushMode: ref<'pre' | 'post'>
```

---

### useLifecycleLogger.ts 設計規範

```ts
// 型別定義
type HookName =
  | 'onBeforeMount'
  | 'onMounted'
  | 'onBeforeUpdate'
  | 'onUpdated'
  | 'onBeforeUnmount'
  | 'onUnmounted'

interface LifecycleLog {
  id: number
  hook: HookName
  timestamp: number       // Date.now()
  relativeTime: number    // 相對於首次 onBeforeMount 的毫秒數
  domAvailable: boolean   // 當下是否可以操作 DOM？
}

export function useLifecycleLogger() {
  const logs = ref<LifecycleLog[]>([])
  const hookCounts = reactive<Record<HookName, number>>({
    onBeforeMount: 0,
    onMounted: 0,
    onBeforeUpdate: 0,
    onUpdated: 0,
    onBeforeUnmount: 0,
    onUnmounted: 0
  })
  let startTime = 0
  let logId = 0

  function recordHook(hook: HookName, domAvailable: boolean) {
    if (startTime === 0) startTime = Date.now()
    logs.value.push({
      id: logId++,
      hook,
      timestamp: Date.now(),
      relativeTime: Date.now() - startTime,
      domAvailable
    })
    hookCounts[hook]++
  }

  // 各個 Lifecycle Hooks 的記錄
  onBeforeMount(() => recordHook('onBeforeMount', false))
  onMounted(() => recordHook('onMounted', true))
  onBeforeUpdate(() => recordHook('onBeforeUpdate', true))
  onUpdated(() => recordHook('onUpdated', true))
  onBeforeUnmount(() => recordHook('onBeforeUnmount', true))
  onUnmounted(() => recordHook('onUnmounted', false))

  function clearLogs() {
    logs.value = []
    startTime = 0
    logId = 0
    Object.keys(hookCounts).forEach(key => {
      hookCounts[key as HookName] = 0
    })
  }

  // ← 注意：這裡回傳時，logs 是 ref（不需要 toRefs）
  //         hookCounts 是 reactive，但我們回傳整個物件（使用端不需要解構它）
  //         clearLogs 是方法，直接放入
  return {
    logs,
    hookCounts,
    clearLogs
  }
}
```

**回傳設計說明**：
- `logs`：獨立的 `ref`，直接回傳，不需要 `toRefs`
- `hookCounts`：`reactive` 物件，但以整體回傳（使用端直接 `hookCounts.onMounted` 存取，不解構），不需要 `toRefs`
- `clearLogs`：函式，直接回傳

這個設計示範了：**當使用端不需要解構 `reactive` 物件時，也不需要 `toRefs`**——`toRefs` 的目的是「讓解構後保持響應性」，如果使用端不解構，就沒有這個需求。

---

### LifecycleLogger.vue 骨架

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useLifecycleLogger } from './useLifecycleLogger'
import { useWatchEffectLogger } from './useWatchEffectLogger'

// Composable 整合
const { logs: lifecycleLogs, hookCounts, clearLogs } = useLifecycleLogger()
const { logs: watchEffectLogs, cleanupCount, isAsync, flushMode } = useWatchEffectLogger(trackedValue)

// 觀察用的響應式資料（供 watchEffect 追蹤）
const trackedValue = ref(0)
const triggerUpdate = ref(0)

function bumpTrackedValue() {
  trackedValue.value++
}

function forceUpdate() {
  triggerUpdate.value++
}
</script>

<template>
  <div class="lifecycle-logger">
    <!-- 控制面板 -->
    <section class="controls">
      <button @click="bumpTrackedValue">觸發 watchEffect（trackedValue++）</button>
      <button @click="forceUpdate">強制更新（觸發 onUpdated）</button>
      <button @click="clearLogs">清除所有 Log</button>
    </section>

    <!-- 生命週期 Log -->
    <section class="lifecycle-logs">
      <h3>Lifecycle Hooks Log</h3>
      <div v-for="log in lifecycleLogs" :key="log.id">
        [+{{ log.relativeTime }}ms] {{ log.hook }} | DOM: {{ log.domAvailable ? '✅' : '❌' }}
      </div>
    </section>

    <!-- watchEffect Log -->
    <section class="watch-effect-logs">
      <h3>watchEffect Log（Cleanup 計數：{{ cleanupCount }}）</h3>
      <div v-for="(log, i) in watchEffectLogs" :key="i">
        [{{ log.triggerSource }}] cleanupTriggered: {{ log.cleanupTriggered }}
      </div>
    </section>
  </div>
</template>
```

---

### 接下來幾天的實作計畫

| 日期 | 目標 | 說明 |
|------|------|------|
| 5/14（今日）| 骨架設計完成 | 本文件規劃 + useLifecycleLogger.ts 骨架 |
| 5/15（Day 4）| useWatchEffectLogger.ts 完整實作 | 參考 Day 2 設計規範 |
| 5/16（Day 5）| useLifecycleLogger.ts 完整實作 + 模板整合 | hook 記錄正確觸發 |
| 5/17（Day 6）| 功能驗收 + TypeScript 審查 + PR 準備 | vue-tsc 零警告 |
| 5/18（Day 7）| PR 提交截止日 | 最晚提交 |

---

## 自我檢核問題

1. 你在 W3 的 `useTasks` 中回傳時沒有用 `toRefs`，但一樣可以解構使用。這是為什麼？如果改成用 `reactive` 集中管理狀態，回傳方式需要怎麼改？

2. 以下程式碼有什麼問題？
   ```ts
   const state = reactive({ count: 0, name: 'Alice' })
   const refs = toRefs({ count: ref(0), name: ref('Alice') })
   // refs.count.value.value === ???
   ```

3. `storeToRefs` 和 `toRefs` 最大的設計差異是什麼？為什麼 Pinia 不直接用 `toRefs`？

4. 當你設計的 Composable 回傳物件包含 `{ results, loading, search, abort }` 時，哪些需要 `toRef/toRefs` 處理，哪些不需要？原則是什麼？

5. 在 `useLifecycleLogger` 的設計中，`hookCounts` 是 `reactive` 物件，但我們回傳時沒有做 `toRefs`。這樣設計合理嗎？在什麼情況下需要改為 `toRefs(hookCounts)`？

---

## 明日預告（W4 Day 4：2026-05-15）

**主題**：`useWatchEffectLogger.ts` 完整實作 × flush 模式動態切換

**帶著這個問題進入明天**：
> `useWatchEffectLogger` 需要支援「動態切換 flush 模式（pre / post）」——當使用者在 UI 切換 flush 選項時，你需要停止舊的 `watchEffect` 並建立新的。但這個「停止與重建」的邏輯要放在哪裡？是在 Composable 裡用 `watch` 監聽 `flushMode` 的變化，還是由使用端重新呼叫 Composable？
