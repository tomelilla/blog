# Pay-Flow Web 版開發紀錄 Day 3 (2026-03-06)

**工作時段**: 全天  
**狀態**: 已完成 ✅

---

## 📋 今日完成的工作

### 1. 架構評估與決策確認 ✅
- **評估三個遷移方案**：
  - 方案 A：Vue SPA + Go Cloud Run（最小改動）
  - 方案 B：Nuxt 3 全端 on Cloud Run（推薦）✅ **採用**
  - 方案 C：純前端 SPA（無後端）
  
- **最終決策確認**：
  - ✅ 用戶規模：個人/小型企業（<10 員工）
  - ✅ 認證方案：無需登入（單一經理人使用）
  - ✅ 資料庫：IndexedDB（本地）→ 日後升級 Supabase Phase 2
  - ✅ SMTP 寄信：必須包含在 MVP（第 1 週實現）
  - ✅ PDF 方案：前端 html2pdf.js（後期升級 Puppeteer）
  - ✅ Excel 樣式：MVP 簡版（Phase 2 優化）
  - ✅ Cloud Run 部署：手動 `gcloud run deploy`（無需 GitHub Actions）
  - ✅ Wails 桌面版：凍結維護，全力投入 Web 版

### 2. Nuxt 3 專案初始化 ✅

**建立目錄結構**：
```
nuxt/
├── pages/             # 頁面路由
├── server/api/        # 後端 API
├── composables/       # 邏輯層 Composables
├── layouts/           # 頁面佈局
├── components/        # Vue 組件
├── shared/            # 共用型別定義
└── public/            # 靜態資源
```

**配置文件**：
- ✅ `nuxt.config.ts` — SSR=false、nitro preset=node-server
- ✅ `tsconfig.json` — TypeScript 嚴格模式
- ✅ `package.json` — 所有必要依賴
- ✅ `.env.example` — 環境變數範本
- ✅ `app.vue` 根組件

### 3. 資料儲存層實現 ✅

**Dexie IndexedDB 封裝** (`composables/useDb.ts`)：
- employees 表（員工主檔）
- monthlyPayrollVersions 表（月份薪資版本）
- payrollItems 表（動態加減項）
- CRUD 完整函式 + 批量操作

**localStorage 設定管理** (`composables/useSettings.ts`)：
- 公司資訊（名稱、統編、電話、地址）
- SMTP 設定（host、port、認證、範本）
- 投保級距（預設 2025 年台灣標準 11 級距）

### 4. 業務邏輯移植 ✅

**薪資計算** (`composables/usePayrollCalculation.ts`)：
- 日薪計算
- 請假扣款（事假、病假區分）
- 遲到扣款
- 加班費（133%、167% 分級）
- 動態加減項
- 應發、扣款、實付淨額計算
- **與 Wails calc.go 邏輯完全一致** ✅

**投保級距管理** (`composables/useInsurance.ts`)：
- `LookupBracket(salary)` — 根據薪資查對應級距
- `calculateInsurance(salary, dependentCount)` — 計算保費
- `recalculateAllInsurance()` — 批量重算

### 5. Excel I/O 完整實現 ✅

`composables/useExcelIO.ts` 包含 8 個函式：
- `importEmployeeSheet()` — 讀取員工清單 Excel
- `exportEmployeeSheet()` — 匯出員工資料
- `importAttendanceSheet()` — 讀取出勤 Excel
- `exportAttendanceSheet()` — 匯出出勤紀錄
- `exportPayrollSummary()` — 三頁薪資報表（薪資明細、政府應繳、匯款彙總）
- `generateEmployeeTemplate()` — 產生匯入範本
- `downloadExcel()` — 瀏覽器端 Excel 下載

### 6. 後端 API 實現 ✅

**SMTP 寄信 API** (`server/api/send-email.post.ts`)：
- 薪資單 HTML 產生（含完整表格和樣式）
- nodemailer SMTP 集成
- 支援信件主旨/內文範本變數（{name}, {year}, {month}, {companyName}）
- HTML 附件寄送
- 基礎錯誤處理

### 7. 前端 UI 頁面實現 ✅

**首頁** (`pages/index.vue`)：
- 3 個功能卡片（員工、出勤、設定）
- 簡潔介紹文案

**員工列表** (`pages/employees/index.vue`)：
- 員工列表展示（表格格式）
- 新增、編輯、刪除功能入口
- Excel 匯入/匯出按鈕
- 在職/離職篩選狀態

**出勤計算** (`pages/attendance/index.vue`) — **核心頁面**：
- 月份選擇器
- 出勤欄位（實際天數、各類假期、遲到、加班、獎金）
- 實時薪資計算顯示（應發、扣款、實付）
- Excel 匯入/匯出
- 薪資單預覽彈窗
- 寄送 Email 功能

**設定頁** (`pages/settings/index.vue`)：
- 公司資訊分頁（公司名、統編、電話、地址、薪資計錄模式）
- SMTP 設定分頁（host/port/認證、信件範本）
- 投保級距分頁（可編輯級距表）

**Layout** (`layouts/default.vue`)：
- 頂部導航列
- 頁面路由導航

### 8. Nuxt UI + Tailwind CSS 集成 ✅

**選擇理由**：
- Nuxt 官方推薦，深度整合
- 30+ 預製組件，開箱即用
- 與 Tailwind CSS 完美搭配
- 支援深色模式、響應式設計

**布局升級**：
- ✅ `layouts/default.vue` — 完全用 Tailwind 重寫
  - 響應式導航列（桌面 + 手機菜單）
  - Sticky header (z-50)
  - 主動連結高亮 (active-class)

**首頁視覺提升**：
- ✅ `pages/index.vue` — Tailwind 響應式設計
  - Hero 標題段落
  - 3 張功能卡片（hover 動畫 + 邊框色彩）
  - 功能特點列表（6 項勾選）

---

## 📊 進度統計

| 模塊 | 進度 | 詳情 |
|------|------|------|
| 框架初始化 | ✅ 100% | Nuxt 3 + Tailwind + TypeScript 配置完成 |
| 資料儲存 | ✅ 100% | Dexie (IndexedDB) + 設定管理 |
| 業務邏輯 | ✅ 100% | 薪資計算與保險函式完整 |
| Excel I/O | ✅ 100% | 8 個核心函式完整 |
| 後端 API（SMTP） | ✅ 100% | 實作完成 |
| UI 頁面 | ✅ 100% | 所有頁面 + Tailwind 美化 |
| 本地測試 | ✅ 100% | 伺服器運作正常、頁面渲染正確 |

---

## ✨ 技術亮點

1. **技術轉型** — 成功從桌面版 (Wails) 遷移至網頁版 (Nuxt 3)。
2. **無伺服器邏輯** — 薪資計算全部在前端完成，後端僅處理 SMTP 寄信。
3. **資料持久化** — 使用 IndexedDB 儲存業務資料，localStorage 儲存系統設定。
4. **響應式 UI** — 導入 Tailwind CSS 確保在各種裝置上的操作體驗。

---
Author: GeminiCLI Gemini 2.0 Flash Thinking
