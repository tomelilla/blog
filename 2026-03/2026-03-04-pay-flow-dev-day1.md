# pay-flow 開發紀錄 Day 1 (2026-03-04)

> 這份文件記錄了這個專案從需求到實作的完整歷程，  
> 包含走彎路、修錯誤、重新設計的真實過程。  
> 給學員的提示：**AI 開發不等於一次到位，它更像是一個會犯錯但很勤快的工程師。**

---

## 一、專案起點：一張 Excel 表

### 初始需求

這專案的起點並非來自一個正式的開發清單，而是源於我看到負責算薪水的人，每個月都在重複一套極其瑣碎且低效的流程：

1. **多維度資料對照**：手邊有一份 Excel，除了記錄員工基本資料與底薪，還要對照每個人的投保級距金額。
2. **複雜的扣薪邏輯**：計算請假扣薪時，必須精確到「按時計」、「按分計」，且病假還需以半價計算。
3. **繁瑣的產出與發送**：計算完成後，還得為每一位員工手動建立一份 PDF 薪資單，再一封一封地建立 Email 寄出。

**當我看到這一切時，心裡只有一個想法：**

> 「這完全是可以、也應該被程式自動化處理的。」

### 核心公式（SPEC.md 裡最早記下的那一行）

```
實付薪資 = 底薪/應上班天數 × 實際上班天數 - 請假扣款 - 其他扣款
         + 加班費 + 獎金 - 勞保自付 - 健保自付 - 眷屬健保 - 獎金稅 - 薪資稅 - 補充保費
```

看起來很簡單。**但這只是開始。**

---

## 二、第一版：能動就好

### 技術選型

選用 **Wails v2**（Go + Vue 3），原因：
- 可以打包成桌面 app，不需要架伺服器
- Go 後端可以直接讀寫本地 SQLite
- 老闆不懂 web，給她一個 `.app` 直接點兩下就能用

### 第一個功能：匯入 Excel → 計算結果

AI 的第一版產出：

```
使用者選擇 Excel 檔 → 解析標題列 → 帶入公式 → 顯示計算結果表 → 匯出 CSV
```

**成果：** 功能可以動。  
**問題：** 每次都要重新匯入 Excel，沒有「記住員工資料」的概念。

### 第一次彎路：標題列格式問題

寫了一個 `inspect-excel` 工具來分析 Excel 結構，發現：

```
❌ 職稱  => 找不到
❌ 職級  => 找不到
```

原來 Excel 欄位名稱帶有前置空白（`"  起調整薪資"` → cleaned: `"起調整薪資"`），  
程式用 `==` 比對字串，當然找不到。

**修法：** 改成 `strings.TrimSpace()` 處理後再比對。

> **學員筆記：** 資料清洗（data cleaning）永遠是第一個坑。  
> AI 寫的程式預設資料是乾淨的——現實不是。

---

## 三、第二版：員工資料管理

### 需求升級

「每次匯 Excel 太麻煩，可以把員工基本資料存起來嗎？」

於是進入了 **CRUD 階段**：新增員工、編輯員工、刪除員工（軟刪除）。

### AI 的做法：migrate() + ensureXxxColumns()

這裡出現了第一個**結構性決策**：

資料庫要怎麼演進？

AI 採用的策略是 **ALTER TABLE 增欄位**，而不是版本化 migration：

```go
func (s *EmployeeStore) ensureExtraColumns() error {
    need := map[string]string{
        "job_title":  "TEXT NOT NULL DEFAULT ''",
        "job_level":  "TEXT NOT NULL DEFAULT ''",
        "leave_date": "TEXT NOT NULL DEFAULT ''",
        // ...
    }
    for col, def := range need {
        _, err := s.db.Exec(`ALTER TABLE employees ADD COLUMN ` + col + ` ` + def)
        // 忽略「欄位已存在」的錯誤
    }
}
```

**好處：** 簡單，舊資料不會不見。  
**壞處：** 長期下去 migrate() 越來越長，而且欄位定義散落各處。

> **學員筆記：** 小工具可以這樣做。  
> 但如果是正式產品，應該用 migrate up/down（flyway、goose 等工具）管理。

---

## 四、第三版：月薪計算草稿（出勤計算頁面）

### 需求

「我希望每個月選一個月份，系統幫我把所有在職員工列出來，  
我只要填實際出勤天數、事假、病假、加班，系統自動算。」

### AI 設計的核心 table：`monthly_payroll`

```sql
CREATE TABLE monthly_payroll (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    year_month TEXT NOT NULL,       -- "2025-02"
    version INTEGER NOT NULL DEFAULT 1,
    salary REAL NOT NULL DEFAULT 0,
    actual_days REAL NOT NULL DEFAULT 0,
    personal_leave_days REAL NOT NULL DEFAULT 0,
    ...
    UNIQUE(user_id, year_month)     -- 每人每月只有一筆
)
```

**Auto-save 設計：**  
每個欄位 `@blur`（失去焦點）就自動存檔，不需要「儲存按鈕」。  
這是 AI 主動建議的 UX，省去使用者忘記按存的問題。

### 這版最大的功能：動態加減項

薪資計算不只有固定欄位，有時候要加「伙食津貼」、扣「宿舍費」……  
每個員工的項目還不一樣。

AI 設計了獨立的 `payroll_items` table：

```sql
CREATE TABLE payroll_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    payroll_version_id INTEGER NOT NULL,  -- 關聯 monthly_payroll.id
    item_name TEXT NOT NULL,
    item_type TEXT NOT NULL,              -- "add" 或 "subtract"
    amount REAL NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL DEFAULT 0
)
```

前端可以針對每個員工展開，新增/刪除動態項目，每次 blur 自動 save。

---

## 五、第一個嚴重 Bug：應上班天數算錯了

### 使用者回報

> 「紅框資料，王小明 應上班 E 應該每個人都一樣，  
>  是實際計算出來的，二月應該是 28 天，但畫面顯示 26。」

### 根本原因分析

原本的函數 `calcWorkDays(yearMonth, leaveDate)` 邏輯：

```go
// 有離職日期的話，取「該月天數」和「離職當日」的較小值
func calcWorkDays(yearMonth, leaveDate string) float64 {
    daysInMonth := ...
    if leaveDate != "" && leaveDate 的年月 == yearMonth {
        return float64(leaveDate 的日)  // ← 問題在這裡
    }
    return float64(daysInMonth)
}
```

王小明 2 月 26 日離職，`calcWorkDays` 就回傳 **26**，  
但「應上班天數」應該是「這個月總共幾天」，跟離職幾號無關！

### 重構後的設計

把一個函數拆成兩個，**職責分離**：

```go
// 應上班天數：永遠是整個月的天數（除非根本還沒就職）
func scheduledWorkDays(yearMonth, leaveDate string) float64 {
    // 只有「離職月份之前」才回傳 0
    // 當月離職 → 仍然是整月天數
}

// 實際出勤預設值：離職當月，預設到離職那天
func defaultActualDays(yearMonth, leaveDate string) float64 {
    // 當月離職 → 預設為離職日
    // 其他情況 → 整月
}
```

> **學員筆記：** 「一個函數一個職責」不是口號。  
> `calcWorkDays` 同時承擔了兩件事，改一個需求就壞掉另一個。

---

## 六、第二個邏輯 Bug：離職員工出現在未來的月份

### 使用者回報

> 「王小明 2 月 26 日離職，  
>  但我選 3 月份，他還是出現在列表裡！」

### 根本原因分析

`GetMonthlyPayrollByMonth` 的 SQL：

```sql
-- 原本的查詢（有問題的版本）
SELECT e.* FROM employees e
WHERE e.deleted_at = '' OR e.deleted_at IS NULL
```

**只排除了「已刪除」的員工**，沒有處理「離職但未刪除」的情況。  
（公司希望保留離職員工的歷史紀錄，所以不能直接刪除）

### 修法

在 WHERE 條件加入離職日期篩選：

```sql
AND (
    e.leave_date = '' OR
    e.leave_date IS NULL OR
    substr(e.leave_date, 1, 7) >= ?   -- 離職月份 >= 當前查詢月份
)
```

同時 Go 端需要傳兩個參數：

```go
// 修改前（只傳一個）
rows, err = s.db.Query(query, yearMonth)

// 修改後（傳兩個，第二個給離職篩選用）
rows, err = s.db.Query(query, yearMonth, yearMonth)
```

> **學員筆記：** 這個 Bug 的根因是**需求沒說清楚**。  
> 一開始只說「列出在職員工」，沒有定義「什麼是在職」。  
> 是「沒被刪除」，還是「這個月份還在職」？  
> 需求模糊 → AI 做了一個合理但不完整的假設 → 上了才發現錯。

---

## 七、隱藏需求浮出：薪資明細表

### 使用者的下一個需求

> 「我不只要算薪資，我還要能寄薪資明細給員工。  
>  每個人的薪資單要長這樣（附上截圖）……」

截圖顯示的格式：左欄「應發項目」，右欄「應扣項目」，底部顯示實發金額，  
右上角有三個按鈕：**下載 PDF ｜ 寄送 ｜ 關閉**。

### AI 的設計過程

這個功能涉及三個層面：

**① 後端：Email 發送**

新建 `email.go`，包含：
- `BuildPayslipHTML()` — 產生 HTML 格式的薪資明細
- `SendPayslipEmail()` — 透過 SMTP（TLS/STARTTLS）寄送

**② 後端：MonthlyPayrollRecord 補欄位**

薪資明細需要顯示職稱、職級、各項保費……  
但原本的 `MonthlyPayrollRecord` 沒有這些欄位（因為當初設計時沒想到要顯示）。

```go
// 原本的結構（不夠）
type MonthlyPayrollRecord struct {
    ID, UserID int64
    Name, YearMonth string
    Salary float64
    // ...出勤欄位...
}

// 補充後
type MonthlyPayrollRecord struct {
    // ... 原有欄位 ...
    JobTitle            string  // 需要 JOIN employees
    JobLevel            string  // 需要 JOIN employees
    LaborSelf           float64 // 需要 JOIN employees
    HealthSelf          float64 // 需要 JOIN employees
    DependentHealthSelf float64 // 需要 JOIN employees
    LaborPension        float64 // 需要 JOIN employees
    SalaryTax           float64 // 需要 JOIN employees
}
```

SQL 也要同步更新，從原本的單表查詢變成 JOIN：

```sql
-- 修改前
SELECT mp.* FROM monthly_payroll mp WHERE mp.year_month = ?

-- 修改後
SELECT mp.*, e.job_title, e.job_level, e.labor_self, ...
FROM monthly_payroll mp
JOIN employees e ON e.id = mp.user_id
WHERE mp.year_month = ?
AND (e.leave_date = '' OR substr(e.leave_date, 1, 7) >= ?)
```

**③ 前端：PayslipDialog 元件**

新建 `PayslipDialog.vue`，全部在 Vue 端計算：

```
應發 = 底薪 + 加班費 + 獎金 + Σ動態加項
應扣 = 事假扣 + 病假扣½ + 勞保 + 健保 + 眷屬健保 + 勞退 + 所得稅 + Σ動態減項
實發 = 應發 - 應扣
```

> **學員筆記：** 這裡出現了一個常見問題——**同樣的計算邏輯寫了兩遍**：  
> Go 後端 `email.go` 裡算一次，Vue 前端 `PayslipDialog.vue` 裡又算一次。  
> 這違反 DRY 原則（Don't Repeat Yourself）。  
> 理想做法是後端回傳計算好的數字，前端只顯示。  
> 但在這個 desktop app 的情境下，網路延遲幾乎是零，  
> 所以這個設計暫時可以接受——但技術債已經埋下了。

---

## 八、一個小但重要的細節：預設月份

### 使用者的觀察

> 「每次打開出勤計算頁面，預設是這個月，  
>  但我通常是月底才算上個月的薪資，可以預設上個月嗎？」

### 修法

```js
// 修改前
const now = new Date()
selectedMonth.value = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`

// 修改後
const now = new Date()
const prev = new Date(now.getFullYear(), now.getMonth() - 1, 1)
selectedMonth.value = `${prev.getFullYear()}-${String(prev.getMonth() + 1).padStart(2, '0')}`
```

> **學員筆記：** 這個需求一開始沒有提，AI 也沒有主動問。  
> 這是「隱性假設」——使用者覺得理所當然，但開發者（包含 AI）不知道。  
> **沒有 100% 完整的需求，只有不斷在使用中發現的新需求。**

---

## 九、TypeScript 型別錯誤：defineEmits 語法

### 問題

AI 第一次寫 `PayslipDialog.vue` 時，用了舊版的 TypeScript 語法：

```typescript
// 舊語法（TS 4.x 可以，但 vue-tsc 嚴格模式下報錯）
const emit = defineEmits<{ (e: 'update:modelValue', val: boolean): void }>()
```

VS Code 的 vue-tsc 報錯：

```
Type literal has only a call signature, you should use a function type instead.
```

### 修法

```typescript
// 正確的 Vue 3.3+ 語法
const emit = defineEmits<{
  'update:modelValue': [val: boolean]
}>()
```

> **學員筆記：** API 語法會隨版本演進，AI 的訓練資料有截止日期。  
> 特別是 Vue 3.3 引入了新的 defineEmits 語法，AI 可能給出舊版寫法。  
> **直接跑 build 或開 linter，比相信 AI 說「這樣沒問題」更可靠。**

---

## 十、目前的系統架構

```
pay-flow/
├── main.go                      # Wails 進入點
├── app.go                       # 所有 Go 函數綁定（前端可呼叫）
├── internal/payroll/
│   ├── types.go                 # 所有資料結構定義
│   ├── employee_store.go        # SQLite CRUD + 月薪計算邏輯
│   ├── email.go                 # HTML薪資明細 + SMTP寄送
│   ├── pdf.go                   # PDF產生
│   ├── calc.go                  # 薪資計算公式
│   ├── io.go                    # Excel匯入/匯出
│   └── settings.go              # 公司設定、SMTP設定
└── frontend/src/
    ├── pages/
    │   ├── AttendanceListPage.vue   # 出勤計算（主要工作頁面）
    │   └── EmployeeFormPage.vue     # 員工管理
    ├── components/
    │   └── PayslipDialog.vue        # 薪資明細彈窗
    └── types.ts                     # 前端型別定義
```

---

## 十一、給學員的總結

### AI 開發的真實樣貌

| 我以為 | 實際上 |
|---|---|
| AI 看懂需求就能直接輸出正確答案 | 需求本身就模糊，AI 做了假設，假設有時候錯 |
| 功能做完就結束了 | 做完才發現邊界條件、隱藏需求 |
| AI 一次寫好，不用改 | Bug 要修、邏輯要重構、型別要補 |
| 有 AI 就不用懂程式 | 要能看懂 AI 寫的東西，才能判斷對不對 |

### 這個專案的 Bug 模式分析

1. **資料清洗假設錯誤** — Excel 欄位有空白，用 `==` 比對失敗
2. **職責不清** — `calcWorkDays` 混合了兩個意義不同的計算
3. **需求未定義清楚** — 「在職員工」的邊界沒有想清楚
4. **技術債** — 計算邏輯被 duplicate（Go + Vue 各一份）
5. **API 版本差異** — defineEmits 語法隨 Vue 版本不同

### AI 最擅長的地方

- **樣板程式**：CRUD、migrate、binding 這類重複性工作，AI 很快
- **公式翻譯**：把 Excel 公式轉成程式碼
- **架構建議**：第一版架構比從零開始快很多

### AI 你還是要盯著

- **需求澄清**：「在職」的定義，你說清楚了嗎？
- **邊界條件**：2 月 26 日離職， 3 月還算不算在職？
- **型別錯誤**：跑 build 才知道，不是看 AI 說的
- **重複邏輯**：AI 不會主動提醒你「這段我剛才已經寫過一次了」

---
Author: GeminiCLI Gemini 2.0 Flash Thinking
