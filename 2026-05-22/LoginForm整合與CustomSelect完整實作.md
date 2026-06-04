# LoginForm.vue 整合 × CustomSelect 完整實作

> **日期**：2026-05-22（M2-W5 Day 4）
> **所屬週次**：W5（5/19–5/25）
> **截止日**：2026-05-25（剩 3 天）

---

## 一、目標技術與核心知識點

| 技術 | 核心知識點 |
|------|-----------|
| Vue 3 Composition API | `defineProps`、`defineEmits`、`defineModel`、`generic="T"` 實戰整合 |
| TypeScript 泛型 | `generic="T extends string \| number"`、型別參數傳遞鏈 |
| 元件通訊 | Props Down / Events Up 在多層元件中的完整體現 |
| Composable 設計 | `useLoginForm` 實際接入 `LoginForm.vue` |
| 表單驗證 | 即時驗證（@input/@blur）vs 提交時驗證選用決策 |

---

## 二、為什麼學這個（與前幾日的連結）

```
W5 知識積累脈絡

Day 1（5/19）理論建立
  └── defineProps TS 三層架構 × defineEmits tuple 型別 × v-model 本質
Day 2（5/20）實作落地
  └── CustomInput.vue 完整實作 × event.target as HTMLInputElement × defineModel
Day 3（5/21）精進層
  └── Event 型別鏈 × $attrs Fallthrough × useLoginForm Composable × CustomSelect 骨架
Day 4（今日）整合層  ← 你在這裡
  └── LoginForm.vue 組合 CustomInput + CustomSelect × 表單提交流程 × 泛型元件完整實作
```

**為什麼今天是關鍵**：前三天分別建立了「零件」（CustomInput、CustomSelect 骨架、useLoginForm）。今天是**組裝**——把零件接起來形成一個完整可運作的表單，這是 W5 作業的核心交付物。

---

## 三、知識說明

### 3-1｜CustomSelect 完整實作

昨天完成了 CustomSelect 骨架，今天把它填滿：

```vue
<!-- CustomSelect.vue -->
<script setup lang="ts" generic="T extends string | number">
interface Option<T> {
  label: string
  value: T
}

interface Props {
  options: Option<T>[]
  modelValue: T | null
  placeholder?: string
  disabled?: boolean
  errorMessage?: string
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: '請選擇',
  disabled: false,
  errorMessage: '',
})

const emit = defineEmits<{
  'update:modelValue': [value: T | null]
  blur: [event: FocusEvent]
}>()

defineOptions({ inheritAttrs: false })

function handleChange(event: Event) {
  const select = event.target as HTMLSelectElement
  // 轉換：HTML select 的 .value 永遠是 string，需要比對 options 還原正確型別
  const matched = props.options.find(
    (opt) => String(opt.value) === select.value
  )
  emit('update:modelValue', matched?.value ?? null)
}

function handleBlur(event: FocusEvent) {
  emit('blur', event)
}
</script>

<template>
  <div class="custom-select-wrapper">
    <select
      v-bind="$attrs"
      :value="modelValue !== null ? String(modelValue) : ''"
      :disabled="disabled"
      @change="handleChange"
      @blur="handleBlur"
    >
      <option value="" disabled>{{ placeholder }}</option>
      <option
        v-for="opt in options"
        :key="String(opt.value)"
        :value="String(opt.value)"
      >
        {{ opt.label }}
      </option>
    </select>
    <p v-if="errorMessage" class="error-message">{{ errorMessage }}</p>
  </div>
</template>
```

**關鍵陷阱**：`<select>` 的 `.value` 永遠是 `string`，即使選項本來是 `number`。因此：
- 綁定時：`String(opt.value)` 確保 HTML value 是字串
- 讀取時：透過 `options.find` 比對還原原始型別，而非直接使用 `select.value`

---

### 3-2｜LoginForm.vue 完整組裝

```vue
<!-- LoginForm.vue -->
<script setup lang="ts">
import CustomInput from './CustomInput.vue'
import CustomSelect from './CustomSelect.vue'
import { useLoginForm } from './useLoginForm'

type RoleOption = { label: string; value: string }

const roleOptions: RoleOption[] = [
  { label: '管理員', value: 'admin' },
  { label: '一般用戶', value: 'user' },
  { label: '訪客', value: 'guest' },
]

const { form, errors, validate, reset } = useLoginForm()

function handleSubmit() {
  if (!validate()) return  // 驗證失敗，停止提交
  // 提交邏輯（API 呼叫等）
  console.log('提交成功：', { ...form })
  reset()
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <CustomInput
      v-model="form.username"
      label="帳號"
      :error-message="errors.username"
      @blur="validate()"
    />

    <CustomInput
      v-model="form.password"
      label="密碼"
      type="password"
      :error-message="errors.password"
      @blur="validate()"
    />

    <CustomSelect
      v-model="form.role"
      :options="roleOptions"
      placeholder="請選擇角色"
      :error-message="errors.role"
    />

    <button type="submit">登入</button>
    <button type="button" @click="reset">重設</button>
  </form>
</template>
```

---

### 3-3｜useLoginForm 完整版（包含 role 欄位）

```ts
// useLoginForm.ts
import { reactive } from 'vue'

interface LoginForm {
  username: string
  password: string
  role: string | null
}

interface LoginErrors {
  username: string
  password: string
  role: string
}

export function useLoginForm() {
  const form = reactive<LoginForm>({
    username: '',
    password: '',
    role: null,
  })

  const errors = reactive<LoginErrors>({
    username: '',
    password: '',
    role: '',
  })

  function validate(): boolean {
    let isValid = true

    // 清除舊錯誤
    ;(Object.keys(errors) as Array<keyof LoginErrors>).forEach((key) => {
      errors[key] = ''
    })

    if (!form.username.trim()) {
      errors.username = '帳號不得為空'
      isValid = false
    }
    if (form.password.length < 6) {
      errors.password = '密碼至少 6 個字元'
      isValid = false
    }
    if (!form.role) {
      errors.role = '請選擇角色'
      isValid = false
    }

    return isValid
  }

  function reset() {
    form.username = ''
    form.password = ''
    form.role = null
    ;(Object.keys(errors) as Array<keyof LoginErrors>).forEach((key) => {
      errors[key] = ''
    })
  }

  return { form, errors, validate, reset }
}
```

---

### 3-4｜即時驗證 vs 提交時驗證的選用決策

| 策略 | 觸發時機 | 優點 | 缺點 | 適用場景 |
|------|---------|------|------|---------|
| 提交時驗證 | 按下 Submit | 用戶填寫中不被打擾 | 錯誤集中在最後出現 | 短表單、簡單驗證 |
| @blur 驗證 | 離開欄位時 | 即時回饋，不打擾輸入中 | 需處理「首次進入就離開」的邊緣 case | 長表單、複雜驗證 |
| @input 驗證 | 每次輸入時 | 即時回饋 | 錯誤在輸入過程中不斷出現（干擾） | 密碼強度等特殊場景 |

**W5 作業採用策略**：@blur 觸發驗證 + 提交再驗證一次（防止繞過）

---

### 3-5｜常見陷阱

**陷阱一：v-model 綁 `null` 的處理**

```ts
// ❌ 危險：直接把 null 傳給 <select>
// <select :value="modelValue">  ← null 轉成字串 "null"

// ✅ 正確：顯式處理 null
:value="modelValue !== null ? String(modelValue) : ''"
```

**陷阱二：reactive 物件的重設**

```ts
// ❌ 整批替換（破壞響應性）
form = { username: '', password: '', role: null }  // 這行根本不能執行，form 是 const

// ✅ 逐項歸零（保留響應性）
form.username = ''
form.password = ''
form.role = null
```

**陷阱三：v-model 與 CustomSelect 的型別對齊**

```ts
// ❌ 型別不一致：form.role 是 string | null，但 CustomSelect 的 T 是 number
const form = reactive({ role: 0 })  // 0 是 number
// <CustomSelect v-model="form.role" ...> ← T 推導為 number，但 HTML .value 是 string

// ✅ 確保 form.role 型別與 options.value 型別一致
const form = reactive({ role: null as string | null })  // string | null 對齊 roleOptions.value: string
```

---

## 四、作業說明

### 情境背景

W5 作業的主交付物：建立一個完整的 **雙向綁定自訂元件** 展示頁面，包含：
1. `CustomInput.vue` — 文字輸入（已於 Day 2 完成主體）
2. `CustomSelect.vue` — 下拉選單（今日完成）
3. `LoginForm.vue` — 整合上述元件的完整表單

### 功能需求

| # | 功能 | 說明 |
|---|------|------|
| 1 | `CustomInput` v-model 雙向綁定 | 父元件 data 即時同步 |
| 2 | `CustomInput` 錯誤訊息顯示 | 透過 `:error-message` prop 傳入 |
| 3 | `CustomInput` $attrs Fallthrough | `placeholder`、`maxlength` 等屬性正確套到 `<input>` |
| 4 | `CustomSelect` v-model 雙向綁定（泛型）| 支援 `string` 及 `number` 型別的選項值 |
| 5 | `CustomSelect` 錯誤訊息顯示 | 同 CustomInput |
| 6 | `LoginForm` 表單提交驗證 | 空帳號、密碼長度不足、未選角色皆有錯誤提示 |
| 7 | `LoginForm` 重設功能 | 點重設後所有欄位與錯誤訊息清空 |
| 8 | TypeScript 型別安全 | `vue-tsc` 零警告、無 `any` |

### 技術規範

- `CustomSelect` 使用 `generic="T extends string | number"` 實作泛型
- `useLoginForm` 使用 `reactive` 管理表單狀態與錯誤
- `@submit.prevent` 防止表單預設提交行為
- 驗證邏輯集中於 `useLoginForm.validate()`，不散落在模板

### 評分標準（100 分）

| 項目 | 配分 | 說明 |
|------|------|------|
| CustomInput 功能完整 | 20 | v-model、errorMessage、$attrs Fallthrough |
| CustomSelect 泛型實作 | 25 | generic="T"、型別安全、null 處理 |
| LoginForm 整合正確 | 20 | 三個欄位正確接線、資料流無誤 |
| 表單驗證邏輯 | 20 | validate/reset 行為正確 |
| TypeScript 品質 | 15 | vue-tsc 零警告、無 any |

---

## 五、自我檢核問題

1. `CustomSelect` 中為什麼不能直接用 `select.value` 作為 `emit` 的值，而要透過 `options.find` 還原？

2. `useLoginForm` 的 `reset()` 方法直接重新賦值 `form.username = ''`，為什麼不用 `Object.assign(form, initialState)` 也可以？何時兩者有差異？

3. 在 `LoginForm.vue` 中，`@blur="validate()"` 在 `CustomInput` 上能觸發是因為什麼機制？（提示：跟 Day 3 學的 `$attrs Fallthrough` 有關嗎？還是 `defineEmits` 中明確宣告了 `blur`？）

4. 如果表單有 10 個欄位，驗證邏輯全寫在 `useLoginForm` 裡，和分散在各個元件的 `@blur` 處理函式裡，哪個設計更好？為什麼？

5. 泛型元件 `CustomSelect<T>` 在父元件使用時，TypeScript 如何推導 `T` 的型別？

---

## 六、明日預告

**5/23（W5 Day 5）：功能驗收腳本 × TypeScript 全審查**

- 建立 8–12 步驟的系統化功能驗收腳本（對應 W5 所有需求）
- 執行 `vue-tsc --noEmit` 零警告確認
- 補強 `CustomInput` 與 `CustomSelect` 的邊緣 case（空值、極端輸入）
- 模擬 Code Review：用「為什麼這樣設計」的角度自審程式碼
- W5 PR 說明草稿（截止 5/25，還剩 2 天）
