# Daily Learning — 每日前端學習計畫

> **目的**：透過系統化、持續性的每日學習，強化前端技術的理解深度與實作能力，並將學習成果直接反映在真實專案開發品質上。

---

## 目標技術


| 技術                       | 學習重點                       | 優先級    |
| ------------------------ | -------------------------- | ------ |
| JavaScript ES6+          | 現代語法、非同步處理、模組化             | ⭐⭐⭐ 最高 |
| Vue.js 3 Composition API | 響應式系統、元件設計、狀態管理            | ⭐⭐⭐ 最高 |
| TypeScript               | 型別系統、泛型、與 Vue 整合           | ⭐⭐⭐ 最高 |
| Tailwind CSS             | Utility-First 設計、RWD、元件樣式化 | ⭐⭐ 高   |
| Unit Test                | Vitest、Vue Test Utils、測試策略 | ⭐⭐ 高   |


---

## 學習曲線設計

以下為 6 個月的漸進式學習路線，依難度與依賴關係排序，前期打穩基礎，後期聚焦整合與實戰。

```
難度
  ▲
高  │                                          ┌──────────┐
    │                              ┌───────────┤ M6 整合   │
    │                   ┌──────────┤ M5 測試   │ 實戰強化  │
    │         ┌─────────┤ M4 TS 整合│           │           │
    │  ┌──────┤ M3 Vue  │           │           │           │
    │  │M1-M2 │ 深入    │           │           │           │
低  │  │基礎  │         │           │           │           │
    └──┴──────┴─────────┴───────────┴───────────┴───────────▶ 時間
       4-5月   5-6月     6-7月       7-8月       8-9月    9-10月
```

---

## 月度學習目標

### M1｜2026 年 4 月下旬 ～ 5 月（基礎建構期）

**主軸**：JavaScript ES6+ 現代語法 × Vue 3 響應式核心


| 週次            | 學習主題                                            | 預期產出     |
| ------------- | ----------------------------------------------- | -------- |
| W1（4/21–4/27） | JS：解構、展開、箭頭函式、Template Literal                  | 小型工具函式實作 |
| W2（4/28–5/4）  | JS：Promise、async/await、模組（ESM）                  | 非同步資料流封裝 |
| W3（5/5–5/11）  | Vue 3：`ref` / `reactive` / `computed` / `watch` | 任務清單元件   |
| W4（5/12–5/18） | Vue 3：Lifecycle Hooks、`watchEffect`、`toRefs`    | 生命週期觀察工具 |


**成效目標**：

- 能獨立撰寫 ES6+ 語法且無需查閱文件
- 能正確選用 `ref` / `reactive`，理解響應式陷阱
- 月底完成第一個具完整功能的 Vue 3 元件

---

### M2｜2026 年 5 月下旬 ～ 6 月（Composition API 深化期）

**主軸**：元件通訊、Composables、進階 API


| 週次            | 學習主題                                         | 預期產出              |
| ------------- | -------------------------------------------- | ----------------- |
| W5（5/19–5/25） | `defineProps` / `defineEmits` / `v-model` 進階 | 雙向綁定自訂元件          |
| W6（5/26–6/1）  | `provide` / `inject`、Slots、動態元件              | 可插拔 UI 元件庫雛形      |
| W7（6/2–6/8）   | Composable 設計（抽取邏輯、可複用 Hook）                 | 3 個 Composable 實作 |
| W8（6/9–6/15）  | Vue Router 整合（動態路由、導航守衛）                     | 多頁面 SPA 骨架        |


**成效目標**：

- 能設計可複用的 Composable，邏輯與 UI 解耦
- 能運用 Slots 建構彈性元件
- 熟悉 Vue Router 路由保護機制

---

### M3｜2026 年 6 月下旬 ～ 7 月（TypeScript 整合期）

**主軸**：TypeScript 型別系統 × Vue 3 型別安全


| 週次             | 學習主題                                | 預期產出       |
| -------------- | ----------------------------------- | ---------- |
| W9（6/16–6/22）  | TS 基礎：型別、介面、聯合型別、型別縮窄               | 型別化的資料模型   |
| W10（6/23–6/29） | TS 泛型：函式泛型、泛型介面、泛型約束                | 通用型 API 封裝 |
| W11（6/30–7/6）  | Vue + TS：型別化 Props、Emits、Composable | 重構 M2 元件   |
| W12（7/7–7/13）  | Vue + TS：defineComponent、型別推導最佳實踐   | 完整型別安全元件   |


**成效目標**：

- 能為 Vue 元件撰寫完整 TypeScript 型別定義
- 減少執行期錯誤，IDE 靜態分析零警告
- 理解 `as const`、型別守衛、工具型別（Partial、Required 等）

---

### M4｜2026 年 7 月下旬 ～ 8 月（Tailwind CSS 精通期）

**主軸**：Utility-First 樣式系統、響應式設計、主題化


| 週次             | 學習主題                               | 預期產出   |
| -------------- | ---------------------------------- | ------ |
| W13（7/14–7/20） | Tailwind 核心 Utility、Flexbox、Grid   | 常用版型實作 |
| W14（7/21–7/27） | RWD 前綴（sm/md/lg）、動態 class 綁定       | 響應式元件集 |
| W15（7/28–8/3）  | 自訂設定（tailwind.config）、Design Token | 統一設計系統 |
| W16（8/4–8/10）  | Dark Mode、動畫 `transition`、JIT 最佳化  | 深色模式元件 |


**成效目標**：

- 不再撰寫客製化 CSS，全面使用 Tailwind
- 能根據設計稿快速還原 UI，切版速度提升 50%
- 熟悉 `@apply` 與元件類提取策略

---

### M5｜2026 年 8 月下旬 ～ 9 月（測試建構期）

**主軸**：Unit Test 思維 × Vitest × Vue Test Utils


| 週次             | 學習主題                                       | 預期產出               |
| -------------- | ------------------------------------------ | ------------------ |
| W17（8/11–8/17） | 測試基礎：Vitest 設定、describe / it / expect      | 第一份測試套件            |
| W18（8/18–8/24） | Vue Test Utils：mount、findComponent、trigger | 元件行為測試             |
| W19（8/25–8/31） | Composable 測試、非同步測試、Mock                   | 10 個 Composable 測試 |
| W20（9/1–9/7）   | 覆蓋率報告、測試策略、TDD 實踐                          | 覆蓋率 ≥ 80%          |


**成效目標**：

- 能為現有元件補寫 Unit Test
- 建立 TDD 習慣，先寫測試再實作
- CI 整合測試，確保程式碼品質門檻

---

### M6｜2026 年 9 月下旬 ～ 10 月（整合實戰期）

**主軸**：Pinia 狀態管理、效能優化、最佳實踐整合


| 週次             | 學習主題                           | 預期產出     |
| -------------- | ------------------------------ | -------- |
| W21（9/8–9/14）  | Pinia：Store 設計、Getters、Actions | 全域狀態管理模組 |
| W22（9/15–9/21） | 效能：`v-memo`、`shallowRef`、懶加載   | 效能優化報告   |
| W23（9/22–9/28） | 程式碼品質：ESLint、Prettier、命名規範     | 代碼規範文件   |
| W24（9/29–10/5） | 綜合實作：仿 ihouseBMS 功能模組重構        | 成果展示作品   |


**成效目標**：

- 能獨立設計 Pinia Store 架構
- 具備效能問題排查能力
- 完成一個整合所有技術的功能模組

---

## 成效衡量指標

### 每週自評（作業提交後）


| 指標    | 說明                  | 目標       |
| ----- | ------------------- | -------- |
| 作業完成率 | 每週作業按時提交比例          | ≥ 90%    |
| 作業平均分 | 每份作業評分均值            | ≥ 75 分   |
| 概念理解度 | 能否以自己的話解釋本週核心知識     | 達到「教學」程度 |
| 實作獨立性 | 完成作業時查閱文件的次數（少 = 好） | 逐月下降     |


### 每月里程碑檢核


| 月份  | 里程碑                  | 驗收方式                   |
| --- | -------------------- | ---------------------- |
| M1  | 能獨立建立 Vue 3 響應式元件    | 實作任務清單（無參考）            |
| M2  | 能設計可複用 Composable    | Code Review            |
| M3  | 元件達到 TypeScript 型別安全 | 靜態分析零錯誤                |
| M4  | 全程以 Tailwind 完成 UI   | 切版速度評估                 |
| M5  | 關鍵模組測試覆蓋率 ≥ 80%      | Vitest Coverage Report |
| M6  | 完成整合作品並提交展示          | Demo + Code Review     |


### 長期能力提升追蹤

詳見 [能力地圖與學習對照.md](./能力地圖與學習對照.md)，每月更新一次個人技術能力地圖。

---

## 專案結構

```
daily-learning/
├── README.md              # 計畫說明（本文件）
├── 能力地圖與學習對照.md     # 技術能力地圖與學習對照
├── YYYY-MM-DD/            # 每日學習目錄
│   ├── 技術主題　　    　　　# 教材與作業（依主題命名文件）
│   └── 今日學習分析.md      # 當日學習後的技術理解更新
└── ...
```

---

## 學習紀錄


| 日期         | 核心主題                                        | 月份  | 作業狀態  | 評分  |
| ---------- | ------------------------------------------- | --- | ----- | --- |
| 2026-04-23 | Vue 3 響應式原理（Reactivity System）               | M1  | ⏳ 待提交 | —   |
| 2026-04-24 | 解構賦值、Spread 展開、Rest 其餘運算子                    | M1  | ⏳ 待提交 | —   |
| 2026-04-27 | 箭頭函式（Arrow Function）× 模板字串（Template Literal） | M1  | ⏳ 待提交 | —   |
| 2026-04-28 | Promise、async/await、ESM 模組系統                 | M1  | ⏳ 待提交 | —   |
| 2026-04-29 | W2 實作工坊：非同步資料流封裝模組（async-utils）         | M1  | ⏳ 待提交 | —   |
| 2026-04-30 | Vue 3 整合 async-utils × 房源列表頁面 × Skeleton UI  | M1  | ⏳ 待提交 | —   |
| 2026-05-01 | W1–W2 作業補交日：TaskList.vue × 工具函式庫 × 格式化工具庫實作 | M1 | 🔄 PR 準備中 | — |
| 2026-05-02 | W2 PR 整理（async-utils）× W3 預習：ref/reactive/computed/watch 深入 | M1 | 🔄 W2 PR 今日發出 | — |
| 2026-05-03 | W2 截止前確認 × W3 預習深化：Lifecycle Hooks 完整梳理 × watchEffect 整合模式 | M1 | ⏳ 確認 PR 狀態（截止明日 5/4）| — |
| 2026-05-04 | W2 截止日確認 × W3 銜接準備：W2 知識總結 × 4 份 PR 狀態確認 | M1 | 🔲 請登入 GitHub 手動確認 PR | — |
| 2026-05-05 | W3 Day 1：ref / reactive / computed / watch 深入精進（底層原理、dirty flag、Effect Graph）| M1 | ⏳ 進行中 | — |
| 2026-05-06 | W3 Day 2：ref auto-unwrap × toRef / toRefs × shallowRef / shallowReactive × markRaw（響應式參考的傳遞與保存）| M1 | ⏳ 進行中 | — |
| 2026-05-07 | W3 Day 3：TaskList.v2.vue 需求拆解 × useTasks Composable 骨架設計（實作期啟動）| M1 | ⏳ 進行中（截止 5/11）| — |
| 2026-05-08 | W3 Day 4：useTasks 核心邏輯完整實作 × TaskList.v2.vue 接入 × 瀏覽器驗證（核心功能可運作）| M1 | ⏳ 進行中（截止 5/11）| — |
| 2026-05-09 | W3 Day 5：LocalStorage 持久化回路（onMounted 讀取）× 字數限制 × W3 Self-review | M1 | ⏳ 進行中（截止 5/11，剩 2 天）| — |
| 2026-05-10 | W3 Day 6：作業收尾整理 × TypeScript 型別審查 × Commit 整理 × PR 準備 × 最終驗收腳本 | M1 | ⏳ 收尾中（截止 明日 5/11）| — |
| 2026-05-11 | W3 Day 7（截止日）：W3 完整知識總結 × PR 提交行動 × W4（Lifecycle Hooks / watchEffect / toRefs）銜接預習 | M1 | 🔴 截止已過，請確認 PR | — |
| 2026-05-12 | W4 Day 1：Vue 3 Lifecycle Hooks 完整時間軸 × onMounted vs watch { immediate } 選用決策 × 父子掛載順序 × onUnmounted 清理模式 | M1 | ⏳ W4 進行中（截止 5/18）| — |
| 2026-05-13 | W4 Day 2：watchEffect 深化 × onCleanup 完整時機（每次重跑前觸發）× AbortController Race Condition 防禦 × flush: post | M1 | ⏳ W4 進行中（截止 5/18，剩 5 天）| — |
| 2026-05-14 | W4 Day 3：toRefs 深化 × Composable 回傳物件設計（個別 ref vs reactive + toRefs）× storeToRefs 概念 × LifecycleLogger.vue 骨架設計完成 | M1 | ⏳ W4 進行中（截止 5/18，剩 4 天）| — |
| 2026-05-15 | W4 Day 4：useWatchEffectLogger.ts 完整實作 × flush 模式動態切換 × Active Instance 清理機制 × AbortController 非同步防禦 | M1 | ⏳ W4 進行中（截止 5/18，剩 3 天）| — |
| 2026-05-16 | W4 Day 5：useLifecycleLogger.ts 完整實作 × LifecycleLogger.vue 模板整合 × 依賴追蹤因果鏈（讀取建立依賴 → re-render → onUpdated）× 雙 Composable 同一元件整合設計 | M1 | ⏳ W4 進行中（截止 5/18，剩 2 天）| — |
| 2026-05-17 | W4 Day 6：TypeScript 型別審查（定義層 → 函式簽名 → 使用層）× 功能驗收腳本（8 步驟系統化）× Commit 整理 × PR 準備 × keyof T × Object.keys 型別限制 × Composable 回傳型別 API 契約 | M1 | ⏳ W4 收尾中（截止 明日 5/18）| — |
| 2026-05-18 | W4 Day 7（截止日）：W4 整週知識總結（Lifecycle→watchEffect→toRefs→Composable實作→型別審查→PR）× onErrorCaptured 擴充性設計分析 × M2 銜接預習（defineProps/defineEmits/v-model 進階） | M1 | 🔴 截止已過，請確認 Push + PR | — |
| 2026-05-19 | M2-W5 Day 1：defineProps（TS 型別宣告 × withDefaults）× defineEmits（payload 型別約束）× v-model 完整展開（基本 / 具名 / 多個）× defineModel（Vue 3.4）× Props Down / Events Up 原則 | M2 | ⏳ W5 進行中（截止 5/25）| — |
| 2026-05-20 | M2-W5 Day 2：CustomInput.vue 完整實作（Props → Input → Emit 資料流閉環）× defineModel 對照版本（選用決策 trade-off）× 泛型 v-model 預備（generic="T"，面向 CustomSelect）× as HTMLInputElement type narrowing × v-if vs v-show 選用決策 | M2 | ⏳ W5 進行中（截止 5/25，剩 5 天）| — |
| 2026-05-21 | M2-W5 Day 3：Event 型別鏈（InputEvent / FocusEvent / instanceof 守衛）× $attrs Fallthrough（inheritAttrs: false + v-bind="$attrs"）× 表單整合狀態設計（reactive 表單物件 × useLoginForm Composable）× CustomSelect 泛型骨架（generic="T extends string \| number"）| M2 | ⏳ W5 進行中（截止 5/25，剩 4 天）| — |
| 2026-05-22 | M2-W5 Day 4：CustomSelect 完整實作（HTML .value 永遠是 string × options.find 還原 T 型別 × null 值 v-model 處理）× LoginForm.vue 三元件整合（CustomInput × 2 + CustomSelect × 1）× 即時驗證策略（@blur + 提交時二次驗證）× 泛型 T 型別推導機制（無需顯式指定）| M2 | ⏳ W5 進行中（截止 5/25，剩 3 天）| — |
| 2026-05-23 | M2-W5 Day 5：功能驗收腳本系統化（12 步驟 × 操作+確認結構 × 人工版 Unit Test）× TypeScript 全審查流程（定義層→函式簽名→使用層→vue-tsc --noEmit）× 邊緣 case 補強清單 × Code Review 自審（Why 角度）× W5 PR 說明草稿結構建立 | M2 | ⏳ W5 進行中（截止 5/25，剩 2 天）| — |
| 2026-05-24 | M2-W5 Day 6：Commit 整理（Conventional Commits 規範 × feat/fix/refactor 選用 × git rebase -i）× PR 說明定稿（交付清單 + Why 決策 × 3 + 未決問題 + 測試方式）× --force-with-lease 協作安全 push × W5 七天知識閉環總結 × W6 銜接預習（provide/inject × Slots × 動態元件） | M2 | ⏳ W5 截止明日（5/25），今日完成 Push + PR | — |
| 2026-05-25 | M2-W5 Day 7（截止日）：W5 完整知識閉環確認 × PR 行動確認（今日截止）× W6 深化預習（provide/inject 機制 × InjectionKey<T> × Slots 三種形式 × 動態元件 × KeepAlive） | M2 | 🔴 截止已過，請確認 PR Open | — |
| 2026-05-26 | M2-W6 Day 1：provide/inject 完整實作模式（InjectionKey<T> × inject fallback 三情境 × provide reactive 物件 × App-level provide）× Slots 三種形式（Default / Named / Scoped × $slots 條件渲染）× 動態元件與 KeepAlive（activated / deactivated × include / exclude / max × LRU 策略） | M2 | ⏳ W6 進行中（截止 6/1）| — |
| 2026-05-27 | M2-W6 Day 2：`PanelCard.vue` 完整實作（三區域 Named Slot × `$slots.footer` 條件渲染 × fallback content）× `injectionKeys.ts` 架構設計（`PanelTheme` interface × `panelThemeKey` Symbol）× `DashboardLayout.vue` 骨架（`provide reactive(theme)` × `<component :is>` × `KeepAlive`）× `inject` 型別縮窄實戰（fallback vs null check vs `!` 斷言）× `as const` + `keyof typeof` 精確型別鏈 | M2 | ⏳ W6 進行中（截止 6/1，剩 5 天）| — |
| 2026-05-28 | M2-W6 Day 3：`FormPanel.vue` 完整實作（inject theme → CSS class 查找表 × Named Slots × KeepAlive 狀態保留驗證）× `ListPanel.vue` 泛型 Scoped Slot（`generic="T"` × 型別推導鏈）× `ChartPanel.vue`（inject headerBg × style 綁定）× `DashboardLayout.vue` 整合（三面板切換 × 響應式 theme 切換按鈕）× KeepAlive 副作用管理（`onDeactivated` vs `onUnmounted`）| M2 | ⏳ W6 進行中（截止 6/1，剩 4 天）| — |
| 2026-05-29 | M2-W6 Day 4：`usePanelTheme.ts` Composable 提取（inject+fallback+computed 封裝，消除三 Panel inject 重複）× TypeScript 精確索引型別（`Record<PanelTheme['borderRadius'], string>`）× W6 功能驗收腳本 12 步驟 × `vue-tsc --noEmit` 零錯誤 | M2 | ⏳ W6 進行中（截止 6/1，剩 3 天）| — |
| 2026-05-30 | M2-W6 Day 5：W6 Commit 整理（`git add -p` × `git rebase -i` × Conventional Commits feat/refactor/chore 選用）× PR 說明定稿（三個 Why 設計決策：Symbol InjectionKey / 泛型 Scoped Slot / Composable 提取時機）× W6 七天知識閉環整理（核心主軸：跨層通訊 × 彈性插槽 × 可維護性）| M2 | ⏳ W6 截止 6/1（剩 2 天）| — |
| 2026-05-29 | M2-W6 Day 4：`usePanelTheme.ts` Composable 提取（inject+fallback+computed 封裝，消除三 Panel inject 重複）× TypeScript 精確索引型別（`Record<PanelTheme['borderRadius'], string>`）× W6 功能驗收腳本 12 步驟 × `vue-tsc --noEmit` 零錯誤 | M2 | ⏳ W6 進行中（截止 6/1，剩 3 天）| — |
| 2026-05-30 | M2-W6 Day 5：W6 Commit 整理（`git add -p` × `git rebase -i` × Conventional Commits feat/refactor/chore 選用）× PR 說明定稿（三個 Why 設計決策：Symbol InjectionKey / 泛型 Scoped Slot / Composable 提取時機）× W6 七天知識閉環整理（核心主軸：跨層通訊 × 彈性插槽 × 可維護性）× `--force-with-lease` 安全推送複習 | M2 | ⏳ W6 截止 6/1（剩 2 天）| — |
| 2026-05-29 | M2-W6 Day 4：`usePanelTheme.ts` Composable 提取（inject+fallback+computed 查找表封裝，消除三個 Panel inject 重複）× TypeScript 精確索引型別（`Record<PanelTheme['borderRadius'], string>`）× inject 物件共享機制（三 Panel 共享同一 reactive Proxy）× W6 功能驗收腳本 12 步驟 × nested object 泛型推導深度確認 × `vue-tsc --noEmit` 零錯誤 | M2 | ⏳ W6 進行中（截止 6/1，剩 3 天）| — |
| 2026-05-31 | M2-W6 Day 6：W6 Push + PR Open（行動確認：`git push --force-with-lease` × GitHub PR Open × CI 確認）× W7 Composable 設計深度預習（抽取時機三訊號系統化 × 四種設計模式：useAsyncData/useLocalStorage/usePagination/useEventListener × Composable vs Pinia 邊界 × 回傳介面設計原則） | M2 | 🔴 W6 截止明日（6/1），今日必須 Push + PR Open | — |
| 2026-06-01 | M2-W6 Day 7（截止日）：W6 七天完整知識閉環（provide/inject × Slots × KeepAlive × usePanelTheme 三主軸整合）× PR 截止確認（今日截止，立即行動）× W7 正式準備（useAsyncData/useLocalStorage API 設計草稿 × 三態管理 × AbortController × JSON 安全讀寫） | M2 | 🔴 截止今日，請立即 Push + PR Open | — |
| 2026-06-02 | M2-W7 Day 1：Composable 抽取時機三訊號系統化（被動識別→主動設計）× useAsyncData<T> 完整設計（三態管理 × AbortController 封裝 × readonly 邊界 × AbortError 過濾）× AsyncDataResult<T> 泛型介面 × Readonly<Ref<T\|null>> 型別組合 | M2 | ⏳ W7 進行中（截止 6/8）| — |
| 2026-06-03 | M2-W7 Day 2：useLocalStorage<T> 完整設計（parseJSON<T> 型別守衛 × 初始化時機優化：ref() vs onMounted × SSR 安全 × watch deep 寫入回路）× Ref<T> vs Readonly<Ref<T>> API 邊界決策原則 × Composable 四層架構（持久化/非同步/UI邏輯/基礎設施）× useTasks.v3 重構對照 | M2 | ⏳ W7 進行中（截止 6/8，剩 5 天）| — |
| 2026-06-04 | M2-W7 Day 3：usePagination 完整設計（純 computed 驅動 × 無副作用 × total: Ref<number> 響應式邊界 × pageRange 省略號邏輯 × goToPage 邊界保護 × totalPages watch 邊界修正）× 與 useAsyncData 整合兩種模式（模式 A：元件 watch / 模式 B：業務 Composable 封裝）× Composable 四層架構第三層補全 | M2 | ⏳ W7 進行中（截止 6/8，剩 4 天）| — |
