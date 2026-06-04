# watchEffect 深化與清理模式

> **日期**：2026-05-13（W4 Day 2）
> **週次**：W4（5/12–5/18）— Vue 3: Lifecycle Hooks × watchEffect × toRefs
> **前置知識**：W4 Day 1 完整生命週期時間軸、onUnmounted 清理配對模式、watch vs onMounted 選用決策
> **帶著這個問題進入今天**：`watchEffect` 和 `watch` 都能監聽響應式資料並執行副作用。當你需要「自動追蹤多個依賴、且這些依賴在執行中才知道」時，`watchEffect` 比 `watch` 更適合——但為什麼它的 `onCleanup` 比 `watch` 的 `onCleanup` 更容易被觸發？它的清理時機與 `onUnmounted` 有何配合關係？

---

## 目標技術

| 技術 | 今日重點 |
|------|---------|
| `watchEffect` | 自動依賴追蹤原理、與 `watch` 的核心差異、執行時機 |
| `onCleanup`（watchEffect 版）| 清理函式的觸發時機、與 watch 的 onCleanup 對比 |
| `flush: 'post'` | DOM 更新後執行的場景、與 `nextTick` 的關係 |
| Race Condition 防禦 | AbortController + onCleanup 的標準防禦模式 |
| `watchEffect` × Composable | 在 Composable 中封裝 watchEffect 的責任邊界 |

---

## 為什麼學這個

W4 Day 1 完整梳理了生命週期 Hooks，解決了 W3 遺留的「`watch { immediate }` vs `onMounted`」懸案。

但 W4 主題還有另一個主角：`watchEffect`。

W3 已在實作中接觸過 `watchEffect` 的概念，但沒有深入：
- 在 5/3 的預習中，了解到 `watchEffect` **早於 `onMounted`** 執行
- 在 5/5 的底層原理中，了解到 `watchEffect` 的**依賴追蹤邊界在首個 `await` 之前**

今天的目的是把這些碎片整合成可以在實作中**主動選用 `watchEffect`** 的完整理解：
- 什麼時候選 `watchEffect` 而不是 `watch`？
- 當 `watchEffect` 的副作用需要清理時，如何正確配對？
- 如何用 `flush: 'post'` 解決「在 watchEffect 中讀取 DOM 尺寸」的問題？
- 如何防止非同步 `watchEffect` 的 Race Condition？

W4 的最終作業是 **LifecycleLogger.vue**——一個整合生命週期觀察與 `watchEffect` 追蹤的工具元件。今天的知識將直接應用在作業的核心邏輯中。

---

## 核心知識點

### 1. watchEffect vs watch：核心差異表

| 維度 | `watch` | `watchEffect` |
|------|---------|---------------|
| 依賴指定方式 | **明確**：`watch(source, fn)` | **隱含**：自動追蹤 fn 執行時存取的所有響應式資料 |
| 初次執行時機 | 預設不立即執行（需 `{ immediate: true }`）| **立即執行**（建立後就執行一次）|
| 能否取得舊值 | ✅ `watch(src, (newVal, oldVal) => {})` | ❌ 無法取得舊值 |
| 適用場景 | 需要「比較新舊值」或「明確知道要監聽哪個資料」| 需要「自動追蹤多個依賴」或「不需要舊值」的副作用 |
| 依賴追蹤邊界 | 不適用（明確指定）| **首個 `await` 之前**的同步程式碼（見下方說明）|

**依賴追蹤邊界的重要性**：

```ts
// ❌ 陷阱：searchQuery 在 await 之後才被存取，不會被追蹤
watchEffect(async () => {
  const data = await fetchSomething()  // await 之後...
  console.log(searchQuery.value)       // ...這裡存取的依賴不會被追蹤！
})

// ✅ 正確：在 await 之前讀取需要追蹤的響應式資料
watchEffect(async () => {
  const query = searchQuery.value      // ← 在 await 之前讀取，會被追蹤
  const data = await fetch(`/api?q=${query}`)
  result.value = data
})
```

---

### 2. watchEffect 的 onCleanup

`watchEffect` 的清理函式透過**回呼參數**傳入，而不是透過回傳值：

```ts
watchEffect((onCleanup) => {
  const timer = setInterval(() => {
    count.value++
  }, 1000)

  // 這個函式在「下次執行前」或「元件卸載時」被呼叫
  onCleanup(() => {
    clearInterval(timer)
  })
})
```

**觸發時機比較**：

| 事件 | `watchEffect` 的 `onCleanup` | `watch` 的 `onCleanup` |
|------|------------------------------|------------------------|
| 依賴改變，下次執行前 | ✅ **每次都觸發**（因為 watchEffect 每次依賴改變就重跑）| ✅ 每次都觸發 |
| 元件卸載（onUnmounted）| ✅ 觸發 | ✅ 觸發 |
| 首次執行前 | ❌ 不觸發（還沒有上一次執行）| ❌ 不觸發 |

**為什麼 `watchEffect` 的 `onCleanup` 更容易被觸發？**

因為 `watchEffect` 自動追蹤所有存取到的響應式資料，即使是在不同的程式路徑中被存取的依賴。只要任何一個被追蹤的響應式資料改變，`watchEffect` 就會重跑——而每次重跑之前，前一次的 `onCleanup` 就會被呼叫。

相比 `watch`，它的觸發來源更廣，所以 cleanup 被觸發的頻率通常也更高。

---

### 3. Race Condition 防禦：AbortController + onCleanup

Race Condition 的場景：

```ts
// ❌ 有 Race Condition 的版本
watchEffect(async () => {
  const query = searchQuery.value
  const data = await fetch(`/api/search?q=${query}`)
  //             ↑ 如果這個請求很慢，而在它回來之前 searchQuery 又改變了
  //               第二個請求可能先回來，然後第一個慢請求覆蓋了結果！
  searchResults.value = data
})
```

**標準防禦模式**：

```ts
// ✅ AbortController + onCleanup 防禦 Race Condition
watchEffect(async (onCleanup) => {
  const query = searchQuery.value
  const controller = new AbortController()

  // 在「下次執行前」取消尚未完成的請求
  onCleanup(() => controller.abort())

  try {
    const response = await fetch(`/api/search?q=${query}`, {
      signal: controller.signal
    })
    searchResults.value = await response.json()
  } catch (err) {
    if (err.name !== 'AbortError') {
      // AbortError 是預期行為，其他錯誤才需要處理
      console.error(err)
    }
  }
})
```

**執行流程圖**：

```
searchQuery 改變（第一次）
  → watchEffect 第一次執行
  → 建立 controller1，發送 request1
  
searchQuery 改變（第二次，request1 尚未回來）
  → 觸發 onCleanup：controller1.abort() ← request1 被取消！
  → watchEffect 第二次執行
  → 建立 controller2，發送 request2
  
request2 回來
  → searchResults 更新為最新結果 ✅
  （request1 已被中止，不會有舊資料覆蓋問題）
```

---

### 4. flush 選項：執行時機的精確控制

`watchEffect` 和 `watch` 都接受 `flush` 選項：

```ts
watchEffect(fn, { flush: 'pre' })   // 預設：DOM 更新前執行
watchEffect(fn, { flush: 'post' })  // DOM 更新後執行
watchEffect(fn, { flush: 'sync' })  // 同步立即執行（謹慎使用）
```

**什麼時候需要 `flush: 'post'`？**

當你的 `watchEffect` 需要**讀取更新後的 DOM 狀態**時：

```ts
const listRef = ref<HTMLElement | null>(null)
const items = ref(['a', 'b', 'c'])

// ❌ 預設 flush: 'pre'：DOM 尚未更新，scrollHeight 是舊值
watchEffect(() => {
  console.log(items.value.length)        // 追蹤這個依賴
  console.log(listRef.value?.scrollHeight)  // 此時 DOM 還沒更新！
})

// ✅ flush: 'post'：DOM 更新後才執行，拿到正確的 scrollHeight
watchEffect(() => {
  console.log(items.value.length)
  console.log(listRef.value?.scrollHeight)  // DOM 已更新 ✅
}, { flush: 'post' })
```

**`watchPostEffect` 是語法糖**：

```ts
// 以下兩種寫法等價
watchEffect(fn, { flush: 'post' })
watchPostEffect(fn)
```

**與 `nextTick` 的關係**：

`flush: 'post'` 等同於「在 `nextTick` 之後執行」。如果你發現自己在 `watchEffect` 中使用了 `await nextTick()`，通常應該改為 `flush: 'post'`。

---

### 5. watchEffect × Composable：封裝副作用的責任邊界

在 Composable 中使用 `watchEffect` 時，有一個重要的規則：

```ts
// ✅ 正確：watchEffect 在 setup() 同步呼叫流中，Active Instance 正確
export function useSearchResults(query: Ref<string>) {
  const results = ref<SearchResult[]>([])
  const loading = ref(false)

  watchEffect(async (onCleanup) => {
    const q = query.value
    if (!q) {
      results.value = []
      return
    }

    loading.value = true
    const controller = new AbortController()
    onCleanup(() => {
      controller.abort()
      loading.value = false
    })

    try {
      const response = await fetch(`/api/search?q=${q}`, {
        signal: controller.signal
      })
      results.value = await response.json()
    } catch (err) {
      if (err.name !== 'AbortError') throw err
    } finally {
      loading.value = false
    }
  })

  return { results, loading }
}
```

**關鍵原則**：
- Composable 內的 `watchEffect` 在元件 `setup()` 呼叫 Composable 時建立
- 元件卸載時，Vue 自動停止這個 `watchEffect`（因為 Active Instance 綁定了）
- 不需要手動呼叫 `stop()`——`onUnmounted` 的自動清理已處理

**需要手動停止的場景**（少見）：

```ts
// 在 setup 之外（如條件式啟動）才需要手動 stop
const stopEffect = watchEffect(() => {
  // ...
})

// 某個條件達成後手動停止
function handleComplete() {
  stopEffect()
}
```

---

## 作業說明：LifecycleLogger.vue —— watchEffect 追蹤器模組

### 情境背景

W4 最終作業是建立一個 **生命週期觀察工具（LifecycleLogger.vue）**。

今天的任務：完成作業中的 **watchEffect 追蹤器**模組——這是整個工具的核心功能之一，讓使用者可以觀察 `watchEffect` 的觸發時機與 cleanup 執行次數。

### 功能需求

1. **自動追蹤模式**：`watchEffect` 自動追蹤傳入的 `trackedValue`（任意響應式資料），每次觸發時記錄一筆 log
2. **清理計數器**：每次 `onCleanup` 被呼叫時，`cleanupCount` +1
3. **非同步模擬**：可選擇開啟「非同步模式」，模擬 fetch + AbortController 防禦模式
4. **flush 選項**：UI 上可切換 `pre` / `post`，觀察執行時機差異

### 技術規範

```ts
// useWatchEffectLogger.ts
interface WatchEffectLog {
  timestamp: number
  triggerSource: string   // 'initial' | 'reactive-change'
  cleanupTriggered: boolean
}

export function useWatchEffectLogger(trackedValue: Ref<unknown>) {
  const logs = ref<WatchEffectLog[]>([])
  const cleanupCount = ref(0)
  const isAsync = ref(false)
  const flushMode = ref<'pre' | 'post'>('pre')

  // 實作 watchEffect，自動追蹤 trackedValue
  // 每次觸發時 push 一筆 log
  // onCleanup 時 cleanupCount++
  // 支援切換 flush 模式（需要停止舊的 watchEffect，建立新的）

  return {
    logs,
    cleanupCount,
    isAsync,
    flushMode
  }
}
```

### 評分標準

| 項目 | 分值 | 說明 |
|------|:---:|------|
| watchEffect 正確追蹤依賴（trackedValue 改變時觸發）| 25 | 不能用 watch，必須是 watchEffect |
| onCleanup 正確計數（每次重跑前 +1）| 20 | 確認清理時機正確 |
| AbortController 防禦模式（非同步開啟時）| 20 | race condition 不出現 |
| flush 模式切換功能（停止舊 effect，建立新 effect）| 20 | 動態切換時行為正確 |
| TypeScript 型別完整（無 any）| 15 | WatchEffectLog 介面清楚 |

---

## 自我檢核問題

1. `watchEffect` 和 `watch(source, fn, { immediate: true })` 都會立即執行一次，但它們的本質差異是什麼？在什麼場景下你會明確選擇 `watchEffect` 而非 `watch + immediate`？

2. 以下程式碼有 Race Condition 風險嗎？如何修正？
   ```ts
   watchEffect(async () => {
     const id = selectedId.value
     const data = await fetchUser(id)
     userProfile.value = data
   })
   ```

3. 當你需要在響應式資料改變後，**讀取更新後的 DOM 高度**，應該用哪個 flush 選項？為什麼不能用預設的 `pre`？

4. 你在 Composable 中建立了一個 `watchEffect`。當使用這個 Composable 的元件被卸載時，這個 `watchEffect` 會自動停止嗎？什麼條件下需要手動呼叫 `stop()`？

5. `onCleanup` 的觸發時機是什麼？`watchEffect` 的 `onCleanup` 為什麼比 `watch` 的 `onCleanup` 更「活躍」？

---

## 明日預告（W4 Day 3：2026-05-14）

**主題**：`toRefs` 深化 × `storeToRefs` 模式 × LifecycleLogger.vue 實作期啟動

**帶著這個問題進入明天**：
> 你已經在 W3 學過了 `toRefs` 的基本概念（Composable 標準回傳模式）。但在 W4 的情境下，有一個更進階的問題：當 Composable 的回傳值包含「函式」和「響應式資料」時，是否需要對整個回傳物件做 `toRefs`？有沒有什麼情況下，`toRefs` 反而是**多此一舉**甚至會造成問題的？
