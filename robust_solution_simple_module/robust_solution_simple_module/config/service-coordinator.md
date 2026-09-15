# Service Coordinator 登錄與循環

`FB_ServiceCoordinator` 是 `FB_ModuleBase` 內部的 Service 登錄與控制分流 Module。衍生 Module 不直接操作 Coordinator，而是覆寫 `H_RegisterServices()`，並透過 `M_RegisterService()` 宣告每個受管理 Service 的持久儲存區。

## 每個 Service 必備儲存區

每筆登錄包含相同 ServiceId 對應的五個儲存位置：

- 上位 `ST_ServiceCtrl`
- 有效 `ST_ServiceCtrl`
- 上位 Param DUT
- 有效 Param DUT
- `ST_ServiceStatus`

Ctrl、Param 與 Status 都必須是 FB instance 或其 `REFERENCE TO` 輸入所指向的持久變數，不得傳入 method 暫存變數。Upper／Effective Param 必須有完全相同的大小與真實 DUT 名稱；Coordinator 初始化時會以位址解析型別，無法解析或只靠大小相同都會拒絕登錄。單欄位 STRUCT 因無法可靠反查 DUT 名稱，暫不支援。

```iecst
METHOD PROTECTED H_RegisterServices

M_RegisterService(
    ServiceId := E_MyServiceId.Move,
    AllowCompositeCall := TRUE,
    UpperCtrl := ServiceCtrl.Move,
    EffectiveCtrl := _EffectiveServiceCtrl.Move,
    UpperParam := ServiceParam.Move,
    EffectiveParam := _EffectiveServiceParam.Move,
    Status := _ServiceStatusStore.Move);
```

登錄只在第一次有效呼叫 `M_UpdateModule()` 時執行，完成後立即 Seal。錯誤採 first-error-wins，該 FB instance 不會在 runtime 重新登錄；修正 composition 後必須重新初始化 FB／PLC。

## 控制與參數分流

Module 不在 Execute 時，所有已登錄 Service 都接收 Module 的 PackML command、mode 與 RequestId。此期間上位維持為 TRUE 的 request 會被鎖住；Module 回到 Execute 後，必須先看到該 request 回到 FALSE，後續新上升沿才會送入 Service，避免舊命令延遲執行。

Module 在 Execute 時，Coordinator 將上位 Ctrl 寫入 Effective Ctrl。只有新的 Start request 上升沿會將 Upper Param 複製到 Effective Param；request 維持 TRUE 期間的參數修改不會影響 Service 本次執行。若 Service 已被 Composite 擁有，Coordinator 改派送 Composite 的內部命令，並屏蔽上位 command 與 mode request；Coordinator 不代寫 ResponseId，因此上位會以 timeout 辨識未送達的命令。

```text
Module Ctrl ───────────────┐
Composite internal Ctrl ───┼── Coordinator ──> Effective Ctrl ──> Service
Upper Ctrl ────────────────┘
Upper／internal Param ── Start snapshot ──> Effective Param ──> Service
Service ──> Status Store ──> Coordinator snapshot／摘要／Alarm／Data
```

控制優先序固定為 Module、Composite、上位。Coordinator 不修改 `ST_ServiceStatus`、`ResponseId` 或 `PackMLOut`。

### MoveVel 執行語意

Chamber1 的 Spin CW、Lift CW 與 Lift CCW MoveVel Service 均採 `NaturalCompletion`。Start 被接受時會鎖存 Velocity、Acceleration 與 Deceleration；Execute 期間修改上位 Param 不會改變當次命令，也不會以降沿／升沿重新觸發 MoveVel。Axis 回報 `InVelocity` 後，Service 即由 Execute 進入 Completing。

Completing 只將 MoveVel request 拉回 FALSE 並釋放 Axis ownership，不會另送 Stop，因此 Service 進入 Complete 時 Axis 仍可能依底層運動控制行為保持目標速度。若需要停止運動，呼叫端必須另外啟動負責停止或改變運動狀態的流程；PackML Stop／Abort 仍會走各 Service 既有的停止／中止處理。

## Composite Service

`FB_CompositeServiceBase` 本身是 `FB_ServiceBase` 與 PackML 狀態機。具體 Composite 透過 `H_DeclareServices()` 宣告最多 `Param_Config.MaxCompositeDependencies` 筆依賴，並在 `H_OnCompositeExecute()` 實作 Execute 步驟。依賴必須在一般 Service 登錄時設定 `AllowCompositeCall := TRUE`，且第一次取得 ownership 時回報 `NaturalCompletion`。

Module 必須先登錄所有 Service，再於 Coordinator Seal 前登錄 Composite definition：

```iecst
_CompositeService.M_RegisterDefinition(
    CompositeServiceId := E_MyServiceId.Composite,
    Coordinator := _ServiceCoordinator);
```

ownership 一次取得整份依賴清單，不允許部分成功。不同 Composite 的依賴集合不重疊時可以並行；Coordinator 不追蹤 BaseUnit，因此不同 Service 共用底層資源時仍由各 Service 如實回報停止或中止。正常完成會先把依賴 Reset 至 Idle 再釋放；Stopped／Aborted 則保留 ownership，直到 Clear／Reset 後全部回到 Idle。

`FB_ServiceCaller` 為每個 Composite 產生 instance-local、非零 RequestId。內部命令的 accepted／rejected 只表示 PackML 是否接受命令，實際執行結果由 Composite 讀取 Coordinator 保存的前一 scan snapshot 判斷。每筆 request 收到回覆後會強制回到 FALSE 一個 scan，才允許下一筆命令。

## Execute 子狀態

`ST_ServiceStatus.ExecuteSubState` 讓 Service 以 `UDINT` 揭露 Execute 內部的子狀態。此欄位只在 `PackMLOut.eStateCurrent = E_PackMLState.Execute` 時有效；其他 PackML 狀態由 `FB_ServiceBase` 統一清為 `0`。未覆寫 hook 的 Service 也固定回傳 `0`。

需要揭露子狀態的 Service 覆寫 `H_GetExecuteSubState()`，將自己的 enum 轉為 `UDINT`：

```iecst
METHOD PROTECTED H_GetExecuteSubState : UDINT

H_GetExecuteSubState := TO_UDINT(_Step);
```

每個子狀態 enum 都必須保留 `0` 表示 `None` 或未揭露。不同 Service 的數值不要求共用語意；上位或 HMI 必須依 `ServiceId` 使用對應的 enum mapping。Coordinator 會將此值複製到前一 scan 的 `ST_ServiceExecutionSnapshot`，讓 Composite 仍透過既有 snapshot 介面觀察子 Service。

新增欄位會改變 `ST_ServiceStatus` 的記憶體配置；實際部署後，上位與 HMI 必須重新載入產生的 PLC symbols，不得沿用舊版結構配置。

## 新增 Service 檢查清單

1. 在 Module 的 Upper Ctrl／Param 與公開 Status DUT 增加具名欄位。
2. 在 Module FB 增加對應 Effective Ctrl、Effective Param 與 Status Store 的持久欄位。
3. 指派非零且不重複的 ServiceId。
4. 決定是否允許 Composite 呼叫，並在 `H_RegisterServices()` 呼叫一次 `M_RegisterService()`。
5. 在 `H_UpdateService()` 只把 Effective Ctrl／Param 傳給受管理 Service，並把輸出寫入已登錄的 Status Store。
6. Module scan 結束後再將 Status Store 複製至公開 Status。
7. 若新增 Composite，先登錄 Composite 本身，再呼叫其 `M_RegisterDefinition()`；依賴數量上限為 10，且 v1 不允許巢狀 Composite。
8. 若需揭露 Execute 子狀態，覆寫 `H_GetExecuteSubState()`，並確保對應 enum 的 `0` 為 `None`。

未登錄的 Service 不參與 Module 的 AllIdle／AllStopped／AllAborted、Alarm 或 Data 彙整。
