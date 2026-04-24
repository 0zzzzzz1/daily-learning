# Daily Learning 報告 — 2026-04-24（週五）

## 執行狀態

本次排程任務為「每日前端學習計畫」，預期流程為：
1. 讀取 Notion 上的計畫內容
2. 根據需求執行對應行動

**本次無法完成讀取步驟**，原因如下：

| 項目 | 狀態 | 說明 |
|------|------|------|
| `web_fetch` 直接抓取 Notion 頁面 | 失敗 | `www.notion.so` 未在網路允許清單內（cowork-egress-blocked）|
| Notion MCP 連接器 | 未認證 | `mcp__plugin_design_notion__*` 需要 OAuth 授權，但排程執行時使用者不在場 |

---

## 建議的後續設定（使用者下次操作時）

為讓未來的排程能順利讀取 Notion 計畫內容，可擇一設定：

1. **（推薦）完成 Notion MCP 認證**：在 Cowork 中對 Claude 說「請連接 Notion」，走完 OAuth 流程。完成後排程即可直接使用 Notion API 讀取頁面，遠比網頁抓取穩定精確。
2. **將 Notion 加入網路允許清單**：Settings → Capabilities 中將 `www.notion.so` 加到允許的 domain。此方式僅能取得網頁的 public HTML，若頁面非公開仍無法讀取。
3. **備援方案**：把每日計畫的重點內容直接寫進 `SKILL.md`（此檔案是排程能直接讀取的本地檔案），Notion 網址可作為參考連結保留。

---

## 今日前端學習建議（暫代方案）

在無法讀取 Notion 原計畫的情況下，依據「每日前端學習」主題，提供今日一則聚焦學習建議，預估時長 30–45 分鐘：

### 今日主題：React 19 的 `use` hook 與資料讀取模式

**為什麼選這題**
- React 19 已正式釋出一段時間，`use` hook 是與過去 `useEffect + fetch` 模式差異最大的新 API
- 是目前業界面試與 code review 的熱門話題，投報率高
- 可直接套用到 Suspense / Server Components 的理解

**學習路徑（建議 30–45 分鐘）**

1. **閱讀（10 分鐘）**：React 官方文件 `use` API 章節，重點看它能吃 Promise 與 Context 兩種用法
2. **動手（20 分鐘）**：在 CodeSandbox 建立一個最小範例
   - 父元件用 `<Suspense fallback={...}>` 包住子元件
   - 子元件呼叫 `const data = use(fetchPromise)` 讀取資料
   - 比較與 `useEffect + useState` 寫法的差異（少了 loading state、少了 race condition 處理）
3. **對比思考（10 分鐘）**：
   - `use` vs `useEffect`：控制權從元件轉移到父層 Suspense
   - `use` vs React Query / SWR：何時該用內建、何時仍該用第三方
   - 在 Server Component 裡用 `use` 有什麼差異

**自我檢核問題**
- `use(promise)` 可以放在條件判斷或迴圈裡嗎？（答案：可以，與 Rules of Hooks 不同）
- 如果每次 render 都產生新 Promise，會怎樣？（答案：會無限 suspend — 需要穩定 Promise 來源）
- Suspense fallback 與 error boundary 該放哪層？

**延伸**
- 若已熟悉 `use`，可往「Server Actions + `useActionState`」延伸
- 也可研究 `useOptimistic` 在表單提交的樂觀更新應用

---

## 備註

- 此份為排程自動產出的備援報告。若 Notion 計畫內容與上述建議不同，請以 Notion 實際內容為準
- 建議先處理 Notion MCP 認證，讓下次排程能讀到真實計畫內容
