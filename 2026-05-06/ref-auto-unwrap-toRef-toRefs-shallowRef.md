# ref auto-unwrap × toRef / toRefs × shallowRef / shallowReactive

> **日期**：2026-05-06（M1-W3 Day 2）
> **類型**：理論深化日（延續 W3 Day 1）
> **週次**：W3（5/5–5/11），主軸：Vue 3 響應式 API 完整掌握

---

## 目標技術

| API / 機制 | W3 Day 1 理解程度 | W3 Day 2 深化目標 |
|------------|:----------------:|------------------|
| `ref` auto-unwrap | 知道「模板會自動解包」 | 能說出 5 種解包/不解包情境，並解釋為什麼 |
| `toRef` | 偶爾用過，不確定差異 | 能解釋它與 `ref()` 的本質不同：保留原物件的「響應式連線」 |
| `toRefs` | 解構 reactive 不會壞響應性的工具 | 完整理解：實作原理、Composable 回傳模式、效能成本 |
| `shallowRef` / `shallowReactive` | 概念聽過 | 能說明何時必須使用、與深度響應的取捨 |
| `markRaw` / `toRaw` | 未接觸 | 知道何時跳出 Vue 響應式系統 |

---

## 為什麼學這個（與前幾日的連結）

W3 Day 1（5/5）我們深入了 `ref` 的 `RefImpl` 類別與 `reactive` 的 `Proxy` 底層，提出了「5 個 `ref` vs `reactive` 決策標準」。

但這套決策有個前提還沒講清楚：**模板裡的 ref 為什麼可以不寫 `.value`？解構 reactive 為什麼會失去響應性？**

今日要回答的是「**響應式參考的傳遞**」這個議題：

```
W3 Day 1                       W3 Day 2（今日）              W3 作業（5/7~）
─────────                      ──────────────                ──────────────
ref / reactive 底層機制    →   ref 的傳遞、解構、auto-unwrap →  TaskList.vue 第二版
                                                                  - 如何設計 store 介面？
                                                                  - 是否拆 Composable？
                                                                  - 哪裡用 shallowRef 優化？
```

**簡單說**：Day 1 解答了「響應式如何運作」，今日要解答「響應式如何傳遞與保存」。

---

## 核心知識說明

### 一、`ref` auto-unwrap：完整規則表

`ref` 在不同情境下有不同的「自動解包」行為。請熟記下表：

| 情境 | 是否 auto-unwrap | 為什麼 | 範例 |
|------|:---------------:|--------|------|
| `<template>` 頂層 ref | ✅ 是 | 模板編譯器特殊處理 | `{{ count }}` ⟶ 取 `count.value` |
| `<template>` 巢狀於物件 | ❌ 否 | 編譯器只看頂層識別字 | `{{ obj.count }}` ⟶ 取 `obj.count`（若是 ref，需自行 `.value`） |
| `<template>` v-for 索引 | ✅ 是 | 編譯器處理頂層 | `v-for="item in list"` 中 `list` 為 ref 自動解包 |
| `reactive` 物件中的 ref 屬性 | ✅ 是 | reactive 的 Proxy 內部會檢查屬性是否為 ref | `const r = reactive({ a: ref(1) }); r.a // 1，不需 .value` |
| `Map` / `Set` 的 ref 元素 | ❌ 否 | 集合型容器不解包 | `new Map([['a', ref(1)]])` ⟶ 取出仍是 ref 物件 |
| `array[index]` | ❌ 否 | 陣列索引不解包 | `arr.value[0]` ⟶ 若元素是 ref，需 `arr.value[0].value` |
| 函式參數 / 物件解構 | ❌ 否 | JS 語意層的賦值/複製 | `const { count } = state`（state 是 reactive）⟶ count 失去響應性 |
| Composable 回傳 ref | ❌ 否 | 一般 JS 變數 | `const { data } = useFetch()` ⟶ data 是 ref，需 `.value` |

**關鍵心法**：

> **「響應式不會自動穿越 JS 的賦值與解構」**

模板裡能省 `.value` 是模板編譯器的便利語法，是「魔法」；JS 程式碼中沒有這個魔法，必須誠實地用 `.value`。

---

### 二、巢狀 ref：`ref(ref(value))` 會發生什麼？

```typescript
const a = ref(1)
const b = ref(a)  // 巢狀！
console.log(b.value)        // 1（直接拿到內部 ref 的 value）
console.log(b.value === 1)  // true
```

**Vue 的處理規則**：當 `ref()` 被傳入另一個 ref 時，Vue 會直接「攤平」，不會建立額外的包裝層。原始碼裡：

```typescript
// ref.ts
function createRef(rawValue, shallow) {
  if (isRef(rawValue)) {
    return rawValue   // 直接回傳原 ref，不再包一層
  }
  return new RefImpl(rawValue, shallow)
}
```

**實務影響**：你不必擔心「不小心包了兩次」會破壞響應性，但這也意味著 `ref(ref(x)) === ref(x)`（指向同一個 RefImpl 實例）。

---

### 三、`toRef`：建立「單一屬性」的響應式參考

`toRef` 解決一個具體痛點：**從 reactive 物件中安全地取出某個屬性，讓它仍能保持響應**。

```typescript
const state = reactive({ count: 0, name: 'Alice' })

// ❌ 失去響應性
const { count } = state
count        // 0，但之後 state.count 改變不會反映

// ❌ 也失去響應性（複製值）
const count = state.count

// ✅ toRef：建立連動的 ref
const count = toRef(state, 'count')
count.value      // 0
state.count = 5
count.value      // 5（自動同步）
count.value = 10
state.count      // 10（雙向同步）
```

**`toRef` 與 `ref` 的本質差異**：

| 比較項目 | `ref(x)` | `toRef(obj, key)` |
|----------|----------|-------------------|
| 內部儲存 | 自有的 `_value` | 不儲存值，每次讀寫都到原物件 |
| 是否與來源連動 | ❌ 獨立 | ✅ 雙向連動 |
| 來源屬性不存在 | — | 仍可建立（lazy），之後設值會生效 |
| 適用場景 | 建立新的響應狀態 | 拆解既有 reactive，給 Composable 或子元件 |

**Vue 3.3+ 的延伸**：可以傳入 getter 函式建立「唯讀的派生 ref」：

```typescript
const fullName = toRef(() => state.firstName + ' ' + state.lastName)
// 類似 computed，但更輕量；無寫入能力
```

---

### 四、`toRefs`：解構 reactive 物件而不失去響應性

`toRefs` 是 `toRef` 的批次版：把整個 reactive 物件的每個屬性都轉成獨立的 ref，回傳同形狀的物件。

```typescript
const state = reactive({ count: 0, name: 'Alice' })
const stateRefs = toRefs(state)
// stateRefs 結構：{ count: Ref<number>, name: Ref<string> }

// 現在可以安全解構
const { count, name } = stateRefs
count.value        // 0
state.count = 5
count.value        // 5（仍然連動）
```

**實作原理**（簡化版）：

```typescript
function toRefs(obj) {
  const result = {}
  for (const key in obj) {
    result[key] = toRef(obj, key)  // 每個屬性都做 toRef
  }
  return result
}
```

**Composable 標準回傳模式**：

```typescript
// useUser.ts
export function useUser() {
  const state = reactive({
    user: null as User | null,
    loading: false,
    error: null as Error | null,
  })

  const fetchUser = async (id: string) => { /* ... */ }

  // 內部用 reactive 寫起來方便（直接 state.loading = true）
  // 對外用 toRefs 讓使用方能解構
  return {
    ...toRefs(state),
    fetchUser,
  }
}

// 元件使用
const { user, loading, error, fetchUser } = useUser()
// user / loading / error 都是 ref，響應性保留
```

**效能成本**：`toRefs` 會走過所有屬性建立 `toRef`，物件巨大時有 O(n) 成本。如果 Composable 內部用 `ref` 各自包裝，可省去 `toRefs` 步驟，但程式碼會比較多 `.value`。

---

### 五、`shallowRef`：只讓「.value 的整體替換」響應

`shallowRef` 與 `ref` 的差別：**不會對內部物件做 `toReactive`**。

```typescript
const r = ref({ a: { b: 1 } })
r.value.a.b = 2      // ✅ 觸發更新（深度響應）

const sr = shallowRef({ a: { b: 1 } })
sr.value.a.b = 2     // ❌ 不會觸發更新（內部不是 reactive）
sr.value = { a: { b: 2 } }  // ✅ 整體替換才會觸發
```

**為什麼需要它？**——效能。

當你的物件巨大（例如：上千筆地圖座標、巨大表格資料、Three.js 場景），`reactive` 的深度 Proxy 化會在初始化和讀取時帶來成本。如果你 **保證只整體替換、不局部修改**，`shallowRef` 可以跳過所有深度代理。

**ihouseBMS 的可能應用**：
- 房源列表的「分頁資料」一次性換整批 ⟶ `shallowRef`
- 大型圖表的資料來源（每次都是新陣列）⟶ `shallowRef`
- Vue Router 的路由元資訊（routeMeta）⟶ `shallowRef`

---

### 六、`shallowReactive`：只代理第一層

```typescript
const obj = shallowReactive({
  topLevel: 0,
  nested: { count: 0 },
})

obj.topLevel = 1      // ✅ 觸發更新
obj.nested.count = 1  // ❌ 不觸發（nested 不是 reactive）
obj.nested = { count: 1 }  // ✅ 觸發（這是第一層的賦值）
```

與 `shallowRef` 的差別：`shallowRef` 連 `.value` 內部的物件都不代理；`shallowReactive` 仍會代理第一層的「設值」操作。

**何時用？**：當你有一個外層需要追蹤但內層是「外部資料、不需要 Vue 接管」的混合物件時。常見於整合第三方類別實例（地圖、編輯器、播放器）：

```typescript
const mapState = shallowReactive({
  visible: true,        // 第一層，需要響應
  mapInstance: null,    // 內部是 mapbox 物件，Vue 不該動它
})
```

---

### 七、`markRaw` / `toRaw`：跳出響應式系統

兩個「逃生艙」工具：

```typescript
// markRaw：標記這個物件「永遠不要被 Vue 響應化」
const heavyClass = markRaw(new Mapbox(...))
const state = reactive({ map: heavyClass })  // map 仍是原始 Mapbox 實例

// toRaw：取出 Proxy 包裝前的原始物件（用於序列化、第三方傳遞）
const raw = toRaw(reactiveState)
JSON.stringify(raw)  // 不會觸發 Vue 內部的 track
```

**典型場景**：
- 引入第三方 SDK 的實例（Mapbox、Monaco Editor、Pixi.js）⟶ `markRaw`
- 把資料丟給非 Vue 的庫（例如 axios、IndexedDB、postMessage）⟶ `toRaw` 避免 Proxy 序列化問題
- 大量靜態設定資料（i18n 字典、enum 表）⟶ `markRaw` 跳過代理

---

## 決策樹：什麼時候用什麼？

```
資料是基本型別？
├── 是 ──────────────────────────▶ ref
└── 否（物件 / 陣列）
     │
     是否需要從外部解構出來？
     ├── 是 ────────────────▶ ref（推薦）或 reactive + toRefs
     └── 否（內部使用）
          │
          物件層級深、需要深度響應？
          ├── 否（只關心整體替換）─▶ shallowRef
          ├── 否（只關心第一層）──▶ shallowReactive
          └── 是
               │
               來自第三方 / 不希望 Vue 接管？
               ├── 是 ───▶ markRaw 包裹後再放入 reactive
               └── 否 ───▶ reactive
```

---

## 常見陷阱

### 陷阱 1：模板巢狀 ref 不會解包

```vue
<script setup>
const data = { count: ref(0) }  // 普通物件中包 ref
</script>

<template>
  <!-- ❌ 顯示「[object Object]」或 ref 物件 -->
  <p>{{ data.count }}</p>
  <!-- ✅ 必須手動解包 -->
  <p>{{ data.count.value }}</p>
</template>
```

**修法**：要嘛把 `data` 改成 `reactive`（reactive 內部的 ref 會解包），要嘛把 `count` 提到頂層。

---

### 陷阱 2：`toRef(state, 'arr')` 不能阻止陣列方法的響應性問題

```typescript
const state = reactive({ list: [1, 2, 3] })
const list = toRef(state, 'list')

list.value = [...list.value, 4]   // ✅ 觸發
state.list.push(4)                // ✅ 觸發（透過 reactive 的 Proxy）
list.value.push(4)                // ✅ 觸發（取的是 state.list 本身，依然是 reactive）
```

`toRef` 不會切斷響應性，這點與 `ref(state.list)` 不同（後者會建立新 RefImpl，與 state 切斷）。

---

### 陷阱 3：`shallowRef` 後又呼叫深層 mutation

```typescript
const sr = shallowRef({ items: [] })

// 整批替換 ✅
sr.value = { items: [...sr.value.items, newItem] }

// 局部修改 ❌
sr.value.items.push(newItem)  // 不會觸發更新！
```

**徵狀**：畫面不更新，但 console.log 印出來資料有改。**檢查**：是不是用了 `shallowRef` 但仍在做局部 mutation。

---

### 陷阱 4：`toRefs` 對非 reactive 物件無意義

```typescript
const obj = { count: 0 }      // 普通物件
const refs = toRefs(obj)       // refs.count 是 ref，但...
refs.count.value = 5
console.log(obj.count)         // 0，沒同步！
```

`toRefs` 的「連動性」來自原物件本身的響應系統。普通物件沒有 Proxy，自然無法觀察修改。

---

## 自我檢核問題

1. **解構 reactive 物件會失去響應性，但解構「`toRefs(reactive物件)`」不會。請用底層機制（不是「就是這樣」）解釋為什麼。**
2. **同一個物件被 `ref` 包後，內部的物件部分是不是 reactive？怎麼驗證？**
3. **`shallowRef` 和 `shallowReactive` 的差異具體在哪一層？請用一個範例說明，把同樣的資料分別用兩者包裝，比較行為。**
4. **如果你有一個 100MB 的 GeoJSON 資料要顯示在地圖上，你會用 `ref` / `reactive` / `shallowRef` / `markRaw` 中的哪個？為什麼？**
5. **下面這段 Composable 有什麼問題？**
   ```typescript
   export function useCounter() {
     const count = ref(0)
     const increment = () => count.value++
     return { count: count.value, increment }  // ← 這裡
   }
   ```

---

## 與 W3 作業（TaskList.vue 第二版）的連結

W3 作業要求：整合 `ref/reactive/computed/watch`，重構 W1 版本的 TaskList。今日所學會在以下決策點派上用場：

| 設計決策 | 應用工具 | 理由 |
|---------|---------|------|
| Composable 回傳介面（`useTasks`）| `toRefs` | 讓元件可解構但不失響應 |
| 篩選器狀態（filter / sort）| `ref` | 基本型別，獨立狀態 |
| 任務集合（tasks 陣列）| `ref` | 整體替換較頻繁（API 重新載入），且需要對外暴露 |
| 大型靜態 enum（任務類型字典）| `markRaw` | 不需響應，避免 Proxy 開銷 |
| 拖曳排序時的 dragState | `shallowRef` | 高頻變動但只整體替換 |
| 從 reactive store 拿單一欄位給子元件 | `toRef` | 雙向連動，子元件改寫可同步回 |

---

## 明日預告（5/7，W3 Day 3）

**主題**：W3 作業正式開工——TaskList.vue 第二版的需求拆解與骨架設計

預計內容：
- 重新閱讀 W1 版 TaskList.vue 的問題（Code Smell 盤點）
- 拆出 `useTasks` Composable 的責任邊界
- 規劃 `ref / reactive / computed / watch` 的具體配置
- 寫出第一份「資料流圖」（State → Computed → Render）

> 實作週啟動。Day 3–5 為主要實作期，Day 6 收尾，Day 7 PR 提交。

---

## 參考資源

- Vue 3 官方文件 — Reactivity in Depth: <https://vuejs.org/guide/extras/reactivity-in-depth.html>
- Vue 3 官方文件 — Reactivity API: Core: <https://vuejs.org/api/reactivity-core.html>
- Vue 3 官方文件 — Reactivity API: Utilities: <https://vuejs.org/api/reactivity-utilities.html>
- Vue 3 原始碼 `packages/reactivity/src/ref.ts`（建議搭配 Day 1 已讀的 `RefImpl` 實作對照閱讀）
