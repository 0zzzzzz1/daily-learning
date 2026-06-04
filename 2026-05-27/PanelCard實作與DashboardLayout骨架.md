# M2-W6 Day 2：`PanelCard.vue` 完整實作 × `DashboardLayout.vue` 骨架建立
## 2026-05-27（W6 實作期啟動）

> **本週主題**：`provide` / `inject`、Slots、動態元件 → 預期產出：可插拔 UI 元件庫雛形
> **今日定位**：W6 第二天，昨日的完整理論今日轉為可運作程式碼

---

## 一、目標技術與核心知識點

| 技術 | 今日核心知識點 |
|------|--------------|
| InjectionKey 設計 | `injectionKeys.ts` 集中管理、`PanelTheme` 型別定義、Symbol + InjectionKey<T> 的 TS 推導 |
| PanelCard.vue Slots | Named Slot 三區域（header/default/footer）、`$slots.footer` 條件渲染、fallback content |
| DashboardLayout.vue | `provide(panelThemeKey, theme)` 架構、`<component :is>` 動態切換、`KeepAlive` 骨架 |
| inject 型別縮窄 | `inject` 回傳 `T | undefined`、必須提供 fallback 或 null check 才能安全使用 |

---

## 二、為什麼學這個（與前幾日的連結）

### 昨日到今日的推進

昨日（W6 Day 1）完成了三個主題的**完整理論層**：
- `provide` / `inject` 的 InjectionKey<T> 設計哲學
- Slots 三種形式（Default / Named / Scoped）的語法與設計範式
- KeepAlive 的運作機制（暫停 vs 銷毀、LRU 策略）

今日的任務是讓概念「落地」：**把每個知識點轉成真實可運作的 `.vue` 和 `.ts` 檔案**。

### 為什麼從 `injectionKeys.ts` 開始？

實作順序：**型別定義 → 葉節點元件 → 容器元件**

```
injectionKeys.ts          ← 先定義型別（供所有元件 import）
    ↓
PanelCard.vue             ← 葉節點（只使用 inject + Slots，不 provide）
    ↓
FormPanel / ListPanel     ← 使用 inject 拿主題 + 使用 PanelCard 的 Slots
    ↓
DashboardLayout.vue       ← 頂層（provide 主題 + 動態切換面板）
```

先定義 `PanelTheme` 型別，後續所有 `provide` / `inject` 才能得到正確的 TypeScript 推導。

---

## 三、知識說明

### 3a. `injectionKeys.ts` 設計

```ts
// src/injectionKeys.ts
import type { InjectionKey } from 'vue'

// 定義 PanelTheme 型別
export interface PanelTheme {
  borderRadius: 'none' | 'sm' | 'md' | 'lg'
  shadow: 'none' | 'sm' | 'md' | 'lg'
  headerBg: string  // CSS 顏色值，如 '#f0f4ff' 或 'var(--color-primary)'
}

// InjectionKey<T> = Symbol，TypeScript 將 T 附在 Symbol 的型別資訊上
// 使用 Symbol() 而非字串：唯一、不可偽造、不衝突
export const panelThemeKey: InjectionKey<PanelTheme> = Symbol('panelTheme')

// 📌 設計決策：為什麼要集中管理？
// 1. 所有 key 在同一檔案，避免散落各元件
// 2. 改 key 的型別時，只需改這一個檔案，所有 inject 點自動更新
// 3. 新人接手時，打開這個檔案就能知道「整個應用 provide 了什麼」
```

---

### 3b. `PanelCard.vue` 完整實作

#### 設計目標

```
┌─────────────────────────────┐
│  [#header]  ← Named Slot   │  → 沒給：顯示「未命名面板」fallback
├─────────────────────────────┤
│                             │
│  [default slot]             │  → 主要內容，無限制
│                             │
├─────────────────────────────┤
│  [#footer]  ← Named Slot   │  → 沒給：整個 <footer> DOM 不出現
└─────────────────────────────┘
```

#### 實作程式碼

```vue
<!-- src/components/PanelCard.vue -->
<script setup lang="ts">
import { inject } from 'vue'
import { panelThemeKey } from '../injectionKeys'
import type { PanelTheme } from '../injectionKeys'

// inject 主題設定，提供 fallback 確保 T 不含 undefined
const theme = inject<PanelTheme>(panelThemeKey, {
  borderRadius: 'md',
  shadow: 'sm',
  headerBg: '#f8fafc',
})

// 將 theme 轉為 CSS inline style
const headerStyle = computed(() => ({
  backgroundColor: theme.headerBg,
}))

// 將 borderRadius / shadow 轉為 CSS class（以 Tailwind 為例）
const borderRadiusClass = computed(() => ({
  'none': 'rounded-none',
  'sm': 'rounded-sm',
  'md': 'rounded-md',
  'lg': 'rounded-lg',
}[theme.borderRadius]))

const shadowClass = computed(() => ({
  'none': 'shadow-none',
  'sm': 'shadow-sm',
  'md': 'shadow-md',
  'lg': 'shadow-lg',
}[theme.shadow]))
</script>

<template>
  <!-- 最外層套用 borderRadius + shadow -->
  <div
    class="border border-gray-200 overflow-hidden"
    :class="[borderRadiusClass, shadowClass]"
  >
    <!-- Header：有 $slots.header 才渲染，否則顯示 fallback -->
    <header
      class="px-4 py-3 border-b border-gray-200 font-semibold text-gray-700"
      :style="headerStyle"
    >
      <slot name="header">
        <!-- fallback：父層沒提供 header 時顯示 -->
        <span class="text-gray-400 text-sm">未命名面板</span>
      </slot>
    </header>

    <!-- Default Slot：主要內容區 -->
    <main class="p-4">
      <slot />
    </main>

    <!-- Footer：只有父層有傳 footer 插槽內容時才渲染 -->
    <footer
      v-if="$slots.footer"
      class="px-4 py-3 border-t border-gray-100 bg-gray-50 flex justify-end gap-2"
    >
      <slot name="footer" />
    </footer>
  </div>
</template>
```

#### 關鍵設計說明

**為什麼 `$slots.footer` 這樣用？**

```vue
<!-- ❌ 不好：即使父層沒提供 footer，<footer> DOM 節點仍存在 -->
<!-- 可能導致元件底部有一條奇怪的邊框線 -->
<footer class="border-t">
  <slot name="footer" />
</footer>

<!-- ✅ 正確：父層沒提供 footer，整個 <footer> 不渲染 -->
<footer v-if="$slots.footer" class="border-t">
  <slot name="footer" />
</footer>
```

**inject fallback 的型別推導差異**：

```ts
// ❌ 回傳 PanelTheme | undefined，使用時需 null check
const theme = inject(panelThemeKey)
// theme.headerBg  // TypeScript 報錯：可能是 undefined

// ✅ 回傳 PanelTheme，使用時可直接存取
const theme = inject(panelThemeKey, { borderRadius: 'md', shadow: 'sm', headerBg: '#f8fafc' })
// theme.headerBg  // ✅ TypeScript 確定是 string
```

---

### 3c. `DashboardLayout.vue` 骨架建立

#### 實作程式碼

```vue
<!-- src/components/DashboardLayout.vue -->
<script setup lang="ts">
import { ref, reactive, provide } from 'vue'
import { panelThemeKey } from '../injectionKeys'
import type { PanelTheme } from '../injectionKeys'

// 定義三種面板（待 Day 3-4 完整實作）
// 使用 defineAsyncComponent 可懶加載（今日先用靜態 import）
import FormPanel from './FormPanel.vue'
import ListPanel from './ListPanel.vue'
import ChartPanel from './ChartPanel.vue'

// 面板設定：tabs 物件確保 TypeScript 推導正確
const panels = {
  form: FormPanel,
  list: ListPanel,
  chart: ChartPanel,
} as const

type PanelKey = keyof typeof panels

const currentPanel = ref<PanelKey>('form')

// 主題設定：使用 reactive 讓後代可以響應更新
const theme = reactive<PanelTheme>({
  borderRadius: 'md',
  shadow: 'md',
  headerBg: '#eff6ff',  // 淡藍色
})

// provide 主題給所有後代（包含多層深的子元件）
provide(panelThemeKey, theme)

// Tab 切換 handler
const switchPanel = (key: PanelKey) => {
  currentPanel.value = key
}
</script>

<template>
  <div class="dashboard-layout">
    <!-- Tab 切換列 -->
    <nav class="flex gap-1 mb-4 border-b border-gray-200">
      <button
        v-for="(_, key) in panels"
        :key="key"
        @click="switchPanel(key)"
        :class="[
          'px-4 py-2 text-sm font-medium transition-colors',
          currentPanel === key
            ? 'border-b-2 border-blue-500 text-blue-600'
            : 'text-gray-500 hover:text-gray-700',
        ]"
      >
        {{ key === 'form' ? '表單面板' : key === 'list' ? '列表面板' : '圖表面板' }}
      </button>
    </nav>

    <!-- 動態元件 + KeepAlive 保留狀態 -->
    <KeepAlive>
      <component :is="panels[currentPanel]" :key="currentPanel" />
    </KeepAlive>
  </div>
</template>
```

#### KeepAlive + `<component :is>` 的組合重點

```vue
<!-- ❌ 沒有 KeepAlive：每次切換都會銷毀並重建元件 -->
<component :is="panels[currentPanel]" />

<!-- ✅ 有 KeepAlive：切走後元件保留在記憶體，切回時狀態還在 -->
<KeepAlive>
  <component :is="panels[currentPanel]" />
</KeepAlive>
```

> **注意**：KeepAlive 只能包含**單一動態元件**（或條件渲染的單一元件）。如果需要同時快取多個，要用 `include` / `exclude` 精細控制。

---

### 3d. inject 型別縮窄實戰（常見痛點）

今日最容易卡關的地方：

```ts
// 狀況一：沒有 provide，inject 拿到 undefined
const theme = inject(panelThemeKey)
// 型別：PanelTheme | undefined

// 如果直接存取：
console.log(theme.headerBg)
// TypeScript 報錯：Object is possibly 'undefined'

// 解法 A：null check
if (theme) {
  console.log(theme.headerBg)  // ✅ 型別縮窄為 PanelTheme
}

// 解法 B：提供 fallback（推薦，最乾淨）
const theme = inject(panelThemeKey, { borderRadius: 'md', shadow: 'sm', headerBg: '#f8fafc' })
// 型別直接是 PanelTheme，不含 undefined

// 解法 C：!（非空斷言，確定一定有 provide 時才用）
const theme = inject(panelThemeKey)!
// 危險：如果真的沒有 provide，執行期會出錯
```

---

## 四、作業說明（W6 進行中，截止 6/1）

今日應完成的作業進度：

| 交付項目 | 今日目標 | 狀態 |
|---------|---------|------|
| `injectionKeys.ts` | 完整定義 PanelTheme + panelThemeKey | 🎯 今日完成 |
| `PanelCard.vue` | 三個 Slot 區域 + `$slots` 條件渲染 | 🎯 今日完成 |
| `DashboardLayout.vue` 骨架 | provide + `<component :is>` + KeepAlive 架構 | 🎯 今日完成 |
| FormPanel / ListPanel / ChartPanel | 骨架建立（內容下午或 Day 3 補完） | ⏳ 進行中 |

---

## 五、自我檢核問題

**Q1**：`inject(panelThemeKey)` 和 `inject(panelThemeKey, fallback)` 的回傳型別有什麼不同？在 `PanelCard.vue` 中，為什麼選擇提供 fallback 而不是 null check？

**Q2**：`DashboardLayout.vue` 中用 `reactive<PanelTheme>(theme)` 然後 `provide(panelThemeKey, theme)`，如果後來執行 `theme.headerBg = '#ff0000'`，後代的 `inject` 拿到的值會更新嗎？為什麼？

**Q3**：`<component :is="panels[currentPanel]" />` 的 `:is` 接受的是什麼型別？如果想改成懶加載（`defineAsyncComponent`），語法應該怎麼改？

**Q4**：`PanelCard.vue` 的 `$slots.footer` 和 fallback content（`<slot name="header"><span>未命名面板</span></slot>`）各自解決什麼問題？能否互換使用？

**Q5**：`KeepAlive` 包裹的元件在「切走」後，觸發的是什麼 lifecycle hook？如果 `FormPanel.vue` 裡有 `watch(someRef, handler)`，切走後這個 watch 還在作用嗎？

---

## 六、明日預告（W6 Day 3）

明日（2026-05-28）將繼續深化：**`FormPanel.vue` 完整實作 × `ListPanel.vue` Scoped Slot 完整版**

重點在於：
- `FormPanel.vue`：inject theme → 套用 borderRadius/shadow CSS class + 使用 `PanelCard` 的 Named Slots
- `ListPanel.vue`：Scoped Slot 完整版（`<slot :item="item" :index="index" />`）+ inject theme
- 在 `DashboardLayout.vue` 中整合三個面板，驗證 KeepAlive 保留 FormPanel 狀態
- 解決 Scoped Slot 的泛型型別問題（`generic="T"` + `defineProps<{ items: T[] }>()`）
