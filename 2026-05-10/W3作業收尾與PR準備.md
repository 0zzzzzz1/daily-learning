# W3 作業收尾 × PR 準備 × 最終功能驗收

> **日期**：2026-05-10（W3 Day 6）
> **類型**：收尾整理 + PR 準備
> **前置知識**：useTasks.ts + TaskList.v2.vue（Day 4-5 完成）、持久化回路（Day 5 完成）
> **截止提醒**：🔴 W3 作業截止 **明日 5/11**，今日必須完成所有準備！

---

## 目標技術

| 技術 | 核心重點 |
|------|---------|
| TypeScript | 型別完整性審查：無 `any`、函式簽名全標注、型別守衛驗證 |
| Composable 設計 | 最終設計文件化：解釋每個設計決策的「為什麼」 |
| Vue 3 整合 | 功能驗收 Checklist：所有 W3 API（ref/reactive/computed/watch）都有實際用到 |
| Git 工作流 | Commit 歷史整理、分支命名規範、PR 說明撰寫 |

---

## 核心知識點

- TypeScript 型別審查的系統化方法
- 程式碼整理的標準順序（清除 debug 殘留 → 型別補全 → 命名一致性）
- 好的 commit message 格式（Conventional Commits）
- PR 說明的結構：What（做了什麼）/ Why（為什麼這樣做）/ How（怎麼驗證）
- 最終功能驗收 Checklist 設計

---

## 為什麼學這個

過去五天我們把 TaskList.v2 從無到有地建起來：

```
Day 1（5/5）底層機制  →  Day 2（5/6）工具選用  →  Day 3（5/7）骨架設計
→  Day 4（5/8）核心實作  →  Day 5（5/9）持久化回路
→  Day 6（5/10）收尾與 PR  ← 今日
```

一個「能跑」的程式和一個「可以被 Review 的作業」之間，還有一段距離：

| 狀態 | 代表什麼 |
|------|---------|
| 能跑（Day 4-5 達成）| 功能正確、邏輯沒有 bug |
| 可 Review（今日目標）| 型別完整、無殘留 debug 程式、設計決策有說明、commit 歷史清晰 |

今日的任務是跨越這個距離。

---

## 知識說明

### 一、TypeScript 型別審查標準流程

完成功能後，系統化地做型別審查：

#### 1a. 找出所有 `any`（絕不允許）

```bash
# 在專案根目錄執行
grep -rn "any" src/ --include="*.ts" --include="*.vue"
```

常見出現 `any` 的位置：

```typescript
// ❌ 不好
function loadFromStorage(key: string): any { ... }

// ✅ 正確
function loadFromStorage<T>(key: string): T | null { ... }
```

#### 1b. 函式簽名完整標注

```typescript
// ❌ 不好：缺少回傳型別
function addTask(text: string) {
  // ...
}

// ✅ 正確：明確回傳型別
function addTask(text: string): void {
  // ...
}

// Composable 回傳型別
export function useTasks(): {
  tasks: Readonly<Ref<Task[]>>
  filter: Ref<FilterType>
  newTaskText: Ref<string>
  filteredTasks: ComputedRef<Task[]>
  taskStats: ComputedRef<TaskStats>
  remainingChars: ComputedRef<number>
  isAtLimit: ComputedRef<boolean>
  addTask: () => void
  removeTask: (id: string) => void
  toggleTask: (id: string) => void
  clearDone: () => void
  setFilter: (f: FilterType) => void
} {
  // ...
}
```

#### 1c. 介面完整定義

```typescript
// 確認所有介面都有完整欄位且型別正確
interface Task {
  id: string          // crypto.randomUUID() 回傳
  text: string        // 任務文字
  done: boolean       // 完成狀態
  createdAt: number   // Date.now() 回傳的時間戳記（毫秒）
}

interface TaskStats {
  total: number
  active: number
  done: number
}

type FilterType = 'all' | 'active' | 'done'
```

---

### 二、程式碼整理的標準順序

#### Step 1：清除 Debug 殘留

```typescript
// 搜尋並移除所有 console.log
// 只保留刻意設計的 console.warn/error（例如 localStorage 失敗警告）
console.log(tasks.value)          // ❌ 移除（開發時的 debug）
console.warn('localStorage 讀取失敗', e)  // ✅ 保留（有意義的錯誤提示）
```

#### Step 2：確認命名一致性

```typescript
// 統一用同一種命名慣例
const STORAGE_KEY_TASKS = 'vue3-tasks'    // ✅ 常數 UPPER_SNAKE_CASE
const storageKeyFilter = 'vue3-filter'    // ❌ 混用
const STORAGE_KEY_FILTER = 'vue3-filter'  // ✅ 統一

// Composable 函式名稱以 use 開頭（Vue 慣例）
export function useTasks() {}  // ✅
export function getTasks() {}  // ❌
```

#### Step 3：確認空狀態語意

```vue
<!-- TaskList.v2.vue -->
<template>
  <div v-if="tasks.length === 0" class="empty-state empty-state--no-tasks">
    <!-- 無任何任務（從未新增）-->
    <p>還沒有任務，新增第一個吧！</p>
  </div>
  
  <div v-else-if="filteredTasks.length === 0" class="empty-state empty-state--filtered">
    <!-- 有任務，但篩選器過濾掉了所有結果 -->
    <p>目前篩選條件下沒有符合的任務</p>
    <button @click="setFilter('all')">顯示全部</button>
  </div>
  
  <ul v-else>
    <li v-for="task in filteredTasks" :key="task.id">
      <!-- 正常渲染任務列表 -->
    </li>
  </ul>
</template>
```

---

### 三、W3 API 使用確認表

確認 TaskList.v2 完整覆蓋 W3 的所有核心 API：

| API | 在哪裡用到 | 使用方式 |
|-----|---------|---------|
| `ref` | `useTasks.ts` | `tasks = ref<Task[]>([])`、`filter = ref<FilterType>('all')`、`newTaskText = ref('')` |
| `reactive` | 視設計而定 | 若改用 `reactive` 管理 state 物件，需說明選用理由 |
| `computed` | `useTasks.ts` | `filteredTasks`、`taskStats`、`remainingChars`、`isAtLimit` |
| `watch` | `useTasks.ts` | `watch(tasks, () => localStorage.setItem(...))` / `watch(filter, ...)` |
| `onMounted` | `useTasks.ts` | 從 localStorage 讀取 tasks 和 filter |

---

### 四、Commit 歷史整理（Conventional Commits）

好的 commit message 結構：

```
<type>(<scope>): <description>

[body - 可選，說明「為什麼」]
[footer - 可選，Breaking Change / 關聯 issue]
```

**type 類型**：

| type | 用途 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修 bug |
| `refactor` | 重構（無新功能、無 bug 修正）|
| `style` | 格式調整（不影響邏輯）|
| `docs` | 文件更新 |
| `chore` | 雜項（設定檔等）|
| `test` | 測試相關 |

**W3 作業可能的 commit 歷史（建議）**：

```
feat(tasks): add useTasks composable with core CRUD operations
feat(tasks): add computed filteredTasks and taskStats
feat(tasks): add watch-based localStorage persistence (write)
feat(tasks): add onMounted-based localStorage restore (read)
feat(tasks): add character count limit with computed remainingChars
fix(tasks): use :checked + @change instead of v-model on checkbox
refactor(tasks): add TypeScript type guards for localStorage data
docs(tasks): add README with design decisions
```

---

### 五、PR 說明模板

```markdown
## W3 作業：TaskList.vue 第二版

### 功能說明

實作一個整合 Vue 3 響應式 API 的任務管理元件，核心為 `useTasks` Composable。

### 技術設計決策

#### 為什麼選 `ref` 管理 `tasks` 陣列，而非 `reactive`？

`ref` 提供整批替換語義（`tasks.value = [...]`），對陣列的 `trigger` 更可預測。
`reactive` 雖然允許局部 mutation（`state.tasks.push(...)`），但失去了整批替換的清晰語義，
且解構後需要 `toRefs`，增加使用端的認知負擔。

#### 為什麼 `onMounted` 負責讀取，`watch` 負責寫入？

兩者語義完全對稱且不可互換：
- `watch` 的語義是「監聽變化，有變化就寫入」
- `onMounted` 的語義是「啟動時執行一次性初始化」

若用 `watch + immediate: true` 做讀取，`immediate` 執行時 `tasks.value` 是初始空陣列，
會把 localStorage 的舊資料直接覆蓋。

#### TypeScript 型別守衛的必要性

`JSON.parse` 回傳 `any`，`as Task[]` 是編譯期斷言，無執行期保護。
透過 `isValidTask` 型別守衛函式，確保每個欄位都符合 `Task` 介面，
防禦版本格式不符或資料損壞的情況。

### 功能驗收 Checklist

- [ ] 新增任務（Enter / 按鈕）
- [ ] 刪除任務
- [ ] 切換完成狀態（checkbox）
- [ ] 一鍵清除完成項目
- [ ] 篩選：全部 / 未完成 / 已完成
- [ ] 統計列顯示正確（X 項未完成）
- [ ] 字數計數即時更新（XX / 50）
- [ ] 達到上限時新增按鈕 disable
- [ ] 重整頁面後任務列表恢復
- [ ] 重整頁面後篩選器狀態恢復
- [ ] 三種空狀態正確顯示
```

---

### 六、最終功能驗收 Checklist（逐步操作）

**驗收腳本**（請按照順序執行，確認每一步都正常）：

```
第一輪：基本功能驗收
□ 1. 開啟頁面，確認：空狀態「還沒有任務」正確顯示
□ 2. 輸入「任務一」，按 Enter，確認：任務出現在列表
□ 3. 再新增「任務二」、「任務三」，確認：統計列顯示「3 項未完成」
□ 4. 點擊「任務一」的 checkbox，確認：劃線樣式、統計列更新為「2 項未完成」
□ 5. 刪除「任務二」，確認：列表剩 2 項
□ 6. 點擊「已完成」篩選器，確認：只顯示任務一
□ 7. 點擊「未完成」篩選器，確認：只顯示任務三
□ 8. 點擊「清除完成」，確認：任務一消失，統計列「1 項未完成」
□ 9. 點擊「全部」篩選器，確認：只剩任務三

第二輪：字數限制驗收
□ 10. 在輸入框輸入長文字，確認：字元計數即時更新
□ 11. 輸入至 ≤ 10 字元剩餘，確認：計數器顏色變化（警示）
□ 12. 輸入滿 50 字元，確認：無法繼續輸入、新增按鈕 disabled

第三輪：持久化驗收
□ 13. 新增 3 個任務，完成 1 個，切換篩選器到「已完成」
□ 14. 重整頁面（Cmd+R / F5）
□ 15. 確認：列表恢復（顯示 1 個已完成任務）、篩選器在「已完成」
□ 16. 切換到「未完成」，確認：未完成的 2 個任務也在

第四輪：空狀態驗收
□ 17. 把所有任務都刪除，確認：「還沒有任務」空狀態
□ 18. 新增 2 個任務，切換到「已完成」篩選器，確認：「篩選無結果」空狀態
```

---

## 作業說明

### W3 Day 6 任務清單

#### 任務 A：TypeScript 型別審查（必做）

對 `useTasks.ts` 和 `TaskList.v2.vue` 執行型別審查：

1. 搜尋並移除所有 `any`
2. 補全所有函式的回傳型別標注
3. 確認 `useTasks` 的回傳型別有明確定義
4. 確認型別守衛（`isValidTask`、`isValidFilter`）正確實作
5. 在 IDE 或執行 `vue-tsc --noEmit` 確認無 TypeScript 警告

#### 任務 B：程式碼整理（必做）

1. 清除所有 `console.log`（保留有意義的 `console.warn`）
2. 確認命名一致性（常數、變數、函式命名風格統一）
3. 整理 `<script setup>` 的結構順序：
   ```
   import → defineProps/defineEmits（如有）→ Composable 呼叫 → 本地 computed/method
   ```
4. 確認 `<template>` 的三種空狀態正確使用不同的 class

#### 任務 C：撰寫作業 README（必做）

在作業分支的根目錄建立 `README.md`，說明：

- 功能概覽（截圖或文字說明）
- 核心技術設計決策（3 個：ref 選用理由、持久化設計、型別守衛）
- 如何在本機執行

#### 任務 D：整理 commit 歷史（必做）

1. 確認分支名稱：`feat/w3-task-list-v2`（或你習慣的命名）
2. 確認 commit message 使用 Conventional Commits 格式
3. 確認沒有 "WIP"、"test"、"aaa" 等無意義的 commit

#### 任務 E：執行最終驗收腳本（必做）

按照上方「最終功能驗收 Checklist」逐一確認，把有問題的地方在今天修完。

---

## 自我檢核問題

1. **執行 `grep -rn "any" src/` 後，你找到幾個 `any`？都改掉了嗎？**

2. **你的 `useTasks` 函式有明確的回傳型別嗎？如果有人看你的程式碼但不看實作，能從回傳型別知道這個 Composable 提供了什麼能力嗎？**

3. **你的 commit 歷史能說明「這個功能是怎麼一步一步被建造起來的」嗎？如果 reviewer 只看 commit log，能理解你的實作思路嗎？**

4. **執行最終驗收腳本的 16 個步驟時，有沒有哪個步驟失敗？你是如何修正的？**

5. **回顧 W3 五天的學習：哪一個知識點在實作中給你帶來最大的「啊哈！」時刻？為什麼？**

---

## 明日預告（5/11，W3 Day 7）

**主題**：W3 PR 提交

- Push 分支到 GitHub：`git push origin feat/w3-task-list-v2`
- 在 GitHub 開 Pull Request，貼上今日準備好的 PR 說明
- 確認 PR 包含所有 W3 功能點
- 同時確認 W1–W2 的 4 份 PR 狀態（登入 GitHub 手動查看）
- 若有人做 Code Review，針對 feedback 修改

> **W3 截止日 5/11**，今日（5/10）完成所有準備後，明日只需執行 `git push` 和開 PR。  
> W3 PR 完成後，W4 開始進入：Lifecycle Hooks 深化 × `watchEffect` × `toRefs`。
