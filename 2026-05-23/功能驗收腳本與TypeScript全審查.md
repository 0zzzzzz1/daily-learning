# 功能驗收腳本 × TypeScript 全審查

> **日期**：2026-05-23（M2-W5 Day 5）
> **所屬週次**：W5（5/19–5/25）
> **截止日**：2026-05-25（剩 **2 天**）

---

## 一、目標技術與核心知識點

| 技術 | 核心知識點 |
|------|-----------|
| TypeScript | `vue-tsc --noEmit` 靜態分析、`any` 消除、函式簽名完整性 |
| Vue 3 元件設計 | 邊緣 case 防禦（空值、極端輸入、快速互動） |
| 工程實踐 | 系統化功能驗收腳本設計、Code Review 自審角度 |
| PR 文化 | 說明設計決策（Why），而非只描述功能（What） |

---

## 二、為什麼學這個（與前幾日的連結）

```
W5 知識積累脈絡

Day 1（5/19）理論建立
  └── defineProps / defineEmits / v-model 本質 × defineModel

Day 2（5/20）實作落地
  └── CustomInput.vue 完整實作 × event.target 型別斷言

Day 3（5/21）精進層
  └── Event 型別鏈 × $attrs Fallthrough × useLoginForm × CustomSelect 骨架

Day 4（5/22）整合層
  └── CustomSelect 完整實作 × LoginForm.vue 三元件組裝 × 驗證策略設計

Day 5（今日）驗收層  ← 你在這裡
  └── 「能跑」→「能 Review」的距離
      功能驗收腳本 × TypeScript 全審查 × 邊緣 case 補強 × PR 說明草稿
```

**為什麼今天是關鍵**：W5 作業在 Day 4 已達到「能跑」狀態。但「能跑」只是功能正確，還不夠交付。
今天要走完「能 Review」需要的最後一哩路：型別安全確認、行為邊界測試、程式碼自審。

> **工程師的專業度，就體現在「能跑」和「能 Review」之間的這段距離。**

---

## 三、知識說明

### 3-1｜系統化功能驗收腳本設計

功能驗收腳本是「人工版的 Unit Test」，目的是確保所有功能路徑都被測試，而不是靠感覺點一點。

**W5 作業的功能驗收腳本（12 步驟）：**

---

#### 模組一：CustomInput 基本行為

**步驟 1：v-model 雙向綁定確認**
```
操作：在帳號欄輸入「testuser」
確認：父元件 form.username 即時更新為 "testuser"（可透過 Vue DevTools 觀察）
```

**步驟 2：$attrs Fallthrough 確認**
```
操作：在 LoginForm.vue 的帳號 CustomInput 加上 maxlength="10"，輸入超過 10 個字
確認：瀏覽器限制輸入（<input> 上有 maxlength="10"），而非 CustomInput 的 wrapper div
```

**步驟 3：errorMessage 顯示確認**
```
操作：點擊「登入」，保持帳號欄位空白
確認：帳號欄下方出現「帳號不得為空」錯誤訊息
確認：錯誤訊息的 DOM 節點是 v-if（非空才存在），不是 v-show（樣式隱藏）
```

**步驟 4：錯誤清除確認**
```
操作：觸發帳號錯誤後，填入正確值再次點擊「登入」
確認：帳號錯誤訊息消失
```

---

#### 模組二：CustomSelect 泛型行為

**步驟 5：v-model 雙向綁定確認**
```
操作：選擇「一般用戶（user）」
確認：form.role 更新為 "user"（字串型別，非 undefined）
確認：型別是 string，不是 "user" 以外的值
```

**步驟 6：null 值初始狀態確認**
```
操作：頁面載入後，直接觀察 CustomSelect
確認：顯示 placeholder「請選擇角色」（<select> 預設無選中項）
確認：form.role 為 null，不是 "" 或 undefined
```

**步驟 7：重設後回到 null 確認**
```
操作：選擇角色 → 點擊「重設」
確認：下拉選單回到 placeholder 狀態（placeholder option 被選中）
確認：form.role 回到 null
```

**步驟 8：CustomSelect 錯誤訊息確認**
```
操作：不選角色，點擊「登入」
確認：CustomSelect 下方出現「請選擇角色」錯誤訊息
```

---

#### 模組三：LoginForm 整合行為

**步驟 9：完整驗證確認（所有欄位空白）**
```
操作：三個欄位全空，點擊「登入」
確認：三個錯誤訊息同時顯示
  - 帳號：「帳號不得為空」
  - 密碼：「密碼至少 6 個字元」
  - 角色：「請選擇角色」
```

**步驟 10：密碼長度邊界確認**
```
操作：密碼輸入剛好 5 個字元，點擊「登入」
確認：顯示密碼錯誤
操作：密碼輸入剛好 6 個字元
確認：密碼錯誤消失（6 個字元是合法邊界）
```

**步驟 11：@blur 觸發驗證確認**
```
操作：輸入帳號後，直接按 Tab 移到密碼欄位（觸發 blur）
確認：若帳號有效，帳號錯誤不顯示；若帳號空白，blur 後立即顯示錯誤
```

**步驟 12：表單完整提交確認**
```
操作：帳號填「admin」、密碼填「123456」、角色選「管理員」，點擊「登入」
確認：console.log 輸出 { username: "admin", password: "123456", role: "admin" }
確認：提交後表單重設（所有欄位清空，錯誤消失）
```

---

### 3-2｜TypeScript 全審查流程

**系統化審查順序（由底向上）：**

```
Step 1：型別定義層
  搜尋：any（全文）、as unknown、!（non-null assertion）
  目標：消除所有不明確的型別斷言

Step 2：函式簽名層
  檢查：每個函式是否有明確的回傳型別標注
  重點：validate(): boolean、handleChange(event: Event): void
  NG：function validate() { ... }（TypeScript 推導，但缺乏 API 契約）
  OK：function validate(): boolean { ... }

Step 3：使用層
  執行：vue-tsc --noEmit
  目標：零警告、零錯誤
  常見問題：props 解構後失去型別、emit 參數型別不符

Step 4：邊緣 case 型別
  確認：null 值、空字串、undefined 在型別定義中是否都被涵蓋
```

**常見需要補強的型別：**

```ts
// ❌ 回傳型別靠推導，不明確
function validate() {
  return isValid
}

// ✅ 顯式標注，成為 API 契約
function validate(): boolean {
  return isValid
}

// ❌ event 型別不明
function handleChange(event: any) { ... }

// ✅ 精確 DOM 事件型別
function handleChange(event: Event) {
  const select = event.target as HTMLSelectElement
}

// ❌ options 型別過於寬鬆
const options: object[] = [...]

// ✅ 精確的泛型介面
interface Option<T> {
  label: string
  value: T
}
const options: Option<string>[] = [...]
```

---

### 3-3｜邊緣 Case 補強清單

在完成基本功能驗收後，需要考慮以下邊緣情境：

| 情境 | 元件 | 防禦方式 |
|------|------|---------|
| 帳號全空白（空格）| CustomInput | `form.username.trim()` 去除首尾空白後判斷 |
| 密碼為空字串 | CustomInput | `form.password.length < 6` 已涵蓋（空字串 length = 0）|
| 快速連點提交 | LoginForm | `validate()` 回傳 `false` 即停止，天然防禦 |
| 選單沒有任何 options | CustomSelect | `placeholder` option 仍可顯示，不會 crash |
| `form.role` 從 string 變回 null | useLoginForm / reset() | 逐項賦值：`form.role = null`，保留響應性 |
| `options` 的 value 是 number | CustomSelect | `generic="T extends string \| number"` 確保型別邊界；`String()` 轉換處理 |

---

### 3-4｜Code Review 自審角度

自審時，不只看程式碼能不能跑，而是問「為什麼這樣設計」：

**CustomInput.vue**
- ✅ 為什麼用 `as HTMLInputElement` 而不是 `instanceof`？
  → 在自訂元件內部，`<input>` 是我們自己放的，型別確定，`as` 已足夠安全
- ✅ 為什麼 errorMessage 用 `v-if` 而不是 `v-show`？
  → 錯誤訊息不需要保留 DOM 佔位；v-if 更精確，不浪費節點
- ✅ 為什麼 `$attrs` 要 `inheritAttrs: false`？
  → 否則 Vue 預設把 attrs 套到根元素（wrapper div），而不是 `<input>`

**CustomSelect.vue**
- ✅ 為什麼用 `options.find` 而不直接 `emit(select.value)`？
  → HTML `.value` 永遠是字串；find 還原原始型別，維持型別安全
- ✅ 為什麼 null 值要處理成空字串給 `:value`？
  → `null` 傳給 `:value` 會被 HTML 轉成字串 "null"，造成視覺錯誤

**useLoginForm.ts**
- ✅ 為什麼 `validate()` 要回傳 `boolean`？
  → 讓呼叫端自主決定後續行為；否則呼叫端需要額外讀取 `errors`，耦合度更高
- ✅ 為什麼 `reset()` 用逐項賦值而非 `Object.assign`？
  → 明確、不依賴外部物件；響應性保留更直觀

---

### 3-5｜W5 PR 說明草稿結構

PR 說明的核心不是「我做了什麼」，而是「為什麼這樣設計」。

```markdown
## W5 作業：雙向綁定自訂元件（CustomInput × CustomSelect × LoginForm）

### 交付清單
- [x] `CustomInput.vue`：v-model 雙向綁定、$attrs Fallthrough、錯誤訊息顯示
- [x] `CustomSelect.vue`：泛型 v-model（`generic="T extends string | number"`）、null 值處理
- [x] `LoginForm.vue`：三元件整合、表單提交與重設、@blur + 提交雙重驗證
- [x] `useLoginForm.ts`：reactive 表單狀態、validate（回傳 boolean）、reset
- [x] TypeScript：vue-tsc 零警告、無 any

### 主要設計決策

**1. CustomSelect 的型別陷阱處理**
HTML `<select>` 的 `.value` 永遠是字串，即使 option 是 number。
解法：emit 前用 `options.find(opt => String(opt.value) === select.value)` 還原原始型別。
這樣泛型 T 的型別安全才能從 Composable 一路保持到父元件。

**2. validate() 回傳 boolean 的設計意圖**
讓呼叫端（LoginForm.vue 的 handleSubmit）可以 `if (!validate()) return` 立即停止。
若 validate 不回傳值，呼叫端需要另外讀取 errors 物件，增加耦合度。

**3. @blur 驗證 + 提交再驗證的雙重策略**
單純 @blur：離開欄位才顯示錯誤，但若用戶直接按 Submit 可能繞過。
單純提交驗證：用戶填完整份表單才知道哪裡錯，體驗差。
雙重策略：兩者取長補短，也是業界表單常見的標準模式。

### 未決問題 / 後續優化
- [ ] `CustomSelect` 尚未支援 `multiple` 多選模式（M2 作業範圍外）
- [ ] 驗證規則目前硬編碼在 `useLoginForm`；未來可抽成 validator 函式陣列（Rule-based validation）

### 測試
- [x] 12 步驟功能驗收腳本全部通過（詳見 2026-05-23/功能驗收腳本與TypeScript全審查.md）
- [x] vue-tsc --noEmit：零警告
```

---

## 四、作業說明

### 今日任務（W5 Day 5 實作清單）

| # | 任務 | 預計時間 |
|---|------|---------|
| 1 | 執行 12 步驟功能驗收腳本，記錄每步驟結果 | 30 分鐘 |
| 2 | 執行 `vue-tsc --noEmit`，消除所有警告 | 20 分鐘 |
| 3 | 補強已識別的邊緣 case（尤其是空白帳號、null 值 reset）| 15 分鐘 |
| 4 | Code Review 自審：對每個設計決策能口頭說出「為什麼」| 20 分鐘 |
| 5 | 撰寫 W5 PR 說明草稿（參考 3-5 節的結構）| 15 分鐘 |

### 評分標準（承接 Day 4，最終作業總分）

| 項目 | 配分 | 說明 |
|------|------|------|
| CustomInput 功能完整 | 20 | v-model、errorMessage、$attrs Fallthrough |
| CustomSelect 泛型實作 | 25 | generic="T"、型別安全、null 處理 |
| LoginForm 整合正確 | 20 | 三個欄位正確接線、資料流無誤 |
| 表單驗證邏輯 | 20 | validate/reset 行為正確、邊緣 case 防禦 |
| TypeScript 品質 | 15 | vue-tsc 零警告、無 any、回傳型別明確 |

---

## 五、自我檢核問題

1. 功能驗收腳本和 Unit Test 有什麼關係？為什麼說腳本是「人工版 Unit Test」？

2. `vue-tsc --noEmit` 和 `tsc --noEmit` 的差異是什麼？為什麼在 Vue 3 專案要用 `vue-tsc`？

3. 執行 TypeScript 全審查時，為什麼審查順序是「型別定義層 → 函式簽名層 → 使用層」，而不是反過來？

4. PR 說明應該著重「為什麼」而非「是什麼」。請試著用一句話解釋：為什麼 CustomSelect 要用 `options.find` 而不直接 `emit(select.value)`？（用讓不了解這個陷阱的人也能看懂的方式）

5. 以下哪個邊緣 case 目前的程式碼沒有處理好？應該怎麼修？
   ```
   情境：帳號輸入「   」（三個空格），點擊登入
   預期：顯示「帳號不得為空」
   ```

---

## 六、明日預告

**5/24（W5 Day 6）：收尾層 — Commit 整理 × PR 說明定稿**

- Conventional Commits 格式整理（feat/fix/refactor/docs 各自歸位）
- PR 說明草稿 → 定稿（根據今日自審結果補充設計決策）
- `git log --oneline` 確認 commit 歷史清晰
- 最終 push 確認（branch 名稱、PR base branch 確認）
- W5 截止 5/25，明日（5/24）是最後一個完整工作日

> **提醒**：W3、W4 的 PR 若尚未確認，明日也一起處理。不要讓舊作業積壓到 W5 截止日。
