# Pay-Flow SaaS 化開發紀錄 Day 4 (2026-03-07)

**今日目標**: 架構全面 SaaS 化、商業化轉型與系統穩定性提升

今天是專案開發最關鍵的一天。我們正式從 **Wails (桌面) + Nuxt (網頁)** 雙軌制，轉向 **純 SaaS Nuxt (網頁版)**，並實作了完整的付費方案體系與多租戶架構。

---

## 📅 今日開發重點

### 1. UI/UX 與部署自動化 ✅
- **UI 一致性優化**：統一 `employees` 與 `attendance` 頁面的表格規範，並實作操作欄 Sticky 功能，解決橫向捲動時按鈕被遮擋的痛點。
- **Cloud Run 部署**：優化 Dockerfile 並修復 Alpine 字型套件問題，成功將服務部署至 GCP 台灣機房。
- **CI/CD 建立**：實作 `.gitlab-ci.yml` 與本地 `deploy-cloud-run.sh` 腳本，實現自動化發佈。

### 2. SaaS 基礎建設 (Cloud Foundation) ✅
- **Supabase 多租戶架構**：設計並實作 `companies`, `profiles`, `employees`, `payrolls` 資料表及其 RLS (Row Level Security) 策略。
- **Auth 系統整合**：導入 Supabase Auth 與 Google OAuth，支援新使用者註冊後自動建立組織流程。
- **管理員後台**：建立 `/admin` 專屬控端，支援超級管理員跨租戶視角管理。
- **審計日誌 (Audit Log)**：實作 `useAuditLog.ts` 追蹤關鍵操作（如薪資匯出、寄信等）。

### 3. 商業化轉型與權限控制 ✅
- **方案權限檢查 (Plan Guard)**：建立 `usePlan.ts` 統一管理 Free/Plus/Pro 方案權限。
- **付費牆與配額**：在關鍵功能加入方案鎖定提示（🔒 Pro），並針對免費版實作 5 人員工數上限限制。
- **效率工具**：開發「一鍵承襲上月資料」功能，大幅減少使用者跨月份重複輸入的時間。

### 4. 系統穩定性與 ORM 治理 ✅
- **Drizzle ORM 導入**：使用 Drizzle 進行資料庫 Entity 管理，提升開發效率並確保 SQL Migration 的可追蹤性。
- **環境模式切換**：實作 `APP_MODE` (dev | testing | prod) 機制，並根據環境動態顯示 Debug Overlay 面板。
- **專案瘦身**：正式移除舊有的 Wails 相關程式碼與 Go 後端代碼，聚焦 Web 版本開發。

---

## 📝 重大技術決策 (ADR)

- **全面 Web 化**：考量到維護成本與 SaaS 擴展性，決定棄用桌面版。
- **SECURITY DEFINER 授權模式**：為解決 Supabase RLS 在查詢權限時的遞迴問題，改用 PostgreSQL 函數封裝查詢邏輯。
- **管理員後台架構**：超級管理員功能採用 Nuxt Server API 搭配 Service Role 密鑰實作，以達成跨租戶管理之需求。

## 💡 技術心得

- **RLS 遞迴陷阱**：在設計多租戶權限時，若 Policy 查詢自身表容易引發無限遞迴，使用 `is_admin()` 函數封裝是最佳實踐。
- **身分驗證競態**：處理了頁面重新整理時，資料查詢早於 Auth 載入導致的列表空白問題，透過 Composable 中的 `ensureAuth` 機制解決。
- **本地隔離**：SaaS 模式下，本地 IndexedDB 亦需根據 `companyId` 進行隔離，確保資料切換帳號時的安全性。

---

## 🚀 下一步規劃
- [ ] **寄送狀態追蹤**：提升薪資單寄送的透明度。
- [ ] **品牌化自定義**：支援上傳公司 Logo 與印章。
- [ ] **兼職計算邏輯**：開始研發時薪與兼職人員的計薪模組。

---
Author: GeminiCLI Gemini 2.0 Flash Thinking
