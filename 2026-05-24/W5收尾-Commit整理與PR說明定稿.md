# W5 收尾：Commit 整理 × PR 說明定稿 × W5 整週知識總結

> **日期**：2026-05-24（M2-W5 Day 6）
> **所屬週次**：W5（5/19–5/25）
> **截止日**：2026-05-25（明日 ⚠️ 最後一個完整工作日）
> **今日角色**：W5 七天學習週期的收尾層

---

## 一、目標技術與核心知識點

| 技術 | 核心知識點 |
|------|-----------|
| 工程實踐 | Conventional Commits 規範、git commit 拆分原則 |
| PR 文化 | PR 說明定稿（Why 角度完整版）、Code Review 自審 |
| Vue 3 Composition API | W5 整週知識鏈回顧（defineProps → defineEmits → v-model → $attrs → 泛型 → 驗收）|
| 學習方法 | 「能跑」→「可 Review」→「值得學習」的距離感知 |

---

## 二、為什麼學這個（與前幾日的連結）

```
W5 完整知識鏈（七天閉環）

Day 1（5/19）理論建立
  └── defineProps / defineEmits / v-model 本質 × defineModel

Day 2（5/20）實作落地
  └── CustomInput.vue 完整實作 × event.target 型別斷言

Day 3（5/21）精進層
  └── Event 型別鏈 × $attrs Fallthrough × useLoginForm × CustomSelect 骨架

Day 4（5/22）整合層
  └── CustomSelect 完整實作 × LoginForm.vue 三元件組裝 × 驗證策略設計

Day 5（5/23）驗收層
  └── 12 步驟功能驗收腳本 × TypeScript 全審查 × 邊緣 case 補強 × PR 草稿

Day 6（今日）收尾層  ← 你在這裡
  └── Commit 整理 × PR 說明定稿 × W5 七天閉環知識總結 × W6 銜接預習
```

**為什麼今天最關鍵**：昨天完成了「能 Review」的最後一哩路（驗收腳本 + TS 審查）。今天的任務是把這七天的工作成果整理成「值得存留的 Git 歷史」與「有意義的 PR 說明」，並做整週知識總結，讓學習成果內化成長期記憶，而不只是「做完就算了」。

> **工程師的成果，最終體現在 Git 歷史與 PR 說明裡——這是留給未來的自己和同事的文件。**

---

## 三、知識說明

### 3-1｜Conventional Commits 規範

Conventional Commits 是一種結構化的 commit message 格式，讓 Git 歷史成為可讀的變更日誌。

**格式**：
```
<type>(<scope>): <description>

[optional body]
[optional footer]
```

**常用 type 說明**：

| type | 用途 | W5 作業適用範例 |
|------|------|---------------|
| `feat` | 新功能 | `feat(form): add CustomInput component with v-model support` |
| `fix` | 修正 bug | `fix(select): handle null modelValue in CustomSelect` |
| `refactor` | 重構（不影響行為）| `refactor(login): extract form logic into useLoginForm composable` |
| `style` | 格式調整（不影響邏輯）| `style: align TypeScript type annotations` |
| `test` | 新增測試 | `test: add validation acceptance tests for LoginForm` |
| `chore` | 維護性工作 | `chore: run vue-tsc verification` |
| `docs` | 文件變更 | `docs: add JSDoc to useLoginForm return type` |

**W5 作業的建議 commit 拆分方式**：

```
feat(input): add CustomInput with defineProps/defineEmits/v-model
feat(select): add CustomSelect with generic T and options.find type restoration
feat(form): add useLoginForm composable with validate and reset
feat(form): integrate LoginForm with CustomInput × 2 and CustomSelect × 1
fix(select): handle null modelValue edge case
style(form): apply inheritAttrs: false and $attrs passthrough to CustomInput
```

**拆分原則**：
- 每個 commit 只做一件「概念上的事」
- 可以在 `git rebase -i` 之後整理（squash 過細的 commit，split 過粗的 commit）
- `feat` 的粒度：能獨立說清楚「加了什麼能力」的最小單位

---

### 3-2｜PR 說明定稿指引（W5 完整版）

**PR 說明結構**（W5 作業版本）：

---

#### PR 標題（範例）
```
feat(M2-W5): 元件通訊完整實作 — CustomInput × CustomSelect × LoginForm
```

#### 交付清單
- [x] CustomInput.vue — v-model 雙向綁定、$attrs Fallthrough、錯誤訊息顯示
- [x] CustomSelect.vue — 泛型 `T`、HTML `.value` 還原、null 值處理
- [x] useLoginForm.ts — reactive 表單狀態、validate()、reset()
- [x] LoginForm.vue — 三元件組裝、@blur 驗證策略
- [x] 功能驗收腳本（12 步驟）— 完整執行
- [x] vue-tsc --noEmit — 零警告

#### 主要設計決策（Why 角度）

**1. 為什麼 CustomSelect 用 `options.find` 而不是直接 emit `select.value`？**
> HTML `<select>` 的 `.value` 永遠是 string（HTML 規範）。若直接 emit，當泛型 `T` 為 `number` 時，父元件收到的是字串 `"42"` 而非數字 `42`。`options.find` 透過比對找回原始型別，讓 TypeScript 能在編譯期確保型別安全。

**2. 為什麼 CustomInput 使用 `inheritAttrs: false` + `v-bind="$attrs"`？**
> 若不設定 `inheritAttrs: false`，Vue 會自動把未宣告的 attrs（如 `maxlength`、`inputmode`）套到根元素（wrapper div）。我們希望它們落在 `<input>` 元素上才有效果。`v-bind="$attrs"` 讓使用端可以直接傳 HTML 屬性，不需要 CustomInput 逐一宣告。

**3. 為什麼 validate() 回傳 boolean 而不是 void？**
> 若 validate() 只執行驗證但不回傳結果，handleSubmit 需要另外讀取 errors 物件來判斷是否繼續——這在設計上引入了不必要的耦合。回傳 boolean 讓呼叫端可以 `if (!validate()) return` 立即決策，介面更乾淨。

**4. 為什麼表單狀態選 `reactive` 而不是多個 `ref`？**
> 表單欄位是固定結構，且需要用 `Object.keys` 做通用驗證迴圈（`keyof typeof form`）。若每個欄位用獨立 `ref`，通用驗證需要特殊處理；`reactive` 讓表單結構本身就是型別定義，結合 `keyof` 更安全。

#### 未決問題 / 後續優化方向
- `CustomSelect` 尚未支援 multiple selection，留待 M3 型別整合後評估
- `useLoginForm` 的 errors 目前為全量驗證，後續可改為 dirty field tracking
- `$attrs` 中的 `class` 目前會同時作用於 input 和 wrapper，需進一步設計

#### 測試方式
1. 執行功能驗收腳本（12 步驟），確認每個「確認」項目通過
2. 執行 `vue-tsc --noEmit`，確認零警告

---

### 3-3｜W5 整週知識鏈總結

```
W5 核心概念樹
│
├── 元件通訊基礎
│   ├── defineProps<Props>() + withDefaults — 型別化 Props
│   ├── defineEmits<{...}>() — tuple 格式 payload 型別
│   └── Props Down / Events Up — 原則與違反後的後果
│
├── v-model 系統
│   ├── 展開：:modelValue（唯讀）+ @update:modelValue（emit）
│   ├── 具名：v-model:name → prop name + emit update:name
│   ├── defineModel（Vue 3.4）— 可寫 ref，限：無法攔截 emit 前
│   └── 泛型 v-model：generic="T" + defineModel<T>()
│
├── 高階 Props 技巧
│   ├── $attrs Fallthrough — inheritAttrs: false + v-bind="$attrs"
│   ├── class/style 也在 $attrs — 套用位置是 API 設計決策
│   └── Event 型別鏈 — EventTarget → UIEvent → InputEvent/FocusEvent
│
├── 表單整合設計
│   ├── useLoginForm — State(reactive) + validate(boolean) + reset
│   ├── CustomInput + CustomSelect + LoginForm — 三層架構
│   ├── @blur + 提交驗證 — 雙重防禦策略
│   └── HTML .value 型別陷阱 — options.find 還原泛型 T
│
└── 工程實踐（W5 特有）
    ├── 功能驗收腳本 — 操作+確認結構，人工版 Unit Test
    ├── TypeScript 全審查 — 定義→簽名→使用→vue-tsc 由底向上
    ├── 邊緣 case 掃描 — 空值/null/型別邊界/狀態轉換/快速操作
    ├── Conventional Commits — 拆分原則與 type 選用
    └── PR 說明 Why 角度 — 設計決策而非功能清單
```

---

## 四、作業說明（今日最重要的行動項目）

### W5 作業提交 Checklist

#### 情境背景
W5 作業截止日為 **明日（5/25）**。今天是最後一個完整工作日，目標是完成 Git 整理與 PR 發出，讓明天只需確認 PR 狀態。

#### 今日行動清單（按順序執行）

**Step 1：確認程式碼狀態**
```bash
# 確認工作目錄乾淨（所有修改已 stage 或 stash）
git status

# 確認目前分支名稱
git branch
```

**Step 2：整理 Commit 歷史（若有過多零散 commit）**
```bash
# 查看最近 N 個 commit
git log --oneline -10

# 若需要整理，使用 interactive rebase
git rebase -i HEAD~N  # N = 要整理的 commit 數量
# 在編輯器中：pick 保留、squash 合併到上一個、reword 改訊息
```

**Step 3：撰寫/確認每個 commit message**

按照 Conventional Commits 格式確認每個 commit：
```
feat(input): add CustomInput with v-model and $attrs passthrough
feat(select): add generic CustomSelect with type-safe value restoration
feat(form): add useLoginForm composable with validate and reset
feat(form): compose LoginForm with CustomInput and CustomSelect
fix(form): handle null modelValue in CustomSelect
```

**Step 4：Push 到遠端**
```bash
git push origin <branch-name>

# 若需要 force push（rebase 後）
git push --force-with-lease origin <branch-name>
```

**Step 5：在 GitHub 建立 PR**
- 標題：使用 Conventional Commits 格式
- Description：使用 3-2 節的 PR 說明模板
- Labels：M2-W5、feat
- Reviewers：（依團隊慣例）

**Step 6：確認 CI 狀態**
- vue-tsc 是否通過
- （若有）Lint 是否通過

---

### 技術規範
- Commit message 必須符合 Conventional Commits
- PR 說明必須有 Why 說明（至少 3 個設計決策）
- PR 說明必須有交付清單（checkbox）

### 評分標準（PR 品質）
| 項目 | 滿分 | 標準 |
|------|------|------|
| Commit 整潔度 | 25 | 每個 commit 概念清晰、格式規範 |
| PR 說明完整性 | 35 | 有交付清單 + Why 決策（≥3） + 測試方式 |
| 程式碼型別安全 | 25 | vue-tsc 零警告 |
| 整體工程品質 | 15 | 無 console.log 殘留、命名一致 |

---

## 五、自我檢核問題

**Q1：Conventional Commits 中，`feat` 和 `refactor` 的差異是什麼？什麼情況下應該用哪個？**

<details>
<summary>參考答案</summary>

`feat` 表示新增了一個使用者可感知的「新能力」（例如新元件、新函式）。`refactor` 表示改了程式碼的組織或結構，但使用者從外部看不出任何行為變化（例如把 inline 邏輯搬進 Composable，但呼叫端和功能完全相同）。

判斷標準：「如果我不告訴使用端，他們會察覺到差異嗎？」有察覺 → feat（或 fix）；無察覺 → refactor。

</details>

---

**Q2：為什麼 git rebase -i 後需要使用 `--force-with-lease` 而不是 `--force`？**

<details>
<summary>參考答案</summary>

`--force` 直接覆蓋遠端分支，如果在你 rebase 的過程中，有其他人 push 了新的 commit 到同一分支，你的 `--force` 會把他的工作覆蓋掉，不留痕跡。

`--force-with-lease` 會先確認「你對遠端分支的認知（你最後一次 fetch 的狀態）是否與遠端當前狀態一致」，如果遠端有你不知道的新 commit，操作會失敗並提示你先 pull，從而保護其他人的工作。

`--force-with-lease` = 「我要強制 push，但前提是沒有我不知道的變更」。

</details>

---

**Q3：一個好的 PR 說明，其「設計決策 Why」段落，如何決定要包含哪些決策點？**

<details>
<summary>參考答案</summary>

包含讓「熟悉技術但不熟悉這個 PR 的人」會感到疑惑的決策：

1. **不明顯的選擇**：為什麼用 `options.find` 而不是直接 emit？為什麼 validate 回傳 boolean？
2. **有取捨的設計**：為什麼選 `reactive` 不選多個 `ref`？（說明 trade-off）
3. **繞過直覺的做法**：`inheritAttrs: false` 通常讓人覺得奇怪，需要解釋
4. **未來會影響的決策**：`T extends string | number` 的邊界限制，以後擴充時要知道

不需要包含：顯而易見的實作細節（「CustomInput 接受 modelValue prop」）、TypeScript 語法解釋（reviewer 應懂）。

</details>

---

**Q4：如果 W3、W4 作業還沒 Push，今天應該先處理哪個？為什麼？**

<details>
<summary>參考答案</summary>

優先處理 **W5**，因為它今天截止（明日截止）。W3、W4 已超過截止日，追加提交也不能改變截止狀態，但至少要讓正在進行的 W5 不要也落後。

處理順序：
1. 先確保 W5 Push + PR 今天完成（最重要）
2. 再視時間，整理 W3 的 branch 並補發 PR（標注 late submission）
3. W4 同上

> **已超期的作業發 PR 仍有意義**：展示完成意願、獲得 Code Review 回饋、建立 Git 歷史。即使評分可能打折，作業品質的提升比不提交更好。

</details>

---

**Q5：W5 七天下來，你的「Composable 設計」能力提升了多少？能舉出最具體的證明？**

<details>
<summary>參考答案</summary>

基線（4/23）：1.0（概念模糊）→ W5 Day 6：4.7（能設計可複用 Composable）

最具體的證明：**useLoginForm** 的設計：
- State 層（reactive 表單物件）與 UI 層（CustomInput/Select）完全解耦
- `validate()` 回傳 boolean — 介面設計考慮了呼叫端的決策流程
- `reset()` 用 Object.keys 逐項賦值 — 不整批替換以保留響應性
- 回傳型別顯式標注 — 作為 API 契約而不只是推導結果

這些設計決策在半年前是無法「獨立想到並說清楚為什麼」的。現在不僅能實作，還能用 Why 角度寫進 PR 說明。

</details>

---

## 六、明日預告（W5 截止日 / W6 Day 1 預備）

### 5/25（明日）雙重任務

**任務一（W5 截止日確認）**：
- 確認 W5 PR 已發出（GitHub 顯示 Open 狀態）
- 確認 CI/Lint 通過
- 若尚未 Push，今日務必完成

**任務二（W6 Day 1 啟動）**：

W6 主題：`provide` / `inject`、Slots、動態元件

```
預習問題（可先思考）：
1. provide/inject 解決什麼問題？
   → Props 需要一層一層傳遞（Props Drilling）；
     provide/inject 讓祖先直接提供資料給任意後代

2. Slots 與 Props 傳資料的根本差異？
   → Props 傳「資料」；Slots 傳「UI 結構」
     Slots 讓父元件決定「這個空間裡放什麼」

3. 動態元件是什麼？
   → <component :is="..." /> 讓元件類型本身成為資料驅動的
     常見於 Tab 切換、動態表單類型切換
```

W6 預期產出：**可插拔 UI 元件庫雛形**（利用 Slots 設計彈性元件）
