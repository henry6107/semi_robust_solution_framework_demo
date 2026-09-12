# Module Alarm 與 SVID Hook

繼承 `FB_ModuleBase` 的 Module 可覆寫 `H_UpdateAlarm()` 與 `H_UpdateVariable()`，直接讀取已綁定的 Base Unit interface 或計算 Module 狀態，不必先將資料註冊成 JSON source。兩個 Hook 預設為空，既有 Module 不需新增覆寫。

## 呼叫時機與介面

每次 `M_UpdateModule()` 在 Service、Base Unit 更新後，執行以下流程；不限於 PackML Execute 狀態：

1. Alarm Manager 開始本輪，清除被有效 JSON 定義接管的舊 Hook lifecycle，處理上層確認請求。
2. 評估 JSON Alarm，呼叫 `H_UpdateAlarm()`，彙整 Service Alarm，再更新生命週期並發布。
3. 執行既有 DVID 彙整，清空 SVID 清單，先發布 JSON SVID，再呼叫 `H_UpdateVariable()`。

JSON snapshot 保持既有取樣時機，位於 Service／Base Unit 更新前；Hook 直接讀取的是 Base Unit 更新後狀態。Module composition 因未綁定必要 reference 而提前返回時，不會執行這些 Hook。

| 方法 | 可見性 | 參數 |
|---|---|---|
| `H_UpdateAlarm()` | `PROTECTED`，可覆寫 | 無 |
| `H_UpdateVariable()` | `PROTECTED`，可覆寫 | 無 |
| `M_AddAlarm(...) : BOOL` | `PROTECTED FINAL` | `MainErrorId: UDINT`、`SourceErrorId: UDINT`、`Message: STRING`、`Path: STRING(255)`、`Severity: E_Severity` |
| `M_AddVariable(...) : BOOL` | `PROTECTED FINAL` | `Id: UDINT`、`Name: STRING(30)`、`DataType: E_DescriptorType`、`Unit: STRING(10)`、`Value: STRING(80)` |

提交方法只能從對應 Hook 的呼叫鏈使用。`FALSE` 表示錯誤呼叫階段、零 ID、Hook 重複 ID 或容量不足；重複時保留第一筆。`TRUE` 表示成功處理，包含被 JSON 整筆覆蓋而略過的項目，因此不保證該項目的 Hook 值出現在輸出。

## 衍生 Module 範例

以下為衍生 Module 的兩個 method body。`Axis` 是 Module 已綁定的 `I_Axis_BaseUnit`，並在既有 `H_UpdateBaseUnit()` 呼叫 `Axis.M_CyclicUpdate()`。範例 ID 只供說明，正式 Module 應使用自己的領域 ID。

```iecst
METHOD PROTECTED H_UpdateAlarm

IF Axis = 0 THEN RETURN; END_IF
IF Axis.AxisIsDisabled THEN
    M_AddAlarm(
        MainErrorId := 9001,
        SourceErrorId := 0,
        Message := 'Axis disabled',
        Path := 'Axis',
        Severity := E_Severity.Alarm);
END_IF
```

```iecst
METHOD PROTECTED H_UpdateVariable

IF Axis = 0 THEN RETURN; END_IF
M_AddVariable(
    Id := 8001,
    Name := 'Axis position',
    DataType := E_DescriptorType.Float_64bit,
    Unit := 'mm',
    Value := LREAL_TO_STRING(Axis.AxisActualPosition));
```

實際可供測試使用的衍生 FB 是 [FB_TestableHookModule](../Untitled1/Tests/Fakes/FB_TestableHookModule.TcPOU)。它另外提供容量、重複 ID 與錯誤呼叫階段的測試控制，未改動正式 Chamber1 的條件或 ID。

SVID 每輪重建，當輪未提供即消失。Alarm 當輪提交表示 Active；未提交表示 Inactive，仍保留到 `Active = FALSE AND Acknowledged = TRUE` 才移除。來源無法讀取時，使用者必須自行決定是否提交；不提交會被解讀為 Inactive。

Hook Alarm 固定使用 `ServiceId = 0`，ModuleId 與首次時間由框架填入。持續成立不增加 `OccurrenceCount`；Inactive 後再次成立會增加次數並取消確認。一次生命週期內維持首次時間、描述與 Severity，移除後再出現才建立新快照。Alarm 本身不新增 Stop／Abort 連動。

## JSON 整筆覆蓋

在有效 RuntimeConfig 中，SVID 依 `id`、Module Alarm 依 `mainErrorId` 決定所有權。JSON 接管同 ID 的全部 metadata、讀值來源及 Alarm 條件，不與 Hook 條件做 OR。Service Alarm 的非零 `ServiceId` 屬於不同 identity，不受同號 Module Alarm 覆蓋。

例如在支援且已註冊 `HwInput.rTemperature` 可讀節點的 Module 中，下列兩個設定項目會接管上述 ID：

```json
{
  "variableList": [
    {
      "source": "HwInput.rTemperature",
      "id": 8001,
      "name": "Temperature",
      "unit": "degC",
      "dataType": "REAL"
    }
  ],
  "alarmList": [
    {
      "source": "HwInput.rTemperature",
      "dataType": "REAL",
      "condition": { "operator": "gt", "value": 100.0, "tolerance": 0.01 },
      "mainErrorId": 9001,
      "sourceErrorId": 0,
      "message": "Temperature too high",
      "severity": "Warning"
    }
  ]
}
```

這是 Module 設定片段，需合併進完整設定，不是可獨立部署的 root JSON。當軸 disabled 而溫度未超標時，9001 不會因 Hook 而觸發；8001 會發布溫度及 JSON 型別／單位，取代軸位置。

JSON 讀值失敗仍保有所有權：SVID 沿用現有空字串值行為；Alarm 保留既有 lifecycle 狀態，不以無效樣本改變 Active，也不退回 Hook。有效設定中沒有同 ID 定義才使用 Hook。

ModuleId 改變會清除三種來源及確認狀態。Configuration revision 改變會重置 JSON Alarm，保留未被覆蓋的 Hook 與 Service lifecycle；JSON 接管的 Hook lifecycle 會在確認處理前移除。移除 JSON 定義後，Hook 下次提交可建立新 lifecycle。設定仍只在 PLC 啟動時套用，本次沒有新增即時 reload 或修改 schema v4。

## 容量、診斷與相容性

| 來源 | 容量 | 公開診斷 |
|---|---:|---|
| JSON Alarm | 30 | `ModuleStatus.Alarm.ConfiguredLatchedCount` |
| Hook Alarm | 30 | `ModuleStatus.Alarm.HookLatchedCount` |
| Service Alarm | 40 | `ModuleStatus.Alarm.ServiceLatchedCount` |
| 合併 SVID | 100 | `ModuleStatus.Variable.PublishedCount` |

Alarm 各組容量包含尚未確認的 Inactive lifecycle，互不借用；發布順序為 Service、JSON、Hook，總上限維持 100。容量滿時保留既有 lifecycle，拒絕新的 identity；釋出的空間可供後續提交使用。

SVID 先依 JSON 順序占用容量，再依 Hook 提交順序補入不同 ID。被 JSON 覆蓋的提交不占額外容量。發布陣列 index 是儲存位置，不是外部 identity。

`ModuleStatus.Alarm.Overflow` 彙整三種 Alarm 來源的溢位。`ModuleStatus.Variable.Overflow` 表示當輪合併 SVID 容量不足。兩者 `DuplicateId` 記錄當輪第一個 Hook 重複 ID；JSON 覆蓋屬正常操作，不產生 DuplicateId。Overflow 與 DuplicateId 每輪重設；LatchedCount 則反映持續保留的 Alarm。

`ST_ExtAlarm`、`ST_ExtDescriptor`、Alarm 確認請求格式與發布陣列長度不變。`ST_ModuleAlarmStatus` 增加欄位，`ST_ModuleStatus` 增加 `Variable`，會改變狀態結構大小及後續欄位 offset；使用固定記憶體布局或自行定義結構的 ADS／C# 用戶端需同步更新型別。Service Alarm 容量由 70 降至 40，應確認所需同時鎖存量。

## 驗證

既有 ModuleBase 與 AlarmManager TcUnit suites 已新增 Hook 時序、軸狀態讀取、確認生命週期、JSON 完整覆蓋、無效樣本、30／30／40 容量及 SVID 100 筆上限案例；既有 Heartbeat、Service、Chamber1 與 DVID 案例保留。

`validate-twincat-plc` 腳本只驗證 XML well-formedness，不代表 Structured Text 已編譯或 TcUnit 已執行。實作交付的靜態檢查與尚未執行的 PLC Runtime 行為驗證須分開記錄。
