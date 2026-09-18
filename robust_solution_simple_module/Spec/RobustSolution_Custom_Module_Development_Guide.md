# Robust Solution Framework 客製 Module 開發指南

| 文件屬性 | 內容 |
|---|---|
| 文件版本 | 1.0 |
| 最後更新 | 2026-09-18 |
| 適用框架 | Robust Solution Framework |
| 範例專案 | `RobustSolutionDemoProject` |
| 範例 Module Type | `Chamber1` |
| Config Schema | Version 4 |

本文件說明客戶取得 Robust Solution Library 後，如何在自己的 TwinCAT PLC 專案中逐步開發 BaseUnit、Service 與 Module，並將 Module 註冊到 Configuration Framework，使已編譯進 PLC 的 Module instance 可以由 `module-config.json` 選擇、啟用、停用及完成 I/O／BaseUnit 綁定。

本文件定位為實作指南。若需要更完整的框架契約、生命週期與相容性規範，請搭配同目錄的 [Robust_Solution_Function_Object_Development_Spec.md](./Robust_Solution_Function_Object_Development_Spec.md) 閱讀。

---

## 1. 文件目的與適用範圍

### 1.1 目標讀者

本文件適用於下列工程師：

- 已取得 Robust Solution Library，準備建立客戶應用 PLC 專案。
- 需要封裝 Axis、Valve、Robot 或其他硬體能力為 BaseUnit。
- 需要建立單一功能 Service 或 Composite Service。
- 需要建立新的 Module Type，並讓 Module instance 可由 JSON 設定啟用。
- 需要將實體 I/O、Shared Memory、BaseUnit reference、Variable 與 Alarm 接入 Framework。

讀者應具備 TwinCAT 3、IEC 61131-3 Structured Text、Function Block、Interface、Reference、Enum、Structure 與 PLC cyclic execution 的基本知識。

### 1.2 完成本指南後的成果

完成所有步驟後，客戶專案應具備：

1. 一個或多個可被 Service 使用的 BaseUnit。
2. 一組由 Module 管理的 Service。
3. 一個繼承 `FB_ModuleBase` 的客製 Module。
4. Module instance、Runtime Config、Control／Status 與 typed references 的全域儲存空間。
5. 一個實作 `I_ModuleConfigurationAdapter` 的 Module Adapter。
6. 一個繼承 `FB_ApplicationConfigRoot` 的應用程式 `FB_ConfigRoot`。
7. 一份 schema v4 的 `module-config.json`。
8. 一個只需呼叫 `ConfigRoot.Run()` 的 `MAIN`。

### 1.3 Framework 與客戶程式的責任分界

Framework 負責：

- PackML 基礎狀態機。
- Service control coordination。
- BaseUnit resource ownership 的共通實作。
- Module Type registry、JSON 載入及 Runtime Config 建立。
- I/O mapping、reference binding、configured Variable／Alarm snapshot。
- Module-level Alarm、Variable 與 Service Data 彙整。
- Application-level cyclic input、Module polling 與 cyclic output 順序。

客戶程式負責：

- 定義機台專屬 BaseUnit interface 與 implementation。
- 定義 Service 參數、錯誤與實際流程。
- 定義 Module 的公開 Ctrl／Param／Status 與硬體 I/O。
- 組合 Service 與 BaseUnit。
- 實作特定 Module Type 的 Configuration Adapter。
- 在 `FB_ConfigRoot` 註冊 I/O、BaseUnit 與 Module Type。
- 提供並部署 `module-config.json`。

### 1.4 目前 Config 動態能力的範圍

本框架所稱的 Config 驅動增減，是指不修改 PLC 程式即可在設定檔中選擇已編譯且已註冊的 Module Type／Slot。設定只在 PLC 啟動初始化時套用；修改 JSON 後必須重新啟動 PLC。目前不支援 PLC 運行中的 hot reload。

---

## 2. 整體架構與組裝關係

客戶端功能由下而上組裝：

```text
Hardware / PLC I/O
        ↓
BaseUnit
        ↓
Service
        ↓
Module
        ↓
Module Configuration Adapter
        ↓
FB_ConfigRoot
        ↓
module-config.json
```

各層責任如下：

| 層級 | 主要責任 |
|---|---|
| BaseUnit | 封裝硬體命令、feedback、cyclic update 與 resource ownership |
| Service | 封裝一項操作能力，管理參數、PackML 生命週期、Error 與 Data |
| Module | 組合 Service／BaseUnit，對外呈現 Module control、status、Variable、Alarm 與 Data |
| Configuration Adapter | 將通用 Config 機制接到特定 Module Type 的 storage、I/O nodes 與 references |
| `FB_ConfigRoot` | 註冊 application resources，並在每個 scan 執行已啟用 Module |
| JSON Config | 選擇 Module instance，設定 mapping、references、Variable 與 Alarm |

### 2.1 三種註冊不要混淆

1. **BaseUnit registration**：在 `H_RegisterBaseUnits()` 中將 BaseUnit instance 註冊成 JSON 可引用的 reference source。
2. **Service registration**：在 Module 的 `H_RegisterServices()` 中將 Service ID、Ctrl、Param 與 Status storage 登錄至 Service Coordinator。
3. **Module Type registration**：在 `H_RegisterModuleTypes()` 中將 `moduleType` 名稱與 Configuration Adapter 登錄至 Module Type Registry。

### 2.2 Slot 與 Module ID

- **Slot**：某一 Module Type 自己的 instance array index，目前範圍為 `1..Param_Config.MaxModulePerType`。
- **Module ID**：啟用 Module 的非零、跨 Module Type 全域唯一識別。
- 不同 Module Type 可以使用相同 Slot，但不能使用相同 Module ID。

---

## 3. 建立客戶 PLC 專案

### 3.1 加入 Library references

在客戶 PLC project 中加入交付的 Robust Solution placeholder／library reference，並加入功能所需的 Beckhoff libraries。`RobustSolutionDemoProject` 目前包含：

- Robust Solution Library placeholder
- `Tc2_MC2`
- `Tc2_Standard`
- `Tc2_System`
- `Tc3_Module`

客戶專案實際需要的 Beckhoff library 依硬體與功能而定；例如使用 NC Axis 時需要 motion 相關型別與 FB。

### 3.2 建議目錄

```text
GVLs/
  GVL_IO
  GVL_Module
  GVL_ShareMemory           // 選用

POUs/
  FB_ConfigRoot
  MAIN

  10_<ModuleType>/
    DUTs/
      Ctrl Status Interface/
      External IO/
      Service Param/
    Services/
    FB_<ModuleType>_Module
    FB_<ModuleType>ModuleConfigurationAdapter
```

BaseUnit 若為客戶專屬 implementation，可放在獨立 `BaseUnits/` 目錄；若使用 Library 已提供的 BaseUnit，直接在 `GVL_IO` 建立 instance 即可。

### 3.3 命名一致性

以下名稱會跨 PLC 程式與 JSON 使用，應在開發初期確定：

| 項目 | Chamber1 範例 |
|---|---|
| Module Type name | `Chamber1` |
| Module FB | `FB_Chamber1_Module` |
| Adapter | `FB_Chamber1ModuleConfigurationAdapter` |
| Service ID enum | `E_Chamber1ServiceId` |
| Runtime array | `GVL_Module.Chamber1_Runtime` |
| Module instance array | `GVL_Module.Chamber1` |

`moduleType` 會以大小寫完全一致的字串比對；更名時必須同步更新 PLC registration 與部署端 JSON。

---

## 4. Step 1：開發自訂 BaseUnit

### 4.1 BaseUnit 的角色

BaseUnit 封裝可被 Service 重複使用的硬體能力，例如：

- Axis motion
- Valve／Cylinder
- Robot command interface
- Pump／Heater／Vacuum controller
- 需要互斥操作權的其他設備資源

Service 應依賴 BaseUnit interface，不應直接操作原廠 FB 或硬體結構。這使 Service 可以在不改變流程程式的情況下替換實體 implementation 或測試 fake。

### 4.2 定義 BaseUnit interface

需要 resource ownership 的 interface 應繼承 `I_ResourceLock`，並只公開 Service 真正需要的命令與 feedback。

以下為簡化範例：

```iecst
INTERFACE I_CustomAxis_BaseUnit EXTENDS I_ResourceLock
```

```iecst
METHOD M_CyclicUpdate
```

```iecst
METHOD M_MoveAbsolute : BOOL
VAR_INPUT
    OwnerId : T_ResourceOwnerId;
    Execute : BOOL;
    Position : LREAL;
    Velocity : LREAL;
END_VAR
VAR_OUTPUT
    Done : BOOL;
    Error : BOOL;
    ErrorId : UDINT;
END_VAR
```

```iecst
PROPERTY ActualPosition : LREAL
```

`OwnerId` 必須傳到所有需要 ownership 的硬體命令，使 BaseUnit 可以拒絕非 Owner 的操作。

### 4.3 實作 BaseUnit

自訂 BaseUnit 繼承 `FB_BaseUnit`，並實作自訂 interface：

```iecst
FUNCTION_BLOCK FB_CustomAxis_BaseUnit
EXTENDS FB_BaseUnit
IMPLEMENTS I_CustomAxis_BaseUnit
VAR
    _Axis : AXIS_REF;
    _MoveAbsolute : MC_MoveAbsolute;
END_VAR
```

在 command method 中先檢查 ownership：

```iecst
M_MoveAbsolute := FALSE;
Done := FALSE;
Error := FALSE;
ErrorId := 0;

IF NOT M_IsResourceOwner(OwnerId := OwnerId) THEN
    RETURN;
END_IF

_MoveAbsolute.Execute := Execute;
_MoveAbsolute.Position := Position;
_MoveAbsolute.Velocity := Velocity;

Done := _MoveAbsolute.Done;
Error := _MoveAbsolute.Error;
ErrorId := _MoveAbsolute.ErrorId;
M_MoveAbsolute := TRUE;
```

在 `M_CyclicUpdate` 中呼叫實際硬體 FB 並更新 feedback：

```iecst
_MoveAbsolute(Axis := _Axis);
_Axis.ReadStatus();
```

### 4.4 在 GVL_IO 建立 instance

```iecst
{attribute 'qualified_only'}
VAR_GLOBAL
    NC_Axis1 : FB_Axis_BaseUnit;
    NC_Axis2 : FB_Axis_BaseUnit;
END_VAR
```

建立 instance 只是配置 PLC storage；若要讓 JSON `references` 可以使用，還必須完成下一章的 registration。

---

## 5. Step 2：將 BaseUnit 註冊為可配置 Reference

### 5.1 在 FB_ConfigRoot 註冊

`FB_ConfigRoot` 繼承 `FB_ApplicationConfigRoot`，並覆寫 `H_RegisterBaseUnits()`：

```iecst
METHOD PROTECTED H_RegisterBaseUnits
```

```iecst
M_RegisterBaseUnit(BaseUnit := GVL_IO.NC_Axis1);
M_RegisterBaseUnit(BaseUnit := GVL_IO.NC_Axis2);
```

Framework 會透過 BaseUnit 提供的 ADS symbol metadata 解析完整 symbol name，呼叫端不需要另外輸入名稱。

### 5.2 JSON reference source

完成註冊後，JSON 可以使用完整 ADS symbol name：

```json
"references": {
  "SpinAxis_BaseUnit": "GVL_IO.NC_Axis1",
  "LiftPinAxis_BaseUnit": "GVL_IO.NC_Axis2"
}
```

- JSON key 是 Module 的 reference port 相對路徑。
- JSON value 是已向 Reference Manager 註冊的 BaseUnit source。
- Module port 名稱區分大小寫。
- 註冊名稱、JSON value 或 typed interface 不相容時，Configuration 初始化會失敗。

Reference registration 不會自動呼叫 BaseUnit 的 `M_CyclicUpdate`；cyclic composition 仍由 Module 負責。

---

## 6. Step 3：開發單一功能 Service

### 6.1 定義 Service 契約

建立 Service 前先定義：

1. Service ID。
2. Parameter DUT；沒有參數時可使用 `ST_EmptyServiceParam`。
3. 可能使用的 Error Code。
4. 所需的 BaseUnit interface 或 I/O reference。
5. Completion Behavior。

Chamber1 使用 `E_Chamber1ServiceId` 管理同一 Module 內的 Service ID：

```iecst
{attribute 'qualified_only'}
{attribute 'to_string'}
TYPE E_Chamber1ServiceId :
(
    None := 0,
    SpinAxisMoveAbs := 2,
    DoorOpen := 9,
    VacuumOn := 11,
    PreparePosition := 13
) UDINT;
END_TYPE
```

`0` 應保留為 None／invalid。每一個註冊至同一 Module 的 Service 必須使用不同的非零 ID。

MoveAbs 參數範例：

```iecst
TYPE ST_Chamber1SpinAxisMoveAbsParam :
STRUCT
    Position : LREAL;
    Velocity : LREAL;
    Acceleration : LREAL;
    Deceleration : LREAL;
END_STRUCT
END_TYPE
```

### 6.2 建立 Service Function Block

Service 繼承 `FB_ServiceBase`，並宣告功能所需的參數與 dependencies：

```iecst
FUNCTION_BLOCK FB_Chamber1SpinAxisMoveAbsServices
EXTENDS FB_ServiceBase
VAR_INPUT
    Param : ST_Chamber1SpinAxisMoveAbsParam;
    Axis_BaseUnit : I_Axis_BaseUnit;
END_VAR
```

Service body 每個 scan 呼叫一次 `M_UpdateService`：

```iecst
M_UpdateService(
    CompletionBehavior := E_ServiceCompletionBehavior.NaturalCompletion);
```

Completion Behavior 的選擇：

| Behavior | 使用時機 |
|---|---|
| `NaturalCompletion` | 工作達成後由 Execute 進入 Completing／Complete |
| `ExecuteUntilExternalStop` | Service 持續執行，直到外部命令要求停止 |

Module 呼叫 Service 時會提供繼承欄位 `Ctrl`、`Context`、`ServiceId`，並接收 `Status`。

### 6.3 實作 PackML State Hooks

Service 依功能需要覆寫 hooks。常用分工如下：

| Hook | 常見用途 |
|---|---|
| `H_OnStarting` | 參數驗證、取得 BaseUnit、確認啟動條件 |
| `H_OnExecute` | 執行主要命令並判斷 Done／Error |
| `H_OnCompleting` | 撤銷完成後仍有效的命令 |
| `H_OnStopping` | 執行正常停止 |
| `H_OnAborting` | 執行錯誤後的安全處理 |
| `H_OnClearing` | 清除 Service Error |
| `H_OnIdle`、`H_OnComplete`、`H_OnStopped`、`H_OnAborted` | 維持穩態或執行簡單防禦性清理 |

回傳 `E_PackMLHookResult.Busy` 表示動作尚未完成；回傳 `Done` 表示 Framework 可以進行下一個 state transition。

簡單 Vacuum Service 只需要在 Execute 設定 output 並等待 feedback：

```iecst
H_OnExecute := E_PackMLHookResult.Busy;
VacuumOnCtrl := TRUE;

IF IsVacuumOn THEN
    H_OnExecute := E_PackMLHookResult.Done;
END_IF
```

當 Service 需要主動進入其他 PackML state，可使用 `RequestCommand`：

```iecst
M_AddError(
    MainErrorId := E_Chamber1ErrorCode.Axis_Command_Error_During_Execute,
    SourceErrorId := nErrorId,
    Message := 'MoveAbsolute failed during Execute.');
RequestCommand(E_PackMLCommand.Abort);
```

### 6.4 BaseUnit Resource Lock

當多個 Service 可能操作同一個 BaseUnit 時，使用 Resource Lock 確保同一時間只有一個 Service 可以控制該資源。

`I_ResourceLock` 提供：

| 方法 | 用途 |
|---|---|
| `M_AcquireResource` | 嘗試取得 BaseUnit 操作權 |
| `M_IsResourceOwner` | 確認目前 Service 是否為 Owner |
| `M_ReleaseResource` | 釋放 BaseUnit 操作權 |

Service 使用 Framework 傳入的 `ServiceId` 作為 `OwnerId`：

```iecst
IF NOT Axis_BaseUnit.M_AcquireResource(OwnerId := ServiceId) THEN
    M_AddError(
        MainErrorId := E_Chamber1ErrorCode.Resource_Acquire_Failed_During_Starting,
        SourceErrorId := 0,
        Message := 'Axis resource acquisition failed during Starting.');
    RequestCommand(E_PackMLCommand.Abort);
    RETURN;
END_IF
```

操作 BaseUnit 時持續傳入相同的 ID：

```iecst
Axis_BaseUnit.M_MoveAbs(
    OwnerId := ServiceId,
    Execute := TRUE,
    Position := Param.Position,
    Velocity := Param.Velocity,
    Acceleration := Param.Acceleration,
    Deceleration := Param.Deceleration,
    Done => bDone,
    Error => bError,
    ErrorId => nErrorId);
```

使用完畢後釋放：

```iecst
Axis_BaseUnit.M_ReleaseResource(OwnerId := ServiceId);
```

實作注意事項：

- `OwnerId = 0` 代表未占用，不可作為有效 Service ID。
- 先驗證 Service 參數，再取得 BaseUnit。
- 取得、操作與釋放必須使用相同的 `ServiceId`。
- 取得失敗時，不應繼續呼叫該 BaseUnit 的操作方法。
- 釋放前應先撤銷仍有效的硬體命令。
- 同一個 BaseUnit 的所有使用者必須使用彼此不同的 Owner ID。
- Resource Lock 只管理操作權；硬體安全處理仍由 Service 負責。

### 6.5 使用 ErrorList 回報與清除錯誤

#### M_AddError

使用 `M_AddError` 將 Service Error 寫入 `Status.ErrorList`：

```iecst
M_AddError(
    MainErrorId := E_Chamber1ErrorCode.Invalid_Param,
    SourceErrorId := 0,
    Message := 'Velocity must be greater than zero.');
```

欄位用途：

- `MainErrorId`：Service 定義的主要錯誤識別。
- `SourceErrorId`：原廠 FB 或底層命令提供的錯誤碼；沒有時使用 `0`。
- `Message`：供診斷使用的文字。

Framework 會填入 Module ID、Service ID、Service instance path、UTC timestamp、severity 與 alarm lifecycle 基本欄位。同一輪中相同 `MainErrorId` 不會重複加入；每個 Service 最多保留 `Param_Config.MaxServiceError` 筆 Error。

#### M_ClearAllError

需要清除 Service Error 時呼叫：

```iecst
METHOD PROTECTED H_OnClearing : E_PackMLHookResult
```

```iecst
M_ClearAllError();
H_OnClearing := E_PackMLHookResult.Done;
```

`M_ClearAllError` 會清除該 Service 的完整 `Status.ErrorList`。目前範例將它當成命令型方法使用，不依賴其回傳值。

### 6.6 使用 DataVariableList 發布 Service Data

ErrorList 用於異常資訊；DataVariableList 用於發布執行結果、量測值或 Service 過程資料。Service Data 儲存在 `Status.Data`，並由 Module 彙整到 `ModuleDataList`。

#### M_UpdateData

```iecst
M_UpdateData(
    DataId := 1,
    Name := 'ActualPosition',
    DataType := E_DescriptorType.Float_64bit,
    Unit := 'mm',
    Value := TO_STRING(Axis_BaseUnit.AxisActualPosition));
```

- `DataId` 必須非零。
- 相同 `DataId` 會更新原本項目。
- 新 ID 會使用第一個空位並增加 `Status.Data.Count`。
- 容量已滿時回傳 `FALSE`，並設定 `Status.Data.Overflow`。

#### M_ClearData

```iecst
bCleared := M_ClearData(DataId := 1);
```

找到 ID 時會清除該項目、減少 Count 並回傳 `TRUE`；找不到或 ID 為零時回傳 `FALSE`。

#### M_ClearAllData

```iecst
M_ClearAllData();
```

此方法清除完整 `Status.Data`，包含 DataList、Count 與 Overflow。

---

## 7. Step 4：開發 Composite Service（選用）

Composite Service 用來協調多個已註冊 Service，不應直接繞過 Service Coordinator 操作其他 Service 的 Ctrl／Param storage。

### 7.1 建立 Composite Service

```iecst
FUNCTION_BLOCK FB_Chamber1PreparePositionService
EXTENDS FB_CompositeServiceBase
VAR_INPUT
    Param : ST_Chamber1PreparePositionParam;
END_VAR
VAR
    _Step : E_Chamber1PreparePositionStep;
    _CommandSubmitted : BOOL;
END_VAR
```

Body 呼叫：

```iecst
M_UpdateCompositeService();
```

### 7.2 宣告 dependencies

```iecst
METHOD PROTECTED H_DeclareServices
```

```iecst
M_AddDependency(ServiceId := E_Chamber1ServiceId.LiftPinAxisMoveAbs);
M_AddDependency(ServiceId := E_Chamber1ServiceId.SpinAxisMoveAbs);
```

被宣告的 Service 必須已註冊至同一個 Module 的 Service Coordinator，且其 registration 必須允許 composite call。

### 7.3 啟動與觀察子 Service

```iecst
SubmitResult := M_StartDependency(
    ServiceId := E_Chamber1ServiceId.LiftPinAxisMoveAbs,
    Parameters := _LiftReadyParam);
```

提交成功後，用以下方法觀察執行：

```iecst
CommandState := M_GetDependencyCommandState(
    ServiceId := E_Chamber1ServiceId.LiftPinAxisMoveAbs);

IF M_GetDependencyServiceSnapshot(
    ServiceId := E_Chamber1ServiceId.LiftPinAxisMoveAbs,
    Snapshot => Snapshot) THEN
    // Read Snapshot.PackMLOut, errors, and other status.
END_IF
```

若流程本身失敗，使用 `M_FailSequence` 提供 Composite error、失敗 Service ID 與來源錯誤：

```iecst
M_FailSequence(
    ErrorCode := E_CompositeServiceErrorCode.InternalCallSubmitFailed,
    FailedServiceId := E_Chamber1ServiceId.LiftPinAxisMoveAbs,
    SourceErrorId := TO_UDINT(SubmitResult),
    Message := 'LiftReady Start could not be queued.');
```

### 7.4 Execute substate

若上位系統需要觀察 Composite step，可覆寫：

```iecst
H_GetExecuteSubState := TO_UDINT(_Step);
H_GetExecuteSubStateName := TO_STRING(_Step);
```

---

## 8. Step 5：建立 Module 的公開資料結構

### 8.1 External I/O DUT

Chamber1 將可配置的 Module input／output 集中在兩個 DUT：

```iecst
TYPE ST_Chamber1_Signal_In :
STRUCT
    bDoorClosed : BOOL;
    bVacuumOn : BOOL;
    rTemperature : REAL;
    bRobotInChamber : BOOL;
    bDIs : ARRAY[1..16] OF BOOL;
    rAIs : ARRAY[1..16] OF REAL;
END_STRUCT
END_TYPE
```

```iecst
TYPE ST_Chamber1_Signal_Out :
STRUCT
    bDoorClosed : BOOL;
    bVaccumOn : BOOL;
    bDOs : ARRAY[1..16] OF BOOL;
    rAOs : ARRAY[1..16] OF REAL;
END_STRUCT
END_TYPE
```

這些欄位稍後由 Adapter 宣告成 Module nodes，供 input mapping、output mapping、configured Variable 與 Alarm 使用。

### 8.2 Service Ctrl／Param／Status

每個 Service 都需要 persistent storage：

```iecst
TYPE ST_Chamber1_ServiceCtrl :
STRUCT
    SpinAxisMoveAbs : ST_ServiceCtrl;
    VacuumOn : ST_ServiceCtrl;
    PreparePosition : ST_ServiceCtrl;
END_STRUCT
END_TYPE
```

```iecst
TYPE ST_Chamber1_ServiceParam :
STRUCT
    SpinAxisMoveAbs : ST_Chamber1SpinAxisMoveAbsParam;
    VacuumOn : ST_EmptyServiceParam;
    PreparePosition : ST_Chamber1PreparePositionParam;
END_STRUCT
END_TYPE
```

```iecst
TYPE ST_Chamber1_ServiceStatus :
STRUCT
    SpinAxisMoveAbs : ST_ServiceStatus;
    VacuumOn : ST_ServiceStatus;
    PreparePosition : ST_ServiceStatus;
END_STRUCT
END_TYPE
```

### 8.3 組合 Module CtrlStatus

Module-specific CtrlStatus 繼承 Framework base structure：

```iecst
TYPE ST_Chamber1_CtrlStatus EXTENDS ST_Module_CtrlStatus_Base :
STRUCT
    ServiceCtrl : ST_Chamber1_ServiceCtrl;
    ServiceParam : ST_Chamber1_ServiceParam;
    ServiceStatus : ST_Chamber1_ServiceStatus;
END_STRUCT
END_TYPE
```

Base structure 已包含 heartbeat、Module Ctrl／Status、ModuleVariableList、ModuleAlarmList 與 ModuleDataList。

---

## 9. Step 6：開發自訂 Module

### 9.1 宣告 Module interface 與 internal storage

```iecst
FUNCTION_BLOCK FB_Chamber1_Module EXTENDS FB_ModuleBase
VAR_INPUT
    HwInput : ST_Chamber1_Signal_In;
    ServiceCtrl : REFERENCE TO ST_Chamber1_ServiceCtrl;
    ServiceParam : REFERENCE TO ST_Chamber1_ServiceParam;
    SpinAxis_BaseUnit : I_Axis_BaseUnit;
    LiftPinAxis_BaseUnit : I_Axis_BaseUnit;
END_VAR

VAR_OUTPUT
    HwOutput : ST_Chamber1_Signal_Out;
    ServiceStatus : ST_Chamber1_ServiceStatus;
END_VAR

VAR
    _EffectiveServiceCtrl : ST_Chamber1_ServiceCtrl;
    _EffectiveServiceParam : ST_Chamber1_ServiceParam;
    _ServiceStatusStore : ST_Chamber1_ServiceStatus;

    _SpinAxisMoveAbsService : FB_Chamber1SpinAxisMoveAbsServices;
    _VacuumOnService : FB_Chamber1VacuumOnService;
    _PreparePositionService : FB_Chamber1PreparePositionService;
END_VAR
```

### 9.2 Module body

Required reference 未綁定時，Module 應保持 inert：

```iecst
IF (SpinAxis_BaseUnit = 0) OR (LiftPinAxis_BaseUnit = 0) THEN
    RETURN;
END_IF

M_UpdateModule();
ServiceStatus := _ServiceStatusStore;
```

`M_UpdateModule()` 由 Framework 管理 Module state、Service Coordinator、configured snapshot、Alarm、Data 與 Variable publication。

### 9.3 覆寫 Module hooks

#### H_UpdateService

在這裡每個 scan 呼叫 Service instance：

```iecst
_SpinAxisMoveAbsService(
    Ctrl := _EffectiveServiceCtrl.SpinAxisMoveAbs,
    Param := _EffectiveServiceParam.SpinAxisMoveAbs,
    Context := _ServiceContext,
    Axis_BaseUnit := SpinAxis_BaseUnit,
    ServiceId := E_Chamber1ServiceId.SpinAxisMoveAbs,
    Status => _ServiceStatusStore.SpinAxisMoveAbs);
```

#### H_UpdateBaseUnit

```iecst
SpinAxis_BaseUnit.M_CyclicUpdate();
LiftPinAxis_BaseUnit.M_CyclicUpdate();
```

每一個實體 BaseUnit 應由 composition owner 每個 scan 呼叫一次。不要因為多個 Service 使用同一 BaseUnit 而重複更新。

#### H_UpdateVariable（選用）

Module 可直接發布計算值或 BaseUnit 狀態：

```iecst
M_AddVariable(
    Id := 10,
    Name := 'Endpoint1_SendReady',
    DataType := E_DescriptorType.Boolean,
    Unit := '',
    Value := TO_STRING(_bEndpoint1_SendReady));
```

#### H_UpdateAlarm（選用）

Module-level 條件可以使用 `M_AddAlarm` 發布。若相同 identity 同時由有效 JSON Alarm 定義，configured Alarm 擁有優先權。

---

## 10. Step 7：將 Service 註冊到 Module

### 10.1 在 H_RegisterServices 登錄

Service 必須在 Module 的 `H_RegisterServices()` 中登錄一次：

```iecst
M_RegisterService(
    ServiceId := E_Chamber1ServiceId.SpinAxisMoveAbs,
    AllowCompositeCall := TRUE,
    UpperCtrl := ServiceCtrl.SpinAxisMoveAbs,
    EffectiveCtrl := _EffectiveServiceCtrl.SpinAxisMoveAbs,
    UpperParam := ServiceParam.SpinAxisMoveAbs,
    EffectiveParam := _EffectiveServiceParam.SpinAxisMoveAbs,
    Status := _ServiceStatusStore.SpinAxisMoveAbs);
```

各參數用途：

| 參數 | 用途 |
|---|---|
| `ServiceId` | Module 內唯一 Service identity |
| `AllowCompositeCall` | 是否允許 Composite Service 呼叫 |
| `UpperCtrl` | 上位控制來源提供的 Ctrl storage |
| `EffectiveCtrl` | Service 實際使用的 Ctrl storage |
| `UpperParam` | 上位控制來源提供的 Param storage |
| `EffectiveParam` | Start 接受後供 Service 使用的參數 snapshot |
| `Status` | Service persistent status storage |

Registration 與 cyclic call 是兩個不同步驟：

- `H_RegisterServices()` 只在第一次 `M_UpdateModule()` 時建立 registration table。
- `H_UpdateService()` 每個 PLC scan 呼叫 Service instance。

### 10.2 註冊 Composite Service definition

```iecst
_PreparePositionService.M_RegisterDefinition(
    CompositeServiceId := E_Chamber1ServiceId.PreparePosition,
    Coordinator := _ServiceCoordinator);
```

### 10.3 新增 Service 時的修改位置

新增一個 Service 通常需要同步修改：

1. `E_<ModuleType>ServiceId`。
2. Service Parameter DUT。
3. Service Ctrl／Param／Status aggregate DUT。
4. Module 內的 Service FB instance。
5. Module 的 effective Ctrl／Param 與 Status storage。
6. `H_RegisterServices()`。
7. `H_UpdateService()`。
8. 若為 Composite Service，再加入 `M_RegisterDefinition()` 與 dependency declarations。

---

## 11. Step 8：建立 Module 全域儲存空間

每種 Module Type 擁有自己的 instance arrays：

```iecst
{attribute 'qualified_only'}
VAR_GLOBAL
    Chamber1 : ARRAY[1..Param_Config.MaxModulePerType]
        OF FB_Chamber1_Module;

    Chamber1_Runtime : ARRAY[1..Param_Config.MaxModulePerType]
        OF ST_ModuleRuntimeConfig;

    Chamber1_Control : ARRAY[1..Param_Config.MaxModulePerType]
        OF ST_Chamber1_CtrlStatus;

    Chamber1_SpinAxis : ARRAY[1..Param_Config.MaxModulePerType]
        OF I_Axis_BaseUnit;

    Chamber1_LiftPinAxis : ARRAY[1..Param_Config.MaxModulePerType]
        OF I_Axis_BaseUnit;
END_VAR
```

用途如下：

| Array | 用途 |
|---|---|
| Module FB | 實際 Module instances |
| Runtime | Adapter 套用的 `ST_ModuleRuntimeConfig` |
| Control | 上位 Ctrl／Param 與公開 Status |
| Typed reference | Adapter 驗證後提交的 BaseUnit interfaces |

Module Type 的所有 arrays、Adapter slot 驗證及 `H_RunModules` 迴圈都應使用 `Param_Config.MaxModulePerType`，不要自行使用不同常數。

---

## 12. Step 9：實作 Module Configuration Adapter

Adapter 是通用 Configuration Framework 與特定 Module Type 之間的 seam。每個新的 Module Type 都需要一個 Adapter。

```iecst
FUNCTION_BLOCK FB_Chamber1ModuleConfigurationAdapter
IMPLEMENTS I_ModuleConfigurationAdapter
```

### 12.1 M_IsSlotSupported

```iecst
M_IsSlotSupported :=
    (Slot >= 1) AND (Slot <= Param_Config.MaxModulePerType);
```

### 12.2 M_ClearAllSlots

初始化新設定前清除 Module Type 的 runtime 與 references：

```iecst
FOR nSlot := 1 TO Param_Config.MaxModulePerType DO
    GVL_Module.Chamber1_SpinAxis[nSlot] := 0;
    GVL_Module.Chamber1_LiftPinAxis[nSlot] := 0;
    MEMSET(
        ADR(GVL_Module.Chamber1_Runtime[nSlot]),
        0,
        SIZEOF(GVL_Module.Chamber1_Runtime[nSlot]));
END_FOR
M_ClearAllSlots := TRUE;
```

### 12.3 M_PrepareInstance：宣告 Module nodes

先開始 Module registration session：

```iecst
bDeclared := LinkVariableManager.M_BeginModuleRegistration(
    Address := ADR(GVL_Module.Chamber1[Slot]),
    Size := SIZEOF(GVL_Module.Chamber1[Slot]),
    ModuleConfig := ModuleConfig);
```

接著宣告 Config 可用的 Module nodes：

```iecst
bDeclared := LinkVariableManager.M_RegisterModuleNode(
    Variable := GVL_Module.Chamber1[Slot].HwInput.rTemperature,
    Access := E_VariableAccess.ReadWrite);

bDeclared := LinkVariableManager.M_RegisterModuleNode(
    Variable := GVL_Module.Chamber1[Slot].HwOutput.bVaccumOn,
    Access := E_VariableAccess.ReadOnly);
```

Access 應符合使用方式：

- Module input 通常需要 mapping 寫入；若也要作為 Variable／Alarm source，使用 `ReadWrite`。
- Module output 作為 output mapping source，需要可讀，通常使用 `ReadOnly`。
- 未宣告的欄位不能出現在 mapping、configured Variable 或 configured Alarm 中。

完成後封存 registration 並取得完整 Module symbol：

```iecst
IF NOT LinkVariableManager.M_EndModuleRegistration(
    ModuleSymbol => ModuleSymbol) THEN
    ErrorMessage := LinkVariableManager.ErrorMessage;
    RETURN;
END_IF
```

### 12.4 設定 Module scope 並取得 references

```iecst
IF NOT ReferenceBindings.M_SetModuleScope(
    ModuleSymbol := ModuleSymbol,
    ErrorMessage => sReferenceError) THEN
    ErrorMessage := sReferenceError;
    RETURN;
END_IF
```

Required reference：

```iecst
BaseUnit := ReferenceBindings.M_TakeRequired(
    Port := GVL_Module.Chamber1[Slot].SpinAxis_BaseUnit,
    PortName => sPortName,
    ErrorMessage => sReferenceError);

IF BaseUnit = 0 THEN
    ErrorMessage := sReferenceError;
    RETURN;
END_IF
```

將通用 `I_BaseUnit` 驗證為 Module 所需的 typed interface：

```iecst
SpinAxis := 0;
IF NOT __QUERYINTERFACE(BaseUnit, SpinAxis) THEN
    ErrorMessage := CONCAT(
        sPortName,
        ' requires I_Axis_BaseUnit.');
    RETURN;
END_IF
```

所有 references 都成功後再提交，避免部分綁定：

```iecst
GVL_Module.Chamber1_SpinAxis[Slot] := SpinAxis;
GVL_Module.Chamber1_LiftPinAxis[Slot] := LiftPinAxis;
M_PrepareInstance := TRUE;
```

選用 reference 可使用 `M_TakeOptional`；未提供時會回傳 `0` 且 `Present = FALSE`。若 JSON 有提供但 source 無效，仍視為設定錯誤。

### 12.5 M_ApplyRuntimeConfig

```iecst
IF NOT M_IsSlotSupported(Slot := Slot) THEN
    ErrorMessage := 'Chamber1 slot is outside the supported range.';
    RETURN;
END_IF

IF RuntimeConfig.ModuleType <> 'Chamber1' THEN
    ErrorMessage :=
        'Runtime configuration type is incompatible with the Chamber1 adapter.';
    RETURN;
END_IF

GVL_Module.Chamber1_Runtime[Slot] := RuntimeConfig;
M_ApplyRuntimeConfig := TRUE;
```

---

## 13. Step 10：在 FB_ConfigRoot 註冊 Module Type

```iecst
FUNCTION_BLOCK FB_ConfigRoot EXTENDS FB_ApplicationConfigRoot
VAR
    _Chamber1ModuleConfigurationAdapter :
        FB_Chamber1ModuleConfigurationAdapter;
END_VAR
```

### 13.1 H_RegisterIo

Infrastructure nodes 是 mapping 的外部 source／target，例如實體 I/O 與 Shared Memory。

```iecst
M_RegisterIoNode(
    Variable := GVL_IO.Term2_EL3602.Chl[nChannel],
    Access := E_VariableAccess.ReadOnly);

M_RegisterIoNode(
    Variable := GVL_IO.Term3_EL2809.Chl[nChannel],
    Access := E_VariableAccess.WriteOnly);

M_RegisterIoNode(
    Variable := GVL_ShareMemory.bChamber1_VacuumOn,
    Access := E_VariableAccess.ReadWrite);
```

- 實體 input 通常為 `ReadOnly`。
- 實體 output 通常為 `WriteOnly`。
- 需要雙向使用的 Shared Memory 使用 `ReadWrite`。

### 13.2 H_RegisterBaseUnits

```iecst
M_RegisterBaseUnit(BaseUnit := GVL_IO.NC_Axis1);
M_RegisterBaseUnit(BaseUnit := GVL_IO.NC_Axis2);
```

### 13.3 H_RegisterModuleTypes

```iecst
M_RegisterModuleType(
    ModuleTypeName := 'Chamber1',
    Adapter := _Chamber1ModuleConfigurationAdapter);
```

`ModuleTypeName` 必須和 JSON `moduleType` 完全一致。

### 13.4 H_RunModules

```iecst
FOR nModule := 1 TO Param_Config.MaxModulePerType DO
    IF GVL_Module.Chamber1_Runtime[nModule].Enabled
        AND GVL_Module.Chamber1_Runtime[nModule].Valid THEN

        GVL_Module.Chamber1[nModule](
            HeartbeatIndex :=
                GVL_Module.Chamber1_Control[nModule].HeartbeatIndex,
            ModuleCtrl :=
                GVL_Module.Chamber1_Control[nModule].ModuleCtrl,
            ServiceCtrl :=
                GVL_Module.Chamber1_Control[nModule].ServiceCtrl,
            ServiceParam :=
                GVL_Module.Chamber1_Control[nModule].ServiceParam,
            SystemContext := SystemContext,
            RuntimeConfig := GVL_Module.Chamber1_Runtime[nModule],
            VariableNodeReader := VariableNodeReader,
            SpinAxis_BaseUnit :=
                GVL_Module.Chamber1_SpinAxis[nModule],
            LiftPinAxis_BaseUnit :=
                GVL_Module.Chamber1_LiftPinAxis[nModule],
            ModuleStatus =>
                GVL_Module.Chamber1_Control[nModule].ModuleStatus,
            ServiceStatus =>
                GVL_Module.Chamber1_Control[nModule].ServiceStatus,
            ModuleAlarmList =>
                GVL_Module.Chamber1_Control[nModule].ModuleAlarmList,
            ModuleDataList =>
                GVL_Module.Chamber1_Control[nModule].ModuleDataList,
            ModuleVariableList =>
                GVL_Module.Chamber1_Control[nModule].ModuleVariableList);
    END_IF
END_FOR
```

每新增一種 Module Type，就需要在 `FB_ConfigRoot` 中建立對應 Adapter、完成 type registration，並在 `H_RunModules()` 加入該類型的 polling loop。

---

## 14. Step 11：建立 MAIN 與理解掃描順序

`MAIN` 只建立 Application Config Root 並呼叫 `Run()`：

```iecst
PROGRAM MAIN
VAR
    ConfigRoot : FB_ConfigRoot;
END_VAR
```

```iecst
ConfigRoot.Run();
```

不要在 `MAIN` 再直接呼叫已由 `FB_ConfigRoot` 管理的 Module 或 BaseUnit。

Configuration Ready 後，`Run()` 固定執行：

```text
Input Mapping Copy
        ↓
H_RunModules
        ↓
Output Mapping Copy
```

可使用以下 properties 觀察初始化狀態：

- `ConfigRoot.Ready`
- `ConfigRoot.Error`
- `ConfigRoot.ErrorCode`
- `ConfigRoot.ErrorMessage`

Module 只有在對應 Runtime Config 同時為 `Enabled = TRUE`、`Valid = TRUE` 時才會進入 cyclic call。

---

## 15. Step 12：撰寫 module-config.json

### 15.1 部署位置

Framework 預設從以下路徑載入 UTF-8 JSON：

```text
C:\ProgramData\RobustSolution\module-config.json
```

專案中的範例檔不會在 PLC build 時自動複製到 target，必須另外部署。

### 15.2 完整 Chamber1 範例

```json
{
  "schemaVersion": 4,
  "modules": [
    {
      "enabled": true,
      "moduleType": "Chamber1",
      "slot": 1,
      "id": 1,
      "inputMappings": [
        {
          "source": "GVL_ShareMemory.bChamber1_Endpoint1_DoorClosed",
          "target": "HwInput.bDoorClosed",
          "sourceDataType": "BOOL",
          "targetDataType": "BOOL"
        },
        {
          "source": "GVL_IO.Term2_EL3602.Chl[1]",
          "target": "HwInput.rTemperature",
          "sourceDataType": "DINT",
          "targetDataType": "REAL",
          "transform": {
            "scale": 0.01,
            "offset": 50.0
          }
        }
      ],
      "outputMappings": [
        {
          "source": "HwOutput.bVaccumOn",
          "target": "GVL_IO.Term3_EL2809.Chl[1]",
          "sourceDataType": "BOOL",
          "targetDataType": "BOOL"
        }
      ],
      "references": {
        "SpinAxis_BaseUnit": "GVL_IO.NC_Axis1",
        "LiftPinAxis_BaseUnit": "GVL_IO.NC_Axis2"
      },
      "variableList": [
        {
          "source": "HwInput.rTemperature",
          "id": 1,
          "name": "Chamber Temperature",
          "unit": "degC",
          "dataType": "REAL"
        }
      ],
      "alarmList": [
        {
          "source": "HwInput.rTemperature",
          "dataType": "REAL",
          "condition": {
            "operator": "gt",
            "value": 100.0,
            "tolerance": 0.01
          },
          "mainErrorId": 123,
          "sourceErrorId": 0,
          "message": "Chamber Temperature too high",
          "severity": "Alarm"
        }
      ]
    }
  ]
}
```

### 15.3 Module 基本欄位

| 欄位 | 說明 |
|---|---|
| `enabled` | 是否建立並執行此 Module instance |
| `moduleType` | 必須與 `M_RegisterModuleType` 名稱完全一致 |
| `slot` | 該 Module Type 的 instance array index |
| `id` | 啟用時必須非零，且跨所有 Module Type 唯一 |

Config 內不需要提供 Module ADS symbol；Framework 會由 Adapter registration 自動解析。

### 15.4 Input mapping

Input mapping 將 infrastructure node 複製到 Module node：

```text
Registered PLC I/O / Shared Memory
        → Module HwInput
```

- `source`：在 `H_RegisterIo()` 註冊且可讀的完整 symbol。
- `target`：Adapter 宣告且可寫的 Module-relative path。
- 無 `transform` 時，來源與目標型別及大小必須完全一致。
- 有 `scale`／`offset` 時，只支援 Framework 可轉換的數值型別。

### 15.5 Output mapping

Output mapping 將 Module node 複製到 infrastructure node：

```text
Module HwOutput
        → Registered PLC output / Shared Memory
```

- `source`：Adapter 宣告且可讀的 Module-relative path。
- `target`：在 `H_RegisterIo()` 註冊且可寫的完整 symbol。

### 15.6 References

```json
"references": {
  "SpinAxis_BaseUnit": "GVL_IO.NC_Axis1"
}
```

- key 必須是實際 Module reference port 的完整相對路徑。
- value 必須是已由 `M_RegisterBaseUnit` 註冊的 BaseUnit symbol。
- Required／Optional 由 Adapter 契約決定，不由 JSON 指定。

### 15.7 VariableList

Variable source 必須是該 enabled Module 已宣告且可讀的 node：

```json
{
  "source": "HwInput.rTemperature",
  "id": 1,
  "name": "Chamber Temperature",
  "unit": "degC",
  "dataType": "REAL"
}
```

`id` 必須非零；`dataType` 必須與註冊 node 的實際 metadata 一致。

### 15.8 AlarmList

Alarm 使用已註冊且可讀的 Module node，加上結構化 condition：

- `gt`
- `lt`
- `eq`
- `ge`
- `le`

`eq` 使用 `tolerance` 判定浮點相等，其他 operator 直接與 `value` 比較。`mainErrorId` 必須非零。

### 15.9 多 Slot 與停用範例

```json
{
  "enabled": false,
  "moduleType": "Chamber1",
  "slot": 2
}
```

Disabled entry 仍占用整份 configuration 的 entry capacity，但不需要非零 Module ID，也不會進入 Module cyclic call。

---

## 16. Config 驅動增減 Module 的實際行為

### 16.1 不修改 PLC 程式即可做的事

- 啟用或停用已註冊 Module Type 的 Slot。
- 為同一 Module Type 啟用多個已配置的 instances。
- 選擇已註冊的 I/O／Shared Memory nodes 建立 mappings。
- 選擇已註冊的 BaseUnit sources 綁定 references。
- 選擇已宣告的 Module nodes 建立 Variable 與 Alarm。

### 16.2 必須修改 PLC 程式的事

- 新增從未實作或註冊的 Module Type。
- 增加 Module Type 的新 I/O node。
- 增加新的 reference port 或改變 required／optional 契約。
- 新增 Service 或改變 Service interface。
- 增加超過 compile-time capacity 的 instances 或 descriptors。

### 16.3 套用時機

Config 只在 PLC 啟動初始化時載入並套用。修改 `enabled`、mapping、reference、Variable 或 Alarm 後，必須重新啟動 PLC。

目前行為不是：

- runtime hot reload
- TwinCAT Online Change
- 由 JSON 動態產生新的 Function Block 型別

JSON 是選擇與組裝已編譯資源的機制，不是執行期程式碼生成機制。

---

## 17. 完整啟動與註冊時序

PLC 啟動後的完整流程如下：

1. `MAIN` 呼叫 `ConfigRoot.Run()`。
2. `FB_ApplicationConfigRoot` 開始 infrastructure registration session。
3. 呼叫 `H_RegisterIo()` 註冊硬體與 Shared Memory nodes。
4. 封存 infrastructure registration；失敗時回滾本次 declarations。
5. 呼叫 `H_RegisterBaseUnits()` 註冊 BaseUnit reference sources。
6. 呼叫 `H_RegisterModuleTypes()` 註冊 Module Type 與 Adapter。
7. Configuration Manager 從 `Param_Config.ModuleConfigFilePath` 載入 JSON。
8. 驗證 schema version、Module Type、Slot、Module ID 與重複項目。
9. 對每個已註冊 Module Type 呼叫 Adapter `M_ClearAllSlots()`。
10. 對 enabled Module 呼叫 Adapter `M_PrepareInstance()`。
11. Adapter 宣告 Module nodes 並取得 Module symbol。
12. Adapter 設定 Module scope、解析 references 並完成 typed-interface 驗證。
13. Framework 驗證並建立 input／output mappings。
14. Framework 建立 configured Variable／Alarm runtime bindings。
15. 呼叫 Adapter `M_ApplyRuntimeConfig()` 寫入 Module Type runtime storage。
16. 所有設定成功後，Configuration Manager 進入 Ready。
17. 每個 PLC scan 執行 cyclic input copy。
18. `H_RunModules()` 執行所有 `Enabled` 且 `Valid` 的 Module。
19. 每個 Module 第一次執行時完成 Service registration。
20. Module 每個 scan 更新 Service、BaseUnit、Alarm、Data 與 Variable。
21. Application Root 執行 cyclic output copy。

任一步驟失敗時，`ConfigRoot.Ready` 不會成立；應從 `Error`、`ErrorCode` 與 `ErrorMessage` 取得第一個初始化錯誤。

---

## 附錄 A：Chamber1 範例檔案索引

| 目的 | 範例檔案 |
|---|---|
| PLC entry point | `RobustSolutionDemoProject/POUs/MAIN.TcPOU` |
| Application registration | `RobustSolutionDemoProject/POUs/FB_ConfigRoot.TcPOU` |
| Module global storage | `RobustSolutionDemoProject/GVLs/GVL_Module.TcGVL` |
| BaseUnit instances | `RobustSolutionDemoProject/GVLs/GVL_IO.TcGVL` |
| Module implementation | `RobustSolutionDemoProject/POUs/10_Chamber1/FB_Chamber1_Module.TcPOU` |
| Configuration Adapter | `RobustSolutionDemoProject/POUs/10_Chamber1/FB_Chamber1ModuleConfigurationAdapter.TcPOU` |
| Service IDs | `RobustSolutionDemoProject/POUs/10_Chamber1/DUTs/E_Chamber1ServiceId.TcDUT` |
| Simple Service | `RobustSolutionDemoProject/POUs/10_Chamber1/Services/Vacuum/FB_Chamber1VacuumOnService.TcPOU` |
| BaseUnit Service | `RobustSolutionDemoProject/POUs/10_Chamber1/Services/SpinAxis/FB_Chamber1SpinAxisMoveAbsServices.TcPOU` |
| Composite Service | `RobustSolutionDemoProject/POUs/10_Chamber1/Services/FB_Chamber1PreparePositionService.TcPOU` |
| JSON example | `config/module-config.json` |

## 附錄 B：Framework 容量速查

以下為目前 `Param_Config` 預設值。Library 升版後應以實際版本為準。

| Parameter | 值 | 範圍 |
|---|---:|---|
| `MaxModuleTypes` | 8 | 可註冊 Module Type 數量 |
| `MaxModulePerType` | 6 | 每一 Module Type 的 Slot 數量 |
| `MaxTotalConfiguredModules` | 48 | 整份 config 的 Module entries |
| `MaxServicesPerModule` | 100 | 每個 Module 的 Service registration 數量 |
| `MaxCompositeDependencies` | 10 | 每個 Composite Service 的 dependency 數量 |
| `MaxServiceError` | 10 | 每個 Service 的 ErrorList 容量 |
| `MaxServiceData` | 30 | 每個 Service 的 persistent Data 容量 |
| `MaxExecuteSubStateRecords` | 30 | 每次 Service run 的 substate history 容量 |
| `MaxModuleVariable` | 100 | 每個 Module 的 VariableList 容量 |
| `MaxConfiguredAlarmsPerModule` | 30 | 每個 Module 的 configured Alarm 容量 |
| `MaxHookAlarmsPerModule` | 30 | 每個 Module 的 Hook Alarm 容量 |
| `MaxServiceAlarmsPerModule` | 40 | 每個 Module 彙整的 Service Alarm 容量 |
| `MaxModuleData` | 50 | 每個 Module 的 DataList 容量 |
| `MaxModuleMappings` | 256 | 每個 Module 每一方向的 mapping 容量 |
| `MaxConfigVariableNodes` | 512 | 全系統可註冊 Variable nodes |
| `MaxModuleReferences` | 50 | 每個 Module 的 reference key 容量 |
| `MaxConfigReferenceNodes` | 100 | 全系統 BaseUnit reference sources |

## 附錄 C：新增 Module Type 最短路徑

若已熟悉各層契約，可依下列順序實作：

1. 定義 Module Type 名稱與 Service IDs。
2. 定義 External I/O、Service Ctrl／Param／Status 與 Module CtrlStatus DUT。
3. 建立或引用所需 BaseUnit interface／implementation。
4. 開發單一功能 Service。
5. 視需要開發 Composite Service。
6. 建立 Module FB，實作 `H_RegisterServices`、`H_UpdateService` 與 `H_UpdateBaseUnit`。
7. 在 `GVL_Module` 建立 Module、Runtime、Control 與 typed reference arrays。
8. 建立 `I_ModuleConfigurationAdapter` implementation。
9. 在 `FB_ConfigRoot` 註冊 infrastructure I/O、BaseUnits 與 Module Type。
10. 在 `H_RunModules()` 加入 Module Type polling loop。
11. 建立 `MAIN` 並只呼叫 `ConfigRoot.Run()`。
12. 部署 `module-config.json`，重新啟動 PLC 並確認 `ConfigRoot.Ready`。
