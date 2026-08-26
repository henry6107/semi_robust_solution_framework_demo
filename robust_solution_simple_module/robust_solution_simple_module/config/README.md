# Module runtime configuration

PLC 啟動時會從 `Param_Config.ModuleConfigFilePath` 載入 UTF-8 JSON；預設目標路徑為：

`C:\ProgramData\RobustSolution\module-config.json`

專案中的 `module-config.json` 是可部署範例，不會在 PLC build 時自動複製到 target。

## 必要欄位

- Root：`schemaVersion`、`modules`；`adsPort` 預設為 851。
- Module：`enabled`、`moduleType`、`slot`。`moduleType` 必須與 `E_ModuleType` enum member 完全同名且大小寫一致；啟用時另須非零 `id`，而且 enabled modules 的 `slot` 與 `id` 不可重複。Module ADS symbol 會由 PLC 自動解析，不需在設定檔提供。
- Mapping：`source`、`target`、`sourceDataType`、`targetDataType`。只有數值型別可使用 `transform`。
- Reference binding：`source`、`targetPort`。`source` 必須是已註冊的 `I_BaseUnit`；`targetPort` 必須與 Module 專屬 reference-port enum member 完全同名且大小寫一致。GC 目前要求 `E_GCReferencePort` 公開的兩個 port 都存在。
- Variable：`source`、非零 `id`、`name`、`unit`、`dataType`。
- Alarm：`source`、`dataType`、非零 `mainErrorId`、`message`、`severity`，以及結構化 `condition`。

`condition.operator` 支援 `gt`、`lt`、`eq`、`ge`、`le`。`eq` 使用 `tolerance` 判定浮點相等；其他 operator 目前直接與 `value` 比較。

## 可連結節點

設定檔只能使用 PLC initial 階段已註冊到 `FB_LinkVariableManager`／`FB_ReferenceManager` 的節點。實體 I/O 與 BaseUnit reference 由 `MAIN` 註冊；Module I/O 則由對應 adapter 依 `enabled` 狀態註冊。使用者仍只需修改 JSON 來選擇已公開的節點。

目前 schemaVersion 為 `2`。舊版 `axisReferences`／`target` 不再接受，必須改為 `referenceBindings`／`targetPort`。Config 不填 interface type，也不接觸 `GVL_Module` 內部 reference 儲存位置；各 Module adapter 會依專屬 enum 契約解析 port，並以 `__QUERYINTERFACE` 驗證實際 BaseUnit 型別。

Module 設定只在 PLC 啟動時套用；變更 `enabled` 狀態後必須重新啟動 PLC。

I/O mapping 的無轉換路徑要求來源與目標的型別和大小完全一致，週期工作只執行 `MEMCPY`。有 `scale`／`offset` 時，manager 會先做明確的數值轉換。
