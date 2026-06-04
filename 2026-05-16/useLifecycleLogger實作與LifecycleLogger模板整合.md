# useLifecycleLogger.ts 完整實作與 LifecycleLogger.vue 模板整合

> **日期**：2026-05-16（W4 Day 5）
> **週次**：W4（5/12–5/18）— Vue 3: Lifecycle Hooks × watchEffect × toRefs
> **前置知識**：W4 Day 1 生命週期完整時間軸、W4 Day 2 watchEffect × onCleanup × Race Condition 防禦、W4 Day 3 toRefs 決策樹 × Composable 回傳設計 × LifecycleLogger.vue 骨架、W4 Day 4 useWatchEffectLogger.ts 完整實作
> **帶著這個問題進入今天**：`LifecycleLogger.vue` 模板中有「強制更新」按鈕，handler 是 `triggerUpdate.value++`，但 `triggerUpdate` 在模板中沒有被渲染（沒有 `{{ triggerUpdate }}`）。Vue 是否會因為「響應式資料改變，但 DOM 實際上沒有任何變化」而觸發 `onUpdated`？

---

## 目標技術

| 技術 | 今日重點 |
|------|---------|
| Lifecycle Hooks | 在 Composable 中封裝所有 8 個 hook 的完整實作 |
| `reactive` 計數物件 | 整體回傳（不需要 toRefs）vs 逐項回傳的選用 |
| Vue 依賴追蹤 | 「響應式資料改變 → DOM 更新 → `onUpdated`」的完整因果鏈 |
| Composable 組合 | 在同一元件中同時使用兩個 Composable（useLifecycleLogger + useWatchEffectLogger）|
| LifecycleLogger.vue 模板整合 | 將兩個 Composable 的輸出整合進同一個 UI |

---

## 為什麼學這個

W4 前四天建立了完整的理論基礎：

```
Day 1：生命週期 8 個 hook 的完整時間軸
Day 2：watchEffect 深化 × onCleanup 時機 × Race Condition 防禦
Day 3：toRefs 完整決策樹 × LifecycleLogger.vue 骨架設計
Day 4：useWatchEffectLogger.ts 完整實作（flush 動態切換 + Active Instance 清理）
```

今天是 **W4 作業實作的主攻日**：完成 `useLifecycleLogger.ts` 並將兩個 Composable 整合進 `LifecycleLogger.vue`。

W4 截止日為 **5/18（周日），距今 2 天**，今天完成核心實作，明天（5/17）進行型別審查與 PR 準備。

---

## 核心問題解答：`triggerUpdate.value++` 會觸發 `onUpdated` 嗎？

### 答案：**不會**（如果 `triggerUpdate` 沒有出現在 render function 的依賴追蹤中）

### 原理拆解

Vue 3 的依賴追蹤遵循一個核心原則：

> **響應式資料必須在 render function（或 template）被「讀取」，才會建立依賴關係。**

```
觸發 onUpdated 的完整因果鏈：
響應式資料改變 → render function 重跑 → Virtual DOM diff → DOM 實際更新 → onUpdated 觸發
```

如果 `triggerUpdate` 沒有在模板中被使用（沒有 `{{ triggerUpdate }}`、沒有綁定到 `:class` 或任何 expression），那麼 render function 根本不知道它的存在，也不會建立依賴關係。

```ts
// ❌ 這樣的按鈕點擊不會觸發 onUpdated
const triggerUpdate = ref(0)
// template: <button @click="triggerUpdate.value++">強制更新</button>
// 但 triggerUpdate 沒有在模板中被讀取 → render function 不追蹤它
// → 點擊後 Vue 根本不知道要重新渲染 → onUpdated 不觸發
```

### 正確解法：讓 `triggerUpdate` 進入 render function 的依賴追蹤

```ts
const triggerUpdate = ref(0)
```

**方案 A（簡單直接）**：在模板中渲染它，即使用 `v-show="false"` 包住也能建立追蹤：

```html
<!-- ✅ render function 讀取 triggerUpdate.value，建立依賴 -->
<span class="text-xs text-gray-400">強制更新計數：{{ triggerUpdate }}</span>
<button @click="triggerUpdate++">強制更新</button>
```

**方案 B（語義更清晰）**：讓 `triggerUpdate` 影響一個在模板中使用的 computed：

```ts
// 這樣 updateMessage 被模板追蹤，而 updateMessage 依賴 triggerUpdate
const updateMessage = computed(() => `已手動觸發 ${triggerUpdate.value} 次更新`)
```

```html
<p>{{ updateMessage }}</p>
<button @click="triggerUpdate++">強制更新</button>
```

### 延伸思考：這個原則的重要性

這個原則解釋了為什麼 Vue 的響應性系統這麼高效：**它只追蹤真正被使用的資料**。沒有被 render function 讀取的響應式資料，改變時不會觸發不必要的重新渲染。

---

## useLifecycleLogger.ts 完整實作

### 設計決策

根據 W4 Day 3 建立的 Composable 回傳設計原則：

```
回傳物件設計決策樹：
├── hookCounts → reactive 整體物件 → 不解構，整體回傳 → 不需要 toRefs
├── logs → ref([]) → 本身是 ref → 直接回傳，不需要 toRefs
└── clearLogs → 函式 → 直接回傳，完全不需要 toRef 處理
```

### 完整實作

```ts
// useLifecycleLogger.ts
import {
  onBeforeMount,
  onMounted,
  onBeforeUpdate,
  onUpdated,
  onBeforeUnmount,
  onUnmounted,
  onActivated,
  onDeactivated,
  reactive,
  ref,
} from 'vue'

// 型別定義
export interface LifecycleLog {
  hook: string
  timestamp: number
  time: string
  count: number  // 此次觸發時該 hook 的第幾次
}

export type HookCounts = {
  onBeforeMount: number
  onMounted: number
  onBeforeUpdate: number
  onUpdated: number
  onBeforeUnmount: number
  onUnmounted: number
  onActivated: number
  onDeactivated: number
}

export function useLifecycleLogger(componentName = 'Component') {
  // 設計決策：reactive 物件整體回傳（使用端用 logger.hookCounts.onMounted 存取）
  // → 不解構 → 不需要 toRefs
  const hookCounts = reactive<HookCounts>({
    onBeforeMount: 0,
    onMounted: 0,
    onBeforeUpdate: 0,
    onUpdated: 0,
    onBeforeUnmount: 0,
    onUnmounted: 0,
    onActivated: 0,
    onDeactivated: 0,
  })

  // 設計決策：陣列用 ref（整批替換語義清晰）
  const logs = ref<LifecycleLog[]>([])

  // 私有輔助函式：記錄 hook 觸發
  function record(hookName: keyof HookCounts): void {
    hookCounts[hookName]++
    const now = new Date()
    logs.value.push({
      hook: hookName,
      timestamp: Date.now(),
      time: now.toLocaleTimeString('zh-TW', { hour12: false }),
      count: hookCounts[hookName],
    })
    // 開發期 console（上線前移除）
    console.log(`[${componentName}] ${hookName} #${hookCounts[hookName]} @ ${now.toLocaleTimeString()}`)
  }

  // 登錄所有生命週期鉤子
  // 注意：必須在 setup() 同步呼叫流中執行（Active Instance 機制）
  onBeforeMount(() => record('onBeforeMount'))
  onMounted(() => record('onMounted'))
  onBeforeUpdate(() => record('onBeforeUpdate'))
  onUpdated(() => record('onUpdated'))
  onBeforeUnmount(() => record('onBeforeUnmount'))
  onUnmounted(() => record('onUnmounted'))

  // onActivated / onDeactivated 只在 <KeepAlive> 包裹的元件中觸發
  // 登錄了也沒關係——不在 KeepAlive 內就永遠不會呼叫
  onActivated(() => record('onActivated'))
  onDeactivated(() => record('onDeactivated'))

  function clearLogs(): void {
    logs.value = []
    // hookCounts 不重置，保留累計計數
  }

  function resetAll(): void {
    logs.value = []
    // 重置所有計數
    ;(Object.keys(hookCounts) as Array<keyof HookCounts>).forEach((key) => {
      hookCounts[key] = 0
    })
  }

  return {
    hookCounts,   // reactive 物件（整體回傳，使用端 hookCounts.onMounted 存取）
    logs,         // Ref<LifecycleLog[]>（本身是 ref，直接使用）
    clearLogs,    // 清除日誌（保留計數）
    resetAll,     // 重置一切
  }
}
```

### 三個設計決策說明

**決策 1：`hookCounts` 選 `reactive` 不選 `ref`**

因為 hookCounts 是一個結構固定的物件（永遠不會整批替換），且在模板中以 `hookCounts.onMounted` 形式存取。用 `reactive` 更語義清晰，且不需要 `.value`。回傳時整體傳出，使用端不解構，所以不需要 `toRefs`。

**決策 2：`logs` 選 `ref` 不選 `reactive`**

`logs.value = []`（整批替換）是最常見的清除操作。`ref` 的整批替換語義清晰，且 `reactive([])` 的整批替換需要用 `splice(0)` 才能保留響應性，比較反直覺。

**決策 3：`record` 設計為私有（不在回傳值中暴露）**

`record` 是 Composable 的「內部邏輯」，使用端不需要知道它的存在。暴露最小介面，使用端只需要 `hookCounts`、`logs`、`clearLogs`、`resetAll`。

---

## LifecycleLogger.vue 模板整合

### 完整元件骨架

```vue
<template>
  <div class="lifecycle-logger p-6 max-w-2xl mx-auto">
    <h2 class="text-xl font-bold mb-4">LifecycleLogger 觀察工具</h2>

    <!-- Hook 計數面板 -->
    <section class="mb-6 p-4 bg-gray-50 rounded-lg">
      <h3 class="text-sm font-semibold text-gray-600 mb-3">Hook 觸發計數</h3>
      <div class="grid grid-cols-2 gap-2 text-sm">
        <div v-for="(count, hookName) in hookCounts" :key="hookName"
             class="flex justify-between items-center px-3 py-1 bg-white rounded border">
          <span class="text-gray-700">{{ hookName }}</span>
          <span class="font-mono font-bold" :class="count > 0 ? 'text-blue-600' : 'text-gray-300'">
            {{ count }}
          </span>
        </div>
      </div>
    </section>

    <!-- 強制更新面板（解答昨日問題的關鍵區塊）-->
    <section class="mb-6 flex items-center gap-4">
      <button
        @click="triggerUpdate++"
        class="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
      >
        強制更新
      </button>
      <!-- ✅ 必須在模板中讀取 triggerUpdate，Vue 才能建立依賴追蹤 -->
      <span class="text-sm text-gray-500">已手動觸發 {{ triggerUpdate }} 次更新</span>
    </section>

    <!-- watchEffect flush 模式選擇器 -->
    <section class="mb-6 p-4 bg-amber-50 rounded-lg">
      <h3 class="text-sm font-semibold text-amber-700 mb-2">watchEffect flush 模式</h3>
      <div class="flex gap-4">
        <label class="flex items-center gap-2 cursor-pointer">
          <input type="radio" v-model="flushMode" value="pre" /> pre（DOM 更新前）
        </label>
        <label class="flex items-center gap-2 cursor-pointer">
          <input type="radio" v-model="flushMode" value="post" /> post（DOM 更新後）
        </label>
      </div>
      <p class="mt-2 text-xs text-amber-600">切換模式時，Composable 內部會停止舊 effect 並以新 flush 建立新 effect。</p>
    </section>

    <!-- 日誌列表 -->
    <section class="mb-4">
      <div class="flex justify-between items-center mb-2">
        <h3 class="text-sm font-semibold text-gray-600">事件日誌（最新在前）</h3>
        <div class="flex gap-2">
          <button @click="clearLogs" class="text-xs text-gray-400 hover:text-red-500">清除日誌</button>
          <button @click="resetAll" class="text-xs text-gray-400 hover:text-red-500">重置全部</button>
        </div>
      </div>

      <div v-if="logs.length === 0" class="text-sm text-gray-400 text-center py-8">
        尚無事件——元件掛載後 onBeforeMount / onMounted 應出現在這裡
      </div>

      <ul class="space-y-1 max-h-64 overflow-y-auto">
        <!-- 最新在前：用 [...logs].reverse() 或直接在 Composable 中 unshift -->
        <li v-for="(log, index) in [...logs].reverse()" :key="index"
            class="flex items-center gap-3 text-xs px-3 py-2 bg-white border rounded font-mono">
          <span class="text-gray-400 w-20">{{ log.time }}</span>
          <span class="font-semibold" :class="hookColorClass(log.hook)">{{ log.hook }}</span>
          <span class="text-gray-400">#{{ log.count }}</span>
        </li>
      </ul>
    </section>

    <!-- watchEffect 日誌 -->
    <section>
      <h3 class="text-sm font-semibold text-gray-600 mb-2">watchEffect 日誌</h3>
      <div v-if="effectLogs.length === 0" class="text-sm text-gray-400 text-center py-4">
        尚無 watchEffect 事件
      </div>
      <ul class="space-y-1 max-h-48 overflow-y-auto">
        <li v-for="(log, index) in [...effectLogs].reverse()" :key="index"
            class="flex items-center gap-3 text-xs px-3 py-2 bg-amber-50 border rounded font-mono">
          <span class="text-gray-400 w-20">{{ log.time }}</span>
          <span class="text-amber-700">watchEffect</span>
          <span class="text-gray-600">{{ log.message }}</span>
        </li>
      </ul>
    </section>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import { useLifecycleLogger } from './composables/useLifecycleLogger'
import { useWatchEffectLogger } from './composables/useWatchEffectLogger'

// ── 強制更新計數（必須在模板中讀取才能建立依賴追蹤）──
const triggerUpdate = ref(0)

// ── Composable 1：生命週期觀察者 ──
const { hookCounts, logs, clearLogs, resetAll } = useLifecycleLogger('LifecycleLogger')

// ── Composable 2：watchEffect 觀察者 ──
const { flushMode, logs: effectLogs } = useWatchEffectLogger(triggerUpdate)

// ── 輔助：hook 名稱對應顏色 ──
function hookColorClass(hook: string): string {
  const map: Record<string, string> = {
    onBeforeMount: 'text-blue-400',
    onMounted: 'text-blue-600',
    onBeforeUpdate: 'text-orange-400',
    onUpdated: 'text-orange-600',
    onBeforeUnmount: 'text-red-400',
    onUnmounted: 'text-red-600',
    onActivated: 'text-green-500',
    onDeactivated: 'text-green-700',
  }
  return map[hook] ?? 'text-gray-600'
}
</script>
```

---

## 兩個 Composable 的整合設計圖

```
LifecycleLogger.vue
│
├── ref: triggerUpdate（強制更新計數）
│   └── ✅ 在模板中被讀取 → 依賴追蹤建立 → 值改變 → re-render → onUpdated 觸發
│
├── useLifecycleLogger('LifecycleLogger')
│   ├── 在 setup 同步流中登錄所有 8 個 hook
│   ├── 回傳：hookCounts（reactive 整體）、logs（ref）、clearLogs、resetAll
│   └── Active Instance 自動綁定 ✅
│
└── useWatchEffectLogger(triggerUpdate)
    ├── 追蹤 triggerUpdate 的變化（pre / post flush 模式）
    ├── flushMode 動態切換：watch + buildEffect 重建機制（Day 4）
    ├── 回傳：flushMode（ref）、logs: effectLogs（ref）
    └── setup 外建立的 effect → onUnmounted 手動清理 ⚠️
```

---

## 作業說明

**W4 作業**：生命週期觀察工具（LifecycleLogger.vue）

### 功能需求

- [ ] **useLifecycleLogger.ts**：封裝所有 8 個 Lifecycle Hook，回傳 `hookCounts`、`logs`、`clearLogs`、`resetAll`
- [ ] **LifecycleLogger.vue**：同時整合 `useLifecycleLogger` 與 `useWatchEffectLogger`
- [ ] **強制更新按鈕**：`triggerUpdate.value++`，且 `triggerUpdate` 有在模板中被渲染（onUpdated 能被觸發）
- [ ] **flush 模式選擇器**：動態切換 pre / post，實際改變 watchEffect 的執行時機
- [ ] **Hook 計數面板**：顯示所有 8 個 hook 的觸發次數
- [ ] **事件日誌列表**：顯示時間戳記、hook 名稱、觸發次數
- [ ] **清除 / 重置功能**：清除日誌 or 重置全部

### 評分標準

| 項目 | 配分 | 說明 |
|------|------|------|
| useLifecycleLogger.ts 完整實作 | 30 分 | 8 個 hook 全部登錄；TS 型別完整；回傳設計符合 W4 Day 3 決策樹 |
| triggerUpdate 依賴追蹤設計正確 | 15 分 | triggerUpdate 在模板中被讀取，onUpdated 能被強制觸發 |
| LifecycleLogger.vue 模板整合 | 25 分 | 兩個 Composable 正確整合；UI 清晰呈現 hook 狀態 |
| flush 模式切換功能正常 | 15 分 | 切換 pre/post 後，watchEffect 實際行為有差異（可在 log 中觀察） |
| TypeScript 型別安全 | 10 分 | 介面定義完整；無 `any`；vue-tsc 零警告 |
| 程式碼整潔度 | 5 分 | 命名清晰；無遺留 console；Composable 回傳設計有說明 |

---

## 自我檢核問題

**Q1**：`triggerUpdate.value++` 的按鈕，在什麼條件下才能讓 `onUpdated` 被觸發？為什麼只改變響應式資料還不夠？

**Q2**：`hookCounts` 選用 `reactive` 而不是 `ref({ ... })`，有什麼設計上的理由？如果 `hookCounts` 需要整批替換（比如 `resetAll`），你的做法是什麼？

**Q3**：在同一個元件中使用兩個 Composable（`useLifecycleLogger` 和 `useWatchEffectLogger`），它們各自的 `onUnmounted` 回呼都能正確觸發嗎？為什麼？

**Q4**：`useLifecycleLogger` 回傳的 `logs` 是 `ref<LifecycleLog[]>`，而 `hookCounts` 是 `reactive<HookCounts>`。如果使用端想在模板中用 `v-for` 遍歷 `logs`，應該寫 `v-for="log in logs"` 還是 `v-for="log in logs.value"`？為什麼？

**Q5**：`onActivated` 和 `onDeactivated` 這兩個 hook 在本元件中永遠不會觸發，但你還是在 `useLifecycleLogger` 中登錄了它們。這樣做合理嗎？有什麼潛在的問題嗎？

---

## 明日預告（W4 Day 6：2026-05-17）

**主題**：W4 收尾整理 × TypeScript 型別審查 × 完整功能驗收 × PR 準備

> W4 截止日為 **5/18（周日）**，明天（5/17）是最後一天整理。

**帶著這個問題進入明天**：

> 你有 `useLifecycleLogger.ts` 和 `useWatchEffectLogger.ts` 兩個 Composable 檔案，以及 `LifecycleLogger.vue`。如果要對這個作業進行 TypeScript 型別審查，審查的順序和重點應該是什麼？有哪些地方是最容易遺漏型別的？
