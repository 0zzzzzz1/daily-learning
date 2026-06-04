# FormPanel × ListPanel 泛型 Scoped Slot × ChartPanel 完整實作

> **日期**：2026-05-28（M2-W6 Day 3）
> **週次**：W6（5/26–6/1），截止 6/1，剩 **4 天**
> **學習階段**：M2 Composition API 深化期

---

## ⚠️ 作業堆積警示

| 週次 | 截止日 | 作業項目 | 狀態 |
|------|--------|---------|------|
| W3 | 2026-05-11 | TaskList.vue v2 | 🔴 截止已過 17 天，未確認 PR |
| W4 | 2026-05-18 | LifecycleLogger.vue | 🔴 截止已過 10 天，未確認 PR |
| W5 | 2026-05-25 | CustomInput + CustomSelect + LoginForm | 🔴 截止已過 3 天，未確認 PR |

**累積 3 份以上作業未確認** → 請在今日學習結束後，登入 GitHub 逐一確認 PR 狀態，並回來更新前端技術分析.md 的評分欄。

---

## 一、目標技術 & 核心知識點

| 目標技術 | 核心知識點 |
|---------|-----------|
| Vue.js 3 Slots | Scoped Slot 泛型設計（`generic="T"`）× 型別穿透 |
| Vue.js 3 provide/inject | inject theme → 動態 CSS class 綁定 |
| Vue.js 3 KeepAlive | 狀態保留驗證（FormPanel 欄位切走後保持） |
| TypeScript | Scoped Slot 的泛型 Props 型別推導 |

---

## 二、為什麼學這個（與昨日的連結）

昨日（Day 2）完成了：
- `injectionKeys.ts`：集中管理 `PanelTheme` interface + `panelThemeKey` Symbol
- `PanelCard.vue`：三區域 Named Slot（header/default/footer）× `$slots.footer` 條件渲染
- `DashboardLayout.vue` 骨架：`provide(reactive(theme))` + `<component :is>` + `KeepAlive` 架構

今日任務是**填入三個 Panel 的實作**，讓整個 `DashboardLayout` 真正能運作。

這個過程有兩個新挑戰：
1. **ListPanel 的泛型 Scoped Slot**：子元件擁有資料（`items: T[]`），但父元件決定渲染 UI —— 這是 Scoped Slot 最典型的設計場景
2. **inject theme → CSS 動態 class**：inject 拿到的是 `PanelTheme` 物件（`borderRadius: 'md'`），如何對應到實際的 CSS class？這是在非 Tailwind 環境下的常見問題

---

## 三、知識說明

### 3.1 Scoped Slot 泛型設計（ListPanel.vue）

**問題情境**：`ListPanel.vue` 要顯示一個列表，但列表中每一項的 UI 應由使用端決定（Parent decides how to render each item）。

#### 基本版（無泛型）

```vue
<!-- ListPanel.vue（基本版） -->
<script setup lang="ts">
const props = defineProps<{
  items: unknown[]  // 任意陣列，但型別不安全
}>()
</script>

<template>
  <ul>
    <li v-for="(item, index) in props.items" :key="index">
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

使用端：
```vue
<ListPanel :items="tasks">
  <template #default="{ item }">
    <!-- TypeScript 不知道 item 是什麼型別 → any -->
    {{ item.title }}  <!-- ⚠️ 紅線：item 型別為 unknown -->
  </template>
</ListPanel>
```

#### 泛型版（Vue 3.3+）

```vue
<!-- ListPanel.vue（泛型版） -->
<script setup lang="ts" generic="T">
const props = defineProps<{
  items: T[]
}>()
</script>

<template>
  <ul>
    <li v-for="(item, index) in props.items" :key="index">
      <!-- Scoped Slot 暴露 item: T 和 index: number -->
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

使用端（在 DashboardLayout.vue 中）：
```vue
<ListPanel :items="taskList">
  <template #default="{ item, index }">
    <!-- TypeScript 從 :items="taskList" 推導 T = Task -->
    <!-- item 型別自動縮窄為 Task，有完整的 IDE 支援 -->
    <span>{{ index + 1 }}. {{ item.title }}</span>
  </template>
</ListPanel>
```

**型別推導機制**：TypeScript 從 `v-bind:items="taskList"` 的實際型別（`Task[]`）推導出 `T = Task`，進而讓 Scoped Slot 的 `item` 型別也被正確推導為 `Task`。

#### 加入型別邊界（更嚴謹版）

```vue
<script setup lang="ts" generic="T extends { id: string | number }">
```

**何時需要邊界**：若 `ListPanel` 的內部實作會用到 `item.id`（例如 `:key="item.id"`），就需要 `extends { id: ... }` 確保 T 一定有 id。若只是 pass-through，無邊界限制即可。

---

### 3.2 FormPanel.vue：inject theme → 動態 CSS class

**問題情境**：`PanelCard.vue` 使用 Named Slots，`FormPanel.vue` 要透過 inject 拿到 `PanelTheme`，然後決定要套用什麼 CSS class。

#### 方案 A：Tailwind 原子 class（適合有 Tailwind 的專案）

```vue
<script setup lang="ts">
import { inject } from 'vue'
import { panelThemeKey } from '../injectionKeys'
import PanelCard from './PanelCard.vue'

// inject with fallback → TypeScript 回傳 PanelTheme（不含 undefined）
const theme = inject(panelThemeKey, {
  borderRadius: 'md',
  shadow: 'sm',
  headerBg: '#f8fafc'
})

// 將 theme 值對應到 Tailwind class
const radiusClass = computed(() => ({
  sm: 'rounded-sm',
  md: 'rounded-md',
  lg: 'rounded-lg'
}[theme.borderRadius] ?? 'rounded-md'))

const shadowClass = computed(() => ({
  sm: 'shadow-sm',
  md: 'shadow-md',
  lg: 'shadow-lg'
}[theme.shadow] ?? 'shadow-sm'))
</script>

<template>
  <PanelCard :class="[radiusClass, shadowClass]">
    <template #header>表單面板</template>
    <!-- 表單內容 -->
    <form @submit.prevent>
      <input v-model="name" placeholder="姓名" class="border rounded px-2 py-1 w-full mb-2" />
      <input v-model="email" placeholder="Email" class="border rounded px-2 py-1 w-full mb-2" />
      <button type="submit" class="bg-blue-500 text-white px-4 py-1 rounded">送出</button>
    </form>
  </PanelCard>
</template>
```

#### 方案 B：動態 style 綁定（不依賴 Tailwind）

```vue
<script setup lang="ts">
const theme = inject(panelThemeKey, { borderRadius: 'md', shadow: 'sm', headerBg: '#f8fafc' })

// 將 theme 的語意值對應到 CSS 值
const panelStyle = computed(() => ({
  borderRadius: {
    sm: '4px',
    md: '8px',
    lg: '16px'
  }[theme.borderRadius] ?? '8px',
  boxShadow: {
    sm: '0 1px 3px rgba(0,0,0,0.1)',
    md: '0 4px 6px rgba(0,0,0,0.1)',
    lg: '0 10px 15px rgba(0,0,0,0.15)'
  }[theme.shadow] ?? '0 1px 3px rgba(0,0,0,0.1)',
}))
</script>

<template>
  <PanelCard :style="panelStyle">
    <!-- ... -->
  </PanelCard>
</template>
```

**選用決策**：
- 有 Tailwind 的專案 → 方案 A（class 查找表）：型別安全 + tree-shaking
- 無 Tailwind 的純 CSS 或行內樣式 → 方案 B（style 綁定）：直接對應 CSS 值
- 本專案學習目的 → **兩種都實作一次，理解差異**

---

### 3.3 KeepAlive 狀態保留驗證

昨日 `DashboardLayout.vue` 已用 `KeepAlive` 包裹 `<component :is="...">`，今日實作後需驗證：

```
驗證步驟：
1. 點選「表單面板」Tab
2. 在 FormPanel 的姓名欄輸入「Alice」
3. 點選「列表面板」Tab（切換到 ListPanel）
4. 再點回「表單面板」Tab
5. 確認：姓名欄仍顯示「Alice」（KeepAlive 保留狀態 ✅）
```

若沒有 KeepAlive：切換 Tab 就等於銷毀 FormPanel，所有欄位資料清空。這是 KeepAlive 解決的典型問題。

---

### 3.4 inject 在子元件中響應祖先改變

**昨日發現**：`DashboardLayout.vue` 使用 `provide(panelThemeKey, reactive(theme))`，Proxy 共享。

今日驗證點：在 `DashboardLayout.vue` 加入一個按鈕來切換 theme，確認所有 Panel 都即時更新：

```vue
<!-- DashboardLayout.vue 新增 -->
<button @click="theme.shadow = theme.shadow === 'sm' ? 'lg' : 'sm'">
  切換陰影（{{ theme.shadow }}）
</button>
```

觀察：點按鈕後，所有 Panel 的陰影是否即時更新 → 這就是 `reactive` provide 的威力。

---

### 3.5 常見陷阱：KeepAlive 內的副作用清理

```vue
<!-- FormPanel.vue 錯誤示範 -->
<script setup>
// ⚠️ 如果有 setInterval，切走不等於卸載，interval 仍在跑
const timer = setInterval(() => {
  console.log('tick') // 切走後繼續印，浪費資源
}, 1000)

onUnmounted(() => {
  clearInterval(timer) // 在 KeepAlive 內，onUnmounted 不會被觸發！
})
</script>
```

**正確做法**：
```vue
<script setup>
let timer: ReturnType<typeof setInterval> | null = null

onActivated(() => {
  timer = setInterval(() => { /* ... */ }, 1000)
})

onDeactivated(() => {
  if (timer) clearInterval(timer)  // ✅ 切走時立即清除
})
</script>
```

---

## 四、作業說明（W6 截止 6/1，剩 4 天）

### 今日應完成的作業進度

| 交付項目 | 說明 | 目標 |
|---------|------|------|
| `FormPanel.vue` | inject theme → 動態 class/style × Named Slots × 欄位能填寫 | 🎯 今日完成 |
| `ListPanel.vue` | 泛型 Scoped Slot（`generic="T"`）× inject theme | 🎯 今日完成 |
| `ChartPanel.vue` | 靜態佔位圖表（div 模擬）× inject theme 套用 headerBg | 🎯 今日完成 |
| `DashboardLayout.vue` 整合 | 三個面板切換 × KeepAlive 驗證 × theme 切換按鈕 | 🎯 今日完成 |

### 技術規範

1. **FormPanel.vue**：
   - inject `panelThemeKey` 使用 fallback 模式（型別安全）
   - 使用 `PanelCard.vue` 的 Named Slots（`#header`、`#default`、`#footer`）
   - 有至少 2 個表單欄位（`v-model` 綁定），驗證 KeepAlive 保留狀態

2. **ListPanel.vue**：
   - `generic="T"` + `defineProps<{ items: T[] }>()`
   - Scoped Slot 暴露 `item: T` 和 `index: number`
   - inject theme 套用樣式

3. **ChartPanel.vue**：
   - inject `panelThemeKey`，使用 `headerBg` 套用到 header 背景
   - 以 `<div>` 模擬圖表（實際不需要真實圖表庫）

4. **DashboardLayout.vue 更新**：
   - 在模板中正確傳 `items` 給 `ListPanel`
   - 新增 theme 切換按鈕，驗證響應式 provide

### 評分標準

| 面向 | 滿分 | 重點 |
|------|------|------|
| 泛型 Scoped Slot 正確性 | 30 | TypeScript 能正確推導 item 型別 |
| inject theme 響應性 | 25 | theme 變動後所有 Panel 即時更新 |
| KeepAlive 驗證 | 20 | FormPanel 欄位在切換後仍保留 |
| 程式碼設計品質 | 15 | inject fallback、型別安全、責任分離 |
| 作業提交完整性 | 10 | 三個 Panel 均完成 + 整合測試通過 |

---

## 五、自我檢核問題

**Q1**：`ListPanel.vue` 的 `generic="T"` 和 `defineProps<{ items: T[] }>()` 為什麼要搭配使用？如果只寫 `defineProps<{ items: unknown[] }>()`，使用端會遇到什麼問題？

**Q2**：在 `FormPanel.vue` 中，若用 `inject(panelThemeKey)` 不帶 fallback，後續存取 `theme.borderRadius` 需要什麼額外處理？若用 `inject(panelThemeKey, defaultTheme)` 的 fallback，情況有何不同？

**Q3**：`DashboardLayout.vue` 的 `provide(panelThemeKey, reactive(theme))` 和 `provide(panelThemeKey, theme)`（普通物件）的行為差異在哪？當 `theme.shadow = 'lg'` 被執行後，各自的後代元件會發生什麼事？

**Q4**：KeepAlive 包裹的元件切換時，如果 FormPanel 有一個 `setInterval`，應該在哪個 lifecycle hook 中清除它？為什麼不能用 `onUnmounted`？

**Q5**：Scoped Slot 的「資料與渲染分離」是什麼意思？以 `ListPanel` 為例，誰「擁有資料」，誰「決定渲染」？這和直接把 UI 寫死在 ListPanel 裡有什麼差異？

---

## 六、明日預告（W6 Day 4）

明日（2026-05-29）：**W6 中期整合驗收 × TypeScript 型別完整審查 × 泛型 Scoped Slot 進階**

重點：
- W6 作業的完整功能驗收腳本（系統化 8–12 步驟）
- TypeScript 審查：泛型 T 的型別邊界、inject 回傳型別、`keyof typeof` 型別鏈
- Scoped Slot 進階：當 `ListPanel<T>` 的 item 有複雜型別（nested object）時，型別推導的限制與解法
- Composable 提取：`usePanelTheme.ts`（封裝 inject + fallback 邏輯，讓每個 Panel 只呼叫一次 Composable 而非直接 inject）
