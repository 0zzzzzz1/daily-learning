# W6 中期整合驗收 × TypeScript 型別完整審查 × usePanelTheme Composable 提取

> **日期**：2026-05-29（M2-W6 Day 4）
> **週次**：W6（5/26–6/1），截止 6/1，剩 **3 天**
> **學習階段**：M2 Composition API 深化期

---

## ⚠️ 作業堆積警示

| 週次 | 截止日 | 逾期天數 | 作業項目 |
|------|--------|---------|---------|
| W3 | 2026-05-11 | 18 天 | TaskList.vue v2 |
| W4 | 2026-05-18 | 11 天 | LifecycleLogger.vue |
| W5 | 2026-05-25 | 4 天 | CustomInput + CustomSelect + LoginForm |

**累積 3 份以上作業未確認** → 請在今日學習結束後，登入 GitHub 逐一確認 W3/W4/W5 的 PR 狀態，並回來更新 `前端技術分析.md` 的評分欄。W6 截止 6/1（剩 3 天），今日務必完成中期驗收！

---

## 一、目標技術 & 核心知識點

| 目標技術 | 核心知識點 |
|---------|-----------|
| Vue.js 3 Composable | `usePanelTheme.ts`：封裝 inject + fallback + computed 轉換 |
| TypeScript | 泛型邊界（`T extends object`）× inject 回傳型別 × `keyof typeof` 型別鏈 |
| Vue.js 3 整合驗收 | 功能驗收腳本（8–12 步驟）× Scoped Slot 複雜型別推導限制 |
| TypeScript 審查流程 | 定義層 → 函式簽名 → 使用層 → `vue-tsc --noEmit` |

---

## 二、為什麼學這個（與昨日的連結）

昨日（W6 Day 3）完成了整個 DashboardLayout 的實作：
- `FormPanel.vue`（inject theme → CSS class 查找表 × Named Slots × KeepAlive 狀態保留）
- `ListPanel.vue`（`generic="T"` 泛型 Scoped Slot × 型別推導鏈）
- `ChartPanel.vue`（inject headerBg × style 綁定）
- `DashboardLayout.vue`（三面板整合 × 響應式 theme 切換確認）

今日進入「**精煉期**」——三個 Panel 都有 `inject(panelThemeKey, fallback)` 的樣板程式碼。這種重複是 Composable 提取的明確訊號。同時，昨日的實作完成後需要做完整的型別審查與功能驗收，確認 W6 作業在截止前（6/1）達到提交標準。

今日三大主軸：
1. **`usePanelTheme.ts` 提取**：消除三個 Panel 的 inject 重複
2. **TypeScript 型別完整審查**：泛型邊界、inject 型別、`keyof typeof` 鏈的正確性
3. **W6 功能驗收腳本**：系統化確認整個 DashboardLayout 的行為

---

## 三、知識說明

### 3.1 usePanelTheme Composable 提取

**問題**：目前 FormPanel、ListPanel、ChartPanel 都有相同的 inject 邏輯：

```typescript
// 三個 Panel 都有這段重複的程式碼
const theme = inject(panelThemeKey, defaultTheme)
const borderRadiusClass = computed(() => borderRadiusMap[theme.borderRadius] ?? 'rounded')
const shadowClass = computed(() => shadowMap[theme.shadow] ?? 'shadow')
```

這違反了 DRY（Don't Repeat Yourself）原則。解決方案是提取成 `usePanelTheme`：

```typescript
// composables/usePanelTheme.ts
import { inject, computed } from 'vue'
import { panelThemeKey, defaultTheme } from '@/injectionKeys'
import type { PanelTheme } from '@/injectionKeys'

// 查找表：語意值 → Tailwind class
const borderRadiusMap: Record<PanelTheme['borderRadius'], string> = {
  none: 'rounded-none',
  sm: 'rounded-sm',
  md: 'rounded-md',
  lg: 'rounded-lg',
}

const shadowMap: Record<PanelTheme['shadow'], string> = {
  none: 'shadow-none',
  sm: 'shadow-sm',
  md: 'shadow-md',
  lg: 'shadow-lg',
}

const headerBgMap: Record<PanelTheme['headerBg'], string> = {
  white: 'bg-white',
  gray: 'bg-gray-100',
  blue: 'bg-blue-50',
}

export function usePanelTheme() {
  const theme = inject(panelThemeKey, defaultTheme)

  const borderRadiusClass = computed(
    () => borderRadiusMap[theme.borderRadius] ?? 'rounded'
  )
  const shadowClass = computed(
    () => shadowMap[theme.shadow] ?? 'shadow'
  )
  const headerBgClass = computed(
    () => headerBgMap[theme.headerBg] ?? 'bg-white'
  )

  return {
    theme,
    borderRadiusClass,
    shadowClass,
    headerBgClass,
  }
}
```

**使用端（重構後的 FormPanel.vue）**：

```vue
<script setup lang="ts">
import { usePanelTheme } from '@/composables/usePanelTheme'
// 一行取代原本的 inject + computed × 3
const { borderRadiusClass, shadowClass, headerBgClass } = usePanelTheme()
</script>
```

**設計決策解析**：

| 決策 | 選擇 | 理由 |
|------|------|------|
| 查找表放在 Composable 內部 | ✅ 內部實作 | 每個 Panel 不需知道查找表的存在，符合封裝原則 |
| 回傳 `theme` 原始物件 | ✅ 一併回傳 | 有些 Panel 可能需要直接存取 theme 屬性（如 ChartPanel 的自訂邏輯） |
| 查找表型別：`Record<PanelTheme['shadow'], string>` | ✅ 精確索引型別 | 比 `Record<string, string>` 更安全；若 PanelTheme 的 union 新增值，TypeScript 會在查找表報錯，強迫開發者補上對應的 class |

---

### 3.2 TypeScript 型別完整審查

#### 審查流程（由根到葉）

```
型別定義層（injectionKeys.ts）
    ↓
函式簽名層（usePanelTheme.ts 回傳型別）
    ↓
使用層（FormPanel.vue / ListPanel.vue / ChartPanel.vue）
    ↓
vue-tsc --noEmit（靜態分析零警告）
```

#### 重點 1：`Record<PanelTheme['borderRadius'], string>` 的精確性

```typescript
// PanelTheme 定義
interface PanelTheme {
  borderRadius: 'none' | 'sm' | 'md' | 'lg'
  shadow: 'none' | 'sm' | 'md' | 'lg'
  headerBg: 'white' | 'gray' | 'blue'
}

// 查找表型別：PanelTheme['borderRadius'] 展開為 'none' | 'sm' | 'md' | 'lg'
const borderRadiusMap: Record<PanelTheme['borderRadius'], string> = {
  none: 'rounded-none',
  sm: 'rounded-sm',
  md: 'rounded-md',
  lg: 'rounded-lg',
  // ✅ 若漏寫任一 key，TypeScript 立即報錯（Property 'lg' is missing）
}
```

vs 錯誤寫法：
```typescript
// ❌ Record<string, string>：key 可以是任何字串，型別保護完全喪失
const borderRadiusMap: Record<string, string> = {
  md: 'rounded-md',
  // 漏寫 none/sm/lg → TypeScript 沉默，執行期才出 undefined
}
```

#### 重點 2：`generic="T"` 的型別邊界限制

**問題**：若 ListPanel 的 items 是 nested object（如 `{ user: { name: string } }`），TypeScript 能正確推導嗎？

```typescript
// 使用端
interface Task {
  id: number
  title: string
  meta: {
    createdAt: Date
    priority: 'high' | 'medium' | 'low'
  }
}

const tasks: Task[] = [...]
```

```vue
<!-- 使用端 -->
<ListPanel :items="tasks">
  <template #default="{ item }">
    <!-- item 型別：Task（TypeScript 正確推導）✅ -->
    {{ item.title }}           <!-- ✅ string -->
    {{ item.meta.priority }}   <!-- ✅ 'high' | 'medium' | 'low' -->
    {{ item.meta.createdAt }}  <!-- ✅ Date -->
  </template>
</ListPanel>
```

**結論**：TypeScript 的泛型推導是深層的——`T = Task` 後，所有存取都有型別保護，不需要額外的型別斷言。

**限制場景**：只有當 `generic="T"` 和 `T extends SomeConstraint` 的約束不匹配時才會報錯：

```typescript
// 若 ListPanel 加了約束
// <script setup lang="ts" generic="T extends { id: number }">
// defineProps<{ items: T[] }>()

// 使用端傳入 string[]：
<ListPanel :items="['a', 'b']">  // ❌ string 不符合 { id: number }
```

#### 重點 3：inject 回傳型別的正確性

```typescript
// 不帶 fallback：T | undefined
const theme = inject(panelThemeKey)
// 型別：PanelTheme | undefined

// 帶 fallback：T（保證有值）
const theme = inject(panelThemeKey, defaultTheme)
// 型別：PanelTheme（不含 undefined）✅

// usePanelTheme 內部用 fallback → 對外暴露的 theme 型別就是 PanelTheme
// 使用端不需做 null check
```

---

### 3.3 W6 功能驗收腳本（12 步驟）

驗收腳本是「人工版 Unit Test」——每個步驟都有「操作」與「確認」兩個部分。

| 步驟 | 操作 | 確認 |
|------|------|------|
| 1 | 啟動開發伺服器，開啟 DashboardLayout 頁面 | 三個 Panel 均正常渲染，預設 theme 套用正確 |
| 2 | 點擊「切換至 ListPanel」 | ListPanel 切換顯示，tasks 列表正確渲染（Scoped Slot 正常） |
| 3 | 點擊「切換至 FormPanel」，填入姓名「測試用戶」 | FormPanel 顯示，輸入欄位可填寫 |
| 4 | 點擊「切換至 ListPanel」再切回「FormPanel」 | FormPanel 的「測試用戶」仍在（KeepAlive 生效）✅ |
| 5 | 點擊「切換至 ChartPanel」 | ChartPanel 顯示，headerBg 正確套用（headerBgClass）|
| 6 | 點擊「切換 Theme」（shadow: 'none' → 'lg'）| 三個 Panel 同步更新 shadow 樣式（響應式 provide 確認）|
| 7 | 再次切換 theme（borderRadius: 'sm' → 'lg'）| 三個 Panel 圓角同步更新 |
| 8 | 開啟 Vue DevTools → Components | 確認 DashboardLayout provides `panelThemeKey` × 三個 Panel injects 它 |
| 9 | 在 Vue DevTools 修改 theme.shadow → 'none' | 三個 Panel 即時反應（Proxy 共享機制驗證）|
| 10 | 執行 `vue-tsc --noEmit` | 零 TypeScript 錯誤 ✅ |
| 11 | 在 ListPanel 傳入不同型別的 items（如 `string[]`）| TypeScript 正確推導 item 型別（或約束限制報錯）|
| 12 | 確認 Console 無 runtime 錯誤、無 Vue warning | 零 warning、零 error ✅ |

---

### 3.4 常見陷阱：usePanelTheme 的使用位置限制

```typescript
// ⚠️ 錯誤：在 setup() 以外的地方呼叫（如事件處理函式內）
function handleClick() {
  const { theme } = usePanelTheme() // ❌ 此時無 Active Instance！
}

// ✅ 正確：在 <script setup> 的頂層同步呼叫
const { borderRadiusClass, shadowClass } = usePanelTheme() // ✅
```

這和所有 Composable 一樣的限制：必須在 `setup()` 同步執行流中呼叫，因為 `inject` 需要 Active Instance 上下文。

---

### 3.5 Composable 回傳型別明確標注（API 契約）

```typescript
// 方式一：TypeScript 自動推導（適合簡單 Composable）
export function usePanelTheme() {
  // ... 回傳型別由 TypeScript 推導
  return { theme, borderRadiusClass, shadowClass, headerBgClass }
}

// 方式二：明確標注（適合作為 API 契約的 Composable）
interface PanelThemeResult {
  theme: PanelTheme
  borderRadiusClass: Readonly<Ref<string>>
  shadowClass: Readonly<Ref<string>>
  headerBgClass: Readonly<Ref<string>>
}

export function usePanelTheme(): PanelThemeResult {
  // ...
}
```

**選用建議**：若 `usePanelTheme` 會被多個元件使用（如本案的三個 Panel），明確標注回傳型別能作為「API 契約」——修改 Composable 內部實作時，TypeScript 會立即檢查回傳值是否符合契約。

---

## 四、作業說明（W6 截止 6/1，剩 3 天）

### 今日應完成的作業進度

| 交付項目 | 說明 | 目標 |
|---------|------|------|
| `composables/usePanelTheme.ts` | inject + fallback + computed 查找表封裝 | 🎯 今日完成 |
| 重構三個 Panel 使用 `usePanelTheme` | FormPanel / ListPanel / ChartPanel 各只需一行 | 🎯 今日完成 |
| W6 功能驗收腳本執行 | 12 步驟完整跑過，截圖或文字紀錄 | 🎯 今日完成 |
| `vue-tsc --noEmit` 零錯誤 | 確認 TypeScript 靜態分析通過 | 🎯 今日完成 |

### 技術規範

1. **`usePanelTheme.ts`**：
   - 查找表型別使用 `Record<PanelTheme['borderRadius'], string>`（精確索引型別）
   - 回傳 `{ theme, borderRadiusClass, shadowClass, headerBgClass }`
   - 選擇性：明確標注 `PanelThemeResult` 回傳型別

2. **重構三個 Panel**：
   - 移除各自的 `inject` + `computed` 查找表重複程式碼
   - 改為 `const { borderRadiusClass, shadowClass } = usePanelTheme()`
   - 確認行為不變（功能驗收腳本再跑一次）

3. **驗收文件**（可選但建議）：
   - 在今日學習目錄下建立簡短的 `驗收結果.md`，記錄 12 步驟的執行結果

### 評分標準（延續 Day 3 的作業基礎）

| 面向 | 滿分 | 今日重點 |
|------|------|---------|
| usePanelTheme 設計正確性 | +10 | 查找表型別精確、inject fallback 正確 |
| 重構後行為不變 | +5 | 功能驗收腳本 12 步驟全 Pass |
| TypeScript 靜態分析通過 | +5 | `vue-tsc --noEmit` 零錯誤 |
| 程式碼整潔度 | +5 | 三個 Panel 的 inject 重複被完全消除 |

---

## 五、自我檢核問題

**Q1**：`usePanelTheme` Composable 為什麼要把查找表（`borderRadiusMap` 等）放在函式外部（module 層級），而不是每次呼叫 `usePanelTheme()` 時都重新建立？

**Q2**：若將 `usePanelTheme` 的查找表型別從 `Record<PanelTheme['borderRadius'], string>` 改為 `Record<string, string>`，會喪失哪些型別保護？何時這個差異會在實務中造成 bug？

**Q3**：`usePanelTheme` 的回傳值中，`borderRadiusClass` 是 `Computed<string>`（包裹在 ref 中），但在 template 中可以直接 `{{ borderRadiusClass }}` 或 `:class="borderRadiusClass"` 而不需要 `.value`，這是因為什麼機制？

**Q4**：若 `ListPanel` 的 `generic="T"` 沒有加任何約束（即 `T` 可以是任何型別），但使用端傳入 `items="hello"`（字串而非陣列），TypeScript 會怎麼處理？

**Q5**：當三個 Panel 都呼叫 `usePanelTheme()` 時，`inject(panelThemeKey, defaultTheme)` 在每個 Panel 中分別執行一次，但最終它們拿到的 `theme` 是同一個 reactive 物件的 Proxy 嗎？為什麼？

---

## 六、明日預告（W6 Day 5）

明日（2026-05-30）：**W6 Commit 整理 × PR 說明定稿 × W6 七天知識閉環預備**

重點：
- `git add -p` 細粒度分批 commit（把 Day 1–4 的實作整理成清晰的 commit 序列）
- Conventional Commits 選用：`feat`（新元件）vs `refactor`（usePanelTheme 提取）vs `chore`（型別審查）
- W6 PR 說明：三個 Why 設計決策（`InjectionKey<T>` 選 Symbol 的理由 / 泛型 Scoped Slot 的設計理由 / usePanelTheme Composable 提取時機）
- W6 七天知識閉環預備：整理 provide/inject × Slots × 動態元件 × KeepAlive 的核心知識鏈
- `--force-with-lease` 安全 push 複習（若有 rebase 整理）
