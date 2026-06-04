# CustomInput TypeScript 精進 × Attrs Pass-through × 表單整合設計

> **日期**：2026-05-21（W5 Day 3）
> **週次主題**：defineProps / defineEmits / v-model 進階 → 雙向綁定自訂元件
> **月份**：M2（Composition API 深化期）

---

## 目標技術

| 技術 | 子主題 |
|------|--------|
| Vue.js 3 | `$attrs` / `useAttrs()` / `v-bind="$attrs"` — Attrs Pass-through |
| TypeScript | Event 型別鏈、FocusEvent 精確型別、`onInput` type narrowing 完整化 |
| Vue.js 3 + TypeScript | 表單整合狀態設計、`reactive` 表單物件 vs 多個 `ref` |
| Vue.js 3（預備）| `CustomSelect` 泛型設計起點（`generic="T"` 搭配 `defineModel<T>()`）|

---

## 為什麼學這個

昨天（Day 2）完成了 `CustomInput.vue` 的核心實作，並建立了 `defineModel` 的對照版本，並預備了 `generic="T"` 的觀念。但距離「可以 Review」的元件，還有幾個關鍵距離：

1. **TypeScript 型別不夠精確**：`onInput` 中使用了 `as HTMLInputElement`，但 `FocusEvent`（`onBlur`）的型別鏈是否同樣精確？型別設計還可以更系統化。
2. **Attrs 沒有 pass-through**：`CustomInput` 是封裝過的 `<input>`，但 `inputmode`、`autocomplete`、`name`、`maxlength` 這些 HTML 原生屬性該怎麼讓父元件傳入？目前的 `defineProps` 無法涵蓋所有可能的屬性。
3. **多個 CustomInput 組合成表單**：一個表單通常有多個欄位，狀態怎麼設計才不會讓 `<script setup>` 充斥大量 `ref`？

這三個問題，今天一起解決。

---

## 知識說明

### 一、Event 型別鏈：從 `EventTarget` 到具體 DOM 元素

#### 問題根源

```ts
// 這樣寫 TypeScript 會報錯
const onInput = (event: Event) => {
  emit('update:modelValue', event.target.value) // ❌ 'value' does not exist on type 'EventTarget'
}
```

`EventTarget` 是所有可以發出事件的 DOM 元素的基底型別（包含 `Window`、`Document`、`XMLHttpRequest` 等），它沒有 `.value` 屬性。

#### 解法一：`as HTMLInputElement`（型別斷言）

```ts
const onInput = (event: Event) => {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}
```

**適用時機**：你確定這個事件一定是從 `<input>` 發出的（因為是你自己綁定的）。

**本質**：TypeScript 的「承諾」，告訴編譯器「相信我，這一定是 HTMLInputElement」。執行期若類型錯誤，不會報錯但會 undefined。

#### 解法二：`InputEvent` 精確型別

```ts
const onInput = (event: InputEvent) => {
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}
```

`InputEvent` 繼承自 `UIEvent` 繼承自 `Event`，比 `Event` 更精確，但 `.target` 仍是 `EventTarget`，所以仍需斷言。

#### FocusEvent 型別鏈

```ts
// onBlur 對應的事件型別是 FocusEvent
const onBlur = (event: FocusEvent) => {
  // FocusEvent 繼承自 UIEvent → Event → EventTarget
  // 同樣需要 as HTMLInputElement 才能存取 .value
  const target = event.target as HTMLInputElement
  console.log('blur, value:', target.value)
  emit('blur', target.value)
}
```

**型別繼承樹**：
```
EventTarget
  └── Event
        ├── UIEvent
        │     ├── InputEvent    → input 事件
        │     ├── FocusEvent    → focus / blur 事件
        │     ├── MouseEvent    → click / mousedown 事件
        │     └── KeyboardEvent → keydown / keyup 事件
        └── ...
```

**關鍵認知**：不管哪種 DOM 事件，`.target` 都是 `EventTarget`，TypeScript 無法自動縮窄到具體元素型別。我們需要**型別斷言（as）**或**型別守衛（instanceof）**。

#### 型別守衛方式（更安全，執行期驗證）

```ts
const onInput = (event: Event) => {
  if (!(event.target instanceof HTMLInputElement)) return
  // 此後 TypeScript 自動縮窄為 HTMLInputElement
  emit('update:modelValue', event.target.value)
}
```

**差異**：`instanceof` 是**執行期守衛**，`as` 是**編譯期斷言**。在自訂元件的受控 `<input>` 中，`as` 已足夠安全，因為綁定對象確定。

---

### 二、`$attrs` / `useAttrs()` — Attrs Pass-through

#### 問題：未宣告的 Props 去哪裡了？

```html
<!-- 父元件 -->
<CustomInput
  v-model="username"
  label="帳號"
  inputmode="email"
  autocomplete="username"
  maxlength="50"
  name="username"
/>
```

`inputmode`、`autocomplete`、`maxlength`、`name` 不在 `defineProps` 中，Vue 會把它們放進 **`$attrs`（Fallthrough Attributes）**。

**預設行為**：Vue 會把 `$attrs` 自動套用到元件的**根元素**。

```html
<!-- CustomInput.vue 的 template -->
<div class="custom-input">           <!-- ← $attrs 預設套到這裡！不是 <input> -->
  <label>{{ label }}</label>
  <input :value="modelValue" ... />  <!-- ← 我們想要的目標 -->
</div>
```

這就是問題所在：`inputmode` 最終套到了 `<div>` 而非 `<input>`。

#### 解法：禁用自動繼承 + 手動套用 `$attrs`

```html
<script setup lang="ts">
// ...
// 禁用自動 fallthrough
defineOptions({ inheritAttrs: false })

// 取得 $attrs（可選，有時直接在 template 用 $attrs 就夠了）
import { useAttrs } from 'vue'
const attrs = useAttrs()
</script>

<template>
  <div class="custom-input">
    <label :for="inputId">{{ label }}</label>
    <input
      :id="inputId"
      :value="modelValue"
      v-bind="$attrs"        <!-- ✅ 手動套到正確位置 -->
      @input="onInput"
      @blur="onBlur"
    />
    <span v-if="errorMessage" class="error">{{ errorMessage }}</span>
  </div>
</template>
```

#### `v-bind="$attrs"` 的展開內容

```ts
// 父元件傳入：inputmode="email" autocomplete="username" maxlength="50"
// $attrs 的值會是：
{
  inputmode: 'email',
  autocomplete: 'username',
  maxlength: '50',
  name: 'username'
  // 注意：class 和 style 也在 $attrs 中！
}
```

**注意事項**：
- `defineProps` 中宣告的屬性**不會**進入 `$attrs`
- `defineEmits` 中宣告的事件**不會**進入 `$attrs`（它們也不會被 fallthrough）
- `class` 和 `style` 預設也在 `$attrs` 中，若不想合併可用 `inheritAttrs: false` + 手動控制

#### 常見陷阱：`class` 雙重套用

```html
<!-- 父元件 -->
<CustomInput class="w-full" ... />

<!-- $attrs 包含 class="w-full"
     若不設 inheritAttrs: false，class 會套到 <div> -->
<!-- 設了 inheritAttrs: false 後，class 只套到 v-bind="$attrs" 的位置（<input>）-->
```

設計時需要思考：你希望父元件的 `class` 套到**外層包裝 div** 還是**內層 input**？這是一個 API 設計決策。

---

### 三、表單整合狀態設計

#### 場景：多個 CustomInput 組成的登入表單

```html
<LoginForm>
  <CustomInput v-model="form.username" label="帳號" />
  <CustomInput v-model="form.password" label="密碼" type="password" />
  <button @click="handleSubmit">登入</button>
</LoginForm>
```

#### 方案一：多個獨立 `ref`（⚠️ 不推薦，欄位多時難維護）

```ts
const username = ref('')
const password = ref('')
const usernameError = ref('')
const passwordError = ref('')
// 欄位一多，這裡就成了 ref 海...
```

#### 方案二：`reactive` 表單物件（✅ 推薦）

```ts
// useLoginForm.ts（Composable 封裝）
import { reactive } from 'vue'

interface LoginForm {
  username: string
  password: string
}

interface FormErrors {
  username: string
  password: string
}

export function useLoginForm() {
  const form = reactive<LoginForm>({
    username: '',
    password: '',
  })

  const errors = reactive<FormErrors>({
    username: '',
    password: '',
  })

  function validate(): boolean {
    errors.username = form.username ? '' : '帳號不能為空'
    errors.password = form.password.length >= 6 ? '' : '密碼至少 6 個字元'
    return !errors.username && !errors.password
  }

  function reset() {
    form.username = ''
    form.password = ''
    errors.username = ''
    errors.password = ''
  }

  return { form, errors, validate, reset }
}
```

```html
<!-- LoginForm.vue -->
<script setup lang="ts">
import { useLoginForm } from './useLoginForm'
const { form, errors, validate, reset } = useLoginForm()

async function handleSubmit() {
  if (!validate()) return
  // 呼叫 API...
}
</script>

<template>
  <CustomInput
    v-model="form.username"
    label="帳號"
    :error-message="errors.username"
  />
  <CustomInput
    v-model="form.password"
    label="密碼"
    type="password"
    :error-message="errors.password"
  />
  <button @click="handleSubmit">登入</button>
</template>
```

**為什麼 `reactive` 適合表單物件？**
- 表單欄位結構固定，不需整批替換（不需 `ref` 的整批替換語義）
- 鍵名對應欄位名，可用 `keyof LoginForm` 做型別安全的迴圈驗證
- `v-model="form.username"` 比 `v-model="username"` 更清楚表達「這是一個表單的一部分」

---

### 四、CustomSelect 泛型設計預備

W5 的最終作業是「雙向綁定自訂元件」，而 `CustomInput` 只是其中一個。真正展現設計能力的是 `CustomSelect`——它需要泛型來確保選項值的型別安全。

```html
<script setup lang="ts" generic="T extends string | number">
// generic="T extends string | number" 限制泛型邊界
// T 只能是字串或數字型別

interface Option<T> {
  label: string
  value: T
}

const props = defineProps<{
  options: Option<T>[]
  modelValue: T | null
  label: string
  placeholder?: string
}>()

const emit = defineEmits<{
  'update:modelValue': [value: T]
}>()
</script>

<template>
  <div class="custom-select">
    <label>{{ label }}</label>
    <select
      :value="modelValue ?? ''"
      @change="emit('update:modelValue', ($event.target as HTMLSelectElement).value as T)"
    >
      <option v-if="placeholder" value="" disabled>{{ placeholder }}</option>
      <option
        v-for="option in options"
        :key="String(option.value)"
        :value="option.value"
      >
        {{ option.label }}
      </option>
    </select>
  </div>
</template>
```

**使用示範**：

```html
<!-- 字串型別的選項 -->
<CustomSelect
  v-model="form.city"
  :options="[
    { label: '台北市', value: 'taipei' },
    { label: '新北市', value: 'new-taipei' },
  ]"
  label="城市"
/>

<!-- 數字型別的選項（TypeScript 確保 v-model 也是 number）-->
<CustomSelect
  v-model="form.categoryId"
  :options="[
    { label: '住宅', value: 1 },
    { label: '商業', value: 2 },
  ]"
  label="類別"
/>
```

---

## 作業說明

### 作業背景

W5 作業目標是「雙向綁定自訂元件」，最終要交付一個可運作的表單系統。今天的任務是讓 `CustomInput.vue` 達到「可 Review」的品質，並為 `CustomSelect.vue` 建立骨架。

### 功能需求

**Task A：CustomInput.vue 型別精進**

1. 將 `@input` 的 handler 改為 `(event: InputEvent)` 型別，確保型別鏈清晰
2. 加入 `@blur` handler（若還沒有），使用 `FocusEvent` 型別
3. 在 `@blur` 中觸發欄位驗證（若 `errorMessage` 由父元件傳入，則 `emit('blur')` 讓父元件決定驗證時機）

**Task B：CustomInput.vue 加入 Attrs Pass-through**

1. 加入 `defineOptions({ inheritAttrs: false })`
2. 將 `v-bind="$attrs"` 套到 `<input>` 元素上
3. 驗證：在父元件傳入 `inputmode="tel"` 和 `maxlength="10"`，確認它們套到 `<input>` 而不是外層 `<div>`（用瀏覽器 DevTools Elements 面板驗證）

**Task C：useLoginForm Composable**

設計一個 `useLoginForm.ts`，包含：
- `form: reactive<{ username: string; password: string }>`
- `errors: reactive<{ username: string; password: string }>`
- `validate()` 函式（空值驗證 + 密碼長度驗證）
- `reset()` 函式

**Task D（選做）：CustomSelect 泛型骨架**

根據知識說明的設計，建立 `CustomSelect.vue`（可以先不實作 UI，先確保 TypeScript 型別正確）

### 技術規範

- TypeScript 嚴格型別（不使用 `any`）
- `defineOptions({ inheritAttrs: false })` 配合 `v-bind="$attrs"`
- `useLoginForm` 使用 `reactive`（不用多個 `ref`）
- `vue-tsc --noEmit` 零警告

### 評分標準（滿分 100 分）

| 項目 | 分數 |
|------|------|
| Task A：型別精進（InputEvent / FocusEvent）| 25 分 |
| Task B：Attrs Pass-through 正確套用位置 | 30 分 |
| Task C：useLoginForm Composable 設計完整 | 30 分 |
| Task D：CustomSelect 泛型骨架（選做）| 15 分 |

---

## 自我檢核問題

1. **`as HTMLInputElement` 和 `instanceof HTMLInputElement` 各適合什麼場景？兩者的本質差異是什麼？**

2. **`inheritAttrs: false` 不設的話，如果父元件傳 `maxlength="10"`，它會套到哪個元素？設了 `inheritAttrs: false` 但沒有 `v-bind="$attrs"` 的話，`maxlength` 又去哪裡了？**

3. **為什麼表單狀態選用 `reactive` 而不是多個 `ref`？在什麼場景下你會改變這個選擇，改用 `ref`？**

4. **`useLoginForm` 回傳 `form` 和 `errors` 時，需要 `toRefs` 嗎？為什麼？**

5. **`CustomSelect` 使用 `generic="T extends string | number"` 的限制是什麼？如果今天選項值是一個物件（例如 `{ id: number, name: string }`），設計應該如何調整？**

---

## 明日預告

**2026-05-22（W5 Day 4）：useLoginForm 整合測試 × CustomSelect 泛型完整實作 × W5 作業整體組裝**

- `LoginForm.vue` 整合 `CustomInput` + `CustomSelect` + `useLoginForm`
- 表單驗證：觸發時機設計（即時 vs 送出時）
- `CustomSelect` 完整實作（含 Attrs Pass-through + 泛型值正確傳遞）
- W5 作業整體審視：功能驗收 + TypeScript 審查 + Commit 整理
