# W4 收尾整理 × TypeScript 型別審查 × 功能驗收 × PR 準備

> **日期**：2026-05-17（W4 Day 6）
> **週次**：W4（5/12–5/18）— Vue 3: Lifecycle Hooks × watchEffect × toRefs
> **前置知識**：W4 Day 1–5 完整實作（Lifecycle Hooks 時間軸、watchEffect × onCleanup、toRefs 決策樹、useWatchEffectLogger.ts、useLifecycleLogger.ts × LifecycleLogger.vue 整合）
> **帶著這個問題進入今天**：你有 `useLifecycleLogger.ts`、`useWatchEffectLogger.ts`、`LifecycleLogger.vue` 三個檔案。如果要對這個作業進行 TypeScript 型別審查，審查的順序和重點應該是什麼？有哪些地方是最容易遺漏型別的？

---

## 目標技術

| 技術 | 今日重點 |
|------|---------|
| TypeScript 型別審查 | 系統化審查順序 × 高風險遺漏點清單 |
| Composable 回傳型別 | 顯式標注 vs 型別推導的取捨 |
| `vue-tsc` 靜態分析 | 零警告驗收標準 |
| 功能驗收腳本 | 系統化地確認所有功能路徑正確 |
| PR 準備 | Commit 整理 × PR 說明設計決策 |

---

## 為什麼學這個

W4 前五天完成了全部核心實作：

```
Day 1：Lifecycle Hooks 完整時間軸 × watch vs onMounted 選用決策
Day 2：watchEffect 深化 × onCleanup 完整時機 × Race Condition 防禦
Day 3：toRefs 決策樹 × Composable 回傳設計 × LifecycleLogger.vue 骨架
Day 4：useWatchEffectLogger.ts 完整實作（flush 動態切換 + Active Instance 清理）
Day 5：useLifecycleLogger.ts 完整實作 × LifecycleLogger.vue 模板整合 × 依賴追蹤因果鏈

     ↓ W4 Day 6 連接點（今日）

Day 6（5/17）：TypeScript 型別審查 × 功能驗收 × Commit 整理 × PR 準備
```

W3 的教訓：「能跑」≠「可 Review」。今天把 LifecycleLogger 作業從「能跑」推到「可 Review」的狀態。

**截止日明日（5/18）**，今天是最後的整理機會。

---

## 核心問題解答：TypeScript 型別審查的順序與重點

### 審查順序

型別審查應從「型別定義層」往「使用層」審查，原因是：型別定義的錯誤會向上傳播影響所有使用端，先修好定義層才能正確判斷使用層。

```
審查順序：
1. useLifecycleLogger.ts → 型別定義（interface / type alias）
2. useLifecycleLogger.ts → 函式簽名 + 回傳型別
3. useWatchEffectLogger.ts → 函式簽名 + 回傳型別
4. LifecycleLogger.vue <script setup> → 輔助函式 × 事件 handler
5. 執行 vue-tsc → 確認零警告
```

---

### Step 1：審查 `useLifecycleLogger.ts`

#### 1a. 型別定義層（LifecycleLog、HookCounts）

```ts
// ✅ 正確
export interface LifecycleLog {
  hook: string        // ← 可以更精確嗎？考慮用 keyof HookCounts
  timestamp: number
  time: string
  count: number
}

// 更精確的版本（可選，視設計目標）
export interface LifecycleLog {
  hook: keyof HookCounts   // ← hook 名稱被約束在 HookCounts 的 key 中
  timestamp: number
  time: string
  count: number
}
```

**高風險遺漏點 1：`hook` 欄位的型別**

`hook: string` 型別太寬泛。理論上 hook 只會是 HookCounts 的 key（`'onMounted' | 'onBeforeMount' | ...`）。使用 `hook: keyof HookCounts` 讓型別更精確，且 `hookColorClass(log.hook)` 的參數型別也能從 `string` 變成 `keyof HookCounts`。

---

#### 1b. 函式簽名

```ts
// ✅ 完整的函式簽名（參數 + 回傳型別）
export function useLifecycleLogger(componentName = 'Component'): {
  hookCounts: HookCounts  // ← reactive 物件，型別是 HookCounts（非 Ref）
  logs: Ref<LifecycleLog[]>
  clearLogs: () => void
  resetAll: () => void
}
```

**高風險遺漏點 2：函式沒有明確標注回傳型別**

如果只靠 TypeScript 推導，回傳型別雖然正確，但沒有「API 契約」效果——使用端需要看實作才知道回傳什麼。顯式標注讓 IDE 直接顯示型別，Composable 的設計意圖也更清晰。

---

#### 1c. 私有函式 `record` 的型別

```ts
// ✅ 完整標注
function record(hookName: keyof HookCounts): void {
  hookCounts[hookName]++  // ← TypeScript 能驗證 hookName 是合法的 key
  // ...
}

// ❌ 問題版本
function record(hookName: string): void {
  hookCounts[hookName]++  // ← TypeScript 可能報錯：string 不能 index reactive<HookCounts>
}
```

**高風險遺漏點 3：`keyof` 在 index access 的重要性**

`hookCounts[hookName]` 的 `hookName` 如果是 `string`，TypeScript 無法確保 `hookName` 是 `HookCounts` 的合法 key，可能報錯或需要用 `as` 強制轉型。使用 `keyof HookCounts` 既安全又不需要斷言。

---

#### 1d. `resetAll` 的 `Object.keys` 轉型

```ts
// ✅ 需要 as 斷言（Object.keys 固定回傳 string[]，TypeScript 刻意設計如此）
;(Object.keys(hookCounts) as Array<keyof HookCounts>).forEach((key) => {
  hookCounts[key] = 0
})
```

**高風險遺漏點 4：`Object.keys` 回傳 `string[]` 的設計限制**

TypeScript 的 `Object.keys()` 永遠回傳 `string[]`，不會回傳 `Array<keyof T>`——這是刻意的設計，因為執行期物件可能有額外 key。在這個特定場景（HookCounts 結構固定），用 `as Array<keyof HookCounts>` 斷言是安全的做法，但需要加上去，否則 `hookCounts[key] = 0` 會報型別錯誤。

---

### Step 2：審查 `useWatchEffectLogger.ts`

#### 函式簽名

```ts
// ✅ 完整標注
export function useWatchEffectLogger(
  target: Ref<number>        // ← 明確說明接受的是 Ref<number>，而非 any
): {
  flushMode: Ref<FlushMode>  // ← FlushMode 是 'pre' | 'post' 的 type alias
  logs: Ref<EffectLog[]>     // ← EffectLog 是 { time: string; message: string } 的 interface
}
```

**高風險遺漏點 5：Composable 參數型別**

`target: Ref<number>` 明確限制只接受 `Ref<number>`，避免傳入 `reactive` 物件或一般 `number` 值造成執行期錯誤。

**高風險遺漏點 6：FlushMode 型別定義**

如果 `flushMode` 只是 `Ref<string>`，切換按鈕的 `v-model` 綁定不會報錯，但允許任意字串賦值。定義 `type FlushMode = 'pre' | 'post'` 能在編譯期防止錯誤值。

```ts
export type FlushMode = 'pre' | 'post'
```

---

### Step 3：審查 `LifecycleLogger.vue <script setup>`

#### 輔助函式 `hookColorClass`

```ts
// ✅ 標注版本：傳入限制在 keyof HookCounts，回傳 string
function hookColorClass(hook: keyof HookCounts): string {
  const map: Record<keyof HookCounts, string> = { ... }
  return map[hook] ?? 'text-gray-600'
}

// ⚠️ 問題版本：傳入 string，map 需要額外型別處理
function hookColorClass(hook: string): string {
  const map: Record<string, string> = { ... }  // Record<string, string> 型別太寬泛
  return map[hook] ?? 'text-gray-600'
}
```

**高風險遺漏點 7：輔助函式的參數型別**

`hookColorClass` 在模板中被呼叫，參數是 `log.hook`。如果 `log.hook` 是 `keyof HookCounts` 型別，`hookColorClass` 的參數也應該是 `keyof HookCounts`，形成型別一致。

---

#### 事件 handler

```ts
// ✅ Vue 中 @click handler 不需要標注型別（void 是預設的）
// 但若有參數，需要明確標注

// 例如：<input @input="onInput">
function onInput(event: Event): void {  // ← Event 需要明確標注
  const value = (event.target as HTMLInputElement).value
}
```

在本作業中，所有 handler 都是簡單的 `triggerUpdate++`、`clearLogs()`、`resetAll()`，不帶 Event 參數，所以這部分風險較低。

---

### Step 4：執行 `vue-tsc` 驗收

```bash
# 安裝（若尚未安裝）
npm install -D vue-tsc typescript

# 執行型別檢查（不產生輸出檔案）
npx vue-tsc --noEmit

# ✅ 目標：零錯誤、零警告
```

---

## W4 作業功能驗收腳本

> **目標**：在提交 PR 前，系統化確認每一個功能路徑都正確運作。
> **方法**：按步驟操作並確認每個「確認：」描述的結果。

### 初始狀態驗收

```
操作 1：首次載入元件（或刷新頁面）
確認：Hook 計數面板顯示「onBeforeMount: 1」× 「onMounted: 1」，其他 hook 為 0
確認：事件日誌中出現兩筆記錄（onBeforeMount → onMounted）
確認：watchEffect 日誌中出現一筆記錄（pre flush 初次執行）
確認：flush 模式選擇器預設為「pre」
```

### 強制更新驗收（核心 onUpdated 功能）

```
操作 2：點擊「強制更新」按鈕一次
確認：「已手動觸發 1 次更新」文字出現（triggerUpdate 依賴追蹤建立 ✅）
確認：Hook 計數面板顯示「onBeforeUpdate: 1」「onUpdated: 1」
確認：事件日誌出現 onBeforeUpdate → onUpdated 兩筆記錄
確認：watchEffect 日誌中新增一筆記錄（triggerUpdate 變動觸發重跑）

操作 3：再次點擊「強制更新」按鈕
確認：計數累加（onBeforeUpdate: 2，onUpdated: 2）
確認：「已手動觸發 2 次更新」
```

### flush 模式切換驗收

```
操作 4：將 flush 模式從「pre」切換至「post」
確認：watchEffect 日誌中出現「onCleanup 觸發（切換前清理）」記錄
確認：新的 watchEffect 日誌出現，flush 標記為 post
確認：點擊強制更新後，post 模式的 watchEffect 執行時機記錄與 pre 不同

操作 5：切換回「pre」
確認：相同的清理 + 重建流程，日誌可觀察到
```

### 清除功能驗收

```
操作 6：點擊「清除日誌」
確認：事件日誌列表清空
確認：Hook 計數面板的數字保持不變（clearLogs 不重置計數）
確認：watchEffect 日誌保持不變（clearLogs 只清除 lifecycle logs）

操作 7：點擊「重置全部」
確認：事件日誌列表清空
確認：Hook 計數面板所有數字歸零
確認：「已手動觸發 X 次更新」仍然存在（triggerUpdate 不屬於 resetAll 範圍）
```

### 卸載驗收（關鍵：確認 onBeforeUnmount × onUnmounted 觸發）

```
操作 8：用 v-if 控制 LifecycleLogger 元件的顯示（需要父元件配合）
確認：切換為 false 時，事件日誌出現 onBeforeUnmount → onUnmounted
確認：兩個 Composable 的 onUnmounted 清理邏輯都執行（可從 console 觀察）
確認：切換回 true 時，元件重新掛載，onBeforeMount → onMounted 重新觸發，計數從 1 開始
```

---

## Commit 整理指南

W4 作業的 commit 建議組織方式：

```
feat: add useLifecycleLogger composable with 8 lifecycle hooks

- Records hook invocations with timestamps and counts
- Returns hookCounts (reactive), logs (ref), clearLogs, resetAll
- Designed for reactive object hookCounts (no toRefs needed)

feat: add useWatchEffectLogger composable with dynamic flush mode

- Tracks triggerUpdate ref changes with configurable flush timing
- Dynamic flush switching: stop old effect + rebuild with new flush
- Manual cleanup via onUnmounted for effects created outside setup

feat: implement LifecycleLogger.vue with dual composable integration

- Integrates useLifecycleLogger and useWatchEffectLogger
- triggerUpdate ref is rendered in template to enable onUpdated tracking
- Flush mode radio selector triggers dynamic effect rebuild

fix: ensure triggerUpdate is rendered in template for dependency tracking

- Without {{ triggerUpdate }} in template, onUpdated never fires
- Added "已手動觸發 N 次更新" display to establish reactive dependency
```

---

## PR 說明模板

```markdown
## W4 作業：生命週期觀察工具（LifecycleLogger.vue）

### 作業目標
建立一個可視化工具，讓開發者能直觀地觀察 Vue 3 元件的生命週期事件與 watchEffect 的執行時機。

### 實作內容

**useLifecycleLogger.ts**
- 封裝 8 個 Lifecycle Hook（含 onActivated/onDeactivated）
- 設計決策：`hookCounts` 選 `reactive`（結構固定 + 不整批替換），`logs` 選 `ref`（整批替換語義清晰）
- `resetAll` 用 `Object.keys` 逐項歸零，不破壞 reactive 響應性

**useWatchEffectLogger.ts**（W4 Day 4 實作）
- 動態 flush 模式切換：停止舊 effect + 以新 flush 重建 effect
- setup 外建立的 effect 無 Active Instance 綁定，需 `onUnmounted` 手動清理

**LifecycleLogger.vue**
- `triggerUpdate` 必須在模板中被讀取，才能建立 Vue 依賴追蹤，`onUpdated` 才能被強制觸發
- 雙 Composable 的 `onUnmounted` 各自獨立登錄，互不干擾

### 關鍵設計決策

1. **`triggerUpdate` 的依賴追蹤問題**：Vue 的響應性系統只追蹤被 render function 讀取的資料。若 `triggerUpdate.value++` 但模板中沒有讀取它，`onUpdated` 永遠不觸發。解法是在模板中顯示 `{{ triggerUpdate }}` 或衍生值。

2. **`hookCounts` 的 `resetAll` 設計**：`reactive` 物件不能整批替換（會丟失響應性），正確做法是 `Object.keys` 逐項歸零，保留原 Proxy 追蹤。

3. **雙 Composable 的 `onUnmounted` 共存**：多個 Composable 在 `setup()` 同步流中各自登錄 hook，都綁定到同一個 Active Instance，卸載時各自觸發，互不影響。

### 測試驗收
- [x] 掛載時 onBeforeMount → onMounted 出現在日誌
- [x] 強制更新觸發 onBeforeUpdate → onUpdated
- [x] flush 模式切換可觀察到 onCleanup + 新 effect 建立
- [x] 清除日誌 vs 重置全部功能區分正確
- [x] vue-tsc --noEmit 零警告
```

---

## 自我檢核問題

**Q1（審查順序）**：TypeScript 型別審查為什麼要從「型別定義層」往「使用層」審查，而不是反過來？

**Q2（keyof 應用）**：在 `record(hookName: keyof HookCounts)` 中，如果把 `keyof HookCounts` 改成 `string`，會在哪一行產生 TypeScript 錯誤？為什麼？

**Q3（Object.keys 限制）**：TypeScript 的 `Object.keys(obj)` 為什麼固定回傳 `string[]`，而不是 `Array<keyof typeof obj>`？在本作業的 `resetAll` 中，你如何安全地繞過這個限制？

**Q4（vue-tsc vs tsc）**：為什麼審查 Vue 元件的型別要用 `vue-tsc --noEmit` 而不是直接 `tsc --noEmit`？兩者的差異是什麼？

**Q5（Composable 回傳型別標注的取捨）**：Composable 回傳型別可以讓 TypeScript 自動推導，也可以手動標注（如 `): { hookCounts: HookCounts; logs: Ref<LifecycleLog[]>; ... }`）。兩種做法各有什麼優缺點？你選哪種？

---

## 明日預告（W4 Day 7 截止日：2026-05-18）

**主題**：W4 PR 提交截止日 × W4 知識總結 × M2 銜接準備

> ⏰ **明日（5/18）為 W4 正式截止日**。今天完成整理後，明天的主要任務：
> 1. Push 分支 + 開 PR（若尚未完成）
> 2. W4 整週知識脈絡總整理（7 天的學習鏈回顧）
> 3. 確認 W1–W3 所有待確認 PR 的 GitHub 狀態
> 4. M2 預習：`defineProps` / `defineEmits` / `v-model` 進階

**帶著這個問題進入明天**：

> W4 作業 PR 說明中，你說「`hookCounts` 使用 `reactive` 而非 `ref`，因為結構固定且整體回傳」。Reviewer 追問：「那如果未來 W4 作業要增加 `onErrorCaptured` hook，你的 `HookCounts` 型別、`resetAll`、以及 Composable 回傳型別，分別需要改哪些地方？改動成本大嗎？」你怎麼回答？
