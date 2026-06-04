# W6 Day 7（截止日）：W6 七天完整知識閉環 × PR 最終截止確認 × W7 Composable 設計正式準備

> **所屬階段**：M2（Composition API 深化期）— W6 收尾 / W7 起點  
> **日期**：2026-06-01（W6 最後一天；W7 明日（6/2）正式啟動）  
> **核心目標**：整合 W6 七天知識鏈 → 確認 W6 PR 今日截止 → 為 W7 Composable 實作做最後準備

---

## 一、W6 七天知識閉環：完整主軸回顧

### 核心主題：跨層通訊 × 彈性插槽 × 可維護性提升

W6 的三個技術主軸不是獨立的，它們解決同一個問題的三個面向：

```
問題：隨著元件樹複雜化，資料傳遞和 UI 彈性變得困難

解法面向：
  ① 資料的跨層傳遞   → provide / inject（由上而下穿透 Props Drilling）
  ② UI 渲染的彈性    → Slots（Default / Named / Scoped，控制權反轉）
  ③ 跨元件邏輯複用   → Composable 提取（usePanelTheme 是 inject + computed 的封裝）
```

### W6 七天知識鏈

| Day | 日期 | 學習內容 | 關鍵突破 |
|-----|------|---------|---------|
| Day 1 | 5/26 | provide/inject 完整模式 × Slots 三種形式 × KeepAlive | `InjectionKey<T>` Symbol 型別安全；Scoped Slot 控制權反轉範式 |
| Day 2 | 5/27 | PanelCard.vue × DashboardLayout.vue 骨架 × injectionKeys.ts | `inject(key, fallback)` 型別縮窄；`as const + keyof typeof` 精確型別鏈 |
| Day 3 | 5/28 | 三個 Panel 完整實作 × 泛型 Scoped Slot × DashboardLayout 整合 | `generic="T"` Scoped Slot 型別推導鏈；KeepAlive 副作用管理（onDeactivated）|
| Day 4 | 5/29 | usePanelTheme Composable 提取 × 精確索引型別 × 驗收腳本 | `Record<PanelTheme['borderRadius'], string>` vs `Record<string, string>` 防禦線差異 |
| Day 5 | 5/30 | Commit 整理（git add -p × rebase -i）× PR 說明定稿 | 三個 Why 設計決策語言化；Scoped Slot vs provide/inject 方向定位清晰 |
| Day 6 | 5/31 | Push + PR Open 確認 × W7 預習（四種 Composable 模式） | 抽取時機三訊號系統化；Composable vs Pinia 邊界 |
| Day 7 | 6/1 | **W6 完整閉環** × **PR 截止確認** × **W7 正式準備** | 整合七天完整知識 → 掌握關鍵決策 |

---

## 二、W6 核心知識精華（Day 7 整合版）

### 2.1 provide / inject 完整決策樹

```
需要跨層傳遞資料？
  ├── 是 → 使用 provide / inject
  │   ├── Key 設計：InjectionKey<T> = Symbol('...') ← 全域唯一 + TS 型別安全
  │   ├── provide 值選擇：
  │   │   ├── 後代需要響應更新 → provide reactive(obj) 或 ref
  │   │   └── 靜態設定值 → provide 普通值（後代拿到 snapshot）
  │   ├── inject 型別縮窄：
  │   │   ├── inject(key, fallback) → 回傳 T（安全，推薦）
  │   │   └── inject(key) → 回傳 T | undefined（需 null check 或 !）
  │   └── 防止後代修改 → provide readonly(reactive(obj))
  └── 否 → 考慮 Props / Composable
```

### 2.2 Slots 三種形式選用

| Slot 類型 | 適用場景 | 範例 |
|-----------|---------|------|
| **Default Slot** | 主體內容替換 | `<PanelCard>自訂內容</PanelCard>` |
| **Named Slot** | 多個具名區域（header/body/footer）| `<template #header>標題</template>` |
| **Scoped Slot** | 子元件有資料，父元件決定渲染 UI | `<template #default="{ item }">...` |

**關鍵設計原則**：
- `v-if="$slots.footer"` 條件渲染：避免空 DOM 節點帶來的空白樣式
- `fallback content`：`<slot name="header"><span>預設標題</span></slot>` 解決未插入時的備案顯示
- 兩者解決不同層面的問題，可以同時使用

### 2.3 泛型 Scoped Slot 型別推導鏈（W6 最複雜的 TS 知識點）

```vue
<!-- ListPanel.vue -->
<script setup lang="ts" generic="T">
defineProps<{ items: T[] }>()
</script>
<template>
  <li v-for="item in items">
    <slot :item="item" />   <!-- 暴露 T 型別資料 -->
  </li>
</template>

<!-- 使用端 -->
<ListPanel :items="users">         <!-- TypeScript 從 users: User[] 推導 T = User -->
  <template #default="{ item }">   <!-- item 型別自動縮窄為 User -->
    {{ item.name }}                 <!-- 安全存取 .name -->
  </template>
</ListPanel>
```

### 2.4 KeepAlive 副作用管理

```
元件切換行為：
  一般元件切走：onBeforeUnmount → onUnmounted（清理副作用）
  KeepAlive 元件切走：onDeactivated（實例保留在記憶體，副作用繼續執行！）
  KeepAlive 元件切回：onActivated

❗ 陷阱：setInterval 在 onDeactivated 後仍然執行
✅ 正確：在 onDeactivated 中清理，在 onActivated 中重建
```

### 2.5 usePanelTheme：W6 提取 Composable 的完整故事

```typescript
// 提取前（三個 Panel 各自 inject + computed，重複 3 次）
const theme = inject(panelThemeKey, defaultTheme)
const borderRadiusClass = computed(() => borderRadiusMap[theme.borderRadius])
const shadowClass = computed(() => shadowMap[theme.shadow])

// 提取後（usePanelTheme.ts）
export function usePanelTheme(): PanelThemeResult {
  const theme = inject(panelThemeKey, defaultTheme)  // inject + fallback 防禦
  return {
    borderRadiusClass: computed(() => borderRadiusMap[theme.borderRadius]),
    shadowClass: computed(() => shadowMap[theme.shadow]),
  }
}

// 精確索引型別防禦
const borderRadiusMap: Record<PanelTheme['borderRadius'], string> = {
  // ← TypeScript 強制補齊所有 union 值，新增值時 TS 立即報錯
  sm: 'rounded-sm',
  md: 'rounded-md',
  lg: 'rounded-lg',
}
```

**提取的三個訊號**：三個元件相同 inject + computed 樣板 ✓ → 提取

---

## 三、W6 PR 截止確認（今日最重要的行動）

> **⚠️ W6 PR 今日（6/1）截止，請立即執行以下步驟：**

### 步驟清單

```bash
# 1. 確認 commit 序列整齊
git log --oneline

# 預期順序（大致如下）：
# feat: add PanelCard.vue with three named slots
# feat: add DashboardLayout.vue with dynamic component and KeepAlive
# feat: add FormPanel ListPanel ChartPanel with inject theme
# refactor: extract usePanelTheme composable
# chore: add injectionKeys.ts type definitions

# 2. 推送到遠端（rebase 後需要 force push）
git push --force-with-lease

# 3. 前往 GitHub 開啟 PR
# ├── Title：feat(W6): provide/inject × Slots × KeepAlive × usePanelTheme
# ├── Body：貼上昨日（5/30）定稿的 PR 說明（三個 Why 設計決策）
# └── CI：確認 vue-tsc --noEmit 通過
```

### 今日截止 W6 PR 說明要點（備忘）

**交付清單**：
- `injectionKeys.ts`：PanelTheme interface + panelThemeKey Symbol
- `PanelCard.vue`：三區域 Named Slot + `$slots.footer` 條件渲染
- `DashboardLayout.vue`：`provide reactive(theme)` + `<component :is>` + `KeepAlive`
- `FormPanel.vue` / `ListPanel.vue` / `ChartPanel.vue`：inject theme + CSS class 查找表
- `usePanelTheme.ts`：inject + fallback + computed 封裝

**三個 Why 設計決策**：
1. Symbol InjectionKey 而非字串：全域唯一 × TypeScript 型別推導雙保障
2. 泛型 Scoped Slot（`generic="T"`）：Headless UI 設計，資料與渲染分離
3. usePanelTheme 提取時機：三個 Panel 重複 inject+computed 樣板是最具體的提取訊號

---

## 四、W7 正式準備：Composable 設計實戰（6/2–6/8）

### W7 核心目標

> 從「看到訊號才提取」進化到「主動設計 Composable API」

W6 的 `usePanelTheme` 是被動提取（看到重複才拆）。W7 要學的是**主動設計**：在動筆寫元件之前，就先想好哪些邏輯適合封裝成 Composable，設計好它的介面。

### W7 四個目標 Composable（3個實作）

| Composable | 封裝的問題 | 核心技術 |
|------------|----------|---------|
| `useAsyncData<T>` | 非同步三態管理 + Race Condition | `ref` × `watchEffect` × `AbortController` |
| `useLocalStorage<T>` | localStorage 持久化 + JSON 安全讀寫 | `ref` × `watch` × `onMounted` × 型別守衛 |
| `usePagination` | 分頁計算邏輯封裝 | `computed` × `ref` × 純計算（無副作用）|
| `useEventListener` | 事件監聽副作用管理 | `onMounted` × `onUnmounted` × 型別安全 |

### useAsyncData<T> API 設計草稿（W7 Day 1 起點）

```typescript
// 目標介面設計
interface UseAsyncDataOptions {
  immediate?: boolean  // 是否立即執行（預設 true）
}

interface UseAsyncDataResult<T> {
  data: Ref<T | null>
  loading: Ref<boolean>
  error: Ref<Error | null>
  execute: () => void   // 手動重新觸發
}

// 使用端預期體驗
const { data: users, loading, error } = useAsyncData<User[]>(
  (signal) => fetchUsers(signal)
)
```

**設計決策要考慮**：
- `execute` 如何實現？（用 `trigger` ref，在 watch source 中使用）
- AbortController 如何在 `execute` 重複呼叫時清理上一個請求？
- `immediate: false` 時如何避免 watch 初始執行？

### useLocalStorage<T> API 設計草稿

```typescript
// 目標介面設計
function useLocalStorage<T>(key: string, defaultValue: T): Ref<T>

// 使用端預期體驗（透明：使用方式和普通 ref 一樣）
const theme = useLocalStorage<'light' | 'dark'>('app-theme', 'light')
theme.value = 'dark'  // 自動寫入 localStorage
// 下次載入頁面自動讀取 'dark'
```

**設計決策要考慮**：
- 如何處理 JSON.parse 失敗？（try/catch + 型別守衛）
- `watch` 的 `deep: true` 何時需要？（物件型別的 T）
- SSR 環境下 `window` 不存在怎麼辦？

---

## 五、自我檢核問題

### W6 知識閉環驗證

1. **provide/inject 的作用域是什麼？** 它和 Pinia 的全域狀態有什麼根本差異？

2. **為什麼說 Scoped Slot 是「控制權反轉」？** 和 Props 傳資料的方向比較，Scoped Slot 改變了什麼？

3. **inject(key, fallback) 和 inject(key) 的 TypeScript 回傳型別有什麼不同？** 對後續程式碼有什麼影響？

4. **KeepAlive 元件切換和一般元件切換在生命週期上有什麼差異？** `setInterval` 在 KeepAlive 切走後還在執行嗎？

5. **`Record<PanelTheme['borderRadius'], string>` 比 `Record<string, string>` 強在哪裡？** 能舉一個 TypeScript 在哪個時機報錯的例子嗎？

### W7 準備驗證

6. 你能說出 `useAsyncData<T>` 需要哪三個核心組成要素，以及為什麼每個都不能省略嗎？

---

## 六、明日預告（W7 Day 1 — 6/2）

**主題**：`useAsyncData<T>` 完整設計與實作

**具體目標**：
- 理解非同步 Composable 的「三態管理」模式（loading / data / error）
- 完整實作 AbortController 防 Race Condition 的 watchEffect 版本
- 比較 `watchEffect` vs `watch + trigger ref` 兩種實作方式的 trade-off
- 為後續 `useLocalStorage<T>` 做設計鋪墊（持久化 vs 非同步的不同關注點）

**銜接點**：
- W2 的 `async-utils`（非同步封裝設計思維）× W4 的 watchEffect + AbortController × W6 的 Composable 設計模式 → W7 的 `useAsyncData<T>` 是這三個週次知識的完整實現
