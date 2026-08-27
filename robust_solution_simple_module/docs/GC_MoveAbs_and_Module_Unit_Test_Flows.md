# GC MoveAbs Service 與 Module 單元測試驗證流程

## 1. 文件目的

本文件描述下列兩個 TcUnit Test Suite 目前各 Test Method 的驗證目的、測試資料、逐 scan 流程與主要斷言：

- `FB_GCSpinAxisMoveAbsServiceTests`
- `FB_GC_ModuleTests`

對應受測物件：

- `Untitled1/POUs/10_Module/11_GC/Services/FB_GCSpinAxisMoveAbsServices.TcPOU`
- `Untitled1/POUs/10_Module/11_GC/FB_GC_Module.TcPOU`

## 2. 共通測試原則

### 2.1 PLC scan 與 Fake Axis

- Service 測試每次呼叫 Service 後，必須再呼叫一次 `FakeAxis.M_CyclicUpdate()`，模擬 BaseUnit scan 與非同步 `Done/Error` 回授。
- Module 測試不可另外呼叫 `FakeAxis.M_CyclicUpdate()`；`FB_GC_Module.H_UpdateBaseUnit()` 已在每個有效 Module scan 更新 Spin 與 LiftPin BaseUnit 各一次。
- Fake Axis script 的 `AfterActiveCycles` 表示命令維持 active 幾個 BaseUnit scan 後才輸出結果。

### 2.2 PackML command request

- 外部命令以 `bCommandChangeRequest = TRUE` 觸發一次。
- 下一個命令前需將 request flag 拉回 `FALSE`，重新武裝 command edge。
- Service 內部呼叫 `RequestCommand()` 時，命令於下一個 Service scan 才會被 PackML base 處理。

### 2.3 Module 測試記憶體配置

- `FB_GC_Module` 與 `FB_FakeAxis_BaseUnit` 都宣告在 `FB_GC_ModuleTests` 的 suite-level `VAR`。
- Test Method 的 `VAR_INST` 只保存 phase、介面 reference、控制結構與 call record。
- 每個非同步 Module 測試使用獨立 Module/Fake fixture，避免 TcUnit 每 scan 呼叫所有 Test Method 時互相污染。
- 本批 Module 測試不使用 `FB_TestableGCModule`。

---

## 3. FB_GCSpinAxisMoveAbsServiceTests

## 3.1 `SuccessfulMoveAbsCompletesAndReleasesAxis`

### 驗證目的

驗證合法 MoveAbsolute 從 PackML 初始化到正常完成的完整流程，包含參數傳遞、非同步等待、ownership 與命令下降緣。

### 測試資料

- ServiceId：`21`
- Position：`125.25`
- Velocity：`15`
- Acceleration：`25`
- Deceleration：`35`
- MoveAbs script：active 2 cycles 後 `Done = TRUE`

### 逐 scan 流程

| Phase | 操作 | 預期結果 |
|---|---|---|
| 0 | 初始化 Fake Axis、參數與 Done script；呼叫 Service | `Undefined → Aborted` |
| 1a | 送 Clear | Service 到 `Stopped` |
| 1b | request flag 拉低 | Clear edge 重新武裝 |
| 1c | 送 Reset | Service 到 `Idle`，ErrorList 清空 |
| 2a | request flag 拉低 | Reset edge 重新武裝 |
| 2b | 送 Start | 參數驗證成功、取得 Axis ownership、Service 到 `Execute` |
| 3 | 第一次 Execute scan | 送出 `MoveAbs.Execute = TRUE`，Fake script active cycle 1 |
| 4 | 第二次 Execute scan | Done 尚未被 Service 觀察，保持 `Execute` 與 ownership；Fake script 在 scan 尾端產生 Done |
| 5 | 第三次 Execute scan | Service 讀到 Done，進入 `Completing`，仍持有 ownership |
| 6 | Completing scan | 送出 `MoveAbs.Execute = FALSE`，再釋放 ownership，進入 `Complete` |

### 主要驗證點

- MoveAbs call 的 Operation、OwnerId 與四個運動參數完全正確。
- `Execute = TRUE` 的 MoveAbs call 必須被 Fake Axis 接受。
- Done pending 期間不得提前進入 Completing 或釋放 ownership。
- Completing 的 MoveAbs falling edge 仍為 `Accepted = TRUE`，證明命令撤銷發生在 release 之前。
- 正常完成不產生 Service error。

## 3.2 `MoveAbsErrorStopsSafelyAndRecordsError`

### 驗證目的

驗證 Execute 中的 MoveAbs error 會留下正確錯誤紀錄，於下一 scan 排程 Stop，並等待非同步 Stop 完成後安全釋放資源。

### 測試資料

- ServiceId：`22`
- MoveAbs script：active 1 cycle 後 `Error = TRUE`
- Axis source ErrorId：`16#1234`
- Stop script：active 2 cycles 後 `Done = TRUE`

### 逐 scan 流程

| Phase | 操作 | 預期結果 |
|---|---|---|
| 0～2 | 初始化並完成 Clear、Reset、Start | Service 到 `Execute` 並持有 Axis |
| 3 | 第一次 MoveAbs active scan | Fake script 在 scan 尾端產生 Error；Service 仍為 `Execute` |
| 4 | Service 再次呼叫 MoveAbs | 讀到 Error、加入 ErrorList、queue Stop；本 scan 仍為 `Execute` |
| 5 | 處理 queued Stop | 進入 `Stopping`；先撤銷 MoveAbs，再送 `Stop.Execute = TRUE`；ownership 保留 |
| 6 | Stop pending scan | Service 保持 `Stopping`；Fake script 在 scan 尾端產生 Done |
| 7 | Service 讀到 Stop Done | 送出 `Stop.Execute = FALSE`、釋放 ownership、進入 `Stopped` |

### 主要驗證點

- ErrorList[1].MainErrorId 為 `Axis_Command_Error_During_Execute`。
- SourceErrorId 保留 `16#1234`。
- Error 發生 scan 只 queue Stop，不會在同一 scan 直接跳入 Stopping。
- Stopping call history 中，MoveAbs falling edge 必須排在 Stop rising edge 之前。
- Stop pending 期間 ownership 不得釋放。
- 最終 Stop falling edge 必須在 ownership release 前被接受。
- ErrorList[2] 保持空白，避免相同錯誤重複加入。

## 3.3 `InvalidParamStopsStartingWithoutAcquiringAxis`

### 驗證目的

驗證 Starting 先檢查 Service-owned parameter；無效參數不得取得 Axis，也不得送出任何實體命令。

### 測試資料

- ServiceId：`23`
- Velocity：`0`
- 其他運動參數合法

### 逐 scan 流程

| Phase | 操作 | 預期結果 |
|---|---|---|
| 0～1 | 初始化、Clear、Reset | Service 到 `Idle` |
| 2 | 送 Start | Starting 檢查到 Velocity 無效；記錄錯誤並 queue Stop；Service 保持 `Starting` |
| 3 | request flag 拉低並處理 queued Stop | 因 Service 從未持有 Axis，Stopping 立即完成並進入 `Stopped` |

### 主要驗證點

- MainErrorId 為 `Invalid_Param`。
- SourceErrorId 為 `0`。
- Message 為 `Velocity must be greater than zero.`。
- ServiceId `23` 從未成為 Axis owner。
- Fake Axis HistoryCount 全程為 `0`。
- Stopped 後錯誤仍保留且沒有 duplicate entry。

## 3.4 `ResourceAcquireFailureStopsStartingWithoutDisturbingOwner`

### 驗證目的

驗證 Axis 已被其他 owner 使用時，Starting 會失敗並回到 Stopped，而且不得干擾既有 owner。

### 測試資料

- ServiceId：`24`
- 既有 Axis OwnerId：`99`
- 所有運動參數合法

### 逐 scan 流程

| Phase | 操作 | 預期結果 |
|---|---|---|
| 0～1 | 初始化、Clear、Reset | Service 到 `Idle` |
| 2a | Owner `99` 預先 Acquire Axis | Axis owner 成為 `99` |
| 2b | Service 送 Start | Acquire 失敗、記錄錯誤並 queue Stop；Service 保持 `Starting` |
| 3 | request flag 拉低並處理 queued Stop | 因 ServiceId `24` 不是 owner，Stopping 不呼叫 Axis 並進入 `Stopped` |

### 主要驗證點

- MainErrorId 為 `Resource_Acquire_Failed_During_Starting`。
- SourceErrorId 為 `0`。
- Owner `99` 在 Starting failure 與 Stopping 後都保持不變。
- ServiceId `24` 從未取得 ownership。
- Fake Axis HistoryCount 全程為 `0`。
- ErrorList 不產生 duplicate entry。

---

## 4. FB_GC_ModuleTests

## 4.1 `MissingAxisPreventsModuleUpdate`

### 驗證目的

驗證 Module composition guard：Spin 或 LiftPin 任一 Axis interface 未綁定時，整個 Module 必須停止更新。

### 逐 scan 流程

| Phase | Axis 綁定狀態 | 預期結果 |
|---|---|---|
| 0 | Spin 已綁定、LiftPin 未綁定 | Module 保持 `Undefined`；七個 Service 保持 `Undefined`；Spin Fake 不更新 |
| 1 | Spin 未綁定、LiftPin 已綁定 | Module 仍保持 `Undefined`；LiftPin Fake 不更新 |

### 主要驗證點

- Module 不得呼叫 `M_UpdateModule()`。
- 已綁定 Fake Axis 的 CurrentCycle 保持 `0`。
- HistoryCount 保持 `0`。
- ServiceStatus 不得出現部分初始化。

## 4.2 `ModuleClearResetAndAutoStartSynchronizesAllServices`

### 驗證目的

驗證 Module command broadcast、七個 Service 狀態聚合，以及 Idle 自動 Start 的 scan timing。

### 逐 scan 流程

| Phase | Module 操作 | Module 狀態 | 七個 Service 狀態 |
|---|---|---|---|
| 0 | 第一次有效 scan | `Aborted` | 全部 `Aborted` |
| 1 | ModuleCtrl 送 Clear | `Clearing` | 全部已到 `Stopped` |
| 2 | request flag 拉低 | `Stopped` | 全部 `Stopped` |
| 3 | ModuleCtrl 送 Reset | `Resetting` | 全部已到 `Idle` |
| 4 | request flag 拉低 | `Idle` | 全部 `Idle` |
| 5 | Idle hook queue Start | `Idle` | 全部 `Idle` |
| 6 | 處理 queued Start | `Execute` | 全部維持 `Idle`，改由外部 ServiceCtrl 控制 |

### 主要驗證點

- Module 必須比 ServiceStatus 晚一個 scan 完成 Clearing/Resetting，證明它確實等待全部 Service。
- Clear RequestId `1001` 與 Reset RequestId `1002` 正確回應到 ModuleStatus。
- Module Execute 不會自動 Start 七個 Service。
- 七次有效 Module call 使兩個 Fake Axis 的 CurrentCycle 都恰好等於 `7`。

## 4.3 `ExecuteRoutesSelectedServiceControlAndParameters`

### 驗證目的

驗證 Module Execute 使用外部 ServiceCtrl，只啟動指定的 SpinAxisMoveAbs，並傳遞正確 ServiceId 與運動參數。

### 測試資料

- SpinAxisMoveAbs RequestId：`2001`
- Position：`125.25`
- Velocity：`15`
- Acceleration：`25`
- Deceleration：`35`

### 逐 scan 流程

| Phase | 操作 | 預期結果 |
|---|---|---|
| 0～6 | 完成 Module 初始化、Clear、Reset、Idle queued Start | Module 到 `Execute`，全部 Service 為 `Idle` |
| 7 | 只對 SpinAxisMoveAbs ServiceCtrl 送 Start | 該 Service 到 `Execute` 並取得 Spin Axis；其他 Service 保持 Idle |
| 8 | Service request flag 拉低 | SpinAxisMoveAbs 送出第一個 active MoveAbs command |

### 主要驗證點

- 只有 SpinAxisMoveAbs 進入 Execute。
- ServiceStatus.ResponseId 等於 `2001`。
- Spin Axis owner 為 `E_GCServiceId.SpinAxisMoveAbs`。
- Fake call 的 OwnerId、Position、Velocity、Acceleration、Deceleration 正確。
- LiftPin Fake Axis HistoryCount 保持 `0`。

## 4.4 `ModuleStopOverridesServiceControlAndWaitsForCleanup`

### 驗證目的

驗證 Module Stop 的安全優先權：離開 Execute 後，ModuleCtrl 必須覆蓋外部 ServiceCtrl，並等待 active Service 完成非同步 Stop cleanup。

### 測試資料

- Active Service：SpinAxisMoveAbs
- Module Stop RequestId：`3001`
- Stop script：active 2 cycles 後 `Done = TRUE`
- 衝突外部命令：SpinAxisJogCW Start

### 逐 scan 流程

| Phase | 操作 | 預期結果 |
|---|---|---|
| 0～6 | 完成 Module 初始化與自動 Start | Module 到 `Execute` |
| 7 | 外部 Start SpinAxisMoveAbs | MoveAbs Service 到 Execute 並取得 Spin Axis |
| 8 | request flag 拉低 | MoveAbs active command 已送出；設定延遲 Stop script |
| 9 | ModuleCtrl 送 Stop；外部同時要求 Start Jog | Module 到 `Stopping`；Jog Start 被忽略；MoveAbs 先撤銷後送 Stop；ownership 保留 |
| 10 | Stop pending scan | Module 與 MoveAbs Service 保持 `Stopping`；Fake 在 scan 尾端產生 Done |
| 11 | Service 觀察 Stop Done | Service 送 Stop falling edge、釋放 ownership並到 Stopped；Module 仍為 Stopping |
| 12 | Module 觀察全部 ServiceStatus | 七個 Service 全部 Stopped 後，Module 才進入 `Stopped` |

### 主要驗證點

- Module Stopping 時不採用外部 ServiceCtrl。
- SpinAxisJogCW 不得取得 ownership。
- MoveAbs falling edge 在 Stop rising edge 前出現。
- Stop pending 期間 active Service 保留 ownership。
- Stop falling edge 在 ownership release 前被 Fake Axis 接受。
- Module 必須比最後一個 Service 晚一個 scan進入 Stopped。
- 最終七個 Service 都是 Stopped，Spin Axis 無殘留 owner。

## 5. 驗證邊界

- 本文件描述的是 TcUnit runtime 預期行為；PLC Build 只能驗證 XML、型別與語法，不能取代 runtime assertion 執行。
- 本批工作不執行 TcUnit runtime，也不連線、下載、啟動或切換 TwinCAT Runtime。
- Alarm aggregation 與 SingleProcess ModuleDataList 測試尚未納入本批 Module suite。
