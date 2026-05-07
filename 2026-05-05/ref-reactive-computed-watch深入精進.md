# ref / reactive / computed / watch 深入精進

> **日期**：2026-05-05（M1-W3 Day 1，W3 正式啟動）
> **類型**：新主題日（理論深化）
> **週次**：W3（5/5–5/11），主軸：Vue 3 響應式 API 完整掌握

---

## 目標技術

| API | W2 末理解程度 | W3 Day 1 深化目標 |
|-----|:-----------:|-----------------|
| `ref` vs `reactive` | 4.4 | 能說出 5 個決策標準、解釋 RefImpl vs Proxy 底層差異 |
| `computed` | 4.5 | 能解釋 lazy evaluation、dirty flag、快取 invalidation 機制 |
| `watch` | 4.5 | 完整掌握 4 種監聽形式 + `deep/immediate/flush/once` 所有選項 |
| `watchEffect` | 3.8 | 理解 Effect Graph 概念、async 依賴追蹤邊界 |

---

## 為什麼學這個（與前幾日的連結）

W1（4/23）我們「認識」了 `ref` 和 `reactive`，能避開基本陷阱。  
W2（4/30–5/3）我們在整合 async-utils 和 Lifecycle Hooks 時，反覆用到 `watch + immediate`、`watchEffect + onCleanup`。

**W3 的轉變**：從「能正確使用」升級到「能解釋設計原因」。

具體來說：
- W1 解決了「什麼時候用哪個 API」
- W3 要解答「這個 API 為什麼要這樣設計」

W3 作業（TaskList.vue 第二版）需要你整合所有這些知識，因此今天理論上的深化，將直接服務於本週的實作。

---

## 核心知識說明

### 一、`ref` 的底層：`RefImpl` 類別

`ref` 並非直接使用 `Proxy`，而是 Vue 自己實作的 `RefImpl` 類別：

```typescript
// Vue 3 原始碼簡化版（packages/reactivity/src/ref.ts）
class RefImpl<T> {
  private _value: T
  private _rawValue: T
  public dep?: Dep  // 依賴收集器

  constructor(value: T) {
    this._rawValue = toRaw(value)
    this._value = toReactive(value)  // 若是物件，內部轉為 reactive
  }

  get value() {
    trackRefValue(this)      // 讀取時：收集依賴
    return this._value
  }

  set value(newVal: T) {
    const useDirectValue = isShallow(newVal) || isReadonly(newVal)
    newVal = useDirectValue ? newVal : toRaw(newVal)
    if (hasChanged(newVal, this._rawValue)) {
      this._rawValue = newVal
      this._value = useDirectValue ? newVal : toReactive(newVal)
      triggerRefValue(this, newVal)  // 變更時：觸發更新
    }
  }
}
```

**關鍵細節**：
- 當你 `ref({ name: 'Alice' })` 時，內部會自動呼叫 `toReactive()`，讓物件部分也具備深度響應性
- 這就是為什麼 `ref` 包裝物件和 `reactive` 包裝物件行為幾乎相同，差異只在 `.value` 那一層

---

### 二、`reactive` 的底層：`Proxy` 代理

`reactive` 直接對物件套用 ES6 `Proxy`：

```typescript
// 簡化版 reactive 實作
function reactive<T extends object>(target: T): T {
  return new Proxy(target, {
    get(target, key, receiver) {
      const res = Reflect.get(target, key, receiver)
      track(target, TrackOpTypes.GET, key)  // 依賴收集
      if (isObject(res)) {
        return reactive(res)  // 深度響應：子物件也自動變 reactive
      }
      return res
    },
    set(target, key, value, receiver) {
      const oldValue = target[key]
      const result = Reflect.set(target, key, value, receiver)
      if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)  // 觸發更新
      }
      return result
    }
  })
}
```

**常見陷阱**：
```typescript
const state = reactive({ user: { name: 'Alice' } })

// ❌ 解構：拿到的是普通 string，不是 Proxy
const { name } = state.user  // name 是 'Alice'，失去響應性

// ✅ 保持路徑存取
console.log(state.user.name)  // 每次都經過 Proxy get trap

// ❌ 整體替換：原 Proxy 參考失效
state = reactive({ user: { name: 'Bob' } })  // TS 報錯，且 UI 不更新

// ✅ 用 ref 包裝，才能整體替換
const state = ref({ user: { name: 'Alice' } })
state.value = { user: { name: 'Bob' } }  // ✅ 觸發更新
```

---

### 三、5 個 ref vs reactive 決策標準

| 情境 | 建議 | 原因 |
|------|------|------|
| **基本型別**（string、number、boolean）| 必須用 `ref` | Proxy 只能代理物件，無法攔截基本型別賦值 |
| **需要解構後仍有響應性** | 用 `ref` + `toRefs` | `reactive` 解構會脫離 Proxy，失去響應性 |
| **需要整體替換物件** | 用 `ref` | `ref.value = newObj` 可觸發更新；`reactive` 無法整體替換 |
| **複雜嵌套物件，不需解構** | 可用 `reactive` | Proxy 自動深度追蹤，語法更乾淨（不需 `.value`）|
| **Composable 的回傳值** | 優先 `ref` | 使用方可以解構（配合 `toRefs`），對外介面友好 |

```typescript
// Composable 最佳實踐
function useUser(id: Ref<number>) {
  const user = ref<User | null>(null)
  const loading = ref(false)
  const error = ref<Error | null>(null)

  watch(id, async (newId) => {
    loading.value = true
    try {
      user.value = await fetchUser(newId)
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  }, { immediate: true })

  return { user, loading, error }  // 回傳 ref，使用方可安全解構
}

// 使用方
const { user, loading, error } = useUser(userId)  // ✅ 仍有響應性
```

---

### 四、`computed` 的 lazy evaluation 與 dirty flag

`computed` 是 Vue 響應式系統中最精妙的設計之一——「懶執行 + 快取」：

```typescript
// ComputedRefImpl 關鍵邏輯（簡化）
class ComputedRefImpl<T> {
  public _value!: T
  public _dirty = true       // ← 核心：標記「快取是否過期」
  public readonly effect: ReactiveEffect<T>

  constructor(getter: ComputedGetter<T>, private readonly _setter) {
    this.effect = new ReactiveEffect(getter, () => {
      // 當依賴改變時，不立即重新執行 getter
      // 而是把 dirty 設為 true，等下次讀取時才真正計算
      if (!this._dirty) {
        this._dirty = true
        triggerRefValue(this)  // 通知依賴此 computed 的地方
      }
    })
  }

  get value() {
    trackRefValue(this)
    if (this._dirty) {
      this._dirty = false
      this._value = this.effect.run()  // 執行 getter，同時收集新依賴
    }
    return this._value  // 快取命中，直接回傳
  }
}
```

**執行流程圖解**：
```
首次讀取 computed.value
  → _dirty = true → 執行 getter → 收集依賴 → _dirty = false → 回傳結果

依賴資料改變
  → _dirty = true → 通知下游（不重新執行 getter！）

再次讀取 computed.value
  → _dirty = true → 重新執行 getter → _dirty = false → 回傳新結果

多次讀取（依賴未改變）
  → _dirty = false → 直接回傳快取 ← 這就是快取的效益
```

**writable computed（getter + setter）**：
```typescript
const firstName = ref('John')
const lastName = ref('Doe')

const fullName = computed({
  get() {
    return `${firstName.value} ${lastName.value}`
  },
  set(val: string) {
    const parts = val.trim().split(' ')
    firstName.value = parts[0]
    lastName.value = parts.slice(1).join(' ')
  }
})

// 讀取
console.log(fullName.value)  // 'John Doe'

// 寫入（觸發 setter）
fullName.value = 'Jane Smith'
console.log(firstName.value)  // 'Jane'
console.log(lastName.value)   // 'Smith'
```

**常見陷阱**：不要在 `computed` getter 裡做「副作用」（Side Effects）：
```typescript
// ❌ 錯誤：getter 裡有副作用
const total = computed(() => {
  console.log('computing...')  // 可以，但要注意頻率
  document.title = `Total: ${items.value.length}`  // ❌ DOM 操作不應在此
  fetch('/api/log')             // ❌ 網路請求絕對不行
  return items.value.reduce((sum, item) => sum + item.price, 0)
})
```

---

### 五、`watch` 完整選項解析

#### 四種監聽形式

```typescript
// 形式 1：監聽單一 ref（最常見）
watch(count, (newVal, oldVal) => {
  console.log(`count: ${oldVal} → ${newVal}`)
})

// 形式 2：監聽 getter 函式（監聽非 ref 的響應式資料路徑）
watch(
  () => user.profile.name,  // getter
  (newName, oldName) => {
    console.log(`name: ${oldName} → ${newName}`)
  }
)
// ⚠️ 不同於 watch(user, ...)，這只追蹤 user.profile.name 這條路徑

// 形式 3：監聽多個來源（陣列）
watch(
  [count, () => user.age],
  ([newCount, newAge], [oldCount, oldAge]) => {
    console.log(`count: ${newCount}, age: ${newAge}`)
  }
)

// 形式 4：監聽整個 reactive 物件
watch(user, (newUser, oldUser) => {
  // ⚠️ newUser === oldUser（同一個 Proxy 參考）
  // 需要深度快照才能比較 old/new 差異
  console.log('user changed:', newUser)
}, { deep: true })
```

#### 所有選項

```typescript
const stop = watch(source, callback, {
  immediate: true,    // 立即執行：setup 階段就執行一次 callback
  deep: true,         // 深度監聽：物件內任何層級改變都觸發
  flush: 'post',      // 執行時機：'pre'（預設）| 'post'（DOM 更新後）| 'sync'（同步）
  once: true,         // 一次性：觸發後自動停止監聽（Vue 3.4+）
})

// 手動停止監聽
stop()
```

**`flush` 選項實際場景**：
```typescript
// flush: 'post' — 需要操作更新後的 DOM
const itemCount = ref(0)
watch(itemCount, async () => {
  await nextTick()  // 不需要！flush: 'post' 已在 DOM 更新後執行
  const list = document.querySelector('.list')
  list.scrollTop = list.scrollHeight
}, { flush: 'post' })

// flush: 'sync' — 同步立即執行（通常只用於除錯）
// ⚠️ 效能警告：依賴每次改變都立即同步觸發，可能造成效能問題
watch(debugValue, (val) => {
  console.log('[debug] value changed:', val)
}, { flush: 'sync' })
```

**`onCleanup` 防 Race Condition（W2 複習，W3 應用）**：
```typescript
watch(searchQuery, (newQuery, oldQuery, onCleanup) => {
  const controller = new AbortController()

  // 當 watch 再次觸發（或元件卸載）時，先執行 onCleanup 取消上次的請求
  onCleanup(() => {
    controller.abort()
  })

  fetch(`/api/search?q=${newQuery}`, {
    signal: controller.signal
  })
    .then(r => r.json())
    .then(data => {
      results.value = data
    })
    .catch(err => {
      if (err.name !== 'AbortError') {
        error.value = err
      }
    })
})
```

---

### 六、`watchEffect` 的 Effect Graph 概念

#### Effect Graph（依賴圖）

Vue 3 的響應式系統維護一個全域的依賴圖：

```
依賴圖示意
─────────────────────────────────────────────────────

響應式資料（節點）           觀察者（Effect）
─────────────────         ─────────────────────────
userId (ref)        ──▶   watchEffect: fetch user
userData (ref)      ──▶   computed: displayName
displayName         ──▶   渲染函式（render effect）
                              │
                              ▼
                           DOM 更新
```

當任何節點的值改變，Vue 沿著依賴圖通知所有相關 Effect，按優先順序排隊執行。

#### `watchEffect` 的同步依賴追蹤

```typescript
const userId = ref(1)
const config = ref({ timeout: 5000 })
const userData = ref(null)

watchEffect(async () => {
  // ✅ 以下在 await 之前，依賴被追蹤：
  const id = userId.value    // ← 追蹤 userId
  const timeout = config.value.timeout  // ← 追蹤 config

  // ↓ 分界線 ↓
  const data = await fetchUser(id, timeout)
  // ↑ await 後，控制權交還 Event Loop ↑

  // ⚠️ 以下在 await 之後，依賴「不被追蹤」：
  const extraData = otherRef.value  // ← 改變 otherRef 不重新觸發
  userData.value = { ...data, extra: extraData }
})
```

**規則**：依賴追蹤只在 Effect 函式的「首個 `await` 之前」有效。

**最佳實踐**：在 `await` 前讀取所有需要監聽的資料：
```typescript
watchEffect(async () => {
  // ✅ 一次性讀取所有需要追蹤的 reactive 資料
  const id = userId.value
  const query = searchQuery.value
  const page = currentPage.value

  // 之後只做非同步操作和賦值
  const result = await fetchData({ id, query, page })
  data.value = result
})
```

#### `watchEffect` vs `watch` 選用時機

| | `watch` | `watchEffect` |
|--|---------|--------------|
| 依賴宣告 | 明確（第一個參數）| 自動收集 |
| 初始執行 | 預設不執行（需 `immediate: true`）| 立即執行 |
| `oldValue` 存取 | ✅ 可以 | ❌ 無法 |
| 適合場景 | 需要比較新舊值、明確追蹤來源 | 簡潔的副作用、依賴多且明確 |

---

## 作業說明

### 作業名稱：TaskList.vue 第二版（W3 里程碑作業）

**情境背景**：
你正在開發 ihouseBMS 的房源管理模組，需要一個能記住篩選條件、支援非同步操作的任務清單元件。

**功能需求**：

| 功能 | 使用的 API | 說明 |
|------|----------|------|
| 任務列表展示 | `reactive` 或 `ref` | 自選最合適的方式組織資料 |
| 新增 / 刪除 / 完成任務 | - | 基本操作 |
| 篩選（全部/進行中/已完成）| `computed` | 快取篩選結果 |
| 統計（總數/完成數/待辦數）| `computed` | 快取計算，避免重複計算 |
| 條件自動持久化到 localStorage | `watch` | 篩選條件改變時自動儲存 |
| 頁面載入時還原篩選條件 | `watch + immediate` | 初始載入 + 後續同步走同一邏輯 |
| 搜尋關鍵字（防 debounce）| `watchEffect` | 練習 watchEffect 的適用場景 |
| 元件卸載清理 | `onUnmounted` + `onCleanup` | 清除計時器或 abort 請求 |

**技術規範**：
- 使用 `<script setup>` + TypeScript
- 每個 Task 物件需有完整型別定義（`interface Task`）
- `ref` vs `reactive` 的選用需有明確理由（寫在元件頂部的註解中）
- `computed` 屬性禁止有副作用
- `watch` 必須至少使用一個 `onCleanup`

**進階（選做）**：
- 將核心邏輯抽取到 `useTaskList.ts` Composable
- Composable 回傳型別需完整標注

**評分標準**（滿分 100 分）：

| 項目 | 配分 |
|------|------|
| TypeScript 型別完整性 | 20 分 |
| `ref`/`reactive` 選用說明 | 10 分 |
| `computed` 正確使用（無副作用、快取有效）| 20 分 |
| `watch` + `onCleanup` 正確實作 | 20 分 |
| `watchEffect` 正確實作（依賴追蹤邊界）| 15 分 |
| 整體程式碼品質（可讀性、命名）| 15 分 |

---

## 自我檢核問題

**Q1**：`ref` 和 `reactive` 都可以包裝物件，為什麼要有兩個？它們的底層實作有何本質差異？

**Q2**：`computed` 為什麼需要 `dirty flag`？如果每次讀取都直接重新執行 getter，會有什麼問題？

**Q3**：以下程式碼有幾個問題？請逐一指出：
```typescript
const state = reactive({ tasks: [], filter: 'all' })
const { tasks, filter } = state  // 解構
watch(tasks, (newTasks) => {
  localStorage.setItem('tasks', JSON.stringify(newTasks))
})
const filtered = computed(() => {
  fetch('/api/sync').then(...)  // 同步到後端
  return tasks.filter(t => filter === 'all' || t.status === filter)
})
```

**Q4**：為什麼在 `watchEffect` 的 `async callback` 中，`await` 之後的響應式資料存取不會被追蹤？這與 JavaScript 的 Event Loop 有什麼關係？

**Q5**：`watch` 的 `flush: 'pre'`（預設）和 `flush: 'post'` 分別在什麼時機執行？請舉出一個真實場景說明何時必須用 `flush: 'post'`。

---

## 明日預告（5/6，W3 Day 2）

**主題**：`ref` auto-unwrap 完整規則 + `toRef` / `toRefs` 使用場景

W3 Day 2 的學習方向：
- `<template>` 中 `ref` auto-unwrap 的觸發條件（為什麼有時需要 `.value`，有時不需要）
- 巢狀 `ref`（`ref` 包 `ref`）的行為
- `toRef`：從 `reactive` 物件建立單一屬性的 `ref`
- `toRefs`：一次性把 `reactive` 物件全部屬性轉為 `ref`（解構安全的核心）
- `shallowRef` / `shallowReactive`：效能優化的響應式工具

> 這些是 W3 作業（TaskList.vue 第二版）中 `ref/reactive` 選用決策的底層支撐。
