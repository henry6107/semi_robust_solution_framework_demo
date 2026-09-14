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
    UpperCtrl := ServiceCtrl.Move,
    EffectiveCtrl := _EffectiveServiceCtrl.Move,
    UpperParam := ServiceParam.Move,
    EffectiveParam := _EffectiveServiceParam.Move,
    Status := _ServiceStatusStore.Move);
```

登錄只在第一次有效呼叫 `M_UpdateModule()` 時執行，完成後立即 Seal。錯誤採 first-error-wins，該 FB instance 不會在 runtime 重新登錄；修正 composition 後必須重新初始化 FB／PLC。

## 控制與參數分流

Module 不在 Execute 時，所有已登錄 Service 都接收 Module 的 PackML command、mode 與 RequestId。此期間上位維持為 TRUE 的 request 會被鎖住；Module 回到 Execute 後，必須先看到該 request 回到 FALSE，後續新上升沿才會送入 Service，避免舊命令延遲執行。

Module 在 Execute 時，Coordinator 將上位 Ctrl 寫入 Effective Ctrl。只有新的 Start request 上升沿會將 Upper Param 複製到 Effective Param；request 維持 TRUE 期間的參數修改不會影響 Service 本次執行。

```text
Upper Ctrl ── Coordinator ──> Effective Ctrl ──> Service
Upper Param ── Start 上升沿 ──> Effective Param ──> Service
Service ──> Status Store ──> Coordinator 摘要／Alarm／Data
```

Coordinator 不修改 `ST_ServiceStatus`、`ResponseId` 或 `PackMLOut`。Composite ownership、內部 Service 呼叫與外部命令屏蔽不屬於本階段。

## 新增 Service 檢查清單

1. 在 Module 的 Upper Ctrl／Param 與公開 Status DUT 增加具名欄位。
2. 在 Module FB 增加對應 Effective Ctrl、Effective Param 與 Status Store 的持久欄位。
3. 指派非零且不重複的 ServiceId。
4. 在 `H_RegisterServices()` 呼叫一次 `M_RegisterService()`。
5. 在 `H_UpdateService()` 只把 Effective Ctrl／Param 傳給受管理 Service，並把輸出寫入已登錄的 Status Store。
6. Module scan 結束後再將 Status Store 複製至公開 Status。

未登錄的 Service 不參與 Module 的 AllIdle／AllStopped／AllAborted、Alarm 或 Data 彙整。Chamber1 的 `SingleProcess` 目前刻意保留為未登錄的直接上位控制。
