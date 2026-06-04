# useTasks 核心邏輯完整實作 × 瀏覽器驗證

> **日期**：2026-05-08（M1-W3 Day 4）
> **類型**：核心實作日（骨架 → 可運作元件）
> **對應週次作業**：W3 作業 — TaskList.vue 第二版（截止 5/11，剩 3 天）

---

## 目標技術 & 核心知識點

| 技術 | 本日焦點 |
|------|---------|
| Vue 3 Composition API | 完整 `setup` 實作：從骨架到可運作 |
| `ref` 整批替換 | 邊界處理（空字串、重複 id、不存在的 id）|
| `computed` | 驗證快取行為（filter 切換 → filteredTasks 重算）|
| `watch` | filter 變化時的副作用（localStorage 可選）|
| Vue DevTools | 觀察 ref 更新、computed 快取、事件觸發 |
| TypeScript | 函式簽名、邊界錯誤的型別安全處理 |

**核心知識點**：
- 邊界處理是「防禦性程式設計」的體現：不信任輸入，驗證後再處理
- `v-if` 搭配空狀態讓 UX 完整（空列表、空篩選結果需分開處理）
- Vue DevTools 的 computed 面板能直接觀察 `_dirty flag` 行為
- `@keyup.enter` + `@click` 雙觸發點需確保邏輯一致（都呼叫同一個 `addTask`）

---

## 為什麼學這個（與前幾日內容的連結）

```
W3 Day 1（5/5）：底層機制
│  RefImpl、dirty flag、watch flush、Effect Graph
▼
W3 Day 2（5/6）：傳遞與保存
│  auto-unwrap 8 規則、toRef/toRefs、shallowRef、markRaw
▼
W3 Day 3（5/7）：骨架設計
│  Code Smell 盤點、useTasks 責任邊界、資料流圖、骨架完成
▼
W3 Day 4（5/8）：核心實作 ← 今日
│  骨架 → 完整邏輯 → 接入元件 → 瀏覽器可運作
▼
W3 Day 5（5/9）：邊界強化 + 細節收尾
W3 Day 6（5/10）：Self-review + Vue DevTools 驗證
W3 Day 7（5/11）：PR 提交
```

**今日是 W3 的關鍵里程碑**：Day 4 成功後，TaskList.v2.vue 在瀏覽器中可實際運作。這是 Day 5–6 進行細節優化的前提。

---

## 知識說明

### 1. 邊界處理設計原則

「防禦性程式設計」的核心：**不相信呼叫者傳入的任何值都是合法的**。

對 `useTasks` 來說：

| 函式 | 邊界情境 | 處理方式 |
|------|---------|---------|
| `addTask(text)` | 空字串 / 全空白 | `trim()` 後長度為 0 → 直接 return |
| `addTask(text)` | 重複文字 | 暫不處理（允許重複，靠 id 唯一）|
| `removeTask(id)` | id 不存在 | `filter` 回傳原陣列，無副作用 |
| `toggleTask(id)` | id 不存在 | `map` 回傳原陣列，無副作用 |
| `setFilter(f)` | 非法 FilterType | TypeScript 型別系統在編譯期攔截 |

**重要洞察**：`removeTask` 和 `toggleTask` 用 `filter`/`map` 的天然特性，使「id 不存在」自動成為 no-op（無操作），不需要額外的防禦邏輯。這是函數式風格的優點之一。

---

### 2. 完整 `useTasks.ts` 實作

```typescript
// composables/useTasks.ts
import { ref, computed, watch } from 'vue'
import type { Task, FilterType } from '@/types/task'

export function useTasks() {
  // ── State ──────────────────────────────────────────────────────────
  const tasks = ref<Task[]>([])
  const filter = ref<FilterType>('all')
  const newTaskText = ref<string>('')

  // ── Computed ────────────────────────────────────────────────────────
  const filteredTasks = computed<Task[]>(() => {
    switch (filter.value) {
      case 'active': return tasks.value.filter(t => !t.done)
      case 'done':   return tasks.value.filter(t => t.done)
      default:       return tasks.value   // 'all'
    }
  })

  const totalCount = computed<number>(() => tasks.value.length)

  const doneCount = computed<number>(() =>
    tasks.value.filter(t => t.done).length
  )

  const activeCount = computed<number>(() =>
    tasks.value.filter(t => !t.done).length
  )

  // ── Watch（副作用）──────────────────────────────────────────────────
  watch(filter, (newFilter) => {
    // 儲存篩選偏好至 localStorage（可在 Day 5 擴充 restore 邏輯）
    try {
      localStorage.setItem('task-filter', newFilter)
    } catch {
      // 靜默處理 localStorage 不可用的情境（如 SSR、隱私模式）
    }
  })

  // ── Methods ─────────────────────────────────────────────────────────

  /**
   * 新增任務
   * @param text - 任務文字（自動 trim，空字串忽略）
   */
  function addTask(text: string): void {
    const trimmed = text.trim()
    if (!trimmed) return                   // 邊界：空字串 / 純空白

    tasks.value = [
      ...tasks.value,
      {
        id: crypto.randomUUID(),           // 唯一 id（非遞增整數，避免競態）
        text: trimmed,
        done: false,
        createdAt: Date.now()
      }
    ]
    newTaskText.value = ''                 // 清空輸入框（副作用：在 method 中處理）
  }

  /**
   * 刪除任務
   * @param id - 要刪除的任務 id（不存在則為 no-op）
   */
  function removeTask(id: string): void {
    tasks.value = tasks.value.filter(t => t.id !== id)
  }

  /**
   * 切換任務完成狀態
   * @param id - 要切換的任務 id（不存在則為 no-op）
   */
  function toggleTask(id: string): void {
    tasks.value = tasks.value.map(t =>
      t.id === id
        ? { ...t, done: !t.done }
        : t
    )
  }

  /**
   * 設定篩選模式
   * @param f - FilterType（TypeScript 型別系統確保合法值）
   */
  function setFilter(f: FilterType): void {
    filter.value = f
  }

  /**
   * 清除所有已完成任務（擴充功能）
   */
  function clearDone(): void {
    tasks.value = tasks.value.filter(t => !t.done)
  }

  // ── Return ───────────────────────────────────────────────────────────
  return {
    // State（直接回傳 ref，解構後保持響應性）
    tasks,
    filter,
    newTaskText,
    // Computed（ComputedRef，也是 ref）
    filteredTasks,
    totalCount,
    doneCount,
    activeCount,
    // Methods
    addTask,
    removeTask,
    toggleTask,
    setFilter,
    clearDone
  }
}
```

---

### 3. 完整 `TaskList.v2.vue` 實作

```vue
<!-- components/TaskList.v2.vue -->
<script setup lang="ts">
import { useTasks } from '@/composables/useTasks'

const {
  filter,
  newTaskText,
  filteredTasks,
  totalCount,
  doneCount,
  activeCount,
  addTask,
  removeTask,
  toggleTask,
  setFilter,
  clearDone
} = useTasks()

// 篩選按鈕清單（避免模板 inline 陣列）
const FILTER_OPTIONS = [
  { value: 'all',    label: '全部' },
  { value: 'active', label: '未完成' },
  { value: 'done',   label: '已完成' },
] as const
</script>

<template>
  <div class="task-list">

    <!-- ── 輸入區 ── -->
    <div class="task-input">
      <input
        v-model="newTaskText"
        placeholder="輸入任務名稱..."
        @keyup.enter="addTask(newTaskText)"
      />
      <button @click="addTask(newTaskText)">新增</button>
    </div>

    <!-- ── 篩選器 ── -->
    <div class="task-filter">
      <button
        v-for="opt in FILTER_OPTIONS"
        :key="opt.value"
        :class="['filter-btn', { active: filter === opt.value }]"
        @click="setFilter(opt.value)"
      >
        {{ opt.label }}
      </button>
    </div>

    <!-- ── 任務列表（有任務時）── -->
    <ul v-if="filteredTasks.length > 0" class="task-items">
      <li
        v-for="task in filteredTasks"
        :key="task.id"
        :class="['task-item', { done: task.done }]"
      >
        <input
          type="checkbox"
          :checked="task.done"
          @change="toggleTask(task.id)"
        />
        <span class="task-text">{{ task.text }}</span>
        <button class="remove-btn" @click="removeTask(task.id)">刪除</button>
      </li>
    </ul>

    <!-- ── 空狀態（有任務但篩選無結果）── -->
    <div v-else-if="totalCount > 0" class="empty-filter">
      <p>篩選條件下沒有任務 🔍</p>
      <button @click="setFilter('all')">查看全部</button>
    </div>

    <!-- ── 空狀態（尚無任何任務）── -->
    <div v-else class="empty-list">
      <p>目前沒有任務，從上方新增一個吧！✨</p>
    </div>

    <!-- ── 統計列 ── -->
    <div class="task-stats">
      <span>共 {{ totalCount }} 項・完成 {{ doneCount }} 項・剩餘 {{ activeCount }} 項</span>
      <button
        v-if="doneCount > 0"
        class="clear-done-btn"
        @click="clearDone"
      >
        清除已完成
      </button>
    </div>

  </div>
</template>
```

---

### 4. 空狀態設計：三種情境

今日加入的空狀態分為三種，需用 `v-if / v-else-if / v-else` 條件鏈處理：

```
情境 1：totalCount === 0 且 filteredTasks.length === 0
        → 尚無任何任務（第一次進入頁面）
        → 顯示「新增任務」引導文字

情境 2：totalCount > 0 且 filteredTasks.length === 0
        → 有任務，但目前篩選條件下沒有符合的
        → 顯示「篩選無結果」提示 + 快速回到「全部」的按鈕

情境 3：filteredTasks.length > 0
        → 正常顯示任務列表
```

**為什麼要區分情境 1 和情境 2？**
- 情境 1 的使用者是「新用戶 / 空狀態」，需要引導行動
- 情境 2 的使用者「知道有任務，只是看不到」，需要解除篩選的路徑

---

### 5. Vue DevTools 觀察指南

安裝 Vue DevTools 後，開啟 Components 面板，選取 `<TaskList>` 元件：

**觀察點 1：computed 快取驗證**
```
操作步驟：
1. 新增 3 個任務
2. 在 DevTools 觀察 filteredTasks（computed）
3. 切換 filter 為 'active'
   → filteredTasks 應重新計算（dirty flag 觸發）
4. 再次點擊 'active'（不切換）
   → filteredTasks 不重計算（快取命中，無閃爍）

預期結果：只有在 tasks 或 filter 改變時，filteredTasks 才重算。
```

**觀察點 2：ref 整批替換驗證**
```
操作步驟：
1. 在 DevTools 的 Setup 區看 tasks ref
2. 新增一個任務
   → tasks.value 的陣列長度應增加 1
   → 注意：整個陣列是新物件（非原陣列 push）

3. 刪除一個任務
   → tasks.value 是全新陣列（filter 結果）
```

**觀察點 3：watch 副作用驗證**
```
操作步驟：
1. 開啟 DevTools > Application > Local Storage
2. 切換 filter 到 'active'
   → localStorage 應出現 key: 'task-filter', value: 'active'
3. 切換 filter 到 'done'
   → localStorage 的值應更新為 'done'
```

---

### 6. 型別安全的細節強化

```typescript
// types/task.ts（完整版）
export type FilterType = 'all' | 'active' | 'done'

export interface Task {
  id: string           // crypto.randomUUID() → UUID v4 字串
  text: string         // 任務文字（已 trim）
  done: boolean        // 完成狀態
  createdAt: number    // Date.now() 時間戳（ms）
}
```

**為什麼 `createdAt` 用 `number` 而非 `Date`？**
- `number` 是 JSON 序列化友好的格式（localStorage / API 傳輸不需 parse）
- 顯示時再 `new Date(task.createdAt).toLocaleDateString()` 轉換即可
- `Date` 物件無法直接做 JSON 序列化，反序列化後會變成 string

---

### 7. 常見陷阱（今日實作常見問題）

**陷阱 1：`v-model` 與 `ref` 的同步**

```vue
<!-- ✅ 正確：v-model 自動連結 ref.value -->
<input v-model="newTaskText" />

<!-- ⚠️ 容易忽略：addTask 後要清空 newTaskText -->
function addTask(text: string): void {
  const trimmed = text.trim()
  if (!trimmed) return
  // ... 新增任務 ...
  newTaskText.value = ''   // ← 這一行很容易漏掉
}
```

**陷阱 2：`:key` 必須是唯一且穩定的值**

```vue
<!-- ❌ 錯誤：用 index 當 key，刪除 / 重排時 Vue diff 效率差 -->
<li v-for="(task, index) in filteredTasks" :key="index">

<!-- ✅ 正確：用 task.id（UUID，唯一且穩定）-->
<li v-for="task in filteredTasks" :key="task.id">
```

**陷阱 3：checkbox 用 `:checked` + `@change` 而非 `v-model`**

```vue
<!-- ❌ 不適合：v-model 需要雙向綁定，但 task.done 是 readonly computed 衍生值 -->
<input type="checkbox" v-model="task.done" />

<!-- ✅ 正確：單向綁定 + 事件觸發 Composable 方法 -->
<input
  type="checkbox"
  :checked="task.done"
  @change="toggleTask(task.id)"
/>
```

> **為什麼不能直接 `v-model="task.done"`？**
> `filteredTasks` 是 `computed` 回傳的陣列，任務物件不應被直接 mutate（違反單向資料流）。應透過 `toggleTask(id)` 更新 `tasks` ref，再由 `filteredTasks` 自動重算。

---

## 作業說明（持續進行中）

### 今日實作目標

| 目標 | 說明 | 完成標準 |
|------|------|---------|
| 完成 `useTasks.ts` | 含邊界處理、`clearDone` 方法 | TypeScript 零紅線 |
| 完成 `TaskList.v2.vue` | 含三種空狀態、統計列、清除按鈕 | 功能可運作 |
| 接入元件 | 在頁面中使用 `<TaskList>` | 瀏覽器可見 |
| Vue DevTools 驗證 | 觀察 computed 快取行為 | 行為符合預期 |

### 今日成功標準

> 在瀏覽器中能：
> - ✅ 新增任務（輸入 + Enter 或點按鈕）
> - ✅ 勾選完成（checkbox → done 狀態切換）
> - ✅ 刪除任務
> - ✅ 切換篩選器（全部 / 未完成 / 已完成）
> - ✅ 統計數字隨操作即時更新
> - ✅ 空狀態正確顯示
> - ✅ Vue DevTools 中 computed 快取行為符合預期

---

## 自我檢核問題

1. **`addTask` 在什麼情況下應該 return 不執行？** 除了空字串，還有哪些你認為應該加入的防禦邏輯？（提示：長度上限？重複文字？）

2. **`removeTask` 和 `toggleTask` 都使用 `filter`/`map` 回傳新陣列，而非直接 mutation。如果改成 `tasks.value.splice()` 或 `tasks.value[index].done = true`，會發生什麼？** 請分別解釋 `ref` 和 `reactive` 在這兩種情況下的行為差異。

3. **三種空狀態（無任務 / 篩選無結果 / 有列表）的 `v-if` 鏈順序可以改變嗎？** 為什麼 `v-else-if="totalCount > 0"` 放在 `v-else-if="filteredTasks.length > 0"` 之後？

4. **在 `watch(filter, ...)` 的 callback 中，`try/catch` 包住 `localStorage.setItem` 有必要嗎？** 什麼情境下 localStorage 會拋出例外？

5. **Vue DevTools 的 computed 快取驗證：切換相同的 filter（例如已在 'active' 再點一次 'active'），`filteredTasks` 會重算嗎？** 從 `watch` + `dirty flag` 的角度解釋原因。

---

## 明日預告（5/9，W3 Day 5）

**主題**：邊界強化 + LocalStorage 持久化 + Self-review

重點方向：
- 實作 localStorage restore：`onMounted` 讀取已儲存的 filter 和 tasks
- 任務文字長度上限（50 字元限制）× 即時字數提示
- Self-review：對照 W3 作業評分標準（100 分制）自評
- 補充：`v-transition` 讓任務新增 / 刪除有動畫效果（可選）

> Day 5 目標：讓 TaskList.v2.vue 具備「重整頁面後資料不遺失」的能力，並完成自評，為 Day 6 的最終 review 做準備。
