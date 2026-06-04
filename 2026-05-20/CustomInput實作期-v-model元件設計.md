# CustomInput 實作期 — v-model 元件設計

> **日期**：2026-05-20（週三）
> **週次**：M2-W5 Day 2（實作日）
> **月份主軸**：Composition API 深化期
> **本週交付目標**：雙向綁定自訂元件（CustomInput.vue）

---

## 目標技術

| 技術 | 重點 |
|------|------|
| `defineProps` + `withDefaults` | 將昨日理論落地，實際宣告 5 個 Props 並確認型別安全 |
| `defineEmits` | 宣告 `update:modelValue` 和 `blur`，驗證 payload 型別約束 |
| `:value` + `@input` 模式 | 實踐「不直接修改 props」，封閉正確的 v-model 資料流迴路 |
| `defineModel`（Vue 3.4）| 建立對照版本，比較實作差異與選用條件 |
| 泛型 v-model 預備思考 | 面向 CustomSelect，思考 `T` 型別參數設計 |

---

## 為什麼學這個

### 昨日（5/19）到今日的銜接

昨天系統化建立了 defineProps / defineEmits / v-model 的概念框架：
- 知道 `v-model` 展開的細節
- 知道 `withDefaults` 解決的型別問題
- 知道「不能直接修改 props」的原因

**今天的核心目標是：讓這些知識變成肌肉記憶**。

知識和實作之間有一段距離。你「知道」不能直接修改 props，但當你真正坐下來寫 `CustomInput.vue` 時，會遇到：
- `@input` 的 `event` 型別是什麼？如何 cast 到 `HTMLInputElement`？
- `disabled` 屬性在模板中應該怎麼寫？`disabled` 還是 `:disabled`？
- `errorMessage` 為空字串時要不要渲染 DOM？`v-if` vs `v-show`？

這些細節只有動手才會遇到，只有遇到才能真正理解。

---

## 實作完整指引

### Step 1：建立專案環境

如果你還沒有 Vue 3 + TypeScript 的測試環境，使用 Vite 快速建立：

```bash
npm create vite@latest custom-input-demo -- --template vue-ts
cd custom-input-demo
npm install
npm run dev
```

---

### Step 2：實作 `CustomInput.vue`（完整版，defineProps + defineEmits）

在 `src/components/CustomInput.vue` 建立以下內容：

```vue
<script setup lang="ts">
// =====================================================
// Props 定義（TypeScript 型別宣告 + withDefaults）
// =====================================================
interface Props {
  label: string           // required：輸入框標籤
  modelValue: string      // required：v-model 綁定值
  placeholder?: string    // optional：預設為空字串
  disabled?: boolean      // optional：預設為 false
  errorMessage?: string   // optional：預設為空字串
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: '',
  disabled: false,
  errorMessage: ''
})

// =====================================================
// Emits 定義（TypeScript 型別宣告）
// =====================================================
const emit = defineEmits<{
  'update:modelValue': [value: string]
  blur: [event: FocusEvent]
}>()

// =====================================================
// 事件處理函式
// =====================================================
function onInput(event: Event) {
  // 注意：event.target 是 EventTarget，需要 type cast 才能取到 .value
  const target = event.target as HTMLInputElement
  emit('update:modelValue', target.value)
}

function onBlur(event: FocusEvent) {
  emit('blur', event)
}
</script>

<template>
  <div class="custom-input">
    <!-- 標籤 -->
    <label class="custom-input__label">{{ props.label }}</label>

    <!-- 輸入框：使用 :value + @input，不用 v-model（Props Down 原則）-->
    <input
      class="custom-input__input"
      :class="{ 'custom-input__input--disabled': props.disabled }"
      :value="props.modelValue"
      :placeholder="props.placeholder"
      :disabled="props.disabled"
      @input="onInput"
      @blur="onBlur"
    />

    <!-- 錯誤訊息：只有 errorMessage 非空才顯示 -->
    <p v-if="props.errorMessage" class="custom-input__error">
      {{ props.errorMessage }}
    </p>
  </div>
</template>

<style scoped>
.custom-input {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.custom-input__label {
  font-size: 14px;
  font-weight: 500;
  color: #374151;
}

.custom-input__input {
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  outline: none;
  transition: border-color 0.15s;
}

.custom-input__input:focus {
  border-color: #6366f1;
}

.custom-input__input--disabled {
  background-color: #f3f4f6;
  color: #9ca3af;
  cursor: not-allowed;
}

.custom-input__error {
  font-size: 12px;
  color: #ef4444;
  margin: 0;
}
</style>
```

---

### Step 3：在父元件使用（驗證 v-model 正確性）

在 `src/App.vue` 建立測試場景：

```vue
<script setup lang="ts">
import { ref } from 'vue'
import CustomInput from './components/CustomInput.vue'

const username = ref('')
const errorMsg = ref('')

function validate() {
  errorMsg.value = username.value.length < 2 ? '名稱至少需要 2 個字元' : ''
}

// 額外測試：disabled 狀態
const isDisabled = ref(false)
</script>

<template>
  <div style="max-width: 400px; margin: 40px auto; padding: 20px;">
    <h2>CustomInput 測試</h2>

    <!-- 基本 v-model 測試 -->
    <CustomInput
      v-model="username"
      label="使用者名稱"
      placeholder="請輸入名稱"
      :error-message="errorMsg"
      @blur="validate"
    />
    <p>目前輸入：「{{ username }}」</p>

    <hr style="margin: 20px 0;" />

    <!-- disabled 狀態測試 -->
    <button @click="isDisabled = !isDisabled" style="margin-bottom: 12px;">
      切換 disabled（目前：{{ isDisabled ? '停用' : '啟用' }}）
    </button>
    <CustomInput
      v-model="username"
      label="測試 disabled"
      :disabled="isDisabled"
    />
  </div>
</template>
```

---

### Step 4：功能驗收腳本（8 步驟）

| # | 操作 | 確認目標 |
|---|------|---------|
| 1 | 頁面載入 | 「使用者名稱」標籤顯示；輸入框為空；無錯誤訊息 |
| 2 | 輸入一個字元 | 輸入框顯示字元；「目前輸入」即時更新；v-model 雙向同步確認 |
| 3 | 點擊其他區域（觸發 blur）| 若輸入 < 2 字元，「名稱至少需要 2 個字元」紅字出現 |
| 4 | 繼續輸入到 ≥ 2 字元後 blur | 錯誤訊息消失（v-if 條件改變，DOM 節點移除）|
| 5 | 點選「切換 disabled」按鈕 | 下方輸入框灰化，游標變為禁止符號 |
| 6 | 嘗試在 disabled 輸入框打字 | 無法輸入，`input` 事件不觸發 |
| 7 | 再點「切換 disabled」恢復啟用 | 輸入框回到可輸入狀態 |
| 8 | 打開 Vue DevTools → 找到 CustomInput 元件 | 觀察 Props（modelValue、disabled、errorMessage）與 Emits 正確觸發 |

---

### Step 5（Bonus）：`defineModel` 對照版本

建立 `src/components/CustomInput.defineModel.vue`：

```vue
<script setup lang="ts">
// defineModel 版本（Vue 3.4+）
// 取代 defineProps + defineEmits + 手動 emit

const modelValue = defineModel<string>({ required: true })

// 其他 props 仍需 defineProps
interface RestProps {
  label: string
  placeholder?: string
  disabled?: boolean
  errorMessage?: string
}

const props = withDefaults(defineProps<RestProps>(), {
  placeholder: '',
  disabled: false,
  errorMessage: ''
})

const emit = defineEmits<{
  blur: [event: FocusEvent]
}>()

function onBlur(event: FocusEvent) {
  emit('blur', event)
}
</script>

<template>
  <div class="custom-input">
    <label class="custom-input__label">{{ props.label }}</label>

    <!-- defineModel 版本：可以直接 v-model，因為 modelValue 是可寫入 ref -->
    <input
      class="custom-input__input"
      v-model="modelValue"
      :placeholder="props.placeholder"
      :disabled="props.disabled"
      @blur="onBlur"
    />

    <p v-if="props.errorMessage" class="custom-input__error">
      {{ props.errorMessage }}
    </p>
  </div>
</template>
```

#### 兩個版本的對照表

| 面向 | defineProps + defineEmits 版 | defineModel 版 |
|------|------------------------------|---------------|
| 程式碼量 | 較多（需手動宣告 emit 和 onInput）| 較少（modelValue 直接可寫）|
| 靈活性 | 高（可在 emit 前做驗證、轉換）| 中（寫入立即 emit，攔截較難）|
| 可讀性 | 意圖清晰（看 onInput 知道流程）| 更簡潔（但隱藏了 emit 邏輯）|
| Vue 版本 | 3.0+ 均可 | 需 Vue 3.4+ |
| 推薦場景 | 複雜驗證、emit 前需攔截 | 簡單雙向綁定、Vue 3.4+ 環境 |

---

## 進階思考：泛型 v-model（CustomSelect 預備）

目前 `CustomInput` 的 `modelValue` 型別是固定的 `string`。但如果要建立 `CustomSelect`（下拉選單），選項的值可能是 `string`、`number`、甚至 `{ id: number; name: string }` 這樣的物件。

### 問題：如何讓 v-model 的值型別是「任意的 T」？

#### 方向一：用 TypeScript 泛型定義 Props

```ts
// 概念程式碼（目前 Vue 3 的 defineProps 支援有限）
interface Props<T> {
  modelValue: T
  options: Array<{ label: string; value: T }>
}
```

#### 方向二：用 `defineModel<T>` 搭配泛型（更實用）

```vue
<script setup lang="ts" generic="T">
const modelValue = defineModel<T>({ required: true })
defineProps<{
  options: Array<{ label: string; value: T }>
}>()
</script>
```

> Vue 3.3+ 的 `generic="T"` 語法讓 `<script setup>` 支援泛型型別參數。

### 今日思考題

- 當 `CustomSelect` 的選項值是物件時，`v-model` 比較的是「參考相等」還是「值相等」？會造成什麼問題？
- 如何為 `CustomSelect` 的每個選項設計 `key`，確保 Vue 能正確追蹤 DOM？

> 這個問題留到 W6 的 `provide/inject` × Slots 章節前，嘗試自行解答。

---

## 自我檢核問題

1. **實作 `onInput` 時，為什麼要 `event.target as HTMLInputElement`？直接用 `event.target.value` 為什麼不行？**  
   提示：TypeScript 知道 `EventTarget` 是什麼型別，但它不知道具體的 DOM 元素類型。

2. **`v-if="props.errorMessage"` 和 `v-show="props.errorMessage"` 哪個更適合？為什麼？**  
   提示：錯誤訊息的 DOM 節點有必要在 `errorMessage` 為空時保留在頁面上嗎？

3. **`defineModel` 版本的 `v-model="modelValue"` 為什麼不違反「不直接修改 props」原則？**  
   提示：`defineModel` 回傳的「可寫入 ref」在寫入時實際上做了什麼？

4. **如果父元件需要在 `CustomInput` 的輸入值被更新前做驗證（例如只允許數字），應該選哪個版本實作？為什麼？**

5. **`blur` 事件的 payload 是 `FocusEvent`，而不是 `Event`。為什麼要用更精確的型別？**  
   提示：想想 FocusEvent 有哪些 `Event` 沒有的屬性（如 `relatedTarget`）。

---

## 明日預告

**2026-05-21（W5 Day 3）：CustomInput TypeScript 精進 × 表單整合設計**

實作完成後，進入「能 review」的距離：
- TypeScript 型別審查：補完 `onInput` 的 type narrowing × 確認 `FocusEvent` 型別鏈
- 表單整合思考：多個 `CustomInput` 組合成一個表單時，狀態管理如何設計？
- 進階思考：加入 `inputmode`（數字鍵盤）、`autocomplete`、`name` 等 HTML 屬性 pass-through
- 開始思考 `CustomSelect` 的泛型設計（為 W5 作業最終版本準備型別安全的選項系統）

---

## 附錄：今日知識地圖（實作視角）

```
CustomInput.vue 實作流程
│
├── Props 設計（defineProps + withDefaults）
│   ├── label（required）────────────────── 渲染 <label>
│   ├── modelValue（required）──────────── :value 綁定
│   ├── placeholder（optional, 預設 ''）── :placeholder
│   ├── disabled（optional, 預設 false）── :disabled + CSS class
│   └── errorMessage（optional, 預設 ''）─ v-if 顯示錯誤文字
│
├── Emits 設計（defineEmits with TS 型別）
│   ├── 'update:modelValue': [value: string] ← @input → onInput → emit
│   └── 'blur': [event: FocusEvent]          ← @blur  → onBlur  → emit
│
└── 資料流方向
    父元件 v-model="val"
      ↓（:modelValue 傳入）
    CustomInput（只讀 props.modelValue）
      ↓（使用者輸入 → @input → onInput）
    emit('update:modelValue', 新值)
      ↓（父元件收到 @update:modelValue）
    父元件 val = $event（更新資料）
      ↓（Vue 重新渲染）
    CustomInput（接收新的 props.modelValue）
```
