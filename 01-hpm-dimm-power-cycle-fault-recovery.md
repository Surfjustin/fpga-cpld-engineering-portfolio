# HPM DIMM Power Cycle 偶發性判定失敗與 Fault Recovery 修正

> **案例類型：** FPGA／CPLD Power Sequence 與 FSM Debug  
> **使用工具：** Verilog／SystemVerilog、Quartus Prime、Signal Tap、示波器  
> **驗證結果：** 修正後完成 1,000 次 Power Cycle，測試通過

## 案例摘要

系統開機時會判斷 DIMM 配置，並依正常、單一 DIMM 或不同 Fault Case 執行對應的 Power／Reset 流程。驗證階段發現，系統在重複 Power Cycle 約 20～100 次後，偶爾會誤進 Fault 流程並卡在錯誤狀態，造成開機失敗。

我從板端雙向訊號、輸入資格處理、DDR5 FSM 一路追蹤到上層 Sequence FSM，最後確認主要問題是 Fault Case 與 Reset／Recovery 流程沒有依規格完整實作。完成訊號路徑與 FSM 修正後，Power Cycle 1,000 次皆通過。

## 我的責任

- 建立可重現條件，比較正常與異常 Cycle 的狀態流程
- 追蹤 DIMM Power-Good／Fail、Sleep State、Reset 與 Fault Case 的完整路徑
- 比對規格、DDR5 FSM 與上層 Sequence FSM 的行為差異
- 修正輸入資格處理、板卡改版後的訊號來源及 Fault Recovery 流程
- 規劃並執行修正後的板端 Regression 與 Power Cycle 壓力測試

## 分析過程

### 1. 建立端到端訊號路徑

我沒有只從判定結果開始修改，而是先確認訊號實際經過的每一層：

```text
DIMM Power-Good／Fail 實體 Pin
                ↓
       Synchronizer／Debounce
                ↓
            DDR5 FSM
                ↓
       Fault Case／Status Output
                ↓
          Sequence FSM
                ↓
      Warm Reset／Global Reset
```

這個追蹤方式可以分辨問題來自板端輸入、同步／濾波、DDR5 狀態判斷，或上層 Sequence 對 Fault 的處理。

### 2. 檢查雙向 Power-Fail 訊號與 Debounce

DIMM Power-Fail 為雙向訊號，分析時必須分開確認：

- CPLD 何時主動拉低訊號
- CPLD 何時釋放為 Hi-Z
- 實體 Pin 回讀值何時穩定
- 回讀訊號是否經過正確的同步與 Debounce

板端量測顯示，Power-Good 釋放／爬升期間可能在尚未穩定前被邏輯取樣。依量測結果加入約 5 µs 的穩定資格判斷，避免過早接受輸入狀態。

### 3. 改版後的 Sleep-State 訊號路徑

新版本改變了平台 Sleep-State 訊號的來源。舊版訊號經其他模組轉送，新版則需要由實體 Pin 經同步與 Debounce 後直接送入 DDR5 FSM。

我重新追蹤 Top-level 到 DDR5 FSM 的連線，修正沿用舊版路徑所造成的來源不一致，並確認 Recovery 條件使用的是新版實際訊號。

### 4. 比對 DDR5 FSM 與 Sequence FSM

我依規格逐項確認：

- DDR5 FSM 是否在正確狀態產生 Fault Case
- 不同 Fault Case 是否確實傳送到 Sequence FSM
- Sequence FSM 是否執行對應的 Warm Reset 或 Global Reset
- Fault 流程是否具有明確的進入、處理、完成與離開條件

檢查結果顯示，正常流程可以反覆運作，但 Fault Case 一旦真的發生，上層 Sequence 的處理與規格不一致，部分狀態缺少正確的判定或後續轉移，因此 FSM 會停在錯誤流程。

## Root Cause

主要根因是 Sequence FSM 對 DDR5 Fault Case 的處理不完整。系統大多數時間沒有進入 Fault 路徑，因此一般開關機看起來正常；經過多次 Power Cycle 後，只要偶發 Fault 條件被觸發，就可能進入缺少有效轉移或 Recovery 條件的狀態，最終造成開機流程卡死。

排查過程中也確認兩個需要一併修正的風險：

- 雙向 Power-Fail 回讀需要足夠的輸入穩定時間，避免在爬升階段過早取樣。
- 改版後的 Sleep-State 訊號來源已改變，不能繼續沿用舊版的模組路徑。

## 修正內容

1. 依規格補齊 DDR5 的不同 Fault Case 與狀態輸出。
2. 確認各 Fault Case 能正確傳送至上層 Sequence FSM。
3. 補齊 Sequence FSM 對應的 Warm Reset、Global Reset 與 Recovery 轉移。
4. 將新版 Sleep-State 實體訊號經同步與 Debounce 後送入 DDR5 FSM。
5. 依板端量測結果調整 Power-Fail 輸入的穩定資格時間。

## 驗證結果

| 驗證項目 | 結果 |
|---|---|
| RTL hierarchy 與端到端訊號路徑確認 | 完成 |
| 雙向訊號 Driver、Hi-Z 與實體回讀確認 | 完成 |
| 板端波形與 Debounce 時間確認 | 完成 |
| DDR5 Fault Case 與 Sequence Reset 流程比對 | 完成 |
| 修正版本建置與基本開機測試 | 通過 |
| Power Cycle 壓力測試 | 1,000 次通過 |

修正前，異常可能在約 20～100 次 Power Cycle 內發生；修正後完成 1,000 次測試，未再出現原本的判定錯誤與 FSM 卡死問題。

## 從這個案例學到的事

- 正常開機通過只能驗證 Happy Path；Fault、Recovery 與重複 Cycle 必須另外測試。
- 分析 Inout 訊號時，要分開看 CPLD 主動拉低、釋放 Hi-Z 與實體 Pin 回讀，不能只依內部控制值判斷線路狀態。
- 改版可能改變訊號來源與路徑，即使功能名稱相同，也必須重新追蹤 Top-level 到使用模組的實際連線。
- Fault FSM 不只要能偵測異常，還需要定義完整的處理、完成與 Recovery 條件。
- 對偶發問題而言，修正後的長時間壓力測試是證明改善有效的重要依據。

