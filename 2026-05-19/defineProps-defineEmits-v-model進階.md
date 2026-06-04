# defineProps / defineEmits / v-model 進階

> **日期**：2026-05-19（週二）
> **週次**：M2-W5 Day 1（新週次起點）
> **月份主軸**：Composition API 深化期
> **本週交付目標**：雙向綁定自訂元件（Custom Input with v-model）

---

## 目標技術

| 技術 | 重點 |
|------|------|
| `defineProps` | TS 型別宣告方式、`withDefaults`、required vs optional |
| `defineEmits` | 型別宣告、payload 型別約束、emit 命名規範 |
| `v-model` | 語法糖本質展開、單一 v-model、多個 v-model（Vue 3.4 `defineModel`）|
| 元件通訊設計 | Props Down / Events Up 單向資料流原則 |

---

## 核心知識點

1. `defineProps` 的 Runtime 宣告 vs TypeScript 型別宣告
2. `withDefaults` 為 TS 型別宣告補充預設值
3. `defineEmits` 的型別宣告與 payload 型別約束
4. `v-model` 展開本質：`:modelValue` + `@update:modelValue`
5. 多個 `v-model`：`v-model:title`、`v-model:content`
6. Vue 3.4 `defineModel` macro（簡化版）
7. 常見陷阱：在子元件直接修改 props

---

## 為什麼學這個

### 從 W4 的「時間維度」走向 M2 的「空間維度」

W4 深入掌握了單一元件的生命週期（Lifecycle Hooks）、副作用管理（watchEffect/onCleanup）和介面設計（Composable 回傳型別契約）。

這些知識的核心問題是：**一個元件如何在時間軸上管理自己的狀態？**

M2 的核心問題則是：**多個元件之間如何安全地交換資料與事件？**

元件通訊是連接這兩個維度的橋樑。從 W3 的 TaskList.vue（單一元件 + Composable）到 M2 的自訂元件庫（多元件組合），需要一套明確的通訊契約——`defineProps` 和 `defineEmits` 正是這份契約的語言。

### 與昨日（5/18）M2 預習的銜接

昨日已建立了對 `defineProps`、`defineEmits`、`v-model` 本質的概念理解。今天從預習進入系統化深化：
- 昨日：知道「`v-model` 是 Props + Emits 的語法糖」
- 今日：理解語法糖的展開細節、TS 型別宣告方式、多個 v-model 的設計決策

---

## 知識說明

### 一、`defineProps` 的兩種宣告方式

#### 1a. Runtime 宣告（JavaScript 風格）

```vue
<script setup>
const props = defineProps({
  label: String,
  modelValue: {
    type: String,
    required: true
  },
  disabled: {
    type: Boolean,
    default: false
  }
})
</script>
```

**優點**：語法直覺，不需 TypeScript。  
**缺點**：型別不夠精確（`String` vs 字串字面量聯合型別），無法表達複雜型別。

#### 1b. TypeScript 型別宣告（推薦）

```vue
<script setup lang="ts">
interface Props {
  label: string
  modelValue: string
  placeholder?: string
  disabled?: boolean
  errorMessage?: string
}

const props = defineProps<Props>()
</script>
```

**重點**：
- `?` 表示 optional（可省略），否則為 required
- 不能同時使用 Runtime 宣告和 TS 型別宣告
- 型別可以是 interface、type alias 或 inline object type

#### 1c. `withDefaults`：為 TS 型別宣告補充預設值

```vue
<script setup lang="ts">
interface Props {
  label: string
  modelValue: string
  placeholder?: string
  disabled?: boolean
  errorMessage?: string
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: '請輸入...',
  disabled: false,
  errorMessage: ''
})
</script>
```

**為什麼需要 `withDefaults`**：
TS 型別宣告無法像 Runtime 宣告那樣直接指定 `default`，所以需要 `withDefaults` 包裹。
`withDefaults` 會讓帶有預設值的 optional props 在型別上被視為已定義（非 `undefined`），讓後續使用更安全。

---

### 二、`defineEmits` 的型別宣告

#### 2a. Runtime 宣告

```vue
<script setup>
const emit = defineEmits(['update:modelValue', 'blur', 'focus'])
</script>
```

#### 2b. TypeScript 型別宣告（推薦）

```vue
<script setup lang="ts">
const emit = defineEmits<{
  'update:modelValue': [value: string]
  blur: [event: FocusEvent]
  focus: [event: FocusEvent]
}>()
</script>
```

**格式**：`事件名稱: [參數名稱: 型別, ...]`（tuple 型別）

**優點**：
- TypeScript 會驗證 `emit('update:modelValue', someValue)` 中的 `someValue` 必須是 `string`
- IDE 能提示 emit 的參數型別

**命名規範**：
- 事件名用 camelCase（`update:modelValue`、`focusIn`），HTML 中用 kebab-case（`:focus-in`）
- `update:propName` 是 `v-model` 的專屬命名規範

---

### 三、`v-model` 語法糖展開

#### 3a. 基本 v-model

在父元件：
```vue
<CustomInput v-model="searchText" />
```

Vue 展開等價於：
```vue
<CustomInput
  :modelValue="searchText"
  @update:modelValue="searchText = $event"
/>
```

在子元件（`CustomInput.vue`）中，必須：
1. 接收 `modelValue` prop
2. 不直接修改 `modelValue`，而是 `emit('update:modelValue', 新值)`

```vue
<script setup lang="ts">
const props = defineProps<{ modelValue: string }>()
const emit = defineEmits<{ 'update:modelValue': [value: string] }>()

function onInput(event: Event) {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}
</script>

<template>
  <input :value="props.modelValue" @input="onInput" />
</template>
```

**⚠️ 常見陷阱：直接修改 props**

```vue
<!-- ❌ 錯誤：直接修改 props -->
<input v-model="props.modelValue" />

<!-- ✅ 正確：只讀取 props，透過 emit 更新 -->
<input :value="props.modelValue" @input="e => emit('update:modelValue', (e.target as HTMLInputElement).value)" />
```

直接修改 props 會觸發 Vue 的 runtime warning，破壞單向資料流原則。

#### 3b. 多個 v-model

```vue
<!-- 父元件 -->
<UserCard
  v-model:name="user.name"
  v-model:email="user.email"
/>
```

展開等價於：
```vue
<UserCard
  :name="user.name"
  @update:name="user.name = $event"
  :email="user.email"
  @update:email="user.email = $event"
/>
```

子元件：
```vue
<script setup lang="ts">
const props = defineProps<{
  name: string
  email: string
}>()

const emit = defineEmits<{
  'update:name': [value: string]
  'update:email': [value: string]
}>()
</script>
```

---

### 四、Vue 3.4 `defineModel`（簡化語法）

Vue 3.4 引入 `defineModel` macro，大幅簡化 v-model 的實作：

```vue
<script setup lang="ts">
// 取代 defineProps + defineEmits + emit('update:modelValue')
const modelValue = defineModel<string>({ required: true })

// 多個 v-model
const name = defineModel<string>('name')
const email = defineModel<string>('email')
</script>

<template>
  <!-- 直接用 v-model 綁定，不需手動 emit -->
  <input v-model="modelValue" />
</template>
```

**`defineModel` 回傳的是可寫入的 ref**：
- 讀取：`modelValue.value` → 等同讀取父元件傳進的 prop
- 寫入：`modelValue.value = '新值'` → 等同 `emit('update:modelValue', '新值')`

**選用建議**：
| 情境 | 建議 |
|------|------|
| Vue 3.4+，簡單雙向綁定 | 使用 `defineModel` |
| 需要在 emit 前做驗證或轉換 | 使用 `defineProps + defineEmits`（控制更精細）|
| 需要相容 Vue 3.3 以下 | 使用 `defineProps + defineEmits` |

---

### 五、Props Down / Events Up 原則

```
父元件
  │
  │  props（向下傳資料）
  ▼
子元件
  │
  │  emit（向上傳事件）
  ▼
父元件（處理事件，更新資料）
```

**規則**：
1. 子元件**只能讀取** props，不能修改
2. 子元件需要改變父元件的資料時，透過 `emit` 通知父元件
3. 父元件監聽事件，自己決定是否更新資料

這個原則讓資料流向可預測：**只要看 emit，就知道子元件能影響父元件的什麼**。

---

### 六、常見陷阱整理

| 陷阱 | 錯誤寫法 | 正確寫法 |
|------|---------|---------|
| 直接修改 props | `props.value = 'new'` | `emit('update:modelValue', 'new')` |
| v-model 綁 computed 屬性 | `<input v-model="props.modelValue">` | `:value` + `@input` 分開 |
| emit 名稱錯誤 | `emit('updateModelValue', ...)` | `emit('update:modelValue', ...)` |
| withDefaults 順序錯誤 | `defineProps<Props>(withDefaults(...))` | `withDefaults(defineProps<Props>(), {...})` |
| defineModel 在 Vue 3.3 以下使用 | — | 檢查 Vue 版本後再使用 |

---

## 作業說明

### 情境背景

ihouseBMS 的表單頁面中，有許多輸入欄位（文字輸入、下拉選單、日期選擇）。目前這些欄位直接散落在頁面元件中，沒有封裝。你被要求建立一個可複用的 `CustomInput` 元件，讓所有表單頁面都能用 `v-model` 操作它。

### 功能需求

建立 `CustomInput.vue`，支援：

1. **基本 v-model 雙向綁定**（文字輸入）
2. **Props**：
   - `label`（string，required）：輸入框的標籤文字
   - `modelValue`（string，required）：v-model 綁定值
   - `placeholder`（string，optional，預設：`''`）
   - `disabled`（boolean，optional，預設：`false`）
   - `errorMessage`（string，optional，預設：`''`）
3. **顯示規則**：
   - `label` 顯示在輸入框上方
   - `disabled` 為 true 時，輸入框灰化不可輸入
   - `errorMessage` 不為空時，顯示紅色錯誤文字（在輸入框下方）
4. **Emit**：
   - `update:modelValue`：每次輸入時觸發
   - `blur`（bonus）：輸入框失焦時觸發（payload: FocusEvent）

### 技術規範

- 使用 `<script setup lang="ts">`
- `defineProps` 使用 **TypeScript 型別宣告 + `withDefaults`**
- `defineEmits` 使用 **TypeScript 型別宣告**
- 不直接修改 props，只透過 emit 更新
- 不使用 `v-model` 直接綁定 props（使用 `:value` + `@input`）

### 範例使用（父元件）

```vue
<script setup lang="ts">
import { ref } from 'vue'
import CustomInput from './CustomInput.vue'

const username = ref('')
const errorMsg = ref('')

function validate() {
  errorMsg.value = username.value.length < 2 ? '名稱至少需要 2 個字元' : ''
}
</script>

<template>
  <CustomInput
    v-model="username"
    label="使用者名稱"
    placeholder="請輸入名稱"
    :error-message="errorMsg"
    @blur="validate"
  />
  <p>目前輸入：{{ username }}</p>
</template>
```

### 評分標準（滿分 100 分）

| 項目 | 分數 | 說明 |
|------|------|------|
| v-model 正確實作（不直接修改 props）| 25 | `:value` + `@input` + `emit('update:modelValue')` |
| TypeScript 型別完整（Props + Emits）| 20 | 無 `any`，`withDefaults` 正確使用 |
| Props 行為正確（disabled / errorMessage）| 20 | disabled 時無法輸入；error 時顯示紅字 |
| 元件結構清晰（模板語意、無多餘邏輯）| 15 | 模板讀得懂，Props 與 UI 職責分離 |
| Bonus：`blur` emit 正確實作並型別標注 | 10 | payload 為 `FocusEvent` |
| Bonus：加入 `defineModel` 版本對照 | 10 | 額外建立 `CustomInput.defineModel.vue`，說明差異 |

---

## 自我檢核問題

1. **為什麼不能直接在子元件對 props 做 `v-model` 綁定？**  
   提示：想想 `v-model` 展開後會做什麼，和 Props Down 原則有什麼衝突。

2. **`withDefaults` 解決了什麼問題？Runtime 宣告方式有什麼不同？**  
   提示：想想 TS 型別宣告裡的 `?` 如何影響後續程式碼的型別推斷。

3. **`v-model:title="post.title"` 展開後等價於什麼？**  
   提示：modelValue 對應哪個 prop 名稱？event 名稱是什麼？

4. **`defineModel` 的回傳值是什麼型別？寫入它時發生了什麼？**  
   提示：它不是普通的 ref，而是 Vue 內部代理了 prop 讀取和 emit 的特殊 ref。

5. **一個元件同時有 `v-model:name` 和 `v-model:email`，子元件的 `defineProps` 和 `defineEmits` 應該怎麼寫？**

---

## 明日預告

**2026-05-20（W5 Day 2）：CustomInput 實作期**

明日進入作業實作：
- 建立 `CustomInput.vue` 完整實作（Props → Input → Emit 資料流）
- 驗證 disabled / errorMessage 的 UI 行為
- 嘗試 `defineModel` 版本對照（bonus）
- 開始思考：若要封裝 `CustomSelect`（下拉選單），v-model 的值型別如何設計泛型版？

---

## 附錄：v-model 快速對照表

| 場景 | 父元件寫法 | 子元件 prop | 子元件 emit |
|------|-----------|------------|------------|
| 基本 | `v-model="val"` | `modelValue` | `update:modelValue` |
| 具名 | `v-model:title="val"` | `title` | `update:title` |
| 多個 | `v-model:a="x" v-model:b="y"` | `a`, `b` | `update:a`, `update:b` |
