# Blog Project Agent Guidelines

此文件定義了與 GeminiCLI 協作時，針對此 Blog 專案的文件管理與 README (TOC) 更新規範。

## 1. 檔案組織與命名 (File Organization & Naming)

- **資料夾結構**：依年份與月份建立資料夾，格式為 `YYYY-MM/` (例如：`2026-03/`)。
- **檔案命名**：所有檔案名稱必須包含發布日期，格式為 `YYYY-MM-DD-標題.extension` (例如：`2026-03-07-pay-flow-dev-day4.md`)。
- **儲存位置**：新文件應放入對應的 `YYYY-MM/` 資料夾中。

## 2. README (TOC) 更新規範

每次新增或修改文件後，必須同步更新根目錄下的 `README.md`。

- **排序規則**：
  - **年份/月份**：由新到舊 (最新月份在最上方)。
  - **文件項目**：由新到舊 (最新日期在最上方)。
- **內容分類**：
  - 在每個月份標題下，依檔案類型區分為 `Markdown 筆記`、`PDF 文件` 或特定專題 (如 `pay-flow 開發紀錄`)。
- **格式規範**：
  - 文件項目格式：`- YYYY/MM/DD - [標題](./path/to/file)`。
  - 確保路徑使用相對路徑。

## 3. 自動化與檢查清單 (Checklist)

當收到指令要求「更新 Blog」或「新增文章」時，請執行以下步驟：

1. **放置檔案**：確認檔案放入正確的 `YYYY-MM/` 資料夾。
2. **更新 README**：
   - 檢查是否需要建立新的月份標題 (H2)。
   - 依日期倒序插入新連結。
3. **驗證連結**：確保 README 中的相對路徑正確無誤。
4. **Git 提交**：將新檔案與 README 一併提交 (Commit)，訊息格式建議為 `docs: add new post [標題] and update TOC`。

---
Author: GeminiCLI Gemini 2.0 Flash Thinking
