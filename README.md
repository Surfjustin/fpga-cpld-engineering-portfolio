# FPGA / CPLD Engineering Portfolio

我主要負責伺服器平台的 FPGA／CPLD 板級控制，工作範圍涵蓋電源與 Reset 時序、跨板訊號、多節點控制、Hot-plug／LED 功能，以及偶發性硬體問題除錯。本作品集以去識別化方式整理代表性經驗。

## 核心能力

| 領域 | 經驗 |
|---|---|
| RTL 設計 | 使用 Verilog／SystemVerilog 修改控制邏輯、FSM、Timer 與模式選擇 |
| 板級整合 | 分析訊號方向、有效極性、Driver、Reset 預設值、Open-Drain／Hi-Z 與跨時脈輸入 |
| 系統除錯 | 從實體 Pin 追蹤至 Synchronizer、Filter、FSM 與輸出，建立並驗證 Root Cause 假設 |
| 驗證工具 | 使用 Quartus Compilation Report、Signal Tap、示波器與板端測試交叉驗證 |

## 代表性專案

### 1. HPM DIMM Power Cycle 偶發性判定失敗與 Fault Recovery 修正

**問題**

系統進行 Power Cycle 壓力測試時，約 20～100 次後可能誤進 DIMM Fault 流程，造成 Sequence FSM 卡在錯誤狀態並導致開機失敗。由於問題並非每次發生，單次正常開機無法證明流程可靠。

**個人貢獻**

- 建立可重現條件，比較正常與異常 Cycle 的 FSM 行為
- 追蹤雙向 Power-Fail、Sleep State、DDR5 Fault Case 與 Reset 路徑
- 確認 Input Synchronizer／Debounce、板卡改版後的訊號來源及 Recovery 條件
- 補齊 DDR5 Fault Case 與 Sequence FSM 的 Warm Reset／Global Reset 流程

**成果**

定位到 Sequence FSM 對 DDR5 Fault Case 的處理與 Recovery 流程不完整，並一併修正輸入資格及改版後的 Sleep-State 訊號路徑。修正前約 20～100 次可能發生異常；修正後完成 1,000 次 Power Cycle，未再出現原本的判定錯誤與 FSM 卡死問題。

**展現能力：** 偶發問題重現、FSM Reachability 分析、Power Sequence Debug、故障恢復設計。

---

### 2. DC-SCM 板卡改版：新增實體訊號與控制功能

**需求**

因應板卡改版，CPLD 需要加入多條新的實體控制訊號，並整合至既有電源、Reset 與狀態控制流程。工作範圍不只包含 Pin assignment，也必須定義新訊號在各系統狀態下的行為。

**個人貢獻**

- 對照硬體需求，確認每條訊號的方向、極性、Driver 與上電預設值
- 評估非同步輸入所需的 Synchronizer 與 Filter
- 完成 Top-level Port、Pin Constraint 及內部控制邏輯整合
- 檢查 Reset、訊號未就緒與異常輸入時的安全狀態
- 執行原有開機流程的 Regression，確認新功能沒有破壞既有行為

**成果**

完成改版訊號與功能整合；板端測試可正常啟動，新功能與原有流程皆可正常運作。

**展現能力：** 規格轉換、板級訊號整合、CDC／Filter 判斷、Regression 驗證。

---

### 3. PDB 雙節點風扇控制與關機電源時序修正

**問題**

雙節點系統的風扇轉速與狀態控制不符合預期；在關機流程中，控制時序也可能觸發過電流保護，造成系統斷電。

**個人貢獻**

- 確認 INIT／STANDBY／S0 階段的 CPLD／BMC 風扇控制權
- 依規格核對單／雙節點條件下的 Fan FSM State 與 PWM Duty
- 使用 Signal Tap 與示波器對照內部 State、風扇反應及電源關斷時序
- 分開處理 Fan FSM 設定錯誤與風扇機械慣性造成的關斷時序
- 協調 Fan FSM 與 PSU FSM 的完成時點，加入必要的穩定等待

**成果**

修正 Fan FSM State／PWM Duty 及關機交握後，風扇控制恢復預期行為，關機流程也不再觸發過電流保護。單／雙節點、單／雙 PSU 及滿載條件皆完成板端測試。

**展現能力：** RTL 與實體量測交叉除錯、機電反應與數位時序整合、保護流程設計。

---

### 4. PDB 雙節點 Master／Slave 控制權切換

**需求**

雙節點伺服器可能以不同順序安裝或上電；系統必須在單節點存在時仍可運作，並在雙節點狀態下明確決定控制權，避免兩端同時驅動或無人接管。

**個人貢獻**

- 定義單節點、雙節點及節點加入時的 Master／Slave 行為
- 新增控制權判斷與相關 RTL 訊號
- 設計狀態轉移及安全預設狀態
- 確認跨板 LTPI 訊號的來源、方向與使用條件

**目前狀態**

RTL 已完成並交付測試團隊；完整板端驗證仍在進行中，因此目前不將此項描述為正式驗證完成。

**展現能力：** 多節點仲裁、跨板通訊、控制權切換、Fail-safe 狀態設計。

---

### 5. HSBP Drive Hot-plug 與 Fault LED 資料映射修正

**問題**

Drive Hot-plug 後偶爾無法被正確偵測；Drive 插入時，Fault LED 也會出現不符合規格的閃爍行為。

**個人貢獻**

- 分開追蹤 Drive 控制／狀態與 Fault LED 的端到端資料路徑
- 核對 SMBus、PCA9555 Device Address、Port／Register、Bit 與 Drive Channel Mapping
- 逐 Slot 確認資料來源，排除 Port／Byte 交換與 Drive Index 錯接
- 修正 Fault／Locate Mapping、控制優先順序及四種 Fault LED 顯示模式

**成果**

確認 Hot-plug 與 LED 異常來自 PCA9555／SMBus Address 與 Drive Channel Mapping 錯誤，以及 LED FSM 模式判斷不完整。修正後，各 Drive Slot Hot-plug 測試通過，Drive 插入時不再誤閃，四種 Fault LED 模式皆符合規格。

**展現能力：** I²C／SMBus 位址分析、Register Mapping、端到端資料路徑追蹤、LED 狀態機除錯。

---

### 6. HSBP 單一 RTL 支援不同平台

**需求**

為降低多份程式分支的維護成本，同一套 CPLD RTL 需要支援兩種平台；兩者的狀態資料定義、輸入方式與 LED 控制語意並不完全相同。

**個人貢獻**

- 比較兩種平台的傳輸協定、Bit Mapping 與 LED 控制需求
- 將平台差異集中於模式解碼與資料映射層，保留共用控制邏輯
- 使用實體 Jumper 作為 Mode Select，於上電時選擇對應平台
- 為未支援的模式組合定義安全輸出，避免錯誤驅動
- 執行兩種模式下的 Drive 偵測與 LED 功能測試

**成果**

完成單一 RTL 的多平台支援；透過 Jumper 可選擇目標模式，板端測試中 Drive 與 LED 功能皆能正常運作。

**展現能力：** 多平台架構、協定映射、組態選擇、共用程式碼維護與 Safe Default 設計。

## 詳細案例

- [HPM DIMM Power Cycle 偶發性判定失敗與 Fault Recovery 修正](01-hpm-dimm-power-cycle-fault-recovery.md)
- [PDB 雙節點風扇控制與關機電源時序修正](02-pdb-dual-node-fan-shutdown-sequencing.md)
- [HSBP Drive Hot-plug 與 Fault LED 資料映射修正](03-hsbp-hotplug-led-address-mapping.md)

## 工作與驗證方式

1. 先定義可觀察的問題、發生條件與成功標準。
2. 確認訊號方向、有效極性、Driver 與 Reset 行為，再追蹤完整 RTL 路徑。
3. 將假設轉換成可觀察的訊號與測試，使用編譯報告、Signal Tap 或示波器驗證。
4. 分開標示 RTL 分析、編譯、模擬與板端測試結果；證據不足時保留為待確認事項。
