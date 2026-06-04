# W6 Push + PR 確認 × W7 Composable 設計深度預習

> **日期**：2026-05-31（M2-W6 Day 6）
> **週次**：W6（5/26–6/1），截止 **明日 6/1**，今日為最後行動視窗
> **學習階段**：M2 Composition API 深化期

---

## ⚠️ 作業堆積警示

| 週次 | 截止日 | 逾期天數 | 作業項目 |
|------|--------|---------|---------|
| W3 | 2026-05-11 | 20 天 | TaskList.vue v2 |
| W4 | 2026-05-18 | 13 天 | LifecycleLogger.vue |
| W5 | 2026-05-25 | 6 天 | CustomInput + CustomSelect + LoginForm |

**🔴 W6 截止日（6/1）剩 1 天** — 今日（Day 6）必須完成 `git push --force-with-lease` + GitHub PR Open。若今日未推送，明日（截止日）將處於非常緊迫的狀態。

---

## 一、目標技術 & 核心知識點

| 目標技術 | 核心知識點 |
|---------|-----------|
| W6 行動 | `git push --force-with-lease` × GitHub PR Open × CI 確認 |
| W7 預習：Composable 設計原則 | 抽取時機三訊號 × 單一職責邊界 × 回傳介面設計 |
| W7 預習：Composable 模式庫 | useAsyncData / useLocalStorage / useForm / usePagination |
| W7 預習：Composable vs Pinia 邊界 | 局部 vs 全域狀態的明確分界 |
| W7 預習：Composable 可測試性 | 與元件解耦後的單元測試結構 |

---

## 二、為什麼學這個（與前幾日的連結）

```
W5（5/19–5/25）：useLoginForm — 第一個完整業務 Composable
    把 reactive form + validate + reset 封裝為可複用 Hook
        ↓
W6（5/26–5/31）：usePanelTheme — 從元件內部提取 Composable
    抽取時機訊號：三個元件有相同的 inject+computed 樣板
        ↓
W7（6/2–6/8）：Composable 設計系統化
    不再只是「能提取就提取」，而是建立設計決策框架
    作業目標：設計 3 個不同場景的 Composable
```

W6 的 `usePanelTheme` 已經讓你體驗了「看到重複 → 提取 → 精確型別」的完整流程。W7 的目標是把這個直覺系統化，建立可重複使用的設計思維。

---

## 三、知識說明

### 3a. W6 Push + PR 行動流程

#### Step 1：確認本地 commit 序列

```bash
# 確認 W6 的 commit 序列是否整齊
git log --oneline -10

# 預期應看到類似：
# abc1234 chore(w6): add PanelThemeResult interface
# def5678 refactor(w6): extract usePanelTheme composable
# ghi9012 feat(w6): implement FormPanel, ListPanel, ChartPanel
# jkl3456 feat(w6): implement PanelCard with named slots
# mno7890 feat(w6): scaffold DashboardLayout with provide and KeepAlive
# pqr1234 feat(w6): add injectionKeys.ts (InjectionKey<PanelTheme>)
```

#### Step 2：推送到遠端

```bash
# 若已做過 rebase，需要 force-with-lease
git push origin <branch-name> --force-with-lease

# --force-with-lease 的安全保障：
# 若遠端有未知 commit（其他人 push 過），指令會中止
# 只有確認遠端狀態與你預期一致時才覆蓋
```

#### Step 3：開 PR

- **PR 標題**：`feat(w6): provide/inject × Slots × KeepAlive × usePanelTheme`
- **PR 說明結構**（昨日定稿的三個 Why）：
  1. Why Symbol InjectionKey（vs 字串 key）
  2. Why `generic="T"` Scoped Slot（vs `unknown[]`）
  3. Why 提取 `usePanelTheme`（vs 三個 Panel 各自 inject）
- **確認 CI 通過**：`vue-tsc --noEmit` 應在 CI 中零錯誤

---

### 3b. W7 預習：Composable 設計原則深化

#### 抽取時機的三個明確訊號

根據 W5（useLoginForm）× W6（usePanelTheme）的實際經驗，Composable 的提取時機可以系統化為：

| 訊號 | 具體表現 | W6 對應案例 |
|------|---------|------------|
| **重複樣板** | 3+ 個元件有相同的響應式初始化 + computed 組合 | 三個 Panel 都有 `inject + computed(themeClasses)` |
| **邏輯與 UI 混在一起** | `<script setup>` 超過 80 行，數據邏輯和渲染邏輯難以區分 | FormPanel 的 inject + CSS 查找邏輯本應獨立 |
| **跨元件複用需求** | 同一邏輯出現在兩個不相關的元件中 | 若另一個頁面的 Panel 也需要 theme inject |

> **關鍵判斷問題**：如果只有一個元件需要這段邏輯，且這段邏輯不複雜，**不要** 提取 Composable — 過早抽象增加維護成本。

#### Composable 回傳介面設計原則

```typescript
// 錯誤示範：回傳 reactive 物件後解構，失去響應性
function useBadDesign() {
  const state = reactive({ count: 0 })
  return state  // ❌ 使用端解構後 count 失去響應性
}

// 正確模式 1：回傳個別 ref（簡單 Composable）
function useCounter() {
  const count = ref(0)
  const increment = () => count.value++
  return { count, increment }  // ✅ ref 解構後維持響應性
}

// 正確模式 2：回傳 toRefs(state)（複雜狀態的 Composable）
function useForm() {
  const state = reactive({ name: '', email: '', errors: {} })
  // ...邏輯
  return { ...toRefs(state), validate, reset }  // ✅ toRefs 讓解構安全
}
```

#### 四種常見 Composable 設計模式

**模式 1：useAsyncData（非同步資料獲取）**

```typescript
// 封裝：loading + data + error 三態管理 + AbortController
function useAsyncData<T>(fetcher: (signal: AbortSignal) => Promise<T>) {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref<Error | null>(null)

  const execute = async () => {
    const controller = new AbortController()
    loading.value = true
    error.value = null
    
    try {
      data.value = await fetcher(controller.signal)
    } catch (e) {
      if (!(e instanceof DOMException && e.name === 'AbortError')) {
        error.value = e as Error
      }
    } finally {
      loading.value = false
    }

    return () => controller.abort()  // 回傳清理函式
  }

  return { data, loading, error, execute }
}
```

**模式 2：useLocalStorage（持久化封裝）**

```typescript
// 封裝：讀取 + 寫入 + JSON 安全解析（結合 W3 的持久化回路）
function useLocalStorage<T>(key: string, defaultValue: T) {
  const stored = localStorage.getItem(key)
  const initial = stored ? (JSON.parse(stored) as T) : defaultValue
  
  const value = ref<T>(initial)
  
  watch(value, (newVal) => {
    localStorage.setItem(key, JSON.stringify(newVal))
  }, { deep: true })
  
  return { value }
}
```

**模式 3：usePagination（分頁邏輯）**

```typescript
// 封裝：頁碼管理 + 分頁計算
function usePagination(totalItems: Ref<number>, pageSize = 10) {
  const currentPage = ref(1)
  
  const totalPages = computed(() => Math.ceil(totalItems.value / pageSize))
  const startIndex = computed(() => (currentPage.value - 1) * pageSize)
  const endIndex = computed(() => Math.min(startIndex.value + pageSize, totalItems.value))
  
  const goToPage = (page: number) => {
    if (page >= 1 && page <= totalPages.value) currentPage.value = page
  }
  const nextPage = () => goToPage(currentPage.value + 1)
  const prevPage = () => goToPage(currentPage.value - 1)
  
  return {
    currentPage: readonly(currentPage),
    totalPages,
    startIndex,
    endIndex,
    goToPage,
    nextPage,
    prevPage
  }
}
```

**模式 4：useEventListener（事件監聽管理）**

```typescript
// 封裝：addEventListener + removeEventListener + onUnmounted 自動清理
function useEventListener<K extends keyof WindowEventMap>(
  target: Ref<HTMLElement | null> | Window,
  event: K,
  handler: (e: WindowEventMap[K]) => void
) {
  onMounted(() => {
    const el = 'value' in target ? target.value : target
    el?.addEventListener(event, handler as EventListener)
  })

  onUnmounted(() => {
    const el = 'value' in target ? target.value : target
    el?.removeEventListener(event, handler as EventListener)
  })
}
```

#### Composable vs Pinia 邊界

| 面向 | Composable（useXxx）| Pinia Store |
|------|--------------------|-----------  |
| 作用範圍 | 元件樹局部（使用它的元件及其子代）| 全應用（任意元件都能 inject）|
| 實例數量 | 每次 `use` 都是新實例 | 單一實例（Singleton）|
| 適用場景 | UI 邏輯封裝、局部狀態管理 | 跨頁面共享狀態（購物車、使用者資訊）|
| 持久化 | 通常靠外部（如 useLocalStorage）| Store 可直接整合 persistence plugin |
| 與 provide/inject 的關係 | 可結合（W6 的 usePanelTheme 就是 inject + Composable）| 不需要 provide/inject，直接全域取用 |

> **決策規則**：「如果這個狀態需要在路由之間保持，用 Pinia；如果只在某個元件樹的生命週期內有效，用 Composable。」

---

## 四、作業說明（W7 預告）

### W7 作業：設計 3 個 Composable（截止 6/8）

根據 README，W7 的預期交付是「3 個 Composable 實作」。以下是建議的三個題目方向（正式作業說明將在 W7 Day 1 發出）：

| 編號 | Composable 名稱 | 封裝的核心邏輯 | 用到的技術 |
|------|----------------|--------------|-----------|
| 1 | `useAsyncData<T>` | 非同步請求的三態管理 + AbortController | ref, watch, onUnmounted |
| 2 | `useLocalStorage<T>` | 持久化讀寫 + 型別安全 | ref, watch, JSON 解析 |
| 3 | `usePagination` | 分頁計算 + 頁碼導航 | ref, computed, readonly |

> 這三個 Composable 涵蓋「非同步、持久化、計算邏輯」三種典型封裝場景，完成後即具備設計任意業務 Composable 的能力基礎。

---

## 五、自我檢核問題

1. **提取時機**：你負責的功能有三個元件都在做相同的「fetch API + loading/error 狀態管理」，你什麼時候應該提取成 Composable？判斷標準是什麼？

2. **回傳設計**：下面兩種寫法哪個更好？為什麼？
   ```typescript
   // 方案 A
   return reactive({ count, increment })
   // 方案 B
   return { count, increment }
   ```

3. **Composable vs Pinia**：使用者登入後的 `userProfile`（名稱、頭像、權限）應該放在 Composable 還是 Pinia Store？理由？

4. **可測試性**：為什麼把邏輯提取成 Composable 後，單元測試會更容易撰寫？

5. **副作用清理**：一個 `useInterval(callback, ms)` Composable 需要在哪裡清理 setInterval？如果這個 Composable 是在 Composable 中呼叫的（非 setup 直接呼叫），有什麼需要特別注意的？

---

## 六、明日預告（W6 Day 7 = 截止日）

**明日（6/1）是 W6 截止日**，主要工作：
- 確認今日 PR 是否已 Open（若未完成，今晚必須執行推送）
- W6 完整七天知識閉環總結（與 W5 截止日格式相同）
- 確認 CI 通過 + 回應可能的 PR review 意見
- W7 正式啟動預備：設計 `useAsyncData` 的 API 草稿

> 若今日 Push + PR 已完成，明日（截止日）將是輕鬆的知識整合日。若未完成，明日將是高壓的緊急補救日。**今日行動決定明日的學習品質。**
