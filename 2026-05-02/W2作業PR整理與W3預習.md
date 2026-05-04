# W2 作業 PR 整理 × W3 預習：Vue 3 核心響應式 API

> **日期**：2026-05-02（M1-W2 Day 5）
> **類型**：實作日（PR 提交） + 預習日（W3 起跑）
> **目標技術**：① 完成 W2 async-utils PR　② 預習 `ref` / `reactive` / `computed` / `watch`
> **截止日提醒**：W2 截止 2026-05-04（剩 **2 天**）⚠️

---

## 今日目標

| 優先度 | 任務 | 預計時間 |
|--------|------|---------|
| ⭐⭐⭐ 高 | 整理 async-utils 作業目錄，發出 W2 PR | 1–1.5 小時 |
| ⭐⭐ 中 | 預習 W3 主題：`ref` / `reactive` / `computed` / `watch` 深入 | 1.5–2 小時 |
| ⭐ 低 | 檢查 W1 三份 PR 狀態，確認已發出 | 15 分鐘 |

---

## 第一部分｜W2 PR 整理：async-utils 模組

### PR 前的自我審查清單

在發出 PR 之前，依序完成以下檢查：

#### ✅ 目錄結構確認

```
2026-04-28/
└── async-utils/
    ├── httpClient.ts     ← createHttpClient, HttpError
    ├── retry.ts          ← withRetry（指數退避）
    ├── cache.ts          ← withCache（TTL 快取）
    ├── concurrent.ts     ← fetchConcurrent（Sliding Window 並行）
    ├── useAsyncState.ts  ← Vue 3 Composable
    └── index.ts          ← 統一 re-export
```

#### ✅ TypeScript 型別安全審查

逐一確認以下問題：

```typescript
// 問題 1：有沒有用到 any？→ 應換成泛型
// ❌ 壞的
async function withRetry(fn: () => Promise<any>): Promise<any>

// ✅ 好的
async function withRetry<T>(fn: () => Promise<T>): Promise<T>

// 問題 2：錯誤型別有沒有縮窄？
// ❌ 壞的
catch (error) {
  console.error(error.message)  // error 型別是 unknown
}

// ✅ 好的
catch (error) {
  if (error instanceof Error) {
    console.error(error.message)
  }
}

// 問題 3：HttpClient interface 有沒有明確標注 method 型別？
interface HttpClient {
  get<T>(path: string): Promise<T>
  post<T>(path: string, body: unknown): Promise<T>
}
```

#### ✅ 關鍵邏輯確認（核心題目）

**重試機制（withRetry）：是否實作指數退避？**

```typescript
// 指數退避公式：delay = baseDelay * 2^(attempt - 1)
// 若 baseDelay = 1000ms：第1次=1s, 第2次=2s, 第3次=4s...
const delay = (ms: number) => new Promise(resolve => setTimeout(resolve, ms))
await delay(baseDelay * Math.pow(2, attempt - 1))
```

**快取機制（withCache）：是否處理 TTL 過期？**

```typescript
// 快取項目結構：{value, expiredAt}
interface CacheEntry<T> {
  value: T
  expiredAt: number  // Date.now() + ttl
}

// 取值時檢查是否過期
if (entry && Date.now() < entry.expiredAt) {
  return entry.value  // 快取命中
}
// 否則重新請求
```

**並行控制（fetchConcurrent）：Sliding Window 是否正確？**

```typescript
// 核心問題：讓「任意時刻最多 N 個 Promise 在執行」
// 正確做法：使用 Set 追蹤 in-flight requests，每完成一個就加入下一個

async function fetchConcurrent<T>(
  requests: Array<() => Promise<T>>,
  limit: number
): Promise<T[]> {
  const results: T[] = new Array(requests.length)
  const executing = new Set<Promise<void>>()

  for (let i = 0; i < requests.length; i++) {
    const p = requests[i]().then(result => {
      results[i] = result
    }).finally(() => {
      executing.delete(p as unknown as Promise<void>)
    })

    executing.add(p as unknown as Promise<void>)

    if (executing.size >= limit) {
      await Promise.race(executing)
    }
  }

  await Promise.all(executing)
  return results
}
```

#### ✅ PR 描述格式（GitHub PR Template）

```markdown
## 作業摘要

實作非同步資料流封裝工具庫（`async-utils`），包含：
- `createHttpClient`：HTTP 客戶端，統一錯誤處理與 baseUrl
- `withRetry`：指數退避重試裝飾器
- `withCache`：帶 TTL 的記憶化快取裝飾器
- `fetchConcurrent`：Sliding Window 並行控制器（上限 N 個）
- `useAsyncState`：Vue 3 Composable，管理 loading/error/data 狀態

## 學習重點

1. `Promise.race()` 在並行控制中的實際應用
2. 裝飾器（Decorator）模式 vs 繼承（Inheritance）的設計選擇
3. TypeScript 泛型在非同步函式中的正確應用

## 自我評分

| 項目 | 說明 | 分數 |
|------|------|------|
| 功能完整性 | 五個模組皆可獨立運作 | /25 |
| TypeScript 型別安全 | 無 any，正確縮窄 | /25 |
| 錯誤處理 | 每個模組均有錯誤邊界 | /25 |
| 程式碼可讀性 | 命名清晰、有適當說明 | /25 |

## 已知限制 / 待改進

- [ ] `withCache` 的 key 目前使用 `JSON.stringify(args)`，遇到循環參考會失敗
- [ ] `fetchConcurrent` 沒有取消機制（AbortController 待加入）
```

---

## 第二部分｜W3 預習：Vue 3 核心響應式 API 深入

> W3（5/5–5/11）主題：`ref` / `reactive` / `computed` / `watch`
> 今天的預習將讓週一開始「不從零開始」，而是直接進入深水區。

---

### `ref` 進階：你還不知道的細節

#### 1. `ref` 的自動解包（Auto-unwrap）

```vue
<script setup lang="ts">
import { ref } from 'vue'

const count = ref(0)
</script>

<template>
  <!-- ✅ 在 template 中不需要 .value，Vue 自動解包 -->
  <p>{{ count }}</p>

  <!-- ❌ 在 script 中必須加 .value -->
  <!-- count++  → 不會響應！ -->
  <!-- count.value++  → 正確 -->
</template>
```

**陷阱：Reactive 物件中的 ref 會自動解包，但陣列中的 ref 不會！**

```typescript
const arr = reactive([ref(1), ref(2)])

// 陣列中的 ref 需要手動 .value
console.log(arr[0].value)  // 1 ← 需要 .value

const obj = reactive({ count: ref(1) })

// 物件中的 ref 自動解包
console.log(obj.count)  // 1 ← 不需要 .value
```

#### 2. Template Ref：操作 DOM 元素

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'

// ref 的另一個用途：獲取 DOM 元素參考
const inputEl = ref<HTMLInputElement | null>(null)

onMounted(() => {
  // 元件掛載後，ref 自動指向對應 DOM
  inputEl.value?.focus()
})
</script>

<template>
  <!-- 同名屬性 ref 與 script 中的 ref 變數自動綁定 -->
  <input ref="inputEl" type="text" />
</template>
```

#### 3. `shallowRef`：效能優化的取捨

```typescript
import { shallowRef, triggerRef } from 'vue'

// shallowRef：只有 .value 本身被追蹤，內層屬性變更不觸發更新
const state = shallowRef({ count: 0 })

state.value.count++  // ❌ 不觸發更新（內層沒有響應性）
state.value = { count: 1 }  // ✅ 觸發更新（整個 .value 換掉了）

// 若確實需要強制更新（少用）
triggerRef(state)
```

**使用時機**：大型陣列或不需要深層響應的靜態資料（如設定物件）。

---

### `reactive` 進階：限制與陷阱

#### 1. 致命陷阱：不可「重新賦值」

```typescript
import { reactive } from 'vue'

let state = reactive({ count: 0 })

// ❌ 重新賦值後，原本的響應性追蹤斷開了！
// 所有讀取 state.count 的地方不再更新
state = reactive({ count: 999 })

// ✅ 正確做法：只修改屬性
state.count = 999

// ✅ 若需要整體替換，用 Object.assign
Object.assign(state, { count: 999, name: 'new' })
```

#### 2. 解構後失去響應性（W1 已學，W3 加深）

```typescript
import { reactive, toRefs } from 'vue'

const pos = reactive({ x: 0, y: 0 })

// ❌ 解構後 x, y 是普通數字，失去響應性
const { x, y } = pos

// ✅ 用 toRefs 保持響應性（每個屬性變成 ref）
const { x: rx, y: ry } = toRefs(pos)
console.log(rx.value)  // 0
```

#### 3. `reactive` 不能包裹基本型別

```typescript
// ❌ 這樣沒有意義
const count = reactive(0)  // 只會回傳 0，不會是響應式的

// ✅ 基本型別一律用 ref
const count = ref(0)
```

---

### `computed` 進階：快取、可寫、副作用警告

#### 1. 快取機制：為什麼 computed 比 method 快？

```typescript
import { ref, computed } from 'vue'

const expensiveList = ref([/* 大量資料 */])

// method：每次 template 重新渲染都執行
function filteredByMethod() {
  return expensiveList.value.filter(item => item.active)
}

// computed：依賴沒變就直接回傳快取，不重新執行
const filteredByComputed = computed(
  () => expensiveList.value.filter(item => item.active)
)

// 規則：computed 的回傳值相同時（依賴沒變），下次讀取直接用快取
```

#### 2. 可寫的 computed（Writable Computed）

```typescript
import { ref, computed } from 'vue'

const firstName = ref('John')
const lastName = ref('Doe')

// 同時定義 getter 和 setter
const fullName = computed({
  get: () => `${firstName.value} ${lastName.value}`,
  set: (newVal: string) => {
    const parts = newVal.split(' ')
    firstName.value = parts[0]
    lastName.value = parts[1] ?? ''
  }
})

fullName.value = 'Jane Smith'
// firstName.value === 'Jane'
// lastName.value === 'Smith'
```

#### 3. ⚠️ computed 不應有副作用

```typescript
// ❌ 在 computed 中修改其他響應式資料
const doubled = computed(() => {
  count.value++     // 💥 副作用！這會導致無限循環或難以追蹤的 bug
  return count.value * 2
})

// ✅ 若需要在值改變時做事，用 watch
watch(count, (newVal) => {
  console.log('count changed to', newVal)
})
```

---

### `watch` 進階：選項、清理、與 `watchEffect` 的差異

#### 1. `watch` 的四種監聽形式

```typescript
import { ref, reactive, watch } from 'vue'

const count = ref(0)
const user = reactive({ name: 'Alice', age: 30 })

// 形式 1：監聽單一 ref
watch(count, (newVal, oldVal) => {
  console.log(`count: ${oldVal} → ${newVal}`)
})

// 形式 2：監聽 reactive 物件的某個屬性（用 getter 函式）
watch(
  () => user.name,  // ✅ 正確
  (newName) => console.log('name changed:', newName)
)

// ❌ 直接傳屬性值不會響應
// watch(user.name, ...)  → 只是 'Alice' 字串，不是響應式

// 形式 3：監聽多個來源（陣列形式）
watch(
  [count, () => user.name],
  ([newCount, newName], [oldCount, oldName]) => {
    console.log(`count: ${oldCount}→${newCount}, name: ${oldName}→${newName}`)
  }
)

// 形式 4：監聽整個 reactive 物件（需加 deep: true）
watch(
  user,
  (newUser) => console.log('user changed:', newUser),
  { deep: true }
)
```

#### 2. 選項：`immediate`、`deep`、`flush`

```typescript
watch(
  source,
  callback,
  {
    immediate: true,  // 立即執行一次（不等待第一次變更）
    deep: true,       // 深層監聽（效能成本較高）
    flush: 'post',    // 'pre'（預設）| 'post'（DOM 更新後）| 'sync'（同步）
  }
)

// 常見情境：需要在元件掛載後立刻執行 API 呼叫
watch(
  () => props.userId,
  async (userId) => {
    const data = await fetchUser(userId)
    user.value = data
  },
  { immediate: true }
)
// 等價但更簡潔：不再需要 onMounted + watch 兩段邏輯！
```

#### 3. 停止監聽 & 清理副作用

```typescript
// watch 回傳停止函式
const stopWatch = watch(count, callback)
stopWatch()  // 手動停止（元件 unmount 時 Vue 會自動停止）

// onCleanup：清理上一次 watch 遺留的副作用
watch(userId, (newId, oldId, onCleanup) => {
  const controller = new AbortController()

  fetch(`/api/user/${newId}`, { signal: controller.signal })
    .then(r => r.json())
    .then(data => { user.value = data })

  // 下次 userId 改變、或元件 unmount 時，取消上一次的請求
  onCleanup(() => controller.abort())
})
```

#### 4. `watch` vs `watchEffect`：如何選擇？

| | `watch` | `watchEffect` |
|---|---------|---------------|
| 依賴追蹤 | 明確指定 | 自動追蹤（執行時讀到的都算）|
| 取得 oldVal | ✅ 有 | ❌ 無 |
| 初次執行 | 需 `immediate: true` | 自動執行一次 |
| 適用場景 | 知道要監聽誰、需要比較新舊值 | 副作用邏輯複雜、依賴不確定 |

```typescript
// watchEffect 範例：自動追蹤所有讀到的響應式資料
const userId = ref(1)
const user = ref<User | null>(null)

watchEffect(async (onCleanup) => {
  const controller = new AbortController()
  onCleanup(() => controller.abort())

  // 自動追蹤 userId.value，不需要指定
  const data = await fetchUser(userId.value, controller.signal)
  user.value = data
})
// 只要 userId.value 改變，watchEffect 就會重新執行
```

---

## 作業說明｜W3 預習作業（非正式，5/5 前完成）

在正式進入 W3 之前，嘗試用「沒有教材」的方式回答以下問題，檢測你的理解深度：

### 自我檢核（請用自己的話回答，不查文件）

**1. `ref(obj)` 和 `reactive(obj)` 的最大差異是什麼？什麼情況下你會選 `reactive`？**

> 提示方向：`.value`、解構行為、重新賦值限制

**2. 下面這段程式碼有什麼問題？**

```typescript
const items = reactive<string[]>([])
const { length } = items

watch(length, () => {  // 這樣有效嗎？
  console.log('length changed')
})
```

> 提示方向：解構後失去響應性，`watch` 監聽的是什麼？

**3. `computed(() => a.value + b.value)` 什麼時候不會重新計算？**

> 提示方向：依賴沒有變化、getter 沒有被讀取

**4. `watch` 的 `immediate: true` 和直接在 `onMounted` 裡呼叫一次有什麼不同？**

> 提示方向：`immediate` 在 `onMounted` 之前執行

---

## 明日預告（5/3）

**W2 截止前的最後衝刺**：
- 確認所有 PR 已提交（W1 × 3、W2 × 1）
- 若有 PR 收到 Review 回饋，整理回覆
- 繼續深化 W3 預習（Lifecycle Hooks 與 `watchEffect` 互動）

**後天（5/5）起正式進入 W3**：
- `ref` / `reactive` / `computed` / `watch` 的完整教材
- 任務清單元件的「第二版」：納入本週預習的所有深入知識
