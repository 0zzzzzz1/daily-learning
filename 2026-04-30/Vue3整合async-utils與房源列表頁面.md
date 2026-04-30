# W2 Day 3：Vue 3 整合 async-utils × 房源列表頁面

> **日期**：2026-04-30（M1-W2 Day 3）
> **目標**：將 4/29 完成的 `async-utils` 工具庫整合進真實 Vue 3 Vite 專案，建立具備 Loading Skeleton 效果的房源列表頁面
> **預計時長**：3–3.5 小時

---

## 今日學習目標

1. 理解如何在 Vite 專案中組織 `async-utils` 模組的目錄結構
2. 實作具備完整狀態管理（loading / error / data）的 `PropertyList` 元件
3. 建立 `SkeletonCard` 元件，掌握 Skeleton Loading UI 的設計邏輯
4. 整合 `useAsyncState` + `withRetry` + `withCache` 完成真實資料流
5. 理解 Vue 3 `onMounted`、`watch` 在非同步場景中的正確用法

---

## 為什麼今天學這個？

過去兩天（4/28–4/29）我們完成了：
- **理論**：Promise、async/await、ESM 模組系統的完整知識建立
- **工具庫**：5 個非同步封裝模組的實作（httpClient、retry、cache、concurrent、useAsyncState）

但「知道怎麼寫」和「知道怎麼用在真實專案」中間，還有一道牆：

> **工具庫只是材料，元件整合才是建築**

今天的目標是把這兩天的成果真正「用起來」，在真實頁面中驗證設計是否正確，並補上 UI 體驗的最後一塊拼圖：**Skeleton Loading**。

---

## 專案結構設計

在開始寫程式碼之前，先確立目錄結構：

```
src/
├── async-utils/              ← 4/28-4/29 完成的工具庫
│   ├── httpClient.ts
│   ├── retry.ts
│   ├── cache.ts
│   ├── concurrent.ts
│   ├── useAsyncState.ts
│   └── index.ts
│
├── types/
│   └── property.ts           ← 型別定義（今日新增）
│
├── services/
│   └── propertyService.ts    ← API 服務層（今日新增）
│
├── components/
│   ├── PropertyCard.vue      ← 單一房源卡片（今日新增）
│   ├── PropertyList.vue      ← 房源列表主元件（今日新增）
│   └── SkeletonCard.vue      ← Loading 佔位元件（今日新增）
│
└── App.vue                   ← 根元件（今日修改）
```

**設計原則**：職責分離
- `async-utils/` 是通用工具，不知道「房源」是什麼
- `services/` 知道房源 API 的細節，不知道 Vue 元件
- `components/` 負責 UI，透過 Composable 取得資料

---

## 步驟一：型別定義（`types/property.ts`）

先定義資料模型，讓後續所有程式碼都有型別安全保護：

```typescript
// src/types/property.ts

export type PropertyType = 'apartment' | 'house' | 'studio' | 'office'

export interface Property {
  id: number
  title: string
  address: string
  price: number           // 月租金（元）
  area: number            // 坪數
  type: PropertyType
  floor: number
  totalFloor: number
  isAvailable: boolean
  imageUrl?: string       // 可選，有些房源沒有圖片
  createdAt: string       // ISO 8601 格式
}

// 列表查詢參數（搜尋/篩選用）
export interface PropertyFilters {
  type?: PropertyType
  minPrice?: number
  maxPrice?: number
  minArea?: number
  isAvailable?: boolean
  page?: number
  pageSize?: number
}

// 分頁回應格式
export interface PaginatedResponse<T> {
  items: T[]
  total: number
  page: number
  pageSize: number
  totalPages: number
}
```

**學習重點**：
- `PropertyType` 使用 Union Type，比 `string` 更安全，IDE 會自動補全
- `imageUrl?: string` 的 `?` 表示此欄位是 optional，型別為 `string | undefined`
- `PaginatedResponse<T>` 是泛型介面，可用於任何列表資料（`PaginatedResponse<Property>` / `PaginatedResponse<User>`）

---

## 步驟二：服務層（`services/propertyService.ts`）

服務層負責封裝 API 呼叫細節，元件不需要知道 URL 怎麼拼：

```typescript
// src/services/propertyService.ts

import { createHttpClient, withRetry, withCache } from '../async-utils'
import type { Property, PropertyFilters, PaginatedResponse } from '../types/property'

// 建立 HTTP 客戶端（在服務層初始化，全域共用一個實例）
const client = createHttpClient('https://api.ihouse.example.com')

// ── 取得房源列表（加上 retry + cache）──────────────────────────────────
const _fetchProperties = (filters?: PropertyFilters) => {
  const params = new URLSearchParams()

  if (filters?.type)         params.set('type', filters.type)
  if (filters?.minPrice)     params.set('minPrice', String(filters.minPrice))
  if (filters?.maxPrice)     params.set('maxPrice', String(filters.maxPrice))
  if (filters?.isAvailable !== undefined) {
    params.set('isAvailable', String(filters.isAvailable))
  }
  if (filters?.page)         params.set('page', String(filters.page))
  if (filters?.pageSize)     params.set('pageSize', String(filters.pageSize))

  const queryString = params.toString()
  const url = `/properties${queryString ? `?${queryString}` : ''}`

  return client.get<PaginatedResponse<Property>>(url)
}

// 加上重試（最多 3 次，間隔 1.5s）
const _fetchWithRetry = (filters?: PropertyFilters) =>
  withRetry(() => _fetchProperties(filters), {
    maxRetries: 3,
    delay: 1500,
    onRetry: (attempt, error) => {
      console.warn(`[PropertyService] 第 ${attempt} 次重試，原因：${error.message}`)
    }
  })()

// 加上快取（3 分鐘 TTL）
export const fetchProperties = withCache(
  (filters?: PropertyFilters) => _fetchWithRetry(filters),
  3 * 60 * 1000
)

// ── 取得單一房源（不快取，確保資料即時）────────────────────────────────
export const fetchPropertyById = withRetry(
  (id: number) => client.get<Property>(`/properties/${id}`),
  { maxRetries: 2, delay: 1000 }
)

// ── Mock 資料（開發時使用，無後端也能測試）───────────────────────────────
export const fetchPropertiesMock = async (
  filters?: PropertyFilters
): Promise<PaginatedResponse<Property>> => {
  // 模擬網路延遲（300–800ms 隨機）
  await new Promise(resolve => setTimeout(resolve, 300 + Math.random() * 500))

  // 模擬偶發失敗（10% 機率）
  if (Math.random() < 0.1) {
    throw new Error('模擬網路錯誤（10% 機率觸發，用於測試 retry 邏輯）')
  }

  const mockData: Property[] = [
    {
      id: 1, title: '台北信義區精裝三房', address: '台北市信義區松仁路 100 號',
      price: 55000, area: 35, type: 'apartment', floor: 12, totalFloor: 20,
      isAvailable: true, imageUrl: 'https://picsum.photos/seed/prop1/400/250',
      createdAt: '2026-04-01T10:00:00Z'
    },
    {
      id: 2, title: '大安區溫馨套房', address: '台北市大安區復興南路一段 200 號',
      price: 18000, area: 10, type: 'studio', floor: 5, totalFloor: 8,
      isAvailable: true, imageUrl: 'https://picsum.photos/seed/prop2/400/250',
      createdAt: '2026-04-05T14:30:00Z'
    },
    {
      id: 3, title: '內湖科學園區獨棟', address: '台北市內湖區瑞光路 300 號',
      price: 120000, area: 80, type: 'house', floor: 1, totalFloor: 3,
      isAvailable: false, imageUrl: 'https://picsum.photos/seed/prop3/400/250',
      createdAt: '2026-03-20T09:00:00Z'
    },
    {
      id: 4, title: '中山區辦公室出租', address: '台北市中山區南京東路二段 50 號',
      price: 35000, area: 25, type: 'office', floor: 8, totalFloor: 15,
      isAvailable: true, createdAt: '2026-04-10T11:00:00Z'
    },
    {
      id: 5, title: '松山區整層住宅', address: '台北市松山區南京東路五段 150 號',
      price: 42000, area: 28, type: 'apartment', floor: 3, totalFloor: 7,
      isAvailable: true, imageUrl: 'https://picsum.photos/seed/prop5/400/250',
      createdAt: '2026-04-15T16:00:00Z'
    },
    {
      id: 6, title: '文山區景觀套房', address: '台北市文山區木柵路四段 80 號',
      price: 15000, area: 8, type: 'studio', floor: 10, totalFloor: 12,
      isAvailable: true, imageUrl: 'https://picsum.photos/seed/prop6/400/250',
      createdAt: '2026-04-20T08:00:00Z'
    },
  ]

  // 套用篩選
  let filtered = filters?.isAvailable !== undefined
    ? mockData.filter(p => p.isAvailable === filters.isAvailable)
    : mockData

  if (filters?.type) {
    filtered = filtered.filter(p => p.type === filters.type)
  }

  const page = filters?.page ?? 1
  const pageSize = filters?.pageSize ?? 6
  const total = filtered.length
  const items = filtered.slice((page - 1) * pageSize, page * pageSize)

  return {
    items,
    total,
    page,
    pageSize,
    totalPages: Math.ceil(total / pageSize)
  }
}
```

**學習重點**：

1. **服務層職責**：只負責「怎麼取得資料」，不涉及 UI 狀態
2. **裝飾器組合**：`withCache(withRetry(...))` — 外層 cache 防止重複請求，內層 retry 處理網路不穩
3. **Mock 資料**：開發階段用 Mock 替換真實 API，切換只需改一行 import

---

## 步驟三：`SkeletonCard` 元件

**為什麼要有 Skeleton？**

傳統的 Loading 是一個旋轉圓圈（Spinner），缺點是：
- 沒有視覺預期 → 使用者不知道會出現什麼形狀的內容
- 「閃爍感」強 → 資料出現時版面劇烈跳動（Layout Shift）

Skeleton Loading 的優點：
- 先顯示「內容的輪廓形狀」→ 使用者有預期，不焦慮
- 版面不跳動 → 資料載入後，內容直接填入相同位置

```vue
<!-- src/components/SkeletonCard.vue -->
<script setup lang="ts">
// 無需任何邏輯，純展示元件
</script>

<template>
  <div class="skeleton-card">
    <!-- 圖片佔位 -->
    <div class="skeleton-image skeleton-pulse" />

    <div class="skeleton-body">
      <!-- 標題佔位（較長） -->
      <div class="skeleton-line skeleton-pulse" style="width: 75%" />

      <!-- 地址佔位（較短） -->
      <div class="skeleton-line skeleton-pulse" style="width: 55%" />

      <!-- 標籤行佔位 -->
      <div class="skeleton-tags">
        <div class="skeleton-tag skeleton-pulse" />
        <div class="skeleton-tag skeleton-pulse" />
      </div>

      <!-- 價格佔位 -->
      <div class="skeleton-price skeleton-pulse" style="width: 40%" />
    </div>
  </div>
</template>

<style scoped>
/* ── 基底樣式 ────────────────────────────────────── */
.skeleton-card {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  overflow: hidden;
  background: #ffffff;
}

.skeleton-image {
  width: 100%;
  height: 180px;
  background: #e5e7eb;
}

.skeleton-body {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.skeleton-line {
  height: 16px;
  background: #e5e7eb;
  border-radius: 4px;
}

.skeleton-tags {
  display: flex;
  gap: 8px;
}

.skeleton-tag {
  height: 24px;
  width: 60px;
  background: #e5e7eb;
  border-radius: 12px;
}

.skeleton-price {
  height: 20px;
  background: #e5e7eb;
  border-radius: 4px;
  margin-top: 4px;
}

/* ── 動畫：shimmer 光暈效果 ──────────────────────── */
.skeleton-pulse {
  background: linear-gradient(
    90deg,
    #e5e7eb 25%,
    #f3f4f6 50%,
    #e5e7eb 75%
  );
  background-size: 200% 100%;
  animation: shimmer 1.5s infinite;
}

@keyframes shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
</style>
```

**學習重點 — `shimmer` 動畫原理**：

```
background: linear-gradient(90deg, #灰色 25%, #淺灰 50%, #灰色 75%)
background-size: 200% 100%   ← 背景比元素寬 2 倍，這樣才有空間滑動
animation: background-position 從 200% 滑到 -200%
```

視覺效果：灰色背景上有一道「光」從右向左掃過，模擬載入中的感覺。

---

## 步驟四：`PropertyCard` 元件

顯示單一房源資訊的卡片，**結構必須和 `SkeletonCard` 相同**，才能無縫切換：

```vue
<!-- src/components/PropertyCard.vue -->
<script setup lang="ts">
import type { Property } from '../types/property'

const props = defineProps<{
  property: Property
}>()

// 型別標籤的中文對應
const typeLabel: Record<Property['type'], string> = {
  apartment: '公寓',
  house: '透天',
  studio: '套房',
  office: '辦公室',
}

// 格式化租金（每千加逗號）
const formatPrice = (price: number): string =>
  price.toLocaleString('zh-TW')

// 樓層顯示
const floorDisplay = (floor: number, total: number): string =>
  `${floor}/${total} 樓`
</script>

<template>
  <div class="property-card" :class="{ 'unavailable': !property.isAvailable }">
    <!-- 房源圖片 -->
    <div class="card-image-wrapper">
      <img
        v-if="property.imageUrl"
        :src="property.imageUrl"
        :alt="property.title"
        class="card-image"
        loading="lazy"
      />
      <div v-else class="card-image-placeholder">
        <span>🏠</span>
      </div>

      <!-- 不可用標籤 -->
      <div v-if="!property.isAvailable" class="unavailable-badge">
        已租出
      </div>
    </div>

    <!-- 房源資訊 -->
    <div class="card-body">
      <h3 class="card-title">{{ property.title }}</h3>

      <p class="card-address">📍 {{ property.address }}</p>

      <!-- 標籤列 -->
      <div class="card-tags">
        <span class="tag tag-type">{{ typeLabel[property.type] }}</span>
        <span class="tag tag-area">{{ property.area }} 坪</span>
        <span class="tag tag-floor">{{ floorDisplay(property.floor, property.totalFloor) }}</span>
      </div>

      <!-- 租金 -->
      <p class="card-price">
        NT$ <strong>{{ formatPrice(property.price) }}</strong> / 月
      </p>
    </div>
  </div>
</template>

<style scoped>
.property-card {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  overflow: hidden;
  background: #ffffff;
  transition: transform 0.2s, box-shadow 0.2s;
  cursor: pointer;
}

.property-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
}

.property-card.unavailable {
  opacity: 0.6;
}

/* ── 圖片 ──────────────────── */
.card-image-wrapper {
  position: relative;
  height: 180px;
  background: #f3f4f6;
}

.card-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.card-image-placeholder {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 48px;
}

.unavailable-badge {
  position: absolute;
  top: 12px;
  right: 12px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 12px;
}

/* ── 卡片內容 ──────────────── */
.card-body {
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.card-title {
  font-size: 16px;
  font-weight: 600;
  color: #111827;
  margin: 0;
  /* 超過 2 行截斷 */
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.card-address {
  font-size: 13px;
  color: #6b7280;
  margin: 0;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.card-tags {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}

.tag {
  padding: 3px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 500;
}

.tag-type  { background: #dbeafe; color: #1d4ed8; }
.tag-area  { background: #dcfce7; color: #15803d; }
.tag-floor { background: #fef9c3; color: #a16207; }

.card-price {
  margin: 4px 0 0;
  font-size: 14px;
  color: #374151;
}

.card-price strong {
  font-size: 18px;
  color: #dc2626;
}
</style>
```

---

## 步驟五：`PropertyList` 元件（整合核心）

這是今日最重要的元件，整合 `useAsyncState`、Skeleton、篩選功能：

```vue
<!-- src/components/PropertyList.vue -->
<script setup lang="ts">
import { ref, watch, computed } from 'vue'
import { useAsyncState } from '../async-utils'
import { fetchPropertiesMock } from '../services/propertyService'
import PropertyCard from './PropertyCard.vue'
import SkeletonCard from './SkeletonCard.vue'
import type { PropertyFilters, PropertyType } from '../types/property'

// ── 篩選狀態 ──────────────────────────────────────────────────────────
const filters = ref<PropertyFilters>({
  isAvailable: undefined,   // undefined = 全部，true = 可租，false = 已租出
  type: undefined,
  page: 1,
  pageSize: 6,
})

// ── 非同步狀態（useAsyncState 整合）────────────────────────────────────
const {
  data: result,
  isLoading,
  error,
  execute: fetchList,
} = useAsyncState(() => fetchPropertiesMock(filters.value))

// ── 計算屬性 ─────────────────────────────────────────────────────────
const properties = computed(() => result.value?.items ?? [])
const totalPages  = computed(() => result.value?.totalPages ?? 0)
const totalCount  = computed(() => result.value?.total ?? 0)

// 顯示幾個 Skeleton（等於 pageSize，視覺上與真實列表一致）
const skeletonCount = computed(() => filters.value.pageSize ?? 6)

// ── 監聽篩選條件變更，自動重新載入 ────────────────────────────────────
watch(
  filters,
  () => {
    fetchList()
  },
  {
    deep: true,   // 深層監聽，filters 內任何子屬性改變都觸發
    immediate: true  // 元件掛載後立即執行一次
  }
)

// ── 篩選操作 ──────────────────────────────────────────────────────────
const setType = (type: PropertyType | undefined) => {
  filters.value = { ...filters.value, type, page: 1 }  // 切換類型時重設到第一頁
}

const setAvailability = (isAvailable: boolean | undefined) => {
  filters.value = { ...filters.value, isAvailable, page: 1 }
}

const goToPage = (page: number) => {
  filters.value = { ...filters.value, page }
  // 自動滾動到頂部
  window.scrollTo({ top: 0, behavior: 'smooth' })
}
</script>

<template>
  <section class="property-list-section">
    <header class="list-header">
      <h2 class="list-title">房源列表</h2>
      <p v-if="!isLoading && !error" class="list-count">
        共 {{ totalCount }} 筆房源
      </p>
    </header>

    <!-- 篩選列 -->
    <div class="filter-bar">
      <!-- 類型篩選 -->
      <div class="filter-group">
        <span class="filter-label">類型</span>
        <button
          class="filter-btn"
          :class="{ active: filters.type === undefined }"
          @click="setType(undefined)"
        >全部</button>
        <button
          class="filter-btn"
          :class="{ active: filters.type === 'apartment' }"
          @click="setType('apartment')"
        >公寓</button>
        <button
          class="filter-btn"
          :class="{ active: filters.type === 'studio' }"
          @click="setType('studio')"
        >套房</button>
        <button
          class="filter-btn"
          :class="{ active: filters.type === 'house' }"
          @click="setType('house')"
        >透天</button>
        <button
          class="filter-btn"
          :class="{ active: filters.type === 'office' }"
          @click="setType('office')"
        >辦公室</button>
      </div>

      <!-- 可用性篩選 -->
      <div class="filter-group">
        <span class="filter-label">狀態</span>
        <button
          class="filter-btn"
          :class="{ active: filters.isAvailable === undefined }"
          @click="setAvailability(undefined)"
        >全部</button>
        <button
          class="filter-btn"
          :class="{ active: filters.isAvailable === true }"
          @click="setAvailability(true)"
        >可租</button>
        <button
          class="filter-btn"
          :class="{ active: filters.isAvailable === false }"
          @click="setAvailability(false)"
        >已租出</button>
      </div>
    </div>

    <!-- 錯誤狀態 -->
    <div v-if="error" class="error-state">
      <p class="error-icon">⚠️</p>
      <p class="error-message">{{ error }}</p>
      <button class="retry-btn" @click="fetchList">重新載入</button>
    </div>

    <!-- 資料格子（loading 時顯示 Skeleton，有資料時顯示卡片）-->
    <div v-else class="property-grid">
      <!-- Loading：顯示 Skeleton -->
      <template v-if="isLoading">
        <SkeletonCard
          v-for="n in skeletonCount"
          :key="`skeleton-${n}`"
        />
      </template>

      <!-- 有資料：顯示卡片 -->
      <template v-else>
        <PropertyCard
          v-for="property in properties"
          :key="property.id"
          :property="property"
        />
      </template>

      <!-- 空結果 -->
      <div
        v-if="!isLoading && properties.length === 0"
        class="empty-state"
      >
        <p>😅 找不到符合條件的房源</p>
        <button @click="setType(undefined); setAvailability(undefined)">
          清除篩選
        </button>
      </div>
    </div>

    <!-- 分頁 -->
    <div
      v-if="!isLoading && !error && totalPages > 1"
      class="pagination"
    >
      <button
        class="page-btn"
        :disabled="filters.page === 1"
        @click="goToPage((filters.page ?? 1) - 1)"
      >← 上一頁</button>

      <span class="page-info">
        第 {{ filters.page }} / {{ totalPages }} 頁
      </span>

      <button
        class="page-btn"
        :disabled="filters.page === totalPages"
        @click="goToPage((filters.page ?? 1) + 1)"
      >下一頁 →</button>
    </div>
  </section>
</template>

<style scoped>
/* ── 整體佈局 ──────────────────── */
.property-list-section {
  max-width: 1200px;
  margin: 0 auto;
  padding: 24px 16px;
}

.list-header {
  display: flex;
  align-items: baseline;
  gap: 12px;
  margin-bottom: 16px;
}

.list-title {
  font-size: 24px;
  font-weight: 700;
  color: #111827;
  margin: 0;
}

.list-count {
  font-size: 14px;
  color: #6b7280;
  margin: 0;
}

/* ── 篩選列 ─────────────────────── */
.filter-bar {
  display: flex;
  gap: 16px;
  flex-wrap: wrap;
  margin-bottom: 24px;
  padding: 12px 16px;
  background: #f9fafb;
  border-radius: 10px;
}

.filter-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

.filter-label {
  font-size: 13px;
  color: #6b7280;
  white-space: nowrap;
}

.filter-btn {
  padding: 5px 14px;
  border: 1px solid #d1d5db;
  border-radius: 20px;
  background: white;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.15s;
}

.filter-btn:hover {
  border-color: #6366f1;
  color: #6366f1;
}

.filter-btn.active {
  background: #6366f1;
  border-color: #6366f1;
  color: white;
}

/* ── 格子 ───────────────────────── */
.property-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
}

/* ── 錯誤 / 空狀態 ──────────────── */
.error-state,
.empty-state {
  grid-column: 1 / -1;
  text-align: center;
  padding: 48px 16px;
  color: #6b7280;
}

.error-icon { font-size: 40px; margin: 0 0 8px; }
.error-message { margin: 0 0 16px; }

.retry-btn {
  padding: 8px 20px;
  background: #6366f1;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
}

/* ── 分頁 ───────────────────────── */
.pagination {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 16px;
  margin-top: 32px;
}

.page-btn {
  padding: 8px 20px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  background: white;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.15s;
}

.page-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.page-btn:not(:disabled):hover {
  border-color: #6366f1;
  color: #6366f1;
}

.page-info {
  font-size: 14px;
  color: #374151;
}
</style>
```

---

## 步驟六：更新 `App.vue`

```vue
<!-- src/App.vue -->
<script setup lang="ts">
import PropertyList from './components/PropertyList.vue'
</script>

<template>
  <div id="app">
    <nav class="navbar">
      <span class="logo">🏠 ihouse BMS</span>
      <span class="subtitle">房源管理系統</span>
    </nav>

    <main>
      <PropertyList />
    </main>
  </div>
</template>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, 'Noto Sans TC', sans-serif;
  background: #f3f4f6;
  color: #111827;
}

#app {
  min-height: 100vh;
}

.navbar {
  background: white;
  border-bottom: 1px solid #e5e7eb;
  padding: 16px 24px;
  display: flex;
  align-items: center;
  gap: 12px;
  position: sticky;
  top: 0;
  z-index: 100;
}

.logo {
  font-size: 20px;
  font-weight: 700;
  color: #6366f1;
}

.subtitle {
  font-size: 14px;
  color: #6b7280;
}

main {
  padding: 0 0 48px;
}
</style>
```

---

## 關鍵設計決策解析

### 決策一：`watch` vs `onMounted` 觸發資料載入

```typescript
// ❌ 只用 onMounted：無法響應篩選變更
onMounted(() => {
  fetchList()
})

// ❌ 只用 watch 且 immediate: false：第一次不載入
watch(filters, () => { fetchList() }, { deep: true })

// ✅ 正確：watch + immediate: true = 首次 + 每次篩選變更都觸發
watch(
  filters,
  () => { fetchList() },
  { deep: true, immediate: true }
)
```

**關鍵理解**：`immediate: true` 讓 `watch` 在元件建立時立刻執行一次，完全取代 `onMounted`。

---

### 決策二：Skeleton vs Spinner 的選擇時機

| 場景 | 推薦 |
|------|------|
| 首次載入、內容形狀已知 | ✅ Skeleton |
| 非同步操作（提交表單、按鈕動作）| ✅ Spinner + disabled |
| 頁面切換（Router 層級）| ✅ Skeleton 或 Progress Bar |
| 載入時間不確定（串流資料）| ✅ Progress Bar |

---

### 決策三：錯誤狀態的 UX 設計

```vue
<!-- ❌ 不好的錯誤訊息（技術性）-->
<div>Error: HTTP 500 Internal Server Error</div>

<!-- ✅ 好的錯誤訊息（友善 + 提供行動）-->
<div>
  <p>⚠️ 資料載入失敗</p>
  <p>{{ friendlyErrorMessage }}</p>   <!-- 轉換為人話 -->
  <button @click="fetchList">重新載入</button>  <!-- 明確的行動 -->
</div>
```

---

## 自我檢核

完成以上步驟後，請確認以下問題你能用自己的話回答：

**技術理解**：
- [ ] `watch` 的 `deep: true` 和 `immediate: true` 各自的作用是什麼？
- [ ] `SkeletonCard` 的 `shimmer` 動畫是如何產生「光掃過」的視覺效果？
- [ ] `computed(() => result.value?.items ?? [])` 為什麼用 `?.` 和 `?? []`？

**設計判斷**：
- [ ] 為什麼服務層要同時加上 `withRetry` 和 `withCache`？兩者解決的是不同的問題嗎？
- [ ] `PropertyCard` 的 `card-title` 為什麼用 `-webkit-line-clamp` 截斷文字而不是 `white-space: nowrap`？

---

## W2 收尾清單

| 項目 | 說明 | 完成？ |
|------|------|--------|
| `async-utils` 完整實作 | httpClient / retry / cache / concurrent / useAsyncState | ⬜ |
| `propertyService.ts` 建立 | API 服務層封裝 | ⬜ |
| `SkeletonCard.vue` 實作 | shimmer 動畫效果 | ⬜ |
| `PropertyCard.vue` 實作 | 完整卡片 UI | ⬜ |
| `PropertyList.vue` 整合 | useAsyncState + watch + 篩選 | ⬜ |
| `App.vue` 更新 | 引入 PropertyList | ⬜ |
| W2 作業 PR 提交 | `[W2] 非同步資料流封裝模組` | ⬜ |
| W1 補交 PR | 3 份待提交作業 | ⬜ |

---

## 延伸挑戰（若時間充裕）

### 挑戰一：無限滾動（Infinite Scroll）

```typescript
// 在 PropertyList.vue 中新增：
import { useIntersectionObserver } from '../composables/useIntersectionObserver'

const loadMoreTrigger = ref<HTMLElement | null>(null)

useIntersectionObserver(loadMoreTrigger, () => {
  if (!isLoading.value && (filters.value.page ?? 1) < totalPages.value) {
    goToPage((filters.value.page ?? 1) + 1)
  }
})
```

### 挑戰二：搜尋防抖（Debounce）

```typescript
// 避免每次 keypress 都觸發 API 請求
import { debounce } from '../utils/debounce'

const searchQuery = ref('')
const debouncedSearch = debounce((query: string) => {
  filters.value = { ...filters.value, query, page: 1 }
}, 300)  // 300ms 後才觸發
```

---

## 明日預告（5/1–5/3）

W2 正式進入收尾階段：
- **5/1（五）**：W1–W2 所有未提交作業補交（目標：作業完成率 > 90%）
- **5/3（六）**：W3 預習 — Vue 3 `ref` / `reactive` / `computed` / `watch` 深入探討
- **5/4（日）**：W2 作業截止，確認所有 PR 已提交

> **重要提醒**：距離 W2 截止還有 4 天（5/4），但目前 W1 的 3 份作業仍未提交。
> 請在 5/1–5/2 安排 2–3 小時集中補交，否則本月作業完成率目標（≥ 90%）將難以達成。
