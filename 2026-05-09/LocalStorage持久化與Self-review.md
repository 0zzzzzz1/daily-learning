# LocalStorage 持久化回路 × 字數限制 × W3 Self-review

> **日期**：2026-05-09（W3 Day 5）
> **類型**：實作深化 + 自我評量
> **前置知識**：useTasks.ts 核心邏輯（Day 4 完成）、watch 副作用寫入 localStorage
> **截止提醒**：⚠️ W3 作業截止 5/11，剩 **2 天**

---

## 目標技術

| 技術 | 核心 API / 概念 |
|------|----------------|
| Vue 3 Lifecycle Hooks | `onMounted`：DOM 掛載後安全存取瀏覽器 API |
| Composable 設計 | 在 `useTasks` 中封裝持久化讀取邏輯 |
| TypeScript | 型別守衛（Array.isArray）× JSON.parse 安全處理 |
| 響應式 API 整合 | `watch`（寫入）+ `onMounted`（讀取）= 完整持久化回路 |

---

## 核心知識點

- `onMounted` 在 Composable 中的應用原則
- localStorage 安全讀取的三個風險點
- 型別驗證：從 JSON 反序列化後如何確保資料結構正確
- 字數 computed 設計：即時字元計數 + 上限保護
- W3 作業自評標準（100 分制）

---

## 為什麼學這個

昨日（Day 4）已完成：
```
使用者操作 → useTasks 更新 tasks.value → watch 副作用 → 寫入 localStorage
```

這是持久化的**寫入端**，但缺少**讀取端**：

```
App 啟動 → ??? → 恢復上次任務資料
```

缺少 `onMounted`，使用者每次重整都會看到空白的任務清單，localStorage 裡的資料永遠沒有機會發揮作用。

今日的目標是封閉這個回路，讓「數據在頁面間存活」真正成立。

---

## 知識說明

### 一、完整持久化回路設計

```
App 啟動
│  └─ onMounted（在 useTasks 中）
│      ├─ 從 localStorage 讀取 tasks
│      │   ├─ 安全解析（JSON.parse + try/catch）
│      │   └─ 型別驗證（Array.isArray + 欄位檢查）
│      └─ 從 localStorage 讀取 filter
│          └─ 合法值驗證（'all' | 'active' | 'done'）
│
使用者操作（Day 4 已完成）
│  ├─ addTask / removeTask / toggleTask / clearDone
│  └─ setFilter
│
watch（Day 4 已完成）
│  ├─ watch(tasks) → JSON.stringify → localStorage.setItem('tasks', ...)
│  └─ watch(filter) → localStorage.setItem('filter', ...)（建議也持久化）
│
重整頁面（循環開始）
└─ onMounted 再次執行 → 讀取上次儲存的資料
```

---

### 二、`onMounted` 在 Composable 中的正確用法

```typescript
// useTasks.ts
import { ref, computed, watch, onMounted } from 'vue'

export function useTasks() {
  const STORAGE_KEY_TASKS = 'vue3-tasks'
  const STORAGE_KEY_FILTER = 'vue3-filter'

  const tasks = ref<Task[]>([])
  const filter = ref<FilterType>('all')

  // onMounted 在 Composable 中可以正常使用
  // 條件：必須在 setup() 執行期間呼叫 useTasks()（Vue 的 Active Instance 機制）
  onMounted(() => {
    // 讀取任務
    const savedTasks = loadFromStorage<Task[]>(STORAGE_KEY_TASKS)
    if (savedTasks && Array.isArray(savedTasks)) {
      // 型別守衛：確認每個元素都有必要欄位
      const validTasks = savedTasks.filter(isValidTask)
      tasks.value = validTasks
    }

    // 讀取篩選器
    const savedFilter = loadFromStorage<string>(STORAGE_KEY_FILTER)
    if (savedFilter && isValidFilter(savedFilter)) {
      filter.value = savedFilter
    }
  })

  // ... 其餘邏輯
}
```

**關鍵原則**：`onMounted` 和 `watch` 都必須在 `setup()` 的同步執行流中呼叫，Composable 函式的呼叫時機必須在元件 `setup()` 中，才能讓 Vue 正確綁定生命週期。

---

### 三、localStorage 安全讀取的三個風險點

```typescript
// 輔助函式：安全讀取 localStorage
function loadFromStorage<T>(key: string): T | null {
  try {
    // 風險 1：localStorage 不存在（SSR、隱私模式）
    const raw = localStorage.getItem(key)

    // 風險 2：key 不存在，getItem 回傳 null
    if (raw === null) return null

    // 風險 3：JSON 損壞（手動編輯、舊版格式）
    return JSON.parse(raw) as T
  } catch {
    return null
  }
}
```

| 風險 | 情境 | 處理方式 |
|------|------|---------|
| localStorage 不存在 | SSR、Safari 隱私模式 | try/catch 包住整個操作 |
| key 不存在 | 初次使用、清除快取 | `=== null` 檢查 |
| JSON 格式損壞 | 手動改過 localStorage、新舊版本格式不符 | JSON.parse 拋出例外，catch 回傳 null |

---

### 四、型別守衛：確保讀取到的資料結構正確

從 localStorage 讀取後，TypeScript 的型別推斷是「信任程式設計師的斷言」（`as T`），執行期實際上是 `any`。需要額外驗證：

```typescript
// Task 介面
interface Task {
  id: string
  text: string
  done: boolean
  createdAt: number
}

// 型別守衛：確認物件有 Task 需要的所有欄位
function isValidTask(item: unknown): item is Task {
  return (
    typeof item === 'object' &&
    item !== null &&
    typeof (item as Task).id === 'string' &&
    typeof (item as Task).text === 'string' &&
    typeof (item as Task).done === 'boolean' &&
    typeof (item as Task).createdAt === 'number'
  )
}

// 篩選器值守衛
const VALID_FILTERS = ['all', 'active', 'done'] as const
type FilterType = typeof VALID_FILTERS[number]

function isValidFilter(value: string): value is FilterType {
  return (VALID_FILTERS as readonly string[]).includes(value)
}
```

**為什麼需要型別守衛？**

假設使用者之前使用舊版本的 TaskList，localStorage 中的任務格式可能沒有 `createdAt` 欄位，或者 `id` 是數字而非字串。直接用 `as Task` 斷言而不驗證，可能導致執行期錯誤（例如對 `undefined.createdAt` 做計算）。

---

### 五、字數限制：即時計算 + 上限保護

```typescript
// useTasks.ts 中（或直接在元件 setup 中管理）
const MAX_TEXT_LENGTH = 50

const newTaskText = ref('')

// computed 計算剩餘字數
const remainingChars = computed(() => MAX_TEXT_LENGTH - newTaskText.value.length)
const isAtLimit = computed(() => remainingChars.value <= 0)

// addTask 中加入上限防禦
function addTask() {
  const text = newTaskText.value.trim()
  if (!text || text.length > MAX_TEXT_LENGTH) return  // 超限也阻擋

  tasks.value = [
    ...tasks.value,
    {
      id: crypto.randomUUID(),
      text,
      done: false,
      createdAt: Date.now()
    }
  ]
  newTaskText.value = ''
}
```

```vue
<!-- TaskList.v2.vue 中的輸入區域 -->
<div class="input-area">
  <input
    v-model="newTaskText"
    :maxlength="MAX_TEXT_LENGTH"
    placeholder="輸入新任務..."
    @keyup.enter="addTask"
  />
  <span :class="{ 'near-limit': remainingChars <= 10, 'at-limit': isAtLimit }">
    {{ remainingChars }} / {{ MAX_TEXT_LENGTH }}
  </span>
  <button @click="addTask" :disabled="isAtLimit || !newTaskText.trim()">
    新增
  </button>
</div>
```

**設計決策**：
- `:maxlength` 在 HTML 層截斷輸入（使用者無法輸入超過 50 字元）
- `addTask` 仍加入 `text.length > MAX_TEXT_LENGTH` 防禦（避免程式呼叫時跳過 HTML 驗證）
- `remainingChars <= 10` 時顯示警示色（接近上限的漸進視覺提示）

---

### 六、W3 作業 Self-review 評分標準

W3 作業：**TaskList.vue 第二版（整合 ref/reactive/computed/watch）**

| 評分面向 | 滿分 | 評分重點 |
|---------|------|---------|
| **Composable 設計**（useTasks）| 25 分 | 責任邊界清晰、State/Computed/Watch/Methods 各司其職、return 介面乾淨 |
| **響應式 API 正確使用** | 20 分 | `ref` 整批替換語義、computed 不可 mutate、watch 副作用設計 |
| **TypeScript 型別完整性** | 15 分 | Task 介面、FilterType、函式簽名全部標注、無 `any` |
| **UI/UX 實作品質** | 20 分 | 三種空狀態、checkbox 正確模式、字數提示、統計列 |
| **持久化設計** | 10 分 | watch 寫入 + onMounted 讀取 + try/catch 安全處理 |
| **程式碼可讀性** | 10 分 | 變數命名語義清晰、函式職責單一、無多餘的複雜度 |

> **目標分數**：≥ 75 分
> **Day 5 成功標準**：持久化完整（重整後任務和篩選器都恢復），自評分數誠實填寫

---

## 作業說明

### W3 Day 5 任務

在 Day 4 完成的 `useTasks.ts` 和 `TaskList.v2.vue` 基礎上，繼續實作：

#### 任務 A：完成持久化讀取（必做）

在 `useTasks.ts` 的 `onMounted` 中實作：
1. 從 `localStorage` 讀取 `tasks`，通過型別守衛驗證後寫入 `tasks.value`
2. 從 `localStorage` 讀取 `filter`，通過合法值驗證後寫入 `filter.value`
3. 確認：重整頁面後，任務列表和篩選狀態完整恢復

**驗收方式**：
- 新增 3 個任務 → 完成 1 個 → 切換篩選器到「已完成」→ 重整頁面
- 重整後：仍顯示 1 個已完成任務，篩選器仍在「已完成」

#### 任務 B：新增字數限制（必做）

1. 任務文字上限 50 字元
2. 即時顯示剩餘字元數（如：`43 / 50`）
3. 剩餘 ≤ 10 字元時，字元計數器顏色變化（警示）
4. 達到上限時，新增按鈕 disable

#### 任務 C：W3 Self-review（必做）

對照上方評分標準，填寫自評表格（誠實填寫，不虛報）：

| 評分面向 | 滿分 | 自評分 | 說明 |
|---------|------|------|------|
| Composable 設計 | 25 | ___ | |
| 響應式 API 正確使用 | 20 | ___ | |
| TypeScript 型別完整性 | 15 | ___ | |
| UI/UX 實作品質 | 20 | ___ | |
| 持久化設計 | 10 | ___ | |
| 程式碼可讀性 | 10 | ___ | |
| **合計** | **100** | **___** | |

#### 任務 D：可選——新增/刪除動畫

```vue
<!-- 使用 Vue 的 <TransitionGroup> -->
<TransitionGroup name="task-list" tag="ul">
  <li v-for="task in filteredTasks" :key="task.id">
    <!-- ... -->
  </li>
</TransitionGroup>
```

```css
.task-list-enter-active,
.task-list-leave-active {
  transition: all 0.3s ease;
}
.task-list-enter-from,
.task-list-leave-to {
  opacity: 0;
  transform: translateX(-20px);
}
```

---

## 自我檢核問題

1. **為什麼 `onMounted` 可以在 Composable 中使用，而不是只能在元件中？**
   （提示：Vue 的 Active Instance 機制）

2. **讀取 localStorage 後，為什麼不能直接 `tasks.value = JSON.parse(raw) as Task[]` 就好，還需要型別守衛？**
   （提示：`as T` 是 TypeScript 的承諾，不是執行期的保證）

3. **watch 的 `immediate: true` 和 `onMounted` 各自負責什麼？可以用 `immediate` 取代 `onMounted` 做初始讀取嗎？**
   （提示：想想 watch 的「寫入」語義 vs `onMounted` 的「讀取」語義）

4. **字數計算用 `computed` 而不是 `method`，理由是什麼？**
   （提示：`newTaskText` 改變頻率高，computed 的快取機制在這裡有什麼影響？）

5. **今日 Self-review 後，你認為最薄弱的面向是什麼？W3 結束前能補強嗎？**

---

## 明日預告（5/10，W3 Day 6）

**主題**：最終收尾 × 作業品質確認

- 根據 Self-review 結果，補強分數最低的面向
- 整理程式碼：清除 `console.log`、確認所有 TypeScript 型別無警告
- 寫作業說明（README 或 commit message）：說明設計決策
- 準備 PR：分支命名、commit 歷史整理
- 最終功能驗收：清單、涵蓋 W3 所有 API（ref/reactive/computed/watch）

> **W3 截止日 5/11（明後天）**，Day 6（5/10）完成所有準備，Day 7（5/11）只需 push 和開 PR。
