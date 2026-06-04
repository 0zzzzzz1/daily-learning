# W6 Commit 整理 × PR 說明定稿 × 七天知識閉環預備

> **日期**：2026-05-30（M2-W6 Day 5）
> **週次**：W6（5/26–6/1），截止 6/1，剩 **2 天**
> **學習階段**：M2 Composition API 深化期

---

## ⚠️ 作業堆積警示

| 週次 | 截止日 | 逾期天數 | 作業項目 |
|------|--------|---------|---------|
| W3 | 2026-05-11 | 19 天 | TaskList.vue v2 |
| W4 | 2026-05-18 | 12 天 | LifecycleLogger.vue |
| W5 | 2026-05-25 | 5 天 | CustomInput + CustomSelect + LoginForm |

**W6 截止日（6/1）剩 2 天** — 今日 Commit 整理完成後，明日（Day 6）執行 Push + PR 發出；後日（6/1，Day 7）為截止日確認。**今日是整理 commit 歷史的最佳時機，請勿推遲。**

---

## 一、目標技術 & 核心知識點

| 目標技術 | 核心知識點 |
|---------|-----------|
| Git 工作流 | `git add -p`（patch-mode 細粒度分批）× `git rebase -i`（互動式整理 commit 序列）|
| Conventional Commits | feat / refactor / chore / docs 選用決策 × scope 命名 × W6 具體 commit 範例 |
| PR 說明撰寫 | 三個 Why 設計決策 × 交付清單 × 未決問題 × 測試方式 |
| 安全推送 | `--force-with-lease` vs `--force` × rebase 後 push 的標準流程 |
| W6 知識閉環 | provide/inject × Slots × 動態元件 × KeepAlive × Composable 提取：完整知識鏈 |

---

## 二、為什麼學這個（與昨日的連結）

W6 四天的學習軌跡：

```
Day 1（5/26）：概念建立（InjectionKey<T> × Slots 三種形式 × KeepAlive）
      ↓
Day 2（5/27）：PanelCard + DashboardLayout 骨架（Named Slot × provide reactive × <component :is>）
      ↓
Day 3（5/28）：三個 Panel 完整實作（泛型 Scoped Slot × inject + CSS 查找表 × KeepAlive 副作用管理）
      ↓
Day 4（5/29）：精煉（usePanelTheme 提取 × 精確索引型別 × 功能驗收腳本 12 步驟 × vue-tsc 通過）
      ↓
Day 5（今日）：收尾（commit 整理 × PR 說明定稿 × W6 知識閉環預備）
```

四天的實作在 git 歷史中可能是一連串「WIP」、「fix」、「test」類的雜亂 commit。今日的任務是把這段歷史整理成**清晰、可 review 的 commit 序列**，並撰寫一份讓 reviewer 真正能評估設計決策的 PR 說明。

這套流程在 W5 Day 6 已練習過（Conventional Commits 規範 × git rebase -i × --force-with-lease × PR 說明 Why 角度）。今日是在 W6 的語境下複習並應用，同時帶入 W6 特有的設計決策：Symbol InjectionKey、泛型 Scoped Slot、Composable 提取時機。

---

## 三、知識說明

### 3.1 `git add -p`：細粒度分批 commit

大多數情況下，一次開發會累積多個邏輯單位的改動混在同一批檔案裡。`git add -p` 讓你用「hunk（差異塊）」為單位選擇暫存哪些改動：

```bash
git add -p
```

互動提示：
- `y` — 暫存這個 hunk
- `n` — 跳過這個 hunk
- `s` — 把這個 hunk 拆成更小的塊
- `e` — 手動編輯這個 hunk
- `q` — 離開（已暫存的部分保留）

**W6 的應用場景**：

假設你在同一個 session 裡同時做了：
1. 新增 `injectionKeys.ts`（feat）
2. 新增 `PanelCard.vue`（feat）
3. 修正 `$slots.footer` 的條件判斷（fix）
4. 提取 `usePanelTheme.ts`（refactor）

用 `git add -p` 可以把這四件事分成四個獨立 commit，而不是一個巨大的「complete W6 implementation」commit，讓 reviewer 更容易逐 commit review。

---

### 3.2 Conventional Commits 在 W6 的應用

W6 的 commit 序列建議如下（參考順序，依實際開發順序調整）：

```
feat(injection): add InjectionKey<PanelTheme> and default theme in injectionKeys.ts

feat(panel): implement PanelCard.vue with three named slots and $slots.footer guard

feat(layout): implement DashboardLayout.vue with provide reactive theme and KeepAlive

feat(panel): implement FormPanel, ListPanel (generic scoped slot), ChartPanel

refactor(composable): extract usePanelTheme to eliminate inject duplication across panels

chore(types): add PanelThemeResult interface and explicit return type annotation

test(w6): run 12-step verification script; confirm vue-tsc --noEmit passes
```

**選用判斷提示**：

| 動作 | 類型 | 判斷依據 |
|------|------|---------|
| 新增元件（PanelCard、FormPanel 等）| `feat` | 新增使用者可見/可用的功能 |
| 新增 injectionKeys.ts | `feat` | 新增共享資料來源（影響功能架構）|
| 提取 usePanelTheme.ts | `refactor` | 行為不變，只是重組程式碼結構 |
| 補 PanelThemeResult 介面 | `chore` 或 `refactor` | 不改行為；若只補型別選 `chore`；若重構介面邊界選 `refactor` |
| 修正 KeepAlive onDeactivated 清理 | `fix` | 修正運行期錯誤行為 |

**scope 命名建議**（W6 語境）：
- `(injection)` — InjectionKey 相關
- `(panel)` — Panel 元件相關
- `(layout)` — DashboardLayout 相關
- `(composable)` — Composable 提取
- `(types)` — 型別定義

---

### 3.3 W6 PR 說明：三個 Why 設計決策

PR 說明的核心是「為什麼這樣設計」，讓 reviewer 能評估 trade-off 而不只是審格式。

#### Why 1：為什麼用 `InjectionKey<T>` + Symbol，而不是字串 key？

**背景**：provide/inject 可以用字串或 Symbol 作為 key。

**選用字串的問題**：
```typescript
// ❌ 字串 key
provide('theme', reactive(theme))
const t = inject('theme') // 回傳 unknown，需手動斷言
```
- 字串 key 可能在大型專案中命名衝突（第三方套件也可能使用相同字串）
- TypeScript 無法推導 inject 的回傳型別（只能拿到 `unknown`）

**選用 `InjectionKey<T>` 的理由**：
```typescript
// ✅ InjectionKey<T>
export const panelThemeKey: InjectionKey<PanelTheme> = Symbol('panelTheme')
provide(panelThemeKey, reactive(theme))
const t = inject(panelThemeKey, defaultTheme) // TypeScript 推導出 PanelTheme
```
- Symbol 全域唯一，不可偽造，不會命名衝突
- TypeScript 從 `InjectionKey<T>` 的泛型參數自動推導 inject 回傳型別
- 集中管理在 `injectionKeys.ts` 讓所有 provide/inject 對一個真相來源

**trade-off**：Symbol 在 DevTools 顯示時辨識度較低（顯示 Symbol(panelTheme) 而非可讀字串）。在小型專案中字串 key 配合型別斷言也可接受，但本案選擇型別安全優先。

---

#### Why 2：為什麼 `ListPanel` 使用 `generic="T"` 泛型 Scoped Slot，而不是 `items: unknown[]`？

**背景**：ListPanel 需要接受任意型別的列表資料，並讓父元件決定每項如何渲染。

**使用 `unknown[]` 的問題**：
```vue
<!-- ❌ unknown[] 讓使用端無法推導 item 的型別 -->
<ListPanel :items="tasks">
  <template #default="{ item }">
    {{ item.title }} <!-- TypeScript：item 是 unknown，.title 報錯 -->
  </template>
</ListPanel>
```
使用端需要手動斷言 `(item as Task).title`，既繁瑣又不安全。

**使用 `generic="T"` 的理由**：
```vue
<!-- ✅ TypeScript 從 :items 綁定值推導 T = Task -->
<ListPanel :items="tasks">
  <template #default="{ item }">
    {{ item.title }} <!-- TypeScript：item 是 Task，型別安全 ✅ -->
  </template>
</ListPanel>
```
- TypeScript 從 `:items="tasks"` 自動推導 `T = Task`，無需顯式指定
- Scoped Slot 的 `item` 型別自動縮窄為 `T`
- 實現「Headless UI」設計：ListPanel 只負責遍歷，UI 邏輯完全交給使用端

**trade-off**：`generic="T"` 語法需要 Vue 3.3+ 和 TypeScript。若需支援舊版，可退回 `PropType<unknown[]>` + 使用端手動斷言。

---

#### Why 3：為什麼提取 `usePanelTheme` Composable，而不讓三個 Panel 各自 inject？

**背景**：FormPanel、ListPanel、ChartPanel 各自都有 `inject + computed 查找表` 的相同樣板程式碼。

**讓三個 Panel 各自 inject 的問題**：
```typescript
// ❌ 三個 Panel 都有這段（共 3 × 3 = 9 行重複）
const theme = inject(panelThemeKey, defaultTheme)
const borderRadiusClass = computed(() => borderRadiusMap[theme.borderRadius] ?? 'rounded')
const shadowClass = computed(() => shadowMap[theme.shadow] ?? 'shadow')
```
- `borderRadiusMap`、`shadowMap` 等查找表也在三個地方重複定義
- 若 `PanelTheme` 新增一個 key（如 `fontSizeClass`），需修改三個檔案

**提取 `usePanelTheme` 的理由**：
```typescript
// ✅ 每個 Panel 只需一行
const { borderRadiusClass, shadowClass, headerBgClass } = usePanelTheme()
```
- 查找表集中在 `usePanelTheme.ts`（module 層級靜態常數）
- `PanelTheme` 新增 key 只需改一個 Composable
- `Record<PanelTheme['borderRadius'], string>` 精確索引型別確保查找表完整性（漏寫 key 時 TypeScript 編譯報錯）

**提取時機訊號**：三個以上元件有**完全相同的 inject + computed 查找表樣板**——這是「樣板重複」比「邏輯重複」更具體的提取訊號。

---

### 3.4 `--force-with-lease` 複習

當你用 `git rebase -i` 整理了 commit 歷史後，本地歷史和遠端歷史已分叉，需要 force push：

```bash
# ❌ 危險：無條件覆蓋遠端，可能蓋掉他人 push 的工作
git push origin feature/w6-panel --force

# ✅ 安全：先確認遠端狀態與你 fetch 時一致，有差異就拒絕
git push origin feature/w6-panel --force-with-lease
```

**為什麼 `--force-with-lease` 更安全**：
- `--force` 直接覆蓋遠端任何內容
- `--force-with-lease` 記錄你最後一次 `git fetch` 時遠端的狀態，若遠端在此後有新 commit（他人 push），推送會被拒絕，提示你先 pull 再決定

**W6 的實際流程**：
```bash
# 1. 整理 commit 序列
git rebase -i HEAD~N  # N = 過去 N 個 commit

# 2. 確認結果
git log --oneline -10

# 3. 安全推送
git push origin feature/w6-panel --force-with-lease
```

---

### 3.5 W6 七天知識閉環：完整知識鏈

W6 七天的知識鏈如下（今日整理，明後兩天 PR 提交後形成完整閉環）：

```
理論建立（Day 1）
├── provide/inject 機制
│   ├── InjectionKey<T>：Symbol + TS 型別推導雙保障
│   ├── inject fallback 三情境（可選/必要/強制）
│   └── provide reactive 物件：Proxy 共享（後代響應更新）
│
├── Slots 三種形式
│   ├── Default Slot：插入單塊內容
│   ├── Named Slot：多區域結構化插入（Header/Body/Footer）
│   └── Scoped Slot：資料與渲染分離（子有資料，父決定 UI）
│
└── 動態元件 × KeepAlive
    ├── <component :is="currentComponent">
    ├── KeepAlive 保留實例（切走不銷毀，activated/deactivated 替代 mounted/unmounted）
    └── LRU 淘汰（max 屬性）× 副作用管理（onDeactivated 清理）

        ↓

實作落地（Day 2–3）
├── injectionKeys.ts：集中管理 PanelTheme interface + panelThemeKey Symbol
├── PanelCard.vue：三區域 Named Slot × $slots.footer 條件渲染 × fallback content
├── DashboardLayout.vue：provide reactive(theme) × <component :is> × KeepAlive
├── FormPanel.vue：inject + CSS class 查找表 × Named Slots
├── ListPanel.vue：generic="T" × Scoped Slot 型別推導鏈（Headless UI）
└── ChartPanel.vue：inject headerBg × style 綁定

        ↓

精煉層（Day 4）
├── usePanelTheme.ts：提取 inject + fallback + computed 查找表（消除 3 份重複）
├── Record<PanelTheme['borderRadius'], string>：精確索引型別（新增 union 值強制補 class）
├── W6 功能驗收腳本 12 步驟：人工 Unit Test
└── vue-tsc --noEmit：TypeScript 靜態分析通過

        ↓

收尾層（Day 5–6）
├── git add -p：細粒度分批 commit
├── Conventional Commits：feat/refactor/chore 語意化序列
├── PR 說明：三個 Why 設計決策（Symbol Key / 泛型 Scoped Slot / Composable 提取時機）
└── --force-with-lease：rebase 後安全推送
```

這條知識鏈的核心主軸：**「跨層通訊（provide/inject）× 彈性插槽（Slots）× 可維護性（Composable 提取）」**——三個面向都是為了讓元件架構在面對真實專案需求時更具彈性與可擴充性。

---

## 四、作業說明（W6 截止 6/1，剩 2 天）

### 今日應完成的 Commit 整理工作

**目標 commit 序列（建議）**：

```
feat(injection): add InjectionKey<PanelTheme> and defaultTheme in injectionKeys.ts
feat(panel): implement PanelCard.vue with named slots and $slots.footer guard
feat(layout): implement DashboardLayout.vue with provide reactive and KeepAlive
feat(panel): implement FormPanel, ListPanel (generic scoped slot), ChartPanel
refactor(composable): extract usePanelTheme from three panels
chore(types): add PanelThemeResult interface as return type contract
```

### 今日 PR 說明草稿

**標題**：`feat(w6): implement Dashboard Panel system with provide/inject, slots, and KeepAlive`

**交付清單**：
- [ ] `injectionKeys.ts` — `InjectionKey<PanelTheme>` + defaultTheme
- [ ] `PanelCard.vue` — 三區域 Named Slot + `$slots.footer` 條件渲染
- [ ] `DashboardLayout.vue` — `provide reactive(theme)` + `<component :is>` + `KeepAlive`
- [ ] `FormPanel.vue` — inject theme → CSS class 查找表 + Named Slots
- [ ] `ListPanel.vue` — `generic="T"` 泛型 Scoped Slot（Headless UI 設計）
- [ ] `ChartPanel.vue` — inject + style 綁定
- [ ] `composables/usePanelTheme.ts` — 封裝 inject + computed 查找表

**設計決策（Why）**：

1. **為什麼用 `InjectionKey<T>` + Symbol 而非字串 key？**
   → Symbol 全域唯一防衝突；`InjectionKey<PanelTheme>` 讓 TypeScript 自動推導 inject 回傳型別，無需手動斷言。

2. **為什麼 `ListPanel` 使用 `generic="T"` 泛型 Scoped Slot？**
   → Headless UI 設計：ListPanel 只負責遍歷資料，UI 完全交給使用端。`generic="T"` 讓 TypeScript 從 `:items` 自動推導型別，Scoped Slot 中的 `item` 型別自動縮窄，使用端無需手動斷言。

3. **為什麼提取 `usePanelTheme` 而不讓三個 Panel 各自 inject？**
   → 三個 Panel 有完全相同的 inject + computed 樣板（Composable 提取時機的明確訊號）；提取後 `PanelTheme` 新增屬性只需改一處；`Record<PanelTheme['borderRadius'], string>` 精確索引型別確保查找表完整性。

**未決問題**：
- `provide(key, readonly(reactive(theme)))` 是否應加 `readonly` 防止後代意外修改？本版本後代只讀取 theme，故暫不加；若後代需要修改 theme（如局部面板自訂主題），應改用雙 key 設計（讀取 key + 更新 action key）。

**測試方式**：
- 執行 W6 功能驗收腳本 12 步驟（見 `2026-05-29/今日學習分析.md`）
- `vue-tsc --noEmit` — 確認零 TypeScript 錯誤

### 評分標準

| 面向 | 滿分 | 說明 |
|------|------|------|
| Commit 序列清晰度 | 20 | 每個 commit 語意獨立、Conventional Commits 格式正確 |
| PR 說明 Why 完整性 | 20 | 三個設計決策有完整 trade-off 說明 |
| 推送安全性 | 5 | 使用 `--force-with-lease`（若有 rebase）|
| W6 作業行為正確性 | 55 | 延續 Day 1–4 評分（功能驗收腳本 + vue-tsc 通過）|

---

## 五、自我檢核問題

**Q1**：你有一個 commit 訊息是 `fix: resolve inject type issue`，沒有 scope。根據 W6 的 commit 規範，這應該改成什麼格式？為什麼加 scope 有幫助？

**Q2**：完成 `git rebase -i` 整理後，直接執行 `git push origin feature/w6 --force` 推送，這有什麼風險？正確的指令是什麼？

**Q3**：PR 說明中「未決問題」（`readonly` 包裝）的討論有什麼實務意義？reviewer 看到這段討論時，他能做出哪些判斷？

**Q4**：`usePanelTheme` 在 PR 說明中被歸類為 `refactor`，而不是 `feat`。請解釋為什麼：新增 `usePanelTheme.ts` 這個新檔案，卻被視為 `refactor` 而非 `feat`？

**Q5**：W6 的知識鏈中，「Scoped Slot」和「provide/inject」都能解決「子元件需要把資料傳遞給外部」的問題，但使用場景完全不同。請描述這兩者解決的是哪個不同的「方向」問題？

---

## 六、明日預告（W6 Day 6）

明日（2026-05-31）：**W6 Push + PR 行動 × 七天知識閉環整合**

重點：
- 執行最終 `git push origin feature/w6 --force-with-lease`
- 在 GitHub 開 PR，貼上今日定稿的 PR 說明
- 確認 CI（若有）通過，確認 PR 開啟狀態
- W6 七天完整知識鏈最終整合（provide/inject × Slots × KeepAlive × Composable 提取 × TypeScript 整合）
- W7 銜接預習：Composable 設計深化（抽取邏輯、可複用 Hook，3 個 Composable 實作）
