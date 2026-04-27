## 目標技術

`JavaScript ES6+` `TypeScript`

## 核心知識

箭頭函式（Arrow Function）與模板字串（Template Literal）

## 知識說明

### 為什麼今天學這個？（與前幾日的連結）

W1 前兩天我們學習了：

- **Day 1（4/23）**：Vue 3 響應式原理，認識 `ref` / `reactive` / `computed` / `watch`，以及 Proxy 攔截機制
- **Day 2（4/24）**：解構賦值、Spread 展開、Rest 其餘運算子——並因此真正理解了「解構 `reactive` 為何失去響應性」

今天是 W1 的最後一天，補完兩個同樣高頻的 ES6+ 語法：

> **箭頭函式**：改變了 `this` 的綁定規則，是 Vue 3 Composition API 中最常用的函式形式。
> **Template Literal**：讓字串拼接、多行字串、標籤字串都有了更優雅的表達方式。

這兩個概念看起來簡單，但箭頭函式的 `this` 陷阱在 Vue 開發中非常容易踩到，值得深入理解。

---

### 一、箭頭函式（Arrow Function）

**定義**：ES6 引入的函式縮寫語法，同時帶來 `this` 綁定語意的根本改變。

---

#### 1. 基本語法

```ts
// 傳統函式
function add(a: number, b: number): number {
  return a + b
}

// 箭頭函式（完整形式）
const add = (a: number, b: number): number => {
  return a + b
}

// 箭頭函式（單一運算式可省略 return 與大括號）
const add = (a: number, b: number): number => a + b

// 單一參數可省略括號（TypeScript 通常仍建議保留以標注型別）
const double = (n: number): number => n * 2

// 無參數
const greet = (): string => 'Hello!'

// 回傳物件時需用括號包裹（避免與函式主體大括號混淆）
const toUser = (name: string) => ({ name, createdAt: new Date() })
```

---

#### 2. `this` 綁定的根本差異

這是箭頭函式最重要的特性，也是最容易踩坑的地方。

**傳統函式：`this` 由呼叫方式決定（動態綁定）**

```ts
const obj = {
  name: 'Alice',
  greet: function() {
    console.log(this.name) // 'Alice'（obj 呼叫）
  },
  greetLater: function() {
    setTimeout(function() {
      console.log(this.name) // undefined 或報錯
      // setTimeout 的 callback 由全域（或 undefined in strict mode）呼叫
      // this 不是 obj
    }, 1000)
  }
}
```

**箭頭函式：`this` 繼承自外層詞法作用域（靜態綁定）**

```ts
const obj = {
  name: 'Alice',
  greetLater: function() {
    setTimeout(() => {
      console.log(this.name) // 'Alice'
      // 箭頭函式沒有自己的 this，繼承外層函式的 this（obj）
    }, 1000)
  }
}
```

---

#### 3. `this` 完整規則對照

| 情境 | 傳統函式 `this` | 箭頭函式 `this` |
|------|---------------|---------------|
| 物件方法呼叫 `obj.fn()` | `obj` | 外層作用域（不推薦作為方法）|
| 一般呼叫 `fn()` | `undefined`（strict）/ global | 外層作用域 |
| `setTimeout` / `setInterval` callback | `undefined`（strict）/ global | 外層作用域（推薦使用）|
| `call` / `apply` / `bind` 強制綁定 | 指定的 this | **無效**，仍是外層作用域 |
| 建構函式 `new Fn()` | 新建立的物件 | **不能用作建構函式**（TypeError）|
| 事件監聽器 callback | 觸發事件的元素 | 外層作用域 |

---

#### 4. 箭頭函式不適合的場景

```ts
// ❌ 物件方法（this 不指向物件）
const user = {
  name: 'Alice',
  getName: () => {
    return this.name // undefined！this 是外層（通常是 window 或 undefined）
  }
}

// ✅ 改用傳統函式或 shorthand
const user = {
  name: 'Alice',
  getName() {
    return this.name // 'Alice'
  }
}

// ❌ 建構函式
const Person = (name: string) => {
  this.name = name // TypeError: Arrow functions cannot be used as constructors
}

// ❌ prototype 方法（this 指向外層，通常不是實例）
function Person(name: string) {
  this.name = name
}
Person.prototype.greet = () => {
  return `Hi, I'm ${this.name}` // undefined
}
```

---

#### 5. 在 Vue 3 中的應用

Vue 3 Composition API 大量依賴箭頭函式，因為 `setup()` 不依賴 `this`：

```ts
import { ref, computed, watch } from 'vue'

const useCounter = (initial: number = 0) => {
  const count = ref<number>(initial)

  // 箭頭函式作為 computed getter（無 this 問題）
  const doubled = computed(() => count.value * 2)

  // 箭頭函式作為方法
  const increment = (): void => {
    count.value++
  }

  const reset = (): void => {
    count.value = initial
  }

  // 箭頭函式作為 watch callback（可取得新舊值）
  watch(count, (newVal, oldVal) => {
    console.log(`count: ${oldVal} → ${newVal}`)
  })

  return { count, doubled, increment, reset }
}
```

---

#### 6. 隱式回傳的細節與常見錯誤

```ts
// ✅ 正確：單一運算式隱式回傳
const square = (n: number) => n * n

// ✅ 正確：回傳物件需加括號
const makePoint = (x: number, y: number) => ({ x, y })

// ❌ 錯誤：遺漏括號，被解析為空的函式主體 + 物件標籤語法
const makePoint = (x: number, y: number) => { x, y } // 回傳 undefined！

// ✅ 正確：多行邏輯仍需 return
const process = (n: number) => {
  const doubled = n * 2
  return doubled + 1
}
```

---

### 二、模板字串（Template Literal）

**定義**：使用反引號（`` ` ``）包裹的字串，支援多行、嵌入表達式、標籤函式。

---

#### 1. 基本語法與字串嵌入

```ts
const name = 'Alice'
const age = 25

// 傳統字串拼接
const msg1 = 'Hello, ' + name + '! You are ' + age + ' years old.'

// Template Literal（更直覺）
const msg2 = `Hello, ${name}! You are ${age} years old.`

// ${} 內可放任意 JavaScript 表達式
const a = 10, b = 20
console.log(`Sum: ${a + b}`)                    // Sum: 30
console.log(`Max: ${Math.max(a, b)}`)           // Max: 20
console.log(`Is adult: ${age >= 18 ? 'Yes' : 'No'}`) // Is adult: Yes
```

---

#### 2. 多行字串

```ts
// 傳統方式（需要 \n）
const html1 = '<div>\n  <p>Hello</p>\n</div>'

// Template Literal（自然換行）
const html2 = `
<div>
  <p>Hello</p>
</div>
`

// 注意：字串會包含縮排與換行，可能需要 .trim()
const sql = `
  SELECT *
  FROM users
  WHERE id = ${userId}
  AND is_active = true
`.trim()
```

---

#### 3. 嵌套模板字串

```ts
const items = ['apple', 'banana', 'cherry']

// 在 ${} 中使用另一個模板字串或陣列操作
const list = `
<ul>
  ${items.map(item => `<li>${item}</li>`).join('\n  ')}
</ul>
`
```

---

#### 4. 標籤模板（Tagged Template Literal）

標籤模板是進階用法，可讓函式處理模板字串的「片段」與「嵌入值」：

```ts
// 標籤函式接收：strings（固定字串陣列）, ...values（嵌入的表達式值）
function highlight(strings: TemplateStringsArray, ...values: unknown[]): string {
  return strings.reduce((result, str, i) => {
    const value = values[i - 1]
    return result + (value !== undefined ? `<mark>${value}</mark>` : '') + str
  })
}

const product = 'TypeScript'
const price = 0

const msg = highlight`學習 ${product} 完全免費，售價 ${price} 元！`
// 學習 <mark>TypeScript</mark> 完全免費，售價 <mark>0</mark> 元！
```

**實際應用場景**：
- CSS-in-JS 函式庫（如 `styled-components`）：`` styled.div`color: red;` ``
- SQL 安全查詢（防止 SQL injection）
- 國際化（i18n）字串處理
- GraphQL 查詢語法（`` gql`query { ... }` ``）

---

#### 5. Raw 字串

```ts
// String.raw 標籤：不處理跳脫字符
const path = String.raw`C:\Users\Alice\Documents`
// 'C:\\Users\\Alice\\Documents'（\ 不被解析為跳脫字符）

// 對比普通字串
const path2 = `C:\Users\Alice\Documents`
// 'C:UsersAliceDocuments'（\U 等被解析，無效時被忽略）

// 常見應用：RegExp 字串、Windows 路徑、LaTeX
const regex = new RegExp(String.raw`\d+\.\d+`)
```

---

#### 6. TypeScript 中的型別應用

```ts
// 模板字串型別（TypeScript 4.1+）
type EventName = 'click' | 'focus' | 'blur'
type HandlerName = `on${Capitalize<EventName>}`
// HandlerName = 'onClick' | 'onFocus' | 'onBlur'

// 用於建立動態屬性名稱型別
type CSSProperty = 'color' | 'fontSize' | 'padding'
type CSSGetter = `get${Capitalize<CSSProperty>}`
// 'getColor' | 'getFontSize' | 'getPadding'
```

---

#### 7. 實際開發中的應用場景

```ts
// ① API URL 組合
const baseUrl = 'https://api.example.com'
const userId = 42
const endpoint = `${baseUrl}/users/${userId}/profile`

// ② Vue 中動態 class 名稱
const size: 'sm' | 'md' | 'lg' = 'md'
const className = `btn-${size} px-4 py-2` // 'btn-md px-4 py-2'

// ③ 錯誤訊息格式化
function createError(field: string, rule: string, value: unknown): string {
  return `欄位 "${field}" 驗證失敗：${rule}（當前值：${JSON.stringify(value)}）`
}

// ④ 多行 SQL / GraphQL 查詢
const query = `
  query GetUser($id: ID!) {
    user(id: $id) {
      name
      email
      role
    }
  }
`

// ⑤ 動態產生 HTML 片段（搭配 Vue render function 或測試）
const createCard = (title: string, content: string): string => `
  <div class="card">
    <h2>${title}</h2>
    <p>${content}</p>
  </div>
`
```

---

### 三、兩者搭配：箭頭函式 + Template Literal

這兩個特性在實際開發中幾乎形影不離：

```ts
interface Product {
  id: number
  name: string
  price: number
  category: string
}

// 格式化函式：箭頭函式 + Template Literal
const formatPrice = (price: number): string =>
  `NT$ ${price.toLocaleString()}`

const formatProduct = (p: Product): string =>
  `[${p.category}] ${p.name} — ${formatPrice(p.price)}`

const createProductCard = (p: Product): string => `
  <div class="product-card" data-id="${p.id}">
    <span class="category">${p.category}</span>
    <h3>${p.name}</h3>
    <p class="price">${formatPrice(p.price)}</p>
  </div>
`.trim()

// 使用
const products: Product[] = [
  { id: 1, name: 'TypeScript 進階指南', price: 1200, category: '書籍' },
  { id: 2, name: 'VS Code 授權', price: 0, category: '工具' },
]

const catalog = products.map(createProductCard).join('\n')
```

---

### 四、W1 小結：ES6+ 語法全景回顧

W1 涵蓋的四大主題彼此環環相扣：

| 主題 | 核心概念 | Vue 3 連結 |
|------|---------|-----------|
| 解構賦值 | 從物件/陣列取出值、重命名、預設值 | `toRefs` 解決響應性、Props 解構 |
| Spread / Rest | 打散與蒐集、淺拷貝、不可變更新 | `{ ...state }` 狀態更新、v-bind 展開 |
| **箭頭函式** | **this 靜態綁定、簡潔語法** | **Composable / computed / watch callback** |
| **Template Literal** | **字串嵌入、多行、標籤函式** | **動態 class、URL 組合、render function** |

---

## 作業說明

### 作業主題：實作「商品目錄格式化工具函式庫」

使用 **箭頭函式 + Template Literal**（加上 TypeScript 型別定義），實作一組處理商品資料的格式化工具函式。

---

### 情境背景

ihouseBMS 前端需要處理以下房屋物件資料，並產生各種格式的輸出：

```ts
interface Property {
  id: number
  title: string
  address: {
    city: string
    district: string
    street: string
  }
  price: number          // 元
  area: number           // 坪
  type: 'apartment' | 'house' | 'studio'
  features: string[]
  isAvailable: boolean
  listedAt: Date
}
```

---

### 功能需求

#### 1. `formatAddress(property)` — 格式化完整地址

要求：
- 使用 Template Literal 組合 `city`、`district`、`street`
- 回傳格式：`台北市信義區忠孝東路一段10號`

```ts
formatAddress(property)
// '台北市信義區忠孝東路一段10號'
```

---

#### 2. `formatPrice(price, unit?)` — 格式化房價

要求：
- 使用**箭頭函式**定義，支援選填 `unit` 參數（預設為 `'萬'`）
- 超過 10000 萬時自動換算為「億」
- 使用 Template Literal 組合單位

```ts
formatPrice(1500)         // 'NT$ 1,500 萬'
formatPrice(25000)        // 'NT$ 2.5 億'
formatPrice(500, '元')    // 'NT$ 500 元'
```

---

#### 3. `createPropertySummary(property)` — 產生物件摘要

要求：
- 使用**多行 Template Literal** 組合摘要文字
- 使用 **`formatAddress`** 和 **`formatPrice`** 組合資訊
- 格式如下（需精確對應）：

```
【公寓】台北市信義區忠孝東路一段10號
售價：NT$ 2,500 萬 | 坪數：35 坪 | 單價：NT$ 71.43 萬/坪
特色：電梯、車位、近捷運
狀態：供應中
```

---

#### 4. `createBatchReport(properties)` — 批次產生報表

要求：
- 使用**箭頭函式 + 陣列方法**產生完整報表
- 使用 Template Literal 產生標題與頁尾
- 每筆物件呼叫 `createPropertySummary` 並以分隔線隔開

```ts
createBatchReport(properties)
/*
=== 物件報表（共 3 筆）| 產生時間：2026-04-27 ===

【公寓】...
...

---
【透天】...
...

===========================
*/
```

---

#### 5. `createHighlightTag(keyword)` — 標籤模板工廠函式

要求：
- 回傳一個**標籤函式（Tagged Template Function）**
- 該標籤函式會將所有 `${表達式}` 用指定關鍵字樣式包裹

```ts
const bold = createHighlightTag('strong')
const result = bold`物件名稱：${'台北豪宅'} 售價：${2500} 萬`
// '物件名稱：<strong>台北豪宅</strong> 售價：<strong>2500</strong> 萬'
```

---

### 技術規範

| 項目 | 要求 |
|------|------|
| 語法 | 所有函式使用**箭頭函式**定義（const fn = () => {}） |
| Template Literal | 所有字串組合禁止使用 `+` 拼接，全部改用 `` ` `` |
| TypeScript | 所有函式需完整型別標注，**禁止使用 `any`** |
| 純函式 | 不得修改輸入參數 |
| 邊界處理 | `features` 為空陣列、`price` 為 0、`isAvailable` 為 false 均需正確處理 |
| 測試 | 附上 5 組以上的手動測試案例（`console.log` 即可），覆蓋上述邊界情況 |

---

### 評分標準

| 項目 | 分數 |
|------|------|
| `formatAddress` Template Literal 組合正確 | 10 分 |
| `formatPrice` 箭頭函式 + 單位換算正確 | 20 分 |
| `createPropertySummary` 多行 Template Literal 格式正確 | 20 分 |
| `createBatchReport` 箭頭函式鏈式操作 + 格式正確 | 20 分 |
| `createHighlightTag` Tagged Template 實作正確 | 15 分 |
| TypeScript 型別完整，無 `any` | 10 分 |
| 邊界情況正確處理 | 5 分 |
| **總計** | **100 分** |

---

### 延伸思考題（選做，不計分）

1. 箭頭函式沒有 `arguments` 物件，若需要不定數量的參數，應該怎麼做？（提示：W1 Day 2 學過）
2. 為什麼 Vue 3 的 `setup()` 中，`this` 不再有意義？這跟箭頭函式的設計有什麼關係？
3. Tagged Template Literal 如何被用來防止 XSS 攻擊？試著設計一個 `safeHtml` 標籤函式，對嵌入值進行 HTML 跳脫處理。

---

### 提交方式

在 `daily-learning` 專案中，於 `2026-04-27/` 目錄下建立 `作業實作.ts`（或 `.js`）檔案，並發 PR 請求審核。
