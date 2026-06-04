# M2-W6 Day 1：`provide` / `inject` × Slots 三種形式 × 動態元件與 KeepAlive
## 2026-05-26（W6 正式啟動）

> **本週主題**：`provide` / `inject`、Slots、動態元件 → 預期產出：可插拔 UI 元件庫雛形
> **今日定位**：W6 第一天，昨日的概念預習今日轉為完整實作模式

---

## 一、目標技術與核心知識點

| 技術 | 今日核心知識點 |
|------|--------------|
| `provide` / `inject` | InjectionKey<T> 型別安全、inject fallback、provide reactive 物件、App-level provide |
| Slots | Default / Named / Scoped 三種形式、`v-slot` 語法、fallback content、`$slots` 條件渲染 |
| 動態元件 | `<component :is>` 動態切換、KeepAlive 狀態保留、`include` / `exclude` / `max`、`activated` / `deactivated` |

---

## 二、為什麼學這個（與前幾日的連結）

### 知識鏈接點

W5（5/19–5/25）完整掌握了元件通訊的**父子垂直鏈**：
- Props Down：父層資料向下流動
- Events Up：子層事件向上通知
- defineModel：雙向綁定的語法糖

但這條鏈有一個隱性假設：**父子是相鄰的**。  
實際專案中，需求更複雜：

```
App
├── ThemeProvider       ← 主題設定在這裡
│   └── Layout
│       └── Sidebar
│           └── NavItem  ← 需要主題顏色（跨 3 層！）
```

如果用 Props，每一層都要宣告 `theme` prop 並往下傳。  
這就是 **Props Drilling 問題**，而 `provide / inject` 就是解法。

同理，Slots 解決了另一個問題：  
> 「我想讓這個 Card 元件能接受任意內容，但不想事先限制格式。」

動態元件則解決：  
> 「Tab 切換時，每個 Tab 渲染不同元件，且希望狀態在切換後不消失。」

**W6 的三個主題，每一個都在擴展元件的彈性邊界。**

---

## 三、知識說明

### 3a. `provide` / `inject`

#### 基本用法

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import { provide, reactive } from 'vue'

// provide 基本形式
provide('message', 'Hello from parent')

// 更好的做法：provide reactive 物件讓後代可以響應更新
const theme = reactive({
  color: 'blue',
  size: 'md',
})
provide('theme', theme)
</script>
```

```vue
<!-- DeepChildComponent.vue（不管中間隔幾層）-->
<script setup lang="ts">
import { inject } from 'vue'

// 基本 inject（回傳 T | undefined）
const message = inject('message')

// 帶 fallback 的 inject（回傳 T，不會是 undefined）
const theme = inject('theme', { color: 'gray', size: 'sm' })
</script>
```

#### ⚠️ 為什麼不能只用字串作為 key？

問題：字串 key 在大型專案中容易衝突，且 TypeScript 無法推導型別：

```ts
inject('theme') // 型別推導為 unknown — TypeScript 不知道值的型別
```

#### InjectionKey\<T\>：型別安全的跨層通訊

```ts
// injectionKeys.ts（集中管理所有 injection key）
import { InjectionKey, reactive } from 'vue'

// 定義型別安全的 key
export interface Theme {
  color: 'blue' | 'red' | 'green'
  size: 'sm' | 'md' | 'lg'
}

// InjectionKey<T> 是一個 Symbol，TypeScript 將 T 附在 Symbol 上
export const themeKey: InjectionKey<Theme> = Symbol('theme')
export const userKey: InjectionKey<{ name: string; role: string }> = Symbol('user')
```

```vue
<!-- ParentComponent.vue -->
<script setup lang="ts">
import { provide, reactive } from 'vue'
import { themeKey, type Theme } from './injectionKeys'

const theme = reactive<Theme>({
  color: 'blue',
  size: 'md',
})

// TypeScript 知道 provide 的值必須是 Theme 型別
provide(themeKey, theme)
</script>
```

```vue
<!-- DeepChildComponent.vue -->
<script setup lang="ts">
import { inject } from 'vue'
import { themeKey } from './injectionKeys'

// 正確推導：inject 回傳 Theme | undefined
const theme = inject(themeKey)

// 帶 fallback：inject 回傳 Theme（非 undefined）
const themeWithFallback = inject(themeKey, { color: 'gray', size: 'sm' })

// 有時候你確定一定會有值，但不想每次都寫 fallback：
// 用 ! 斷言（小心使用）
const themeAsserted = inject(themeKey)!
</script>
```

#### inject fallback 的設計哲學

```ts
// 三種 inject 使用情境

// 1. 可選依賴：有就用，沒有用預設
const theme = inject(themeKey, { color: 'gray', size: 'sm' })

// 2. 必要依賴：沒有就警告（工廠函式 fallback + 除錯提示）
const theme = inject(themeKey, () => {
  console.warn('[DeepChild] themeKey 未被 provide，使用預設值')
  return { color: 'gray', size: 'sm' }
}, true) // 第三個參數 true = 使用工廠函式（避免每次都執行昂貴計算）

// 3. 強制依賴：找不到就拋例外（極端情況）
const theme = inject(themeKey)
if (!theme) throw new Error('ThemeKey must be provided by a parent component')
```

#### provide reactive 物件 vs 普通值

```vue
<script setup>
import { provide, ref, reactive } from 'vue'

// ❌ 普通值：後代 inject 後拿到的是一個凍結的 snapshot，不響應更新
provide('count', 42)

// ✅ ref：後代 inject 拿到的是 ref，修改 .value 後代會更新
const count = ref(0)
provide('count', count)

// ✅ reactive：後代 inject 拿到的是 reactive proxy，修改屬性後代會更新
const state = reactive({ count: 0, name: '' })
provide('state', state)
</script>
```

#### App-level provide（全域設定）

```ts
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import { globalConfigKey } from './injectionKeys'

const app = createApp(App)

// 應用層 provide，所有元件都能 inject
app.provide(globalConfigKey, {
  apiBaseUrl: 'https://api.example.com',
  locale: 'zh-TW',
})

app.mount('#app')
```

> **適用場景**：全域設定（API URL、語言、主題）；Plugin 注入能力；第三方庫的服務注入

---

### 3b. Slots — 三種形式完整程式碼

#### Default Slot（預設插槽）

最簡單的插槽形式，父層直接插入內容：

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot />  <!-- 插槽出口 -->
  </div>
</template>
```

```vue
<!-- 使用 Card.vue -->
<Card>
  <p>這段話會出現在 Card 元件內部</p>
  <button>按鈕也可以</button>
</Card>
```

**Fallback content**（當父層沒有提供內容時顯示）：

```vue
<!-- Card.vue -->
<template>
  <div class="card">
    <slot>
      <!-- fallback：父層沒插入內容時顯示 -->
      <p class="text-gray-400">尚無內容</p>
    </slot>
  </div>
</template>
```

---

#### Named Slots（具名插槽）

當一個元件需要多個插槽時：

```vue
<!-- LayoutPanel.vue -->
<template>
  <div class="panel">
    <!-- 具名插槽：name 屬性指定名稱 -->
    <header class="panel-header">
      <slot name="header">
        <span>預設標題</span>
      </slot>
    </header>

    <main class="panel-body">
      <!-- 未命名的 slot = name="default" -->
      <slot />
    </main>

    <footer class="panel-footer">
      <slot name="footer" />
    </footer>
  </div>
</template>
```

```vue
<!-- 使用 LayoutPanel.vue -->
<LayoutPanel>
  <!-- v-slot:name 指定對應插槽 -->
  <template v-slot:header>
    <h2>自訂標題</h2>
  </template>

  <!-- 簡寫：# 等同 v-slot: -->
  <template #footer>
    <button>確定</button>
    <button>取消</button>
  </template>

  <!-- 預設插槽：不需要 template 包裝（但加了也可以）-->
  <p>這是主要內容區塊</p>
</LayoutPanel>
```

---

#### Scoped Slots（作用域插槽）⭐ 最重要的設計範式

核心概念：**子元件有資料，但父元件決定如何渲染**

```vue
<!-- DataList.vue：負責「有什麼資料」，不管「怎麼顯示」-->
<script setup lang="ts" generic="T">
defineProps<{
  items: T[]
}>()
</script>

<template>
  <ul>
    <li v-for="(item, index) in items" :key="index">
      <!-- Scoped Slot：把 item 和 index 暴露給父元件 -->
      <slot :item="item" :index="index" />
    </li>
  </ul>
</template>
```

```vue
<!-- 使用 DataList.vue：父層決定如何渲染每個 item -->
<DataList :items="users">
  <!-- #default 接收 Scoped Slot 暴露的資料 -->
  <template #default="{ item, index }">
    <span class="index">{{ index + 1 }}.</span>
    <strong>{{ item.name }}</strong>
    <em>{{ item.role }}</em>
  </template>
</DataList>

<!-- 同一個 DataList，用不同方式渲染 -->
<DataList :items="products">
  <template #default="{ item }">
    <div class="product-card">
      <img :src="item.image" :alt="item.name" />
      <p>NT$ {{ item.price }}</p>
    </div>
  </template>
</DataList>
```

**這就是「資料與渲染分離」的設計範式**：
- `DataList` 只管迭代邏輯（什麼資料、如何排序）
- 父層只管 UI（怎麼呈現每個 item）
- 兩者職責清晰，高度可複用

#### 用 `$slots` 做條件渲染

```vue
<!-- ConditionalPanel.vue -->
<template>
  <div class="panel">
    <!-- 只有父層有傳 header 插槽內容時才顯示 header 區塊 -->
    <header v-if="$slots.header" class="panel-header">
      <slot name="header" />
    </header>

    <main>
      <slot />
    </main>

    <footer v-if="$slots.footer" class="panel-footer">
      <slot name="footer" />
    </footer>
  </div>
</template>
```

> **為什麼重要**：父層沒有提供 header 時，連 `<header>` 的 DOM 節點都不要建立，確保元件樣式不因空 slot 而產生奇怪的空白或邊框。

---

### 3c. 動態元件與 KeepAlive

#### `<component :is>` — 動態切換元件

```vue
<script setup lang="ts">
import { ref, defineAsyncComponent } from 'vue'
import HomeTab from './HomeTab.vue'
import SettingsTab from './SettingsTab.vue'
import ProfileTab from './ProfileTab.vue'

// 可以是元件物件、元件名稱字串（已全域註冊）、或 async component
const tabs = {
  home: HomeTab,
  settings: SettingsTab,
  profile: ProfileTab,
} as const

type TabKey = keyof typeof tabs

const currentTab = ref<TabKey>('home')
</script>

<template>
  <nav>
    <button
      v-for="tab in (Object.keys(tabs) as TabKey[])"
      :key="tab"
      @click="currentTab = tab"
      :class="{ active: currentTab === tab }"
    >
      {{ tab }}
    </button>
  </nav>

  <!-- :is 接受元件定義物件或已註冊的元件名稱 -->
  <component :is="tabs[currentTab]" />
</template>
```

#### KeepAlive — 保留切換後的狀態

**問題**：每次 `currentTab` 改變，舊的元件會被銷毀（`onUnmounted`），新的元件從頭建立（`onMounted`）。  
用戶在表單裡填到一半的資料會消失。

```vue
<template>
  <!-- 用 KeepAlive 包裹動態元件 -->
  <KeepAlive>
    <component :is="tabs[currentTab]" />
  </KeepAlive>
</template>
```

**KeepAlive 的行為**：
- 第一次進入：`onMounted` 執行，元件建立
- 切走：元件**不被銷毀**，改為「暫停」，觸發 `onDeactivated`
- 切回來：元件**不重建**，改為「恢復」，觸發 `onActivated`

#### activated / deactivated 生命週期

```vue
<script setup lang="ts">
import { onActivated, onDeactivated } from 'vue'

// 只在 KeepAlive 中才會觸發（普通元件不會）
onActivated(() => {
  console.log('Tab 被切回來了，可以做資料刷新')
  // 適合：重新 fetch 可能過期的資料
  fetchLatestData()
})

onDeactivated(() => {
  console.log('Tab 被切走了，可以暫停計時器/停止動畫')
  // 適合：暫停 setInterval、停止 WebSocket 監聽（節省資源）
  clearInterval(timer)
})
</script>
```

#### KeepAlive 精細控制

```vue
<template>
  <!-- include：只快取名稱符合的元件 -->
  <KeepAlive include="HomeTab,ProfileTab">
    <component :is="tabs[currentTab]" />
  </KeepAlive>

  <!-- exclude：排除指定元件不快取 -->
  <KeepAlive exclude="SettingsTab">
    <component :is="tabs[currentTab]" />
  </KeepAlive>

  <!-- max：最多快取 N 個元件（超過時淘汰最久未使用的，LRU 策略）-->
  <KeepAlive :max="3">
    <component :is="tabs[currentTab]" />
  </KeepAlive>
</template>
```

> **注意**：`include` / `exclude` 匹配的是元件的 **`name` 選項**（或 `<script setup>` 的檔案名）：
> ```vue
> <!-- HomeTab.vue -->
> <script>
> export default { name: 'HomeTab' }
> </script>
> ```

---

## 四、作業說明

### W6 作業：可插拔 UI 元件庫雛形（截止：2026-06-01）

**情境背景**：

你正在為 ihouseBMS 設計一套可複用的面板系統。PM 說：
> 「我們需要一個 `DashboardLayout` 元件，它能接受任意類型的面板卡片，每張卡片的樣式可以客製化，而且切換面板時不要失去填入的資料。」

這個需求正好涵蓋今天學的三個技術：
1. **provide / inject**：`DashboardLayout` 提供全域 layout 設定（如：欄數、間距主題）
2. **Slots**：每張面板卡片可以接受客製化 header、body、footer
3. **動態元件 + KeepAlive**：不同面板類型動態切換，且保留狀態

---

### 功能需求

#### 1. `PanelCard.vue`（基礎面板元件，使用 Slots）

```
┌─────────────────────────────┐
│  [#header]                  │  ← Named Slot：標題區
├─────────────────────────────┤
│                             │
│  [default slot]             │  ← Default Slot：主要內容
│                             │
├─────────────────────────────┤
│  [#footer]                  │  ← Named Slot：操作區（可選）
└─────────────────────────────┘
```

需求：
- 使用 Named Slots 定義三個區域（header / default / footer）
- 若父層未提供 footer，該區域的 DOM 不應出現（使用 `$slots.footer`）
- 若父層未提供 header，顯示 fallback 文字「未命名面板」

#### 2. `panelThemeKey`（InjectionKey，類型安全的設定共享）

```ts
// injectionKeys.ts
export interface PanelTheme {
  borderRadius: 'none' | 'sm' | 'md' | 'lg'
  shadow: 'none' | 'sm' | 'md' | 'lg'
  headerBg: string  // CSS 顏色值
}

export const panelThemeKey: InjectionKey<PanelTheme> = Symbol('panelTheme')
```

#### 3. `DashboardLayout.vue`（provide 設定 + 動態元件切換）

```
┌──────────────────────────────────────────────┐
│  [Tab 1] [Tab 2] [Tab 3]                     │  ← 切換面板
├──────────────────────────────────────────────┤
│                                              │
│  <KeepAlive>                                 │
│    <component :is="currentPanel" />          │  ← 動態元件 + 保留狀態
│  </KeepAlive>                                │
│                                              │
└──────────────────────────────────────────────┘
```

需求：
- provide `panelThemeKey` 讓所有子面板可以 inject 主題設定
- 使用 `<component :is>` 動態切換三種面板（FormPanel / ListPanel / ChartPanel）
- 使用 `KeepAlive` 保留各面板的填寫狀態

#### 4. 三種面板元件（inject 主題設定）

- **FormPanel.vue**：一個有輸入欄位的表單（inject 主題 → 套用 borderRadius / shadow）
- **ListPanel.vue**：一個清單，Scoped Slot 暴露每個 item 給父層客製化渲染
- **ChartPanel.vue**：靜態圖表佔位（可以用 `<div>` 模擬），inject 主題 → headerBg

---

### 技術規範

| 規範 | 要求 |
|------|------|
| InjectionKey | 必須使用 `Symbol` + `InjectionKey<T>`，不可用字串 |
| inject fallback | 所有 inject 必須提供 fallback 值 |
| Slot 條件渲染 | footer slot 使用 `$slots.footer` 判斷 |
| Scoped Slot | ListPanel 的 item 渲染必須使用 Scoped Slot |
| KeepAlive | 切換面板後，FormPanel 的輸入內容不可消失 |
| TypeScript | 無 `any`，InjectionKey 型別安全 |

---

### 評分標準

| 項目 | 分數 |
|------|------|
| `PanelCard.vue` Slots 正確運作（header / default / footer / fallback）| 25 分 |
| `panelThemeKey` 使用 `InjectionKey<T>` 型別安全 | 15 分 |
| `DashboardLayout.vue` provide + 動態元件切換 | 20 分 |
| KeepAlive 保留 FormPanel 狀態 | 15 分 |
| ListPanel Scoped Slot 正確暴露 item | 15 分 |
| TypeScript 無 `any`，整體型別安全 | 10 分 |

---

## 五、自我檢核問題

**Q1**：`inject(key)` 和 `inject(key, fallback)` 的回傳型別有什麼差異？在 TypeScript 中，什麼情況下必須提供 fallback？

**Q2**：為什麼要用 `Symbol` 作為 injection key，而不是用字串？`InjectionKey<T>` 的 `T` 在 TypeScript 推導中扮演什麼角色？

**Q3**：Scoped Slot 的 `<slot :item="item" />` 和 Named Slot 的 `<slot name="header" />` 可以同時使用嗎？語法上如何在父層同時使用兩種特性？

**Q4**：KeepAlive 的元件是真的在「暫停」還是其實保留在記憶體中？切回來時觸發的是什麼生命週期？如果裡面的元件有 `setInterval`，切走後 interval 還在跑嗎？

**Q5**：`<component :is="tabs[currentTab]" />` 的 `:is` 接受哪些類型的值？如果想要懶加載（lazy load）某個 tab 的元件，應該怎麼改寫？

---

## 六、明日預告（W6 Day 2）

明日將進入 W6 的第二個面向：**實作 `PanelCard.vue` + `ListPanel.vue`**

重點在於：
- 把今日的概念轉為可運作的程式碼
- 實際體驗 Scoped Slot 的「資料與渲染分離」設計感
- 解決 `inject` 的 TypeScript 型別推導問題（常見痛點：型別縮窄）
- 初步整合 `DashboardLayout.vue` 的 provide 架構

> 昨日（W5 Day 7）預習已建立概念底座，今日是第一層實作，明日起逐步完整。
