# Module runtime configuration

PLC 啟動時會從 `Param_Config.ModuleConfigFilePath` 載入 UTF-8 JSON；預設目標路徑為：

`C:\ProgramData\RobustSolution\module-config.json`

專案中的 `module-config.json` 是可部署範例，不會在 PLC build 時自動複製到 target。

## 必要欄位

- Root：`schemaVersion`、`modules`；`adsPort` 預設為 851。
- Module：`enabled`、`moduleType`、`slot`。`moduleType` 必須與 `E_ModuleType` enum member 完全同名且大小寫一致；`slot` 是該 Module type 自己的 array index，同一 type 內不可重複，但不同 type 可使用相同 slot。啟用時另須非零且跨 type 全域唯一的 `id`。Module ADS symbol 會由 PLC 自動解析，不需在設定檔提供。
- Mapping：`source`、`target`、`sourceDataType`、`targetDataType`。只有數值型別可使用 `transform`。
- Reference binding：`source`、`targetPort`。`source` 必須是已註冊的 `I_BaseUnit`；`targetPort` 必須與 Module 專屬 reference-port enum member 完全同名且大小寫一致。GC 目前要求 `E_GCReferencePort` 公開的兩個 port 都存在。
- Variable：`source`、非零 `id`、`name`、`unit`、`dataType`。
- Alarm：`source`、`dataType`、非零 `mainErrorId`、`message`、`severity`，以及結構化 `condition`。

`condition.operator` 支援 `gt`、`lt`、`eq`、`ge`、`le`。`eq` 使用 `tolerance` 判定浮點相等；其他 operator 目前直接與 `value` 比較。

## 可連結節點

設定檔只能使用 PLC initial 階段已註冊到 `FB_LinkVariableManager`／`FB_ReferenceManager` 的節點。實體 I/O 與 BaseUnit reference 由 `MAIN` 註冊；兩個 manager 都會從實際變數位址自動取得 ADS symbol，呼叫端不需手寫名稱。Module I/O 則由對應 adapter 依 `enabled` 狀態註冊。使用者仍只需修改 JSON 來選擇已公開的節點。

目前 schemaVersion 為 `2`。舊版 `axisReferences`／`target` 不再接受，必須改為 `referenceBindings`／`targetPort`。Config 不填 interface type，也不接觸 `GVL_Module` 內部 reference 儲存位置；各 Module adapter 會依專屬 enum 契約解析 port，並以 `__QUERYINTERFACE` 驗證實際 BaseUnit 型別。

Module 設定只在 PLC 啟動時套用；變更 `enabled` 狀態後必須重新啟動 PLC。

I/O mapping 的無轉換路徑要求來源與目標的型別和大小完全一致，週期工作只執行 `MEMCPY`。有 `scale`／`offset` 時，manager 會先做明確的數值轉換。

## Module 容量

目前所有 Module type 共用相同的單一類型容量上限：

| Parameter | 目前值 | 作用範圍 |
|---|---:|---|
| `MaxModuleTypes` | 8 | 系統最多可註冊的 Module type 數量。 |
| `MaxModulePerType` | 6 | 每一種 Module type 可建立的實例數量，也是該 type 的 slot 範圍 `1..6`。各 type 擁有自己的 arrays，因此不同 type 可以使用相同 slot。 |
| `MaxTotalConfiguredModules` | 48 | 整份 config 的 `modules[]` 最大 entry 數量，包含 enabled 與 disabled entries；目前由 `MaxModulePerType * MaxModuleTypes` 計算。 |

Module type 自己的 FB、Runtime、Control 與 reference arrays，以及 adapter 的 slot 驗證和 MAIN cyclic loop，都必須使用 `MaxModulePerType`。Config parser、`ST_SystemFileConfig.Modules` 與全系統 link table 則使用 `MaxTotalConfiguredModules`。

## 資料與 Registry 容量

| Parameter | 目前值 | 作用範圍 |
|---|---:|---|
| `MaxModuleVariable` | 100 | 每個 Module 可發布的 SVID／VariableList 數量。 |
| `MaxModuleAlarm` | 100 | 每個 Module 可發布的 ALID／AlarmList 數量。 |
| `MaxModuleData` | 50 | 每個 Module 可發布的 DVID／ModuleDataList 數量。 |
| `MaxServiceError` | 10 | 每個 Service 每輪可提供的 Error entry 數量。 |
| `MaxModuleMappings` | 256 | 每個 Module 的 `inputMappings` 與 `outputMappings` 各自可配置的最大數量。 |
| `MaxConfigVariableNodes` | 512 | `FB_LinkVariableManager` 可註冊的實體 I/O 與 Module I/O 節點總數。 |
| `MaxModuleReferenceBindings` | 50 | 每個 Module 的 `referenceBindings` 最大數量。 |
| `MaxConfigReferenceNodes` | 100 | `FB_ReferenceManager` 可註冊的 BaseUnit reference source 總數。 |

`MaxConfigSources` 目前為 `MaxModuleVariable + MaxModuleAlarm`，因此每個 Module 的 ADS Sum Command 最多處理 200 個 Variable／Alarm sources。`MaxAdsSumResponseSize` 會依 source 數量與單筆最大資料大小在編譯期計算。
