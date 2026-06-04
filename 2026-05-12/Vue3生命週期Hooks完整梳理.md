# Vue 3 生命週期 Hooks 完整梳理

> **日期**：2026-05-12（W4 Day 1）
> **週次**：W4（5/12–5/18）— Vue 3: Lifecycle Hooks × watchEffect × toRefs
> **前置知識**：W3 Composition API 完整實作（useTasks、onMounted、watch 持久化回路）
> **帶著這個問題進入今天**：`watch(source, fn, { immediate: true })` 和 `onMounted + watch` 有何本質差異？什麼情境下各自適用？

---

## 目標技術

| 技術 | 今日重點 |
|------|---------|
| Vue 3 Lifecycle Hooks | 完整時間軸、各鉤子職責、SSR 安全性 |
| `onMounted` vs `watch({ immediate })` | 核心選用決策（W3 遺留問題解答） |
| `onUnmounted` | 副作用清理：clearInterval / AbortController / removeEventListener |
| `onUpdated` / `onBeforeUpdate` | DOM 更新後操作的正確時機 |
| 生命週期 + Composable | Active Instance 機制深化 |

---

## 為什麼學這個

W3 已大量使用 `onMounted`（讀取 localStorage）和 `watch`（寫入 localStorage），
但只把它們當作工具使用，沒有系統性地理解它們在整個生命週期裡的**執行時機與職責邊界**。

W3 結束時留了一個問題：
> `watch(source, handler, { immediate: true })` 和 `onMounted + watch` 的組合，什麼時候用哪個？

這個問題的答案，需要對生命週期時間軸有**精確**的理解——不是「大概知道哪個先跑」，而是「清楚每個鉤子在 DOM 建立前後、資料初始化前後的確切執行時機」。

W4 的核心升級：從「會用 onMounted」→「理解完整 Lifecycle，能主動選擇正確鉤子，並在 Composable 中正確封裝副作用」。

---

## 核心知識點

### 1. 生命週期完整時間軸

```
元件建立
   │
   ▼
【setup()】← Composition API 的入口點，所有初始化在這裡發生
   │
   ▼
【onBeforeMount】← DOM 尚未建立，模板已編譯，$el 不存在
   │              適合：做最後的資料初始化（但通常在 setup 裡做）
   │              注意：此時操作 DOM 會失敗
   ▼
【DOM 建立完成】← Vue 把虛擬 DOM 渲染到真實 DOM
   │
   ▼
【onMounted】← DOM 已存在，$el 可存取
   │           適合：
   │           - 讀取 localStorage / sessionStorage
   │           - 初始化第三方 DOM 庫（ECharts、D3 等）
   │           - 發送「初次載入」API 請求
   │           - 取得元素尺寸（getBoundingClientRect）
   │
   │ ← 使用者互動 / 資料變化觸發更新
   ▼
【onBeforeUpdate】← 資料已改變，DOM 尚未更新
   │               適合：在 DOM 更新前讀取某些 DOM 狀態（如滾動位置）
   ▼
【DOM 更新完成】
   │
   ▼
【onUpdated】← DOM 已反映最新資料
   │           適合：在 DOM 更新後操作 DOM 元素
   │           ⚠️ 警告：不要在 onUpdated 修改響應式資料（會造成無限循環！）
   │
   │ ← 元件即將被移除
   ▼
【onBeforeUnmount】← 元件仍然完整，所有狀態可存取
   │                適合：最後的資料儲存
   ▼
【onUnmounted】← 子元件已卸載，事件監聽已移除
   │             適合：清理副作用（clearInterval、abort fetch、removeEventListener）
   ▼
元件銷毀完畢
```

---

### 2. 各 Lifecycle Hook 對照表

| Hook | 執行時機 | DOM 可存取 | 常見用途 | 陷阱 |
|------|---------|:--------:|---------|------|
| `setup()` | 最早 | ❌ | 初始化狀態、定義方法 | — |
| `onBeforeMount` | DOM 建立前 | ❌ | 少用，通常在 setup 做 | — |
| `onMounted` | DOM 建立後 | ✅ | 讀取 storage、初始化 DOM 庫、首次 API | SSR 中不執行 |
| `onBeforeUpdate` | 資料改變後、DOM 更新前 | ✅（舊 DOM）| 捕捉更新前的 DOM 狀態 | — |
| `onUpdated` | DOM 更新後 | ✅（新 DOM）| DOM 相關的後處理 | **不可修改響應式資料** |
| `onBeforeUnmount` | 元件卸載前 | ✅ | 最後儲存 | — |
| `onUnmounted` | 元件卸載後 | ❌ | 清理計時器、事件、訂閱 | — |
| `onErrorCaptured` | 子元件拋錯 | — | 錯誤邊界處理 | — |

---

### 3. 核心問題解答：`watch({ immediate })` vs `onMounted + watch`

這是 W3 留下的問題，今天完整解答。

#### 場景 A：讀取 localStorage 並監聽後續變化

```typescript
// ❌ 錯誤做法：watch + immediate
watch(tasks, (newVal) => {
  localStorage.setItem('tasks', JSON.stringify(newVal))
}, { immediate: true }) // 第一次執行時，tasks 是空陣列初始值，會覆蓋 localStorage！
```

```typescript
// ✅ 正確做法：onMounted 讀取 + watch 寫入（職責分離）
onMounted(() => {
  const stored = localStorage.getItem('tasks')
  if (stored) tasks.value = JSON.parse(stored)
})

watch(tasks, (newVal) => {
  localStorage.setItem('tasks', JSON.stringify(newVal))
}, { deep: true })
```

**為什麼 `{ immediate: true }` 在這裡是錯的？**

`watch` 的 `immediate: true` 讓 handler 在「監聽建立時」立刻以**當前值**執行一次。
但此時 `tasks.value` 是初始值（空陣列 `[]`），
立刻把空陣列寫進 localStorage，就把使用者之前存的資料**覆蓋掉**了。

---

#### 場景 B：元件載入後立刻發送 API 請求，且 URL 參數會變化時重新請求

```typescript
// ✅ 適合使用 watch + immediate：
// 「初次執行」和「URL 改變後重新執行」邏輯完全相同時
const userId = computed(() => route.params.id)

watch(userId, async (id) => {
  userData.value = await fetchUser(id)
}, { immediate: true }) // 第一次載入就執行，URL 改變也執行，行為一致
```

**為什麼 `{ immediate: true }` 在這裡是正確的？**

因為「初次載入的邏輯」和「參數改變後重新載入的邏輯」**完全相同**——都是「用當前 userId 取得資料」。`immediate: true` 讓兩件事合而為一，不需要 `onMounted` 再多寫一次同樣的 fetch。

---

#### 選用決策樹

```
需要在「元件初始化」時執行某段邏輯
            │
            ▼
  後續是否還需要「當依賴改變時重複執行」？
     │                    │
    否                    是
     │                    │
     ▼                    ▼
使用 onMounted     「初次執行」和「後續執行」的邏輯是否完全相同？
                        │                    │
                       是                    否
                        │                    │
                        ▼                    ▼
               watch + immediate      onMounted（初次）
               （兩者合而為一）      + watch（後續變化）
                                    （職責分離，避免副作用衝突）
```

**判斷核心**：
- 第一次執行的「輸入值」是否是有意義的初始值？→ 是就適合 `immediate`
- 第一次執行是否有特殊動作（讀取而非寫入）？→ 有就需要 `onMounted`

---

### 4. `onUnmounted` 的副作用清理模式

Composable 中封裝副作用時，必須同時封裝「啟動」和「清理」。

```typescript
// ✅ 完整的副作用封裝模式
function useInterval(callback: () => void, ms: number) {
  let timer: ReturnType<typeof setInterval> | null = null

  onMounted(() => {
    timer = setInterval(callback, ms)
  })

  onUnmounted(() => {
    if (timer !== null) {
      clearInterval(timer)
      timer = null
    }
  })
}
```

```typescript
// ✅ fetch 防 Race Condition：onUnmounted + AbortController
function useFetch(url: Ref<string>) {
  const data = ref(null)
  let controller: AbortController | null = null

  watchEffect((onCleanup) => {
    controller = new AbortController()

    fetch(url.value, { signal: controller.signal })
      .then(r => r.json())
      .then(json => { data.value = json })
      .catch(err => {
        if (err.name !== 'AbortError') console.error(err)
      })

    onCleanup(() => controller?.abort())
  })

  onUnmounted(() => controller?.abort())

  return { data }
}
```

---

### 5. SSR 安全的 Lifecycle Hooks

在伺服器端渲染（SSR）環境中，只有 `setup()` 和 `onServerPrefetch()` 會執行。

| Hook | 瀏覽器 | SSR |
|------|:------:|:---:|
| `setup()` | ✅ | ✅ |
| `onServerPrefetch()` | ❌（無效）| ✅（SSR 專用）|
| `onMounted` | ✅ | ❌ |
| `onUpdated` | ✅ | ❌ |
| `onUnmounted` | ✅ | ❌ |

**實務影響**：
- `localStorage`、`window`、`document` 只能在 `onMounted` 中存取（不是 `setup` 頂層）
- W3 的 `useTasks.ts` 把 `localStorage.getItem()` 放在 `onMounted` 是正確的！
- 但如果直接在 `setup()` 頂層呼叫 `window.xxx`，在 SSR 環境會報錯

---

### 6. Lifecycle Hooks 在 Composable 中的使用（Active Instance 複習）

W3 已知：只要在 `setup()` 的**同步執行流**中呼叫 Composable，Composable 內部的 Lifecycle Hooks 就會正確綁定到元件上。

```typescript
// ✅ 正確：setup() 同步呼叫 Composable
export default defineComponent({
  setup() {
    const { tasks } = useTasks() // onMounted 在這裡同步執行，能綁定到元件
  }
})

// ❌ 錯誤：異步呼叫 Composable（Active Instance 已失效）
export default defineComponent({
  async setup() {
    await someAsyncOperation() // ← Active Instance 在 await 後就斷了
    const { tasks } = useTasks() // onMounted 無法正確綁定！
  }
})
```

**結論**：Composable 必須在 `setup()` 的頂層、任何 `await` 之前同步呼叫。

---

## 常見陷阱

### 陷阱 1：在 `onUpdated` 修改響應式資料

```typescript
// ❌ 無限循環！
onUpdated(() => {
  count.value++ // 修改資料 → 觸發更新 → onUpdated 又跑 → 又修改...
})

// ✅ 如果必須在更新後操作，加條件守衛
onUpdated(() => {
  if (someCondition) {
    nextTick(() => { /* 操作 DOM */ }) // 用 nextTick 確保在正確時機
  }
})
```

### 陷阱 2：在 `setup()` 頂層存取 DOM

```typescript
// ❌ 錯誤：setup() 執行時 DOM 尚未建立
const el = document.querySelector('.my-element') // null！

// ✅ 正確：在 onMounted 中存取
const myRef = ref<HTMLElement | null>(null)
onMounted(() => {
  console.log(myRef.value) // 此時 DOM 已存在
})
```

### 陷阱 3：忘記清理計時器 / 事件監聽

```typescript
// ❌ 記憶體洩漏：元件卸載後 interval 仍在執行
onMounted(() => {
  setInterval(() => { count.value++ }, 1000)
  // 沒有清理！元件被移除後 interval 繼續佔用資源
})

// ✅ 配對清理
onMounted(() => {
  const timer = setInterval(() => { count.value++ }, 1000)
  onUnmounted(() => clearInterval(timer)) // 在 onMounted 內部宣告清理
})
```

---

## 作業說明

### W4 Week 作業：生命週期觀察工具（LifecycleLogger）

#### 情境背景

在實際開發中，新人常常不清楚「到底 `onMounted` 先跑，還是子元件的 `onMounted` 先跑？」
「更新觸發時，`onBeforeUpdate` 和 `onUpdated` 的 DOM 狀態有什麼差異？」

我們要做一個**可視化的生命週期觀察工具**：能即時顯示生命週期鉤子的觸發順序與時間戳，讓使用者能互動觸發更新、掛載子元件，親眼觀察生命週期行為。

#### 功能需求

**元件一：`LifecycleLogger.vue`（主元件）**

1. **生命週期日誌顯示**：
   - 顯示一個日誌列表，記錄每次 hook 觸發（鉤子名稱 + 時間戳 + 備注）
   - 格式：`[HH:MM:SS.mmm] onMounted — DOM 已就緒`
   - 支援的 hooks：`onBeforeMount`、`onMounted`、`onBeforeUpdate`、`onUpdated`、`onBeforeUnmount`、`onUnmounted`

2. **互動控制**：
   - 「觸發更新」按鈕：修改某個響應式資料，觸發 `onBeforeUpdate` / `onUpdated`
   - 「掛載子元件」按鈕：條件式顯示子元件 `ChildLogger.vue`，觀察父子掛載順序
   - 「卸載子元件」按鈕：移除子元件，觀察 `onUnmounted` 觸發
   - 「清空日誌」按鈕：清除所有日誌

3. **watchEffect 整合**：
   - 加入一個用 `watchEffect` 監聽資料的副作用
   - 在日誌中顯示 `watchEffect` 的觸發（標記為不同顏色）
   - 展示 `onCleanup` 的呼叫時機

**元件二：`ChildLogger.vue`（子元件）**

1. 同樣記錄自身的 `onMounted`、`onUnmounted`
2. 在日誌中標記「子元件」來源，便於觀察父子順序

**Composable：`useLifecycleLog.ts`**

1. 封裝日誌狀態（`logs: Ref<LogEntry[]>`）
2. 提供 `addLog(hookName: string, note?: string)` 方法
3. 提供 `clearLogs()` 方法
4. `LogEntry` 型別：
   ```typescript
   interface LogEntry {
     id: number
     timestamp: string  // 'HH:MM:SS.mmm'
     hookName: string
     source: 'parent' | 'child' | 'watchEffect'
     note?: string
   }
   ```

#### 技術規範

- 使用 Vue 3 Composition API（`<script setup>`）
- `useLifecycleLog.ts` 必須有完整 TypeScript 型別（無 `any`）
- 時間戳使用 `new Date().toISOString()` 或手動格式化
- 父子掛載順序應正確顯示：`父 onBeforeMount → 子 onMounted → 父 onMounted`
- `watchEffect` 的 `onCleanup` 需要在日誌中可見
- 元件卸載時，`useLifecycleLog` 的清理動作（如有）要在 `onUnmounted` 中執行

#### 評分標準（100 分）

| 項目 | 分數 | 說明 |
|------|:----:|------|
| 生命週期日誌完整性（6 個 hooks 均記錄）| 25 | 每缺一個 hook -4 分 |
| 父子掛載順序正確 | 20 | 應為：父 onBeforeMount → 子 onMounted → 父 onMounted |
| `watchEffect` + `onCleanup` 整合 | 20 | 需在日誌中可見 onCleanup 觸發時機 |
| TypeScript 型別完整性 | 20 | `LogEntry` 型別、Composable 回傳型別、無 `any` |
| UX 清晰度（時間戳格式、來源標記、互動按鈕）| 15 | 日誌可讀性、顏色區分 |

---

## 自我檢核問題

完成學習後，嘗試不看文件回答以下問題：

**Q1：為什麼 `watch(tasks, saveFn, { immediate: true })` 不能用來「初次讀取 localStorage」？**

> 參考答案：`immediate: true` 讓 handler 以「初始值」立刻執行一次。若 `tasks` 初始值是空陣列 `[]`，handler 會立刻把空陣列寫進 localStorage，**覆蓋掉之前存的資料**。正確做法是 `onMounted` 負責讀（DOM 就緒後，以 storage 資料初始化 state），`watch` 負責寫（state 有變化就存）——方向相反，職責對稱，不可混用。

**Q2：在 `onUpdated` 中直接寫 `count.value++` 會發生什麼事？**

> 參考答案：無限循環。`onUpdated` 在 DOM 更新後執行，修改 `count.value` 又觸發更新，更新又執行 `onUpdated`，進入死循環。如果必須在更新後操作，應加條件守衛，或改用 `watchEffect` 並自己管理觸發條件。

**Q3：父元件和子元件的 `onMounted` 誰先執行？**

> 參考答案：**子元件先**。Vue 的掛載是深度優先（depth-first）：Vue 先完整渲染子樹，每個子元件都掛載完成後，父元件的 `onMounted` 才執行。順序：`父 onBeforeMount → 子 onBeforeMount → 子 onMounted → 父 onMounted`。

**Q4：為什麼 Composable 中的 Lifecycle Hooks 必須在 `setup()` 的「任何 await 之前」同步呼叫？**

> 參考答案：Vue 用 Active Instance 機制追蹤「現在哪個元件正在初始化」。`setup()` 執行時 Vue 設定 Active Instance；遇到 `await` 後，同步執行流中斷，Active Instance 清空。若 Composable 在 `await` 後才被呼叫，`onMounted` 等 hooks 找不到要綁定的元件，會被忽略（開發模式會報警告）。

**Q5：`onBeforeUnmount` 和 `onUnmounted` 分別適合做什麼清理？**

> 參考答案：`onBeforeUnmount` 在元件卸載前執行，此時子元件、DOM、資料都還完整——適合做「需要讀取現有狀態才能清理」的工作（如儲存最後狀態）。`onUnmounted` 在所有子元件都已卸載後執行——適合「不依賴元件資料」的清理（clearInterval、abort fetch、removeEventListener），這是大多數副作用清理的正確位置。

---

## 明日預告（W4 Day 2：2026-05-13）

**主題**：`watchEffect` 深化 × `onCleanup` × `flush: 'post'` × Race Condition 處理模式

**帶著這個問題進入明天**：
> `watchEffect` 和 `watch` 都可以處理副作用，但它們的依賴追蹤方式根本不同——`watchEffect` 是「自動追蹤」，`watch` 是「明確指定」。這個差異在什麼場景下會讓你選擇其中一個而非另一個？

W4 Day 2 會深入 `watchEffect` 的 `onCleanup` 模式（已在 W3 Day 3 預覽過），
並透過實際的 fetch + AbortController 案例，把「防 Race Condition」從概念變成熟練的實作模式。
