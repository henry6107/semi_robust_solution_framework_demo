# Module runtime configuration

PLC 啟動時會從 `Param_Config.ModuleConfigFilePath` 載入 UTF-8 JSON；預設目標路徑為：

`C:\ProgramData\RobustSolution\module-config.json`

專案中的 `module-config.json` 是可部署範例，不會在 PLC build 時自動複製到 target。

## 必要欄位

- Root：`schemaVersion`、`modules`；`adsPort` 預設為 851。
- Module：`enabled`、`moduleType`、`slot`。啟用時另須 `id`、`symbol`，而且 enabled modules 的 `slot` 與 `id` 不可重複。
- Mapping：`source`、`target`、`sourceDataType`、`targetDataType`。只有數值型別可使用 `transform`。
- Axis reference：`source`、`target`。目前 GC 實作要求 `SpinAxis_BaseUnit` 與 `LiftPinAxis_BaseUnit` 都存在。
- Variable：`source`、非零 `id`、`name`、`unit`、`dataType`。
- Alarm：`source`、`dataType`、非零 `mainErrorId`、`message`、`severity`，以及結構化 `condition`。

`condition.operator` 支援 `gt`、`lt`、`eq`、`ge`、`le`。`eq` 使用 `tolerance` 判定浮點相等；其他 operator 目前直接與 `value` 比較。

## 可連結節點

設定檔只能使用 PLC initial 階段已註冊到 `FB_LinkVariableManager`／`FB_AxisReferenceManager` 的節點。新增硬體或 Module I/O 欄位時，必須在 `MAIN` 的註冊區加入該節點；使用者仍只需修改 JSON 來選擇已公開的節點。

I/O mapping 的無轉換路徑要求來源與目標的型別和大小完全一致，週期工作只執行 `MEMCPY`。有 `scale`／`offset` 時，manager 會先做明確的數值轉換。
