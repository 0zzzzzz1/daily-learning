# useWatchEffectLogger.ts 完整實作與 flush 模式動態切換

> **日期**：2026-05-15（W4 Day 4）
> **週次**：W4（5/12–5/18）— Vue 3: Lifecycle Hooks × watchEffect × toRefs
> **前置知識**：W4 Day 2 watchEffect 深化 × onCleanup × Race Condition 防禦；W4 Day 3 toRefs 決策樹 × Composable 回傳物件設計 × LifecycleLogger.vue 骨架
> **帶著這個問題進入今天**：`useWatchEffectLogger` 需要支援「動態切換 flush 模式（pre / post）」——當使用者在 UI 切換 flush 選項時，你需要停止舊的 `watchEffect` 並建立新的。這個「停止與重建」的邏輯要放在哪裡？是在 Composable 裡用 `watch` 監聽 `flushMode` 的變化，還是由使用端重新呼叫 Composable？

---

## 目標技術

| 技術 | 今日重點 |
|------|---------|
| `watchEffect` | 動態停止（`WatchStopHandle`）與重建流程 |
| `watch` + `watchEffect` 協同 | 用 `watch` 監聽設定變化，觸發 effect 重建 |
| `onUnmounted` 手動清理 | setup 之外建立的 effect 需手動停止 |
| Race Condition 防禦 | `AbortController` + `onCleanup` 非同步模式實作 |
| Composable 封裝設計 | 對外介面乾淨（使用端只改 `flushMode.value`）|

---

## 為什麼學這個

W4 Day 2 學了 `watchEffect` 的深化原理：
- `onCleanup` 每次重跑前觸發（不只是卸載）
- `AbortController` 防禦 Race Condition 的標準模式
- `flush: 'post'` 用於讀取更新後 DOM 狀態

W4 Day 3 解決了 Composable 回傳設計的最後一個疑問：
- `toRefs` 的使用時機（reactive 物件 + 需要解構屬性）
- 函式不需要 `toRef`
- `hookCounts`（reactive 整體回傳）不需要 `toRefs`

今天是 **LifecycleLogger.vue 實作的核心日**：完成 `useWatchEffectLogger.ts` 的完整實作。

這個作業的關鍵設計挑戰正是昨天帶入的問題：**flush 模式動態切換**。

---

## 核心問題解答：flush 模式動態切換要怎麼實作？

### 為什麼這個問題有趣？

`watchEffect` 的 `flush` 選項在建立時就固定了——沒有辦法事後修改。所以「動態切換 flush 模式」本質上就是「停止舊 effect、用新 flush 設定建立新 effect」。

問題是：**這個「停止與重建」的邏輯要放在哪裡？**

### 方案 A：由使用端控制（不推薦）

```ts
// 使用端（LifecycleLogger.vue）
const { logs, flushMode } = useWatchEffectLogger(trackedValue)

// 當 flushMode 改變時，重新呼叫整個 Composable？
watch(flushMode, () => {
  // ❌ 這樣做會遺失之前的 logs，使用端也不知道要怎麼重啟
})
```

**問題**：邏輯洩漏到使用端，破壞了 Composable 的封裝性。使用端不應該知道「切換 flush 模式需要重建 watchEffect」這種實作細節。

### 方案 B：在 Composable 內用 `watch` 監聽 flushMode（推薦）

```ts
export function useWatchEffectLogger(trackedValue: Ref<unknown>) {
  const flushMode = ref<'pre' | 'post'>('pre')

  // 內部維護一個「停止函式」的參考
  let stopEffect: WatchStopHandle | null = null

  function buildEffect() {
    if (stopEffect) stopEffect()   // 停止舊的 effect
    stopEffect = watchEffect(       // 用新的 flush 設定重建
      (onCleanup) => { /* ... */ },
      { flush: flushMode.value }
    )
  }

  buildEffect()  // 初始建立（在 setup 同步流中 → 自動綁定 Active Instance）

  // 監聽 flushMode 變化，自動重建 effect
  watch(flushMode, () => {
    buildEffect()  // 在 watch callback 中建立的 effect 需要手動清理
  })

  // 確保元件卸載時清理最後一個 effect（watch callback 中建立的不會自動停止）
  onUnmounted(() => {
    stopEffect?.()
  })

  return { logs, cleanupCount, isAsync, flushMode }
}
```

**優點**：
- 使用端只需改 `flushMode.value` → 自動重建，介面乾淨
- 所有重建邏輯封裝在 Composable 內部
- `onUnmounted` 確保清理，不洩漏記憶體

---

## 關鍵設計問題：Active Instance 與手動清理

### 為什麼初始 `buildEffect()` 不需要手動停止？

初始 `buildEffect()` 在 `setup()` 同步呼叫流中執行，Vue 的 Active Instance 機制確保這個 `watchEffect` 在元件卸載時自動停止。

### 為什麼 `watch(flushMode, buildEffect)` 中建立的 `watchEffect` 需要手動停止？

`watch` 的 callback 是在**響應式改變後非同步執行**的，此時已不在 `setup()` 的同步呼叫流中。在 callback 中呼叫 `buildEffect()` 建立的 `watchEffect` 沒有 Active Instance 綁定，**不會自動停止**。

```
setup() 同步呼叫流  ← Active Instance 存在，watchEffect 自動綁定
  ↓ buildEffect()    ← 初始建立，自動綁定 ✅
  ↓ watch(flushMode, callback)  ← callback 稍後非同步執行
  
flushMode.value 改變（非同步）
  ↓ callback 執行  ← 此時 Active Instance 已不存在
  ↓ buildEffect()    ← 新建 watchEffect 無自動綁定 ⚠️
```

**解決方案**：在 `setup()` 同步流中（即 Composable 初始化時）登錄 `onUnmounted`，它會在元件卸載時清理最後持有的 `stopEffect` 參考，無論它是在哪裡建立的。

```ts
// 在 Composable 初始化時（setup 同步流）登錄清理
onUnmounted(() => {
  stopEffect?.()  // 清理當前持有的任何 effect
})
```

---

## 完整實作：useWatchEffectLogger.ts

```ts
import { ref, watch, watchEffect, onUnmounted } from 'vue'
import type { Ref, WatchStopHandle } from 'vue'

// 型別定義
interface WatchEffectLog {
  id: number
  timestamp: number
  triggerSource: 'initial' | 'reactive-change'
  cleanupTriggered: boolean  // 此次執行前，上一次的 cleanup 是否有被呼叫？
  flushMode: 'pre' | 'post'  // 此次執行時的 flush 模式（記錄用）
}

export function useWatchEffectLogger(trackedValue: Ref<unknown>) {
  const logs = ref<WatchEffectLog[]>([])
  const cleanupCount = ref(0)
  const isAsync = ref(false)
  const flushMode = ref<'pre' | 'post'>('pre')

  // 跨執行輪次的狀態
  let stopEffect: WatchStopHandle | null = null
  let wasCleanedUp = false  // 上一次 effect 執行的 cleanup 是否有被觸發
  let logId = 0

  function buildEffect(): void {
    // 停止並清理舊的 effect
    if (stopEffect) {
      stopEffect()
      stopEffect = null
    }
    wasCleanedUp = false  // 重建時重置 cleanup 狀態

    const currentFlush = flushMode.value  // 快取當前 flush 模式

    stopEffect = watchEffect(
      async (onCleanup) => {
        // ⚠️ 在首個 await 之前讀取所有需要追蹤的依賴
        const _trackedVal = trackedValue.value  // 追蹤 trackedValue
        const triggerSource: WatchEffectLog['triggerSource'] =
          logs.value.length === 0 ? 'initial' : 'reactive-change'
        const cleanupTriggered = wasCleanedUp
        wasCleanedUp = false  // 重置，為下一次執行準備

        if (isAsync.value) {
          // ── 非同步模式：模擬 fetch + AbortController 防禦 Race Condition ──
          const controller = new AbortController()

          onCleanup(() => {
            cleanupCount.value++
            wasCleanedUp = true
            controller.abort()  // 取消尚未完成的模擬請求
          })

          try {
            // 模擬一個 100ms 的非同步操作（可被 AbortController 取消）
            await new Promise<void>((resolve, reject) => {
              const timer = setTimeout(resolve, 100)
              controller.signal.addEventListener('abort', () => {
                clearTimeout(timer)
                reject(new DOMException('Aborted', 'AbortError'))
              })
            })
          } catch (err) {
            if ((err as DOMException).name !== 'AbortError') throw err
            return  // AbortError 是預期行為，直接返回，不記錄 log
          }
        } else {
          // ── 同步模式：直接記錄 cleanup 次數 ──
          onCleanup(() => {
            cleanupCount.value++
            wasCleanedUp = true
          })
        }

        // 記錄這次執行的 log
        logs.value.push({
          id: logId++,
          timestamp: Date.now(),
          triggerSource,
          cleanupTriggered,
          flushMode: currentFlush
        })
      },
      { flush: currentFlush }  // 使用當前 flush 模式
    )
  }

  // ── 初始建立（在 setup 同步流中，Active Instance 自動綁定）──
  buildEffect()

  // ── 監聽 flushMode 變化 → 停止舊 effect，用新 flush 設定重建 ──
  watch(flushMode, () => {
    buildEffect()
    // 注意：此處建立的 watchEffect 不在 setup 同步流中
    // 由下方的 onUnmounted 負責最終清理
  })

  // ── 手動清理：確保元件卸載時清理最後持有的任何 effect ──
  onUnmounted(() => {
    if (stopEffect) {
      stopEffect()
      stopEffect = null
    }
  })

  return {
    logs,
    cleanupCount,
    isAsync,
    flushMode
  }
}
```

---

## 設計決策分析

### 為什麼用 `watch(flushMode, buildEffect)` 而不是在 `watchEffect` 內部讀取 `flushMode`？

```ts
// ❌ 錯誤的直覺做法
watchEffect(() => {
  const flush = flushMode.value  // 追蹤 flushMode
  // ...
  // 但 watchEffect 的 flush 選項是建立時就固定的
  // 在 callback 內讀取 flushMode 不會改變 flush 選項！
}, { flush: ??? })  // 這個選項在建立時已確定，無法動態改變
```

`watchEffect` 的 `flush` 選項決定的是「這個 watchEffect 本身何時執行」，而不是 callback 內部的邏輯。要換 flush 模式，**唯一的方式就是建立一個新的 watchEffect**。

### `wasCleanedUp` 的設計原理

`cleanupTriggered` 欄位記錄「此次執行前，前一次的 cleanup 是否有被呼叫」。

- 第一次執行（`triggerSource: 'initial'`）：沒有前一次，`cleanupTriggered: false`
- 後續執行：前一次的 `onCleanup` 在本次執行前必定已被呼叫，`cleanupTriggered: true`
- 非同步模式被中止（`AbortError`）：cleanup 已被呼叫，但不記錄 log → 下次執行時 `cleanupTriggered: true`

這個 flag 需要**跨執行輪次持久**，所以不能宣告在 `watchEffect` 的 callback 內部（每次執行都是新的作用域），而要宣告在 `buildEffect` 的外部閉包中。

---

## 作業說明：今日進度確認

### W4 作業整體架構回顧

```
LifecycleLogger.vue（W4 作業，截止 5/18）
├── useLifecycleLogger.ts      ✅ 骨架設計完成（5/14）
├── useWatchEffectLogger.ts    ✅ 完整實作（今日，5/15）
└── LifecycleLogger.vue        ⏳ 模板整合（明日，5/16）
```

### 今日完成：useWatchEffectLogger.ts

- [x] `watchEffect` 正確追蹤 `trackedValue`（在首個 await 前讀取）
- [x] `onCleanup` 正確計數（`cleanupCount++`，且 `wasCleanedUp` 跨輪次記錄）
- [x] 非同步模式：`AbortController` 防禦 Race Condition
- [x] `flush` 模式動態切換：`watch(flushMode, buildEffect)` + `onUnmounted` 手動清理
- [x] TypeScript 型別完整（`WatchEffectLog` 介面 × `WatchStopHandle` × `Ref<unknown>`）

### 評分標準預估

| 項目 | 分值 | 預估狀況 |
|------|:---:|---------|
| watchEffect 正確追蹤依賴 | 25 | ✅ 在 await 之前讀取 trackedValue |
| onCleanup 正確計數 | 20 | ✅ cleanupCount++ + wasCleanedUp 跨輪次 |
| AbortController 防禦模式 | 20 | ✅ AbortError 捕獲 + 不記錄 log |
| flush 模式切換功能 | 20 | ✅ watch + buildEffect + onUnmounted 清理 |
| TypeScript 型別完整 | 15 | ✅ 介面清楚，無 any |

**預估分數**：95/100（待瀏覽器實際驗證後確認）

---

## 自我檢核問題

1. 為什麼 `watchEffect` 的 `flush` 選項必須在建立時就確定，不能在 callback 內動態改變？動態切換 flush 的正確做法是什麼？

2. 以下程式碼有什麼問題？
   ```ts
   function buildEffect() {
     stopEffect = watchEffect(() => {
       const val = trackedValue.value
       // ...
     }, { flush: flushMode.value })
   }
   watch(flushMode, buildEffect)
   // 沒有 onUnmounted
   ```

3. `wasCleanedUp` 為什麼必須宣告在 `watchEffect` callback 的外部（閉包中），而不是在 callback 內部宣告？

4. 當 `isAsync` 被開啟後，使用者快速點擊「觸發 watchEffect」5 次，最終 `logs` 中應該有幾筆記錄？為什麼？（提示：非同步模式 + AbortController）

5. `onUnmounted` 中的清理邏輯（`stopEffect?.()` ）在什麼時候是「多餘的」（即 Vue 已自動處理），什麼時候是「必要的」（Vue 不會自動處理）？

---

## 明日預告（W4 Day 5：2026-05-16）

**主題**：`useLifecycleLogger.ts` 完整實作 × `LifecycleLogger.vue` 模板整合

**帶著這個問題進入明天**：
> `LifecycleLogger.vue` 需要整合兩個 Composable（`useLifecycleLogger` 和 `useWatchEffectLogger`）。模板中有一個設計細節：當使用者點擊「觸發強制更新」按鈕時，應該觸發 `onUpdated` 鉤子。要觸發 `onUpdated`，需要改變某個響應式資料讓 Vue 重新渲染。`triggerUpdate` 的值改變了，但它沒有在模板裡被渲染出來——Vue 會因為「有響應式資料改變但 DOM 實際沒有變化」而觸發 `onUpdated` 嗎？
