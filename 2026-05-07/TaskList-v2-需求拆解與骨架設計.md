# TaskList.vue 第二版 — 需求拆解與骨架設計

> **日期**：2026-05-07（M1-W3 Day 3）
> **類型**：實作期啟動日（理論轉實作）
> **對應週次作業**：W3 作業 — TaskList.vue 第二版（截止 5/11）

---

## 目標技術 & 核心知識點

| 技術 | 本日焦點 |
|------|---------|
| Vue 3 Composition API | `setup` 內的資料流設計：State → Computed → Watch → Render |
| `ref` / `reactive` | 在元件與 Composable 中正確選型 |
| `computed` / `watch` | 篩選邏輯、副作用分離設計 |
| Composable 設計 | `useTasks` 責任邊界、對外介面（`toRefs` 回傳模式）|
| TypeScript 基礎 | Props 型別、Task 資料型別定義 |

**核心知識點**：
- Composable 的責任邊界 = 「狀態 + 衍生狀態 + 副作用」，不包含 UI 邏輯
- 資料流方向：單向（State → Computed → UI），副作用由 Watch 處理
- `useTasks` 應對外暴露：狀態（`toRefs`）+ 方法（純函式）
- Code Smell 識別：在元件直接暴露 reactive 物件、watch 分散在元件層

---

## 為什麼學這個（與前幾日內容的連結）

```
W3 Day 1（5/5）：底層機制
│  ref 的 RefImpl、dirty flag、watch flush、Effect Graph
▼
W3 Day 2（5/6）：傳遞與保存
│  auto-unwrap 8 規則、toRef/toRefs、shallowRef、markRaw
▼
W3 Day 3（5/7）：實作應用 ← 今日
│  將所有理論知識應用到真實元件設計上
│  理論的終點 = TaskList.vue 第二版的骨架
▼
W3 Day 4–5（5/8–5/9）：核心功能實作
W3 Day 6（5/10）：Self-review + 測試
W3 Day 7（5/11）：PR 提交
```

**W1 版的問題根源**，正是 Day 1–2 所學到的陷阱的具體展現：
- 用 `reactive({ list: [] })` 包陣列 → 解構失去響應性（W3 Day 1：Proxy 不能複製）
- 沒有 `computed` 篩選器 → 把篩選邏輯寫在模板，難以維護
- `watch` 散落在元件頂層 → 副作用與 UI 混雜，測試困難
- 沒有 Composable → 邏輯無法複用

今日的設計就是「以正確的方式重建這些結構」。

---

## 知識說明

### 1. Code Smell 盤點（W1 版 TaskList.vue 假設缺陷）

W1 階段尚未學習 Composable 與 computed，常見的 W1 版本模式：

```vue
<!-- ❌ W1 版常見問題 -->
<script setup lang="ts">
import { reactive, watch } from 'vue'

// 問題 1：用 reactive 包陣列，解構後失去響應性
const state = reactive({
  tasks: [],
  filter: 'all'
})

// 問題 2：watch 散落在 setup 頂層，難以知道它在監控什麼
watch(() => state.filter, () => {
  console.log('filter changed')
})

// 問題 3：篩選邏輯直接在元件，無法複用
const getFiltered = () => {
  if (state.filter === 'active') return state.tasks.filter(t => !t.done)
  if (state.filter === 'done') return state.tasks.filter(t => t.done)
  return state.tasks
}
</script>

<template>
  <!-- 問題 4：模板直接呼叫函式（每次 render 都執行，沒有快取）-->
  <li v-for="task in getFiltered()">...</li>
</template>
```

**W1 版問題總結**：

| 問題 | 根本原因 | 正確解法 |
|------|---------|---------|
| `reactive` 包陣列，解構後更新不觸發 | Proxy 不能追蹤複製後的參考 | 用 `ref<Task[]>([])` 或 `toRefs` 正確傳遞 |
| `getFiltered()` 在模板呼叫 | 函式每次 render 重新執行，無快取 | 改成 `computed(filteredTasks)` |
| `watch` 分散在元件頂層 | 副作用與 UI 混雜 | 封裝進 `useTasks` Composable |
| 邏輯無法複用 | 沒有抽取成 Composable | 設計 `useTasks.ts` |

---

### 2. 設計 `useTasks` Composable 的責任邊界

**Composable 只管**：狀態、衍生狀態、副作用、對外方法。**不管**：UI 顯示、CSS、事件綁定細節。

```
useTasks 責任圖
─────────────────────────────────────────
  State（ref）
  ├── tasks: ref<Task[]>([])        ← 主列表，用 ref 而非 reactive（整批替換）
  ├── filter: ref<FilterType>('all') ← 篩選狀態
  └── newTaskText: ref<string>('')   ← 輸入框狀態

  Computed（衍生，有快取）
  ├── filteredTasks                  ← 由 tasks + filter 衍生
  ├── totalCount                     ← tasks.length
  └── doneCount                      ← tasks.filter(t => t.done).length

  Watch（副作用）
  └── watch(filter) → 儲存至 localStorage（可選）

  Methods（純函式，不含 UI）
  ├── addTask(text: string): void
  ├── removeTask(id: string): void
  ├── toggleTask(id: string): void
  └── setFilter(f: FilterType): void
─────────────────────────────────────────
  對外回傳（toRefs 模式）
  return {
    ...toRefs(state),   // ← 解構安全
    filteredTasks,
    totalCount,
    doneCount,
    addTask,
    removeTask,
    toggleTask,
    setFilter
  }
```

---

### 3. 型別定義（TypeScript）

```typescript
// types/task.ts

export type FilterType = 'all' | 'active' | 'done'

export interface Task {
  id: string
  text: string
  done: boolean
  createdAt: number
}
```

**為什麼 `id` 用 `string` 而非 `number`？**
- `crypto.randomUUID()` 回傳 UUID 字串，避免手動遞增 ID 的競態問題
- 序列化到 localStorage 時不會有整數精度問題

---

### 4. 資料流圖（Data Flow Diagram）

```
使用者輸入
    │
    ▼
[ newTaskText: ref<string> ]
    │ addTask()
    ▼
[ tasks: ref<Task[]> ] ──────────────────────────────────┐
    │                                                    │
    │ computed                                           │ watch（副作用）
    ▼                                                    ▼
[ filteredTasks ]                              localStorage.setItem(...)
[ totalCount   ]
[ doneCount    ]
    │
    ▼
[ <TaskList> 模板 ]
    │
    ├── v-for="task in filteredTasks"
    ├── @change → toggleTask(task.id)
    └── @click → removeTask(task.id)

[ filter: ref<FilterType> ]
    │ setFilter()
    ├──▶ filteredTasks（computed 依賴）自動重算
    └──▶ watch(filter) → localStorage（副作用）
```

**響應式工具選擇說明**：

| 資料 | 工具 | 理由 |
|------|------|------|
| `tasks` | `ref<Task[]>` | 陣列需整批替換（filter/add/remove 都回傳新陣列），適合 ref |
| `filter` | `ref<FilterType>` | 基本型別（string union），ref 最直觀 |
| `newTaskText` | `ref<string>` | 基本型別 |
| `filteredTasks` | `computed` | 由 tasks + filter 衍生，需快取，不重複計算 |
| `totalCount` | `computed` | 同上 |
| `doneCount` | `computed` | 同上 |

---

### 5. 骨架程式碼

#### `useTasks.ts`

```typescript
// composables/useTasks.ts
import { ref, computed, watch, reactive, toRefs } from 'vue'
import type { Task, FilterType } from '@/types/task'

export function useTasks() {
  // ── State ──────────────────────────────────────────
  const tasks = ref<Task[]>([])
  const filter = ref<FilterType>('all')
  const newTaskText = ref<string>('')

  // ── Computed ────────────────────────────────────────
  const filteredTasks = computed<Task[]>(() => {
    switch (filter.value) {
      case 'active': return tasks.value.filter(t => !t.done)
      case 'done':   return tasks.value.filter(t => t.done)
      default:       return tasks.value
    }
  })

  const totalCount = computed(() => tasks.value.length)
  const doneCount  = computed(() => tasks.value.filter(t => t.done).length)

  // ── Watch（副作用）─────────────────────────────────
  watch(filter, (newFilter) => {
    // 儲存篩選狀態（可選，後續可擴展）
    // localStorage.setItem('task-filter', newFilter)
  })

  // ── Methods ─────────────────────────────────────────
  function addTask(text: string): void {
    const trimmed = text.trim()
    if (!trimmed) return
    tasks.value = [
      ...tasks.value,
      {
        id: crypto.randomUUID(),
        text: trimmed,
        done: false,
        createdAt: Date.now()
      }
    ]
    newTaskText.value = ''
  }

  function removeTask(id: string): void {
    tasks.value = tasks.value.filter(t => t.id !== id)
  }

  function toggleTask(id: string): void {
    tasks.value = tasks.value.map(t =>
      t.id === id ? { ...t, done: !t.done } : t
    )
  }

  function setFilter(f: FilterType): void {
    filter.value = f
  }

  // ── Return（toRefs 回傳模式）────────────────────────
  return {
    tasks,           // ref（直接回傳，不需 toRefs）
    filter,
    newTaskText,
    filteredTasks,   // computed ref
    totalCount,
    doneCount,
    addTask,
    removeTask,
    toggleTask,
    setFilter
  }
}
```

> **為什麼這裡不需要 `toRefs`？**
> `toRefs` 用於把 `reactive` 物件的屬性轉為 ref，以便解構時保持響應性。
> 這裡 `tasks`、`filter`、`newTaskText` 本來就是 `ref`，直接回傳即可安全解構。
> `toRefs` 才是在「`reactive` + 解構」的情況下需要的。

---

#### `TaskList.v2.vue`（骨架）

```vue
<!-- components/TaskList.v2.vue -->
<script setup lang="ts">
import { useTasks } from '@/composables/useTasks'

// 解構 Composable 回傳值（全部都是 ref，解構安全）
const {
  tasks,
  filter,
  newTaskText,
  filteredTasks,
  totalCount,
  doneCount,
  addTask,
  removeTask,
  toggleTask,
  setFilter
} = useTasks()
</script>

<template>
  <div class="task-list">
    <!-- 輸入區 -->
    <div class="task-input">
      <input
        v-model="newTaskText"
        placeholder="新增任務..."
        @keyup.enter="addTask(newTaskText)"
      />
      <button @click="addTask(newTaskText)">新增</button>
    </div>

    <!-- 篩選器 -->
    <div class="task-filter">
      <button
        v-for="f in (['all', 'active', 'done'] as const)"
        :key="f"
        :class="{ active: filter === f }"
        @click="setFilter(f)"
      >
        {{ f === 'all' ? '全部' : f === 'active' ? '未完成' : '已完成' }}
      </button>
    </div>

    <!-- 任務列表 -->
    <ul class="task-items">
      <li
        v-for="task in filteredTasks"
        :key="task.id"
        :class="{ done: task.done }"
      >
        <input
          type="checkbox"
          :checked="task.done"
          @change="toggleTask(task.id)"
        />
        <span>{{ task.text }}</span>
        <button @click="removeTask(task.id)">刪除</button>
      </li>
    </ul>

    <!-- 統計 -->
    <div class="task-stats">
      <span>共 {{ totalCount }} 項，已完成 {{ doneCount }} 項</span>
    </div>
  </div>
</template>
```

---

### 6. 常見陷阱提醒

**陷阱 1：`tasks.value.push()` 不能觸發更新（用 ref 的情況）**

```typescript
// ❌ 錯誤：直接 mutation，ref 的 .value 沒有變（位址沒變）
tasks.value.push(newTask)  // ref 不追蹤內部 mutation

// ✅ 正確：整批替換（ref 觸發的機制是 .value 的位址變化）
tasks.value = [...tasks.value, newTask]
```

> 補充：如果用 `reactive({ list: [] })`，push 是可以的，因為 reactive 使用 Proxy 代理陣列的 push 方法。但我們選 ref 的原因是「整批替換」的語義更清晰，且在 TypeScript 中型別推導更乾淨。

**陷阱 2：`computed` 回傳的 ref 要加 `.value`（在 `<script>` 中）**

```typescript
// ✅ 在 script 中需要 .value
console.log(filteredTasks.value.length)

// ✅ 在 template 中不需要（auto-unwrap）
// <span>{{ filteredTasks.length }}</span>
```

**陷阱 3：`useTasks()` 必須在 `setup()` / `<script setup>` 的同步頂層呼叫**

```typescript
// ❌ 錯誤：在非同步後呼叫（Active Instance 已失效）
async function init() {
  await fetchSomething()
  const { tasks } = useTasks()  // ⚠️ onMounted 等 Lifecycle Hook 無法正常運作
}

// ✅ 正確：在 setup 頂層同步呼叫
const { tasks, addTask } = useTasks()
```

---

## 作業說明

### 情境背景

你正在開發 ihouseBMS 的「待辦任務」模組。PM 要求：
- 房仲可以新增、完成、刪除任務
- 可依「全部 / 未完成 / 已完成」篩選
- 顯示任務統計（總數、完成數）
- 邏輯需抽離為 Composable，方便未來其他頁面複用

### 功能需求

| 功能 | 說明 |
|------|------|
| 新增任務 | 輸入文字後按 Enter 或點擊「新增」按鈕 |
| 完成任務 | 勾選 checkbox，`done` 改為 `true` |
| 刪除任務 | 點擊刪除按鈕，從列表移除 |
| 篩選任務 | 三種模式：all / active / done |
| 統計顯示 | 「共 N 項，已完成 M 項」 |
| 空狀態 | 列表為空時顯示提示文字 |

### 技術規範

| 規範 | 要求 |
|------|------|
| 響應式工具 | 任務列表用 `ref<Task[]>`，篩選用 `ref<FilterType>` |
| 衍生狀態 | 篩選結果、計數一律用 `computed` |
| 邏輯封裝 | 所有狀態與方法封裝於 `useTasks.ts` |
| 型別安全 | `Task` 介面、`FilterType` 聯合型別完整定義 |
| 副作用隔離 | `watch` 只放在 Composable 中，不放在元件 |
| 不可 mutation | 陣列操作使用展開語法或 filter/map，不直接 push/splice |

### 評分標準

| 項目 | 配分 | 說明 |
|------|:----:|------|
| 功能完整性 | 30 | 6 個功能需求全部可運作 |
| 響應式工具選型正確 | 20 | ref/computed/watch 選用有說明理由 |
| Composable 設計 | 20 | 責任邊界清晰，對外介面乾淨 |
| TypeScript 型別完整 | 15 | Task 介面、FilterType、函式簽名皆有型別 |
| 不直接 mutation | 10 | 無 push/splice，全用展開語法 |
| 程式碼可讀性 | 5 | 命名一致、縮排整齊、適當註解 |
| **總分** | **100** | |

---

## 自我檢核問題

1. **為什麼 `tasks` 選用 `ref<Task[]>` 而非 `reactive({ list: Task[] })`？** 請從「整批替換 vs 局部 mutation」的角度說明，並解釋各自的 trigger 機制差異。

2. **`filteredTasks` 為什麼要用 `computed` 而非普通函式？** 如果把它改成 `const getFilteredTasks = () => { ... }` 並在模板中 `v-for="task in getFilteredTasks()"` 呼叫，會有什麼效能問題？

3. **`useTasks` 回傳時直接寫 `{ tasks, filter, ... }` 而非 `{ ...toRefs(state) }`，兩者的本質差異是什麼？** 什麼情況下才需要用 `toRefs`？

4. **若需要將任務列表持久化到 `localStorage`，應該把這個邏輯放在 `useTasks.ts` 還是 `TaskList.v2.vue`？** 理由是什麼？

5. **`toggleTask` 使用 `map` 回傳新陣列，而非直接 `task.done = !task.done`。** 除了 ref 的 trigger 機制外，還有什麼設計理由支持不可變更新（immutable update）？

---

## 明日預告（5/8，W3 Day 4）

**主題**：`useTasks` 完整實作 × 測試自己的設計

重點方向：
- 完成 `addTask` / `removeTask` / `toggleTask` 的邊界處理（空字串、不存在 id）
- 完成篩選 `computed` + 統計 `computed`
- 在 `TaskList.v2.vue` 接入 Composable，驗證整個資料流
- 加入空狀態（`v-if filteredTasks.length === 0` → 顯示提示）
- 第一次手動測試：用 Vue DevTools 觀察 computed 快取是否正常（filter 切換時 filteredTasks 是否重算）

> Day 4 目標：讓 TaskList.v2.vue 在瀏覽器中**實際可運作**。不追求樣式，追求邏輯正確性。
