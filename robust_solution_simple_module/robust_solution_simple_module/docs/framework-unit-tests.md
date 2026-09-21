# Robust Solution Framework 單元測試

## 1. 範圍與判定原則

本測試專案涵蓋 `RobustSolutionFramework` 的 19 個正式 Function Block；Demo FB、一般工具 Function 與測試 Fake 不計入覆蓋數。每個正式 FB 都有同名的 `FB_<Name>Tests` Test Suite，並由 `TEST_PROG` 明確具現化後交由 `TcUnit.RUN()` 驅動，不以其他 suite 的間接呼叫取代。

測試分成兩層：

- **Deterministic**：只使用記憶體內 fake、harness 與具名變數；不依賴 NC 軸、外部檔案或目標機 ADS symbol 解析。
- **RuntimeDependent**：依賴 TwinCAT Runtime、ADS symbol、UTC、`Tc3_JsonXml` 檔案 I/O 或模擬 NC 軸。未啟用前置條件時，案例不呼叫 `TEST()`，因此不會產生誤導性的通過紀錄。

狀態定義必須分開解讀：

1. **XML 驗證完成**：TwinCAT object 與 project XML 可解析，GUID 與 compile entry 靜態檢查通過。
2. **TwinCAT 編譯完成**：在 XAE 對目前安裝的 `[RobustSolution]` library 執行 Build 且無 ST compiler error。
3. **TcUnit Runtime 通過**：下載至 Runtime 後，所有已註冊案例均完成且沒有 failure。

XML 驗證不等同於 ST 編譯，也不代表 NC、ADS、檔案 I/O 或 TcUnit Runtime 已通過。

## 2. 測試案例矩陣

表中的「前置／操作」是執行 Arrange 與 Act 的摘要；程式碼內另有繁體中文 `Arrange／Act／Assert` 及跨 scan 狀態說明。

| ID | Function Block / Suite | 場景與前置／操作 | 預期結果 | 層級 |
|---|---|---|---|---|
| BU-01～03 | `FB_BaseUnit` / `FB_BaseUnitTests` | 初始狀態、OwnerId 0、同 Owner 重入、競爭取得、錯誤與正確釋放；讀取 properties 與 ADS metadata | ownership 互斥；非法操作不改狀態；identity/metadata 穩定 | Deterministic + ADS metadata |
| AX-01～03 | `FB_Axis_BaseUnit` / `FB_Axis_BaseUnitTests` | 六種 motion method 未持有 ownership；未連軸 feedback；啟用模擬 NC 後執行命令與 cyclic update | 非 Owner 被拒且輸出中性；初值安全；NC 層不會同時 Done/Error | Deterministic + RuntimeDependent |
| AXF-01～13 | `FB_FakeAxis_BaseUnitTests`（支援 suite） | scripted Halt/Jog、延遲完成／錯誤、撤回、history/script overflow | fake 合約可重現且可供其他服務測試使用 | Deterministic |
| UTC-01 | `FB_UtcClock` / `FB_UtcClockTests` | 連續取樣 UTC 字串 | 固定 17 位、純數字、可排序且不倒退；目標機驗證時與系統 UTC 比對 | RuntimeDependent |
| PML-01～04 | `FB_PackMLBase` / `FB_PackMLBaseTests` | Undefined→Aborted→Stopped→Idle→Execute；Busy hook；非法命令 latch/re-arm；disabled、external fault；Hold/Suspend/Stop/Abort | 狀態轉移與 hook guard 正確；accepted/rejected feedback 保持至 request 回低；fault 優先 | Deterministic |
| PMM-01～03 | `FB_PackMLModeSet` / `FB_PackMLModeSetTests` | Manual/Production/Unknown；Manual 白名單；兩種 mode 的 state matrix | 支援 mode 被接受；Unknown 與不相容命令回傳固定 rejection | Deterministic |
| SB-01～03 | `FB_ServiceBase` / `FB_ServiceBaseTests` | ServiceId/Completion/Response；error 去重、容量與清除；data upsert/delete/overflow | response 只在有效 edge 推進；first occurrence 與既有資料在 overflow 時保留 | Deterministic |
| SBS-01～04 | `FB_ServiceExecuteSubStateTests`（`FB_ServiceBase` 支援 suite） | Execute step 開始/結束、重入、離開 Execute、下一次 Starting 與 30 筆 overflow | timeline 僅記錄實際 transition；carryover 不污染下一輪；overflow 保留前 30 筆 | Deterministic |
| CS-01～02 | `FB_CompositeServiceBase` / `FB_CompositeServiceBaseTests` | definition 一次性註冊、空/重複/溢位；未配置時 Start | 宣告窗口外不可變更；無依賴的 composite 走可診斷 Stop 路徑 | Deterministic |
| MAT-01～06 | `FB_SetMaterialService` / `FB_SetMaterialServiceTests` | Modify snapshot；Clear/ClearAll；Target=0、不存在、重複；未知 CmdType | 只修改指定 wafer；Starting 後輸入變更不影響 snapshot；錯誤進安全 Stop 並給固定 ID | Deterministic |
| CALL-01～02 | `FB_ServiceCaller` / `FB_ServiceCallerTests` | 未設定 caller/CallerId 0；設定後 Start、command、request id 與 snapshot query | 只有成功排入的 request 才提交遞增 ID；委派與查詢一致 | Deterministic |
| COORD-01～03 | `FB_ServiceCoordinator` / `FB_ServiceCoordinatorTests` | registration lifecycle、大小/型別/重複；upper Start snapshot；Execute 外 lifecycle 白名單；composite handshake | 註冊錯誤 first-error；edge re-arm；ownership 原子化且 request feedback 可追蹤 | Deterministic |
| ALM-01～08 | `FB_ModuleAlarmManager` / `FB_ModuleAlarmManagerTests` | Configured/Service/Hook identity；active/inactive/ack/remove/recur；單筆/全部確認；30/40/30 容量；revision/ModuleId reset | occurrence 只在 active edge 增加；identity 不混用；發布順序為 Service→Configured→Hook；容量區隔 | Deterministic |
| MOD-01～05 | `FB_ModuleBase` / `FB_ModuleBaseTests` | heartbeat；Service/BaseUnit/hook 順序；PackML aggregate；alarm/data/variable snapshot；invalid sample、Hook duplicate/容量 | timeout 與恢復正確；資料彙整穩定；JSON identity 覆蓋完整且錯誤階段固定 | Deterministic + ADS symbol |
| SNAP-01～03 | `FB_ConfigValueSnapshotReader` / `FB_ConfigValueSnapshotReaderTests` | disabled/invalid/no reader；Variable 後接 Alarm；BOOL/DINT/UDINT/REAL/LREAL/STRING；讀取失敗 | configured order 穩定；轉換正確；first-error-wins | Deterministic |
| LINK-01～03 | `FB_LinkVariableManager` / `FB_LinkVariableManagerTests` | infrastructure/module registration 與 rollback；type/access/scope/duplicate；MEMCPY/numeric transform；single writer；phase isolation；resolve/read error | 註冊具交易性；input/output 只在各自 phase 生效；不合法 link 不污染既有設定 | Deterministic + ADS symbol |
| REF-01～02 | `FB_ReferenceManager` / `FB_ReferenceManagerTests` | 空 interface、metadata/provider 錯誤；註冊/同名更新/容量；存在與不存在 resolve | 錯誤可診斷；symbol identity 與 resolve 結果一致 | RuntimeDependent (ADS) |
| BIND-01～03 | `FB_ModuleReferenceBindingContext` / `FB_ModuleReferenceBindingContextTests` | Begin/scope/End；required/optional；unknown、duplicate、未消費；nested/array 名稱與長度 | scope 隔離；optional missing 不報錯；required 與 first-error 規則固定 | Deterministic + ADS symbol |
| TYPE-01～02 | `FB_ModuleTypeRegistry` / `FB_ModuleTypeRegistryTests` | 空名稱/adapter；register/resolve/replace/capacity/unknown；ClearAll adapter failure | 同名更新 deterministic；清除失敗保留 first error | Deterministic |
| CFG-01～06 | `FB_ModuleConfigurationManager` / `FB_ModuleConfigurationManagerTests` | missing/malformed file；unsupported schema；舊 `adsPort`；duplicate enabled ModuleId；合法最小設定 | 非同步 state machine 最終 Ready 或 Error；錯誤時清理；成功只 apply 一次並提升 revision | RuntimeDependent (JSON/ADS) |
| ROOT-01～02 | `FB_ApplicationConfigRoot` / `FB_ApplicationConfigRootTests` | IO→BaseUnit→Module type 一次性註冊及 first-error；合法 fixture 下 input copy→Module→output copy；SystemContext | Ready 前不執行 Module；失敗 latch；成功 cycle 的資料流順序、UTC 與 heartbeat context 正確 | Deterministic + RuntimeDependent |

### 2.1 需在模擬 NC 上完成的 AX 延伸矩陣

`AX-03` 是正式 `FB_Axis_BaseUnit` 與 MC 層的接縫測試；在設備專案驗收時，需對 MoveAbs、MoveVel、Jog、Halt、Stop、Reset 各執行一次成功、一次錯誤注入及 Execute/方向撤回。每次記錄 ownership、Done/InVelocity、Error/ErrorId 與撤回後輸出。記憶體內相同的延遲、錯誤、latch、withdrawal 合約由 `FB_FakeAxis_BaseUnitTests` 固定驗證，但 fake 通過不能取代 NC 驗收。

## 3. Suite、fake 與 fixture 對照

| 類別 | 測試輔助物 | 用途 |
|---|---|---|
| PackML | `FB_TestPackMLBase`, `FB_TestPackMLModeSet` | 控制 hook 結果、計數及公開 protected policy seam |
| Service | `FB_TestableServiceBase`, `FB_TestableCarryoverSubStateService`, `FB_TestCompositeService` | 公開 error/data/substate/definition/sequence 測試 seam |
| Module | `FB_TestableModuleBase`, `FB_TestableHookModule`, `FB_FakeHookValueReader` | 驗證 heartbeat、彙整與 Hook contract |
| Configuration | `FB_FakeVariableNodeReader`, `FB_FakeModuleConfigurationAdapter`, `FB_FakeModuleReferenceBindings`, `FB_ReferencePortFixture`, `FB_TestApplicationConfigRoot` | 模擬 node read、adapter、reference port 與 root hook |
| BaseUnit | `FB_FakeBaseUnit`, `FB_FakeAxis_BaseUnit` | ownership dependency 與可 script 的 motion contract |
| ADS symbols | `GVL_TestSymbols`, `ST_TestModuleFixture`, `ST_ReferencePortTestBranch`, `ST_TestServiceParam` | 提供具名 symbol、Module storage、nested reference 及真實參數 DUT |

JSON fixture 位於 `FrameworkUnitTest/TestData/Configuration`：

- `valid-minimal.json`：合法最小 Module 設定。
- `application-root.json`：具一條 input 及一條 output mapping 的 root cycle 設定。
- `unsupported-schema.json`：不支援的 schemaVersion。
- `legacy-ads-port.json`：schema v4 禁止的舊欄位。
- `duplicate-enabled-id.json`：重複 enabled ModuleId。
- `malformed.json.invalid`：刻意損壞的 JSON；`.invalid` 表示它不應通過 JSON lint。

## 4. XAE 建置與執行

### 4.1 安裝目前工作區的 Framework library

1. 在 XAE 開啟 solution，先單獨建置 `RobustSolutionFramework`。
2. 將本工作區版本儲存／安裝成 `[RobustSolution]` library，確認 `FrameworkUnitTest` 的 `RobustSolution_BATW` placeholder 解析到剛安裝的版本。
3. 確認可解析 `TcUnit`、`Tc2_MC2`、`Tc2_Standard`、`Tc2_System`、`Tc2_Utilities`、`Tc3_Module` 與 `Tc3_JsonXml`。
4. 再 Build `FrameworkUnitTest`。若未先更新 library，測試可能編譯或執行在舊版 Framework 上。

### 4.2 Deterministic 執行

保持 library parameter：

```text
GVL_TestConfig.RunNcAxisRuntimeTests = FALSE
GVL_TestConfig.RunConfigurationFileRuntimeTests = FALSE
```

Activate configuration、Login、Run，於 TcUnit ADS logger/測試 runner 讀取結果。兩個開關為 FALSE 時，外部前置條件案例不會註冊；其餘 suite 仍全部執行。

### 4.3 JSON／ADS RuntimeDependent 執行

1. 建立 `C:\ProgramData\RobustSolution\Tests\`。
2. 將 `FrameworkUnitTest\TestData\Configuration\` 下所有 fixture 原樣複製到該目錄。
3. 將測試專案 parameter list 中的 `GVL_TestConfig.RunConfigurationFileRuntimeTests` 設為 `TRUE`。
4. 為執行 `ROOT-02`，將測試使用之 `[RobustSolution]` library parameter `Param_Config.ModuleConfigFilePath` 覆寫為 `C:\ProgramData\RobustSolution\Tests\application-root.json`。
5. 完整 Build、Activate、Login、Run；確認 `GVL_TestSymbols` 已輸出 ADS symbols。

若 fixture 不存在或 library parameter 未覆寫，runtime case 會以 prerequisite/timeout 訊息失敗，而不是顯示假成功。

### 4.4 模擬 NC RuntimeDependent 執行

1. 在 XAE 建立模擬 NC PTP 軸，完成 encoder/drive simulation 與 enable 條件。
2. 將 `FB_Axis_BaseUnitTests` 內正式 `_Axis` instance 的 `Axis` reference 連至該模擬軸。
3. 先在 `RunNcAxisRuntimeTests = FALSE` 下驗證 deterministic cases，再設為 `TRUE` 執行 NC case。
4. 依 2.1 的六命令矩陣注入成功、錯誤與撤回，保留 TcUnit 與 NC diagnostic 紀錄。

### 4.5 Repository XML 靜態驗證

在 PowerShell 執行：

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File "C:\Users\HenryH\.codex\skills\validate-twincat-plc\scripts\Test-TwinCatPlc.ps1" -WorkspacePath "<repository workspace>"
```

此命令只解析 `.TcPOU`、`.TcDUT`、`.TcGVL`、`.TcTTO`、`.plcproj` 與 `.tsproj` XML；不會開啟 XAE、不會 Build、Download、Activate Configuration 或執行 TcUnit。

## 5. 結果紀錄

每次驗收至少保存下列資訊：Framework library version/hash、TwinCAT build、TcUnit version、target AMS NetId、兩個 runtime 開關、fixture checksum、NC simulation 設定，以及 TcUnit tests/succeeded/failed/ignored 數量。若只做 repository 靜態檢查，結果欄必須寫「XML validated only」，不得填成「TcUnit passed」。
