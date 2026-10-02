# HSBP Drive Hot-plug 與 Fault LED 資料映射修正

> **案例類型：** FPGA／CPLD Interface Decode、Register Mapping 與 LED FSM Debug  
> **使用工具：** Verilog／SystemVerilog、Quartus Prime、PCA9555／SMBus Register Analysis、板端 Hot-plug 測試  
> **驗證結果：** 板端測試通過；Drive 可正常偵測，Fault LED 顯示符合規格

## 案例摘要

系統驗證時發現，部分 Drive 在 Hot-plug 後無法被正確偵測；Drive 插入後，Fault LED 也會出現不符合規格的閃爍行為。

我分別追蹤 Drive 控制／狀態資料與 Fault LED 路徑，從 SMBus、PCA9555 Device Address、Port／Register、Bit Mapping 一路確認到對應 Drive Channel 與 LED FSM。最後定位到資料映射及 LED 狀態邏輯錯誤，完成修正後，Hot-plug 與四種 Fault LED 模式皆通過板端驗證。

## 我的責任

- 建立不同 Drive Slot 的 Hot-plug 與 LED 異常重現條件
- 追蹤 SMBus／PCA9555 資料到 Drive 控制與 LED 輸出的完整路徑
- 核對 Device Address、Port、Register、Bit 與 Drive Channel Mapping
- 依規格重新整理 Fault LED 的狀態與閃爍模式
- 執行各 Slot 的 Hot-plug、LED Pattern 與基本 Regression 測試

## 分析過程

### 1. 將 Hot-plug 與 LED 路徑分開追蹤

雖然兩個問題同時出現，但我沒有直接假設它們一定來自同一個邏輯，而是先建立兩條資料路徑：

```text
Hot-plug／Drive Control Path

Management Interface
        ↓
SMBus Device／Register Decode
        ↓
Per-Drive Control／Status Mapping
        ↓
Drive Slot


Fault LED Path

Host／Management LED Status
        ↓
PCA9555 Device Address
        ↓
Port／Register／Bit Decode
        ↓
Fault／Locate Mapping
        ↓
LED FSM／Physical LED Output
```

這個方式可以確認異常發生在 Bus Decode、Drive Channel Mapping，還是 LED FSM 本身。

### 2. 核對 PCA9555 Address 與 Register Mapping

針對 PCA9555，我逐層確認：

- 實際使用哪一組 SMBus
- Device Address 是否對應正確的 Expander
- 使用的是 7-bit Address，還是已包含 R／W Bit 的 Address Byte
- Port 0／Port 1 與內部 Register 是否對應正確
- 每個 Bit 最後對應哪一個 Drive Slot 與功能

檢查後發現，部分解碼後的資料被送到錯誤的 Address／Channel Mapping，因此只有特定 Drive 受到影響。這也解釋了為什麼問題不是所有 Slot 同時發生。

### 3. 確認 Drive Channel 的端到端對應

我將每個 Drive Slot 的資料來源、Expander Port、Bit Index 與最終控制／狀態逐項列表比對，確認：

- 每個 Slot 使用正確的 Device 與 Port
- 不同 Byte／Port 沒有交換
- Drive Index 沒有 Off-by-one 或跨組誤接
- 修改一個 Slot 時不會影響其他 Slot

修正 Mapping 後，原本無法穩定偵測的 Drive 可以正常完成 Hot-plug 流程。

### 4. 比對 Fault LED 的四種模式

Address Mapping 修正後，我再依規格檢查 Fault LED FSM，確認不同 Fault／Locate 狀態組合是否選到正確的 LED 行為。

原本部分狀態的對應與規格不一致，導致 Drive 插入時就出現非預期閃爍，且錯誤狀態無法呈現指定 Pattern。我重新整理輸入條件、控制優先順序與四種顯示模式，使每種狀態都有明確且互斥的輸出行為。

## Root Cause

本案例包含兩個相互關聯的原因：

1. **Hot-plug／Drive 異常：** PCA9555／SMBus 資料解碼後的 Address 與 Drive Channel Mapping 錯誤，部分資料被送到不正確的 Slot，導致特定 Drive 無法正常完成控制或偵測流程。
2. **Fault LED 異常：** Fault／Locate 資料映射與 LED FSM 的模式判斷不完整，造成 Drive 插入時誤觸發閃爍，並使部分錯誤狀態無法顯示規格要求的 Pattern。

## 修正內容

1. 修正 PCA9555 Device Address、Port／Register 與 Drive Channel 的對應。
2. 逐 Slot 核對資料 Bit 與實體 Drive 的 Mapping。
3. 修正 Fault／Locate 輸入到 LED FSM 的資料位置與優先順序。
4. 依規格重新整理四種 Fault LED 顯示模式。
5. 對所有 Drive Slot 執行 Hot-plug 與 LED Regression，避免只驗證原本異常的 Slot。

## 驗證結果

| 驗證項目 | 結果 |
|---|---|
| SMBus、Device Address 與 Register 路徑確認 | 完成 |
| Port／Bit／Drive Channel Mapping 比對 | 完成 |
| 修正版本建置 | 通過 |
| 各 Drive Slot 插入／拔除 Hot-plug 測試 | 通過 |
| Drive 插入時 Fault LED 不再誤閃 | 通過 |
| 四種 Fault LED 顯示模式 | 符合規格 |
| 其他 Drive Slot Regression | 通過 |

修正後，原本受影響的 Drive Slot 可以正常完成 Hot-plug 偵測；Drive 插入時不再出現非預期 Fault LED 閃爍，各 Fault Case 也能呈現對應的 LED Pattern。

## 從這個案例學到的事

- 分析 I²C／SMBus 問題時，要明確區分 7-bit Device Address、8-bit Address Byte、內部 Register、Port 與 Bit Mapping。
- 只有部分 Slot 發生異常時，應優先比較各 Instance 的 Address、Port 與 Index 差異，而不是先修改共用 FSM。
- 訊號名稱相同不代表資料來源相同，必須從 Bus Decode 一路追蹤到實體輸出。
- LED 功能不只要確認閃爍頻率，也要驗證輸入組合、控制優先順序、輸出極性與 Pattern 選擇。
- 修正共用介面或 Mapping 後，必須對所有 Slot 做 Regression，避免修好一組卻影響其他 Channel。


