# pay-flow 開發紀錄 Day 2 (2026-03-05)

> Day 2 的主軸是「把功能做完整」。  
> Day 1 做了基本的薪資計算與寄信，Day 2 則在使用中發現：有些設定沒有實際生效、  
> 有些資料顯示是 0、UI 操作太分散、還缺少一張月核算總表。  
> 這天修了不少「功能看起來有，但其實沒用」的問題。

---

## 一、發現舊員工資料中 勞保/健保 雇主負擔全部是 0

### 使用者回報

> 「怎麼有些員工的勞保雇主負擔是 0？那是資料庫問題嗎？」

### 根本原因分析

這批員工是「更早期建立的資料」——當時 `employees` table **還沒有雇主負擔欄位**，  
後來用 `ALTER TABLE ADD COLUMN` 補上時，舊員工的欄位預設值就是 `0`，  
而且從來沒有「觸發重新計算」的機會。

每次新增或修改員工，建檔時會 call `LookupInsuranceBracket()` 把所有欄位填好；  
但舊員工從來沒有再被更新過，所以帶著 0 一路活到現在。

### 修法：新增批次重算功能

新增 `RecalculateAllInsurance()` 函式：

```go
// employee_store.go
func (s *EmployeeStore) RecalculateAllInsurance(brackets []InsuranceBracket) (int, error) {
    rows, _ := s.db.Query(`SELECT id, salary FROM employees WHERE deleted_at = ''`)
    count := 0
    for rows.Next() {
        var id int64
        var salary float64
        rows.Scan(&id, &salary)
        b := LookupInsuranceBracket(salary, brackets)
        s.db.Exec(`UPDATE employees SET
            labor_self=?, labor_employer=?, health_self=?, health_employer=?,
            labor_pension=?, dependent_health_self=?
            WHERE id=?`,
            b.LaborSelf, b.LaborEmployer, b.HealthSelf, b.HealthEmployer,
            b.LaborPension, 0, id)
        count++
    }
    return count, nil
}
```

前端在「系統設定」頁面加了「維護工具」區塊，放一個藍色卡片 + 按鈕，  
點下去會跳確認 Dialog，確認後批次更新所有員工。

> **學員筆記：** 這類「補資料」的工具，在正式產品裡通常叫做 **data migration**。  
> 只跑一次，跑完就沒用了——但沒有它，舊資料就一直是錯的。  
> AI 做的是「加功能」，但資料面的問題要你自己發現。

---

## 二、UI 改善：把匯入/匯出合成一個下拉按鈕

### 使用者的觀察

> 「出勤頁面上方有兩個按鈕：匯入出勤、匯出 Excel，  
>  可以合成一個下拉式按鈕嗎？比較整齊。」

### 修法

把兩個獨立的 `<el-button>` 改成 `<el-dropdown>` 組合：

```vue
<el-dropdown split-button type="primary" @click="importAttendance">
  📥 匯入出勤
  <template #dropdown>
    <el-dropdown-menu>
      <el-dropdown-item @click="exportAttendance">📤 匯出出勤 Excel（範本）</el-dropdown-item>
      <el-dropdown-item @click="exportPayrollSummary">📊 匯出薪資總報表</el-dropdown-item>
      <el-dropdown-item divided @click="handleResetMonth">
        <span style="color: var(--el-color-danger)">🗑 清除該月份薪資資料</span>
      </el-dropdown-item>
    </el-dropdown-menu>
  </template>
</el-dropdown>
```

最終下拉選單包含四個項目：
1. **📥 匯入出勤**（主按鈕，點一下就觸發）
2. **📤 匯出出勤 Excel（範本）**
3. **📊 匯出薪資總報表**（新功能，後面說明）
4. **🗑 清除該月份薪資資料**（紅色，有確認 Dialog）

> **學員筆記：** UI 改動看起來很小，但對使用者的感受影響很大。  
> 「整齊」不是美觀問題，而是認知負擔——  
> 按鈕越少，使用者找對的那個越快。

---

## 三、出勤匯入永遠 0 筆：JSON Tag 的坑

### 使用者回報

> 「我匯入出勤 Excel，系統說成功，但實際筆數是 0，資料沒有進來。」

### 除錯過程

追蹤程式碼，發現前端送出的 JSON 欄位是 camelCase，  
Go 後端的 `AttendanceImportRow` struct 卻**沒有加 JSON tag**：

```go
// ❌ 原本的（沒有 json tag）
type AttendanceImportRow struct {
    EmployeeName    string
    ActualDays      float64
    PersonalLeave   float64
    // ...
}
```

Go 的 JSON 反序列化預設會做 **大小寫不敏感** 的匹配，  
但 Wails 的傳遞層在某些版本會先嚴格比對，導致全部匹配不到，  
`rows` 是空的，後端「成功」但什麼都沒寫進去。

### 修法

加上明確的 JSON tag：

```go
// ✅ 修正後
type AttendanceImportRow struct {
    EmployeeName    string  `json:"employeeName"`
    ActualDays      float64 `json:"actualDays"`
    PersonalLeave   float64 `json:"personalLeave"`
    SickLeave       float64 `json:"sickLeave"`
    LateMinutes     float64 `json:"lateMinutes"`
    OvertimePay     float64 `json:"overtimePay"`
    // ...
}
```

> **學員筆記：** 這種 Bug 最難抓——不會 crash、不會報錯，只是靜靜地什麼都沒發生。  
> Go struct 的 JSON tag 雖然不是必填，但在跨語言的 RPC 邊界（這裡是 Wails Go ↔️ Vue）  
> **強烈建議每個欄位都加上明確的 tag**，不要相信「預設行為」。

---

## 四、清除月份功能：ResetMonthlyPayroll

### 需求

> 「我想清掉某個月的薪資計算結果，從頭重填。現在只能一筆一筆改，很麻煩。」

### 設計考量

「清除」這個動作要謹慎——使用者點下去之後，當月所有人的出勤計算都沒了。  
所以設計上加了兩層保護：

1. 按鈕放在下拉選單最底部，用分隔線隔開，文字紅色
2. 點下去出現 `ElMessageBox.confirm`，要求二次確認

SQL 使用 **transaction** 確保原子性：

```go
func (s *EmployeeStore) ResetMonthlyPayroll(yearMonth string) error {
    tx, _ := s.db.Begin()
    // 先刪 items（有外鍵關聯）
    tx.Exec(`DELETE FROM monthly_payroll_items
             WHERE payroll_version_id IN (
               SELECT id FROM monthly_payroll_versions WHERE year_month = ?
             )`, yearMonth)
    // 再刪 version
    tx.Exec(`DELETE FROM monthly_payroll_versions WHERE year_month = ?`, yearMonth)
    return tx.Commit()
}
```

> **學員筆記：** 刪除要用 transaction，否則如果第一個 DELETE 成功、第二個失敗，  
> 資料就會殘留半截——`items` 沒了但 `versions` 還在，或反過來。  
> AI 第一版沒有加 transaction，是後來 code review 時補上的。

---

## 五、WorkDaysMode 設定沒有實際生效

### 使用者回報

> 「我在設定裡把「應上班計算方式」改成「固定 30 天」，  
>  但出勤頁面的「應上班」欄位還是顯示當月實際天數。」

### 根本原因分析

這是一個**設定讀了但沒傳進去**的問題。追蹤整個呼叫鏈：

```
前端 → GetMonthlyPayrollByMonth(yearMonth)
      ↓
app.go → GetMonthlyPayrollByMonth(yearMonth)        ← ① 沒讀 WorkDaysMode
      ↓
employee_store.go → GetMonthlyPayrollByMonth(yearMonth) ← ② 沒接收參數
      ↓
scheduledWorkDays(yearMonth, leaveDate)             ← ③ 完全沒有 mode 邏輯
```

三層都沒有處理。`workDaysMode` 設定存進了 SQLite，但從來沒有人去讀它、傳遞它、使用它。

### 修法

把 `workDaysMode` 一路貫穿下去：

```go
// app.go：從設定讀取後傳入
func (a *App) GetMonthlyPayrollByMonth(yearMonth string) ([]payroll.MonthlyPayrollRecord, error) {
    settings, _ := a.settings.Load()
    return a.store.GetMonthlyPayrollByMonth(yearMonth, settings.WorkDaysMode)
}

// employee_store.go：scheduledWorkDays 加上 mode 參數
func scheduledWorkDays(yearMonth, leaveDate, workDaysMode string) float64 {
    if workDaysMode == "30" {
        return 30
    }
    // 原本的按月實際天數邏輯...
}
```

> **學員筆記：** 這種 Bug 叫做**「功能孤島」**—— UI 可以操作、資料庫有存，  
> 但中間的連接從來沒有建立。  
> AI 在做設定功能時，很容易只做「存設定」，而沒有同步考慮「設定在哪裡被使用」。  
> **每新增一個設定項，都要問：「這個設定什麼時候、在哪裡、被誰讀取？」**

---

## 六、新功能：匯出薪資總報表（三頁 Excel）

### 使用者需求

> 「我需要一份給會計的月結報表，要有薪資明細、  
>  還要有政府應繳費用（勞保健保雇主負擔）和匯款彙總。  
>  這張表跟出勤 Excel 是不同用途的，要分開匯出。」

### 設計決策：分開匯出 vs 合併

一開始有考慮兩個方案：
- **方案 A**：在現有出勤 Excel 增加額外工作表
- **方案 B**：獨立的「薪資總報表」按鈕，只在計算完成後使用

選擇方案 B，因為出勤範本是「給人工填寫用的」，總報表是「計算結果的彙整」，  
兩者語義不同，放在一起會讓使用者搞混。

### 資料層需要補欄位

`MonthlyPayrollRecord` 原本只有員工自付的保費，沒有**雇主負擔**：

```go
// 補充前（缺少雇主欄位）
type MonthlyPayrollRecord struct {
    LaborSelf   float64 `json:"laborSelf"`
    HealthSelf  float64 `json:"healthSelf"`
    // LaborEmployer 不在這裡！
}

// 補充後
type MonthlyPayrollRecord struct {
    LaborSelf     float64 `json:"laborSelf"`
    LaborEmployer float64 `json:"laborEmployer"`  // 新增
    HealthSelf    float64 `json:"healthSelf"`
    HealthEmployer float64 `json:"healthEmployer"` // 新增
}
```

SQL 查詢也要同步 JOIN `employees` 表取得這兩個欄位：

```sql
SELECT mp.*, e.labor_employer, e.health_employer
FROM monthly_payroll_versions mp
JOIN employees e ON e.id = mp.user_id
WHERE mp.year_month = ?
```

### 三個工作表的設計

**Sheet 1「薪資明細」**：每位員工一列，欄位包含：
月薪、應上班、實際出勤、事假扣款、病假扣款、遲到扣、加班費、動態項目、獎金、
出勤應付、勞保(員工)、健保(員工)、眷屬健保、勞退自提、所得稅、**實發薪資**

**Sheet 2「政府應繳費用」**：每位員工一列，欄位包含：
月薪、勞保(員工)、**勞保(雇主)**、勞保合計、健保(員工)、眷屬健保、**健保(雇主)**、健保合計、勞退提繳、**雇主應繳合計**

以綠色 `#E2EFDA` 底色標示雇主負擔欄，與員工自付欄作視覺區隔。

**Sheet 3「匯款彙總」**：垂直式摘要，分三段：
- 員工薪資（各員工實發 + 總計）
- 代扣轉繳（勞保+健保員工自付、所得稅 → 轉繳政府）
- 雇主應繳（公司應繳的勞保+健保+勞退）
- **本月費用合計 = 實發薪資 + 所得稅 + 補充保費 + 雇主負擔**

> **學員筆記：** 這個功能的需求一開始只說了「要有政府費用」，
> 直到仔細問才知道要分三張表，還要有「匯款彙總」給老闆看。  
> **薪資計算是業務邏輯最密集的領域之一，一定要讓 domain expert（這裡是會計/老闆）確認每個欄位的定義。**

---

## 七、匯出功能的兩個 Bug：無聲的失敗 + 樣式不一致

### Bug 1：Go 端根本沒有 build 成功

前端按下「匯出薪資總報表」，出現錯誤：

```
window['go']['main']['App']['ExportPayrollSummary'] is not a function
```

這個訊息不是「功能壞了」，而是「這個函式根本不存在」——  
代表 Wails 後端沒有重新 build。

追查原因：`excel_extras.go` 裡有一個 Go 編譯錯誤：

```go
// ❌ 宣告了 style 但從來沒用到
style := func(sheet, col string, row int, s int) {
    cell, _ := excelize.CoordinatesToCellName(colToNum(col), row)
    f.SetCellStyle(sheet, cell, cell, s)
}
// 沒有 _ = style，Go 不允許 declared and not used
```

Go 的「宣告但未使用」是**編譯錯誤**，不是警告。  
`wails dev` 遇到這個錯誤，後端 build 失敗，但繼續用舊版 binary 跑，  
所以 UI 看起來 app 還在，只是新函式完全不見了。

**修法：** 加上 `_ = style`。

> **學員筆記：** Wails 的熱更新（hot reload）是前端用的。  
> Go 後端出現 compile error 時，它**不會重啟後端**——就算你儲存了檔案，  
> 畫面還是舊的、錯的、但看起來好好的。  
> 養成習慣：改了 Go 檔案之後，**看一眼終端機有沒有 compile error**。

---

### Bug 2：Sheet 2 樣式前後不一致

匯出的「政府應繳費用」表格，同一欄的數字有時候置中、有時候靠右——  
偶數列（有底色）和奇數列的對齊方式不同。

根本原因：`boldStyle` 有兩個版本，一個有 Alignment、一個沒有：

```go
// ✅ subHStyle（套用在偶數列合計欄）— 有 Alignment
subHStyle, _ := f.NewStyle(&excelize.Style{
    Fill:      excelize.Fill{...},
    Font:      &excelize.Font{Bold: true},
    Alignment: &excelize.Alignment{Horizontal: "center"},
})

// ❌ boldStyle（套用在奇數列合計欄）— 沒有 Alignment
boldStyle, _ := f.NewStyle(&excelize.Style{
    Font: &excelize.Font{Bold: true},
    // 沒有 Alignment → 預設靠右
})
```

奇偶列用了不同的樣式，一個有指定對齊、一個用 Excel 預設值（數字靠右）。

**修法：** 統一所有樣式都加上 `Alignment: &excelize.Alignment{Horizontal: "center"}`。

另外還有一個子問題：原本的 switch case 把「勞退提繳」(j=9) 也標上了綠色底：

```go
// ❌ 錯的：j=9 是勞退提繳，不是雇主欄
case 3, 7, 9:

// ✅ 對的：只有勞保雇主(3) 和健保雇主(7) 才是雇主負擔
case 3, 7:
```

> **學員筆記：** 樣式 Bug 是「視覺上發現、邏輯上才找到背後錯誤」的代表案例。  
> 使用者說「格式不一致」，追進去才發現是兩個不同的 style object、還有錯誤的 index。  
> 這類 Bug 很難靠 code review 發現，**只有跑起來看 Excel 輸出才能抓到**。

---
Author: GeminiCLI Gemini 2.0 Flash Thinking
