# Module runtime configuration

PLC 啟動時會從 `Param_Config.ModuleConfigFilePath` 載入 UTF-8 JSON；預設目標路徑為：

`C:\ProgramData\RobustSolution\module-config.json`

專案中的 `module-config.json` 是可部署範例，不會在 PLC build 時自動複製到 target。

## 必要欄位

- Root：`schemaVersion`、`modules`。schema v4 不再支援 `adsPort`。
- Module：`enabled`、`moduleType`、`slot`。`moduleType` 必須與註冊至 `FB_ModuleTypeRegistry` 的名稱完全同名且大小寫一致；`slot` 是該 Module type 自己的 array index，同一 type 內不可重複，但不同 type 可使用相同 slot。啟用時另須非零且跨 type 全域唯一的 `id`。Module ADS symbol 會由 PLC 自動解析，不需在設定檔提供。
- Mapping：`source`、`target`、`sourceDataType`、`targetDataType`。只有數值型別可使用 `transform`。
- Reference：`references` 是 port name 到 BaseUnit source name 的 JSON object。key 必須與 adapter 使用的語意名稱完全同名且大小寫一致；value 必須是已註冊至 `FB_ReferenceManager` 的 `I_BaseUnit` source。GC 目前要求 `SpinAxis_BaseUnit` 與 `LiftPinAxis_BaseUnit`。
- Variable：`source`、非零 `id`、`name`、`unit`、`dataType`。
- Alarm：`source`、`dataType`、非零 `mainErrorId`、`message`、`severity`，以及結構化 `condition`。

`condition.operator` 支援 `gt`、`lt`、`eq`、`ge`、`le`。`eq` 使用 `tolerance` 判定浮點相等；其他 operator 目前直接與 `value` 比較。

## 可連結節點

設定檔只能使用 PLC initial 階段已註冊到 `FB_LinkVariableManager`／`FB_ReferenceManager` 的節點。Beckhoff 實體 I/O 與 BaseUnit reference 由 `MAIN` 註冊；兩個 manager 都會從實際變數位址自動取得 ADS symbol，呼叫端不需手寫名稱。Module I/O 則由對應 adapter 透過通用 registration interface 宣告；`FB_LinkVariableManager` 不包含 GC 或其他應用 Module Type 的 concrete registration method。使用者仍只需修改 JSON 來選擇已公開的節點。

Module adapter 以 `M_RegisterModuleNode(Variable, Access)` 宣告完整 I/O surface；manager 會從變數位址與大小取得完整 ADS symbol，所有 declarations 都會寫入 registry。未被 configuration 使用的 nodes 仍占用 node-table 容量，但不會建立 link，也不增加 cyclic copy 或 configured-value read。初始化每個 Module instance 時，adapter 依語意名稱呼叫 `M_TakeRequired`／`M_TakeOptional` 取得 reference；framework 統一處理 JSON lookup、source resolve、重複取用與未取用的 unknown key，adapter 只負責以 `__QUERYINTERFACE` 驗證並轉成 Module 所需的 typed interface。

目前 schemaVersion 為 `4`。schema v3 與更早版本不再接受；舊版 `axisReferences`／`referenceBindings` 必須改為 `references` object。Config 不填 interface type，也不接觸 `GVL_Module` 內部 reference 儲存位置；required／optional 是各 Module adapter 的固定契約，JSON key 順序不影響綁定結果。

Module 設定只在 PLC 啟動時套用；變更 `enabled` 狀態後必須重新啟動 PLC。

I/O mapping 的無轉換路徑要求來源與目標的型別和大小完全一致，週期工作只執行 `MEMCPY`。有 `scale`／`offset` 時，manager 會先做明確的數值轉換。

VariableList 與 AlarmList 的 `source` 也必須是該 enabled Module 註冊的可讀節點。Configuration Manager 只在初始化時以完整 Symbol Name 解析節點，並將不透明的 `NodeHandle` 寫入 Runtime Binding；Module 週期執行透過 `I_VariableNodeReader` 直接建立本機資料 snapshot，不再建立 ADS handle 或執行 ADS Sum Command。Config 的 `dataType` 是預期型別，必須與註冊節點的真實 metadata 相同。

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
| `MaxConfigVariableNodes` | 512 | eager registration 的 Beckhoff 實體 I/O，加上 enabled Module adapters 宣告的全部 Module nodes。現有 6 個 GC instances 與 Beckhoff hardware 最多使用 436 個。 |
| `MaxModuleReferences` | 50 | 每個 Module 的 `references` member 最大數量。 |
| `MaxConfigReferenceNodes` | 100 | `FB_ReferenceManager` 可註冊的 BaseUnit reference source 總數。 |

`MaxConfiguredValueSources` 為 `MaxModuleVariable + MaxConfiguredAlarmsPerModule`，限制每個 Module 一次本機 snapshot 可包含的 Variable／Alarm sources 總數。`MaxConfigValueSize` 則限制單一節點的原始資料大小，目前可容納 `STRING(80)` 與結尾字元。
