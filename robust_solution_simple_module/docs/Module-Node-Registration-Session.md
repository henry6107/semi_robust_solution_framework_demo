# Module Node Registration Session 說明

## 1. 文件目的

本文件說明 [`FB_GCModuleConfigurationAdapter.M_PrepareInstance`](../robust_solution_simple_module/Untitled1/Configuration/POUs/FB_GCModuleConfigurationAdapter.TcPOU) 中，為什麼 Module node declarations 必須放在以下兩個呼叫之間：

```iecst
bDeclared := LinkVariableManager.M_BeginModuleRegistration(
	Address := ADR(GVL_Module.GC[Slot]),
	Size := SIZEOF(GVL_Module.GC[Slot]),
	ModuleConfig := ModuleConfig);

// M_RegisterModuleNode declarations

IF NOT LinkVariableManager.M_EndModuleRegistration(
	ModuleSymbol => ModuleSymbol) THEN
	ErrorMessage := LinkVariableManager.ErrorMessage;
	RETURN;
END_IF
```

這兩個方法構成一次 Module node registration session 的頭尾：

```text
Begin：建立本次 Module instance 的註冊上下文
  ↓
Declare：Adapter 宣告可提供的 scalar／array nodes
  ↓
End：驗證宣告完整性、回傳 Module symbol、結束 session
```

## 2. `M_BeginModuleRegistration` 的用途

[`FB_LinkVariableManager.M_BeginModuleRegistration`](../robust_solution_simple_module/Untitled1/Configuration/POUs/FB_LinkVariableManager.TcPOU) 會建立本次註冊所需的共同上下文。

它會執行下列工作：

1. 清除上一個 registration session 的錯誤狀態。
2. 確認 Beckhoff hardware 等 infrastructure nodes 已完成註冊並 sealed。
3. 檢查 Module instance 的 `Address` 與 `Size` 是否有效。
4. 透過 `GetSymbolNameByAddress()` 自動解析 Module instance 的 ADS root symbol，例如：

   ```text
   GVL_Module.GC[1]
   ```

5. 保存本次 `ModuleConfig`，供 End 檢查 configuration 引用的 nodes 是否都有宣告。
6. 將 registration session 標示為 active。

建立這個上下文後，中間的 declaration 只需要提供變數與 access：

```iecst
bDeclared := LinkVariableManager.M_RegisterModuleNode(
	Variable := GVL_Module.GC[Slot].HwInput.bDoorClosed,
	Access := E_VariableAccess.ReadWrite);
```

`M_RegisterModuleNode` 會由變數的 address 與 size 直接取得完整 ADS symbol，例如：

```text
GVL_Module.GC[1].HwInput.bDoorClosed
GVL_Module.GC[1].HwInput.bDIs[3]
GVL_Module.GC[1].HwOutput.rAOs[4]
```

它不會先轉成 relative path，也不會使用 root symbol 重新組合 node name。Begin 保存的 root 只用來確認該完整 symbol 屬於目前的 Module instance。

## 3. 中間 declarations 的行為

GC adapter 可以宣告完整的 I/O surface，例如：

- scalar input／output；
- BOOL array；
- REAL array。

每個 declaration 都會 materialize 到 `_Nodes`。Array 則由 adapter 依實際 bounds 逐元素呼叫相同的 `M_RegisterModuleNode`，因此不需要帶 relative path 的 array-specific interface。

未被 configuration 使用的 nodes 仍會占用 node-table 容量，但不會建立 link，也不會增加 cyclic copy 或 configured-value read。現有每個 GC instance 會註冊 67 個 nodes；6 個 GC 與 34 個 Beckhoff hardware nodes 合計最多 436 個，低於目前 `MaxConfigVariableNodes = 512`。

## 4. `M_EndModuleRegistration` 的用途

[`FB_LinkVariableManager.M_EndModuleRegistration`](../robust_solution_simple_module/Untitled1/Configuration/POUs/FB_LinkVariableManager.TcPOU) 是整個 session 的最終驗證點。

它會執行下列工作：

1. 檢查 Begin 是否成功，且 registration session 仍然 active。
2. 檢查中間 declarations 是否曾發生錯誤。
3. 確認 configuration 引用的每個 input mapping、output mapping、Variable 與 Alarm node 都確實被 adapter 宣告。
4. 回傳本次 Module instance 的完整 root symbol。
5. 將 registration session 標示為 inactive。

回傳的 `ModuleSymbol` 會由 Configuration Manager 用來建立後續完整路徑，例如：

```text
<ModuleSymbol>.HwInput.bDoorClosed
```

這些完整路徑接著用於建立 links，以及解析 Variable／Alarm 的 node handles。

## 5. 為什麼需要 Begin／End

### 5.1 避免每次 declaration 重複傳遞共同參數

Begin 讓每個 `M_RegisterModuleNode` 只需傳入變數與 access，不必重複傳入 Module address、size、configuration 與 root symbol。manager 也能以 root prefix 驗證所有 declarations 屬於相同的 Module instance。

### 5.2 集中檢查 configuration 與 adapter 是否一致

如果只有 declarations 而沒有 End，manager 只能知道 adapter 宣告了哪些 nodes，無法確認 configuration 要求的 nodes 是否全部存在。

例如 JSON 將 node path 誤寫為：

```json
{
  "target": "HwInput.bDoorClosd"
}
```

但 adapter 實際只宣告：

```text
HwInput.bDoorClosed
```

`M_EndModuleRegistration` 會因為 configuration 引用的 `HwInput.bDoorClosd` 沒有對應 declaration 而失敗，避免錯誤延後到 cyclic runtime 才出現。

### 5.3 保持 LinkVariableManager 不認識 concrete Module type

Begin／Declare／End 提供的是通用 registration seam。`FB_LinkVariableManager` 只理解：

- Module root symbol；
- 由變數解析出的完整 node symbol；
- primitive data type；
- access direction；
- configuration relative path 的完整性驗證。

它不需要知道 GC、未來新增的 Module type，或該 Module 的 DUT 欄位結構。Module-specific 知識保留在各自的 Configuration Adapter 中。

## 6. `bDeclared` 為什麼會反覆被覆寫

目前 GC adapter 的 declaration 寫法如下：

```iecst
bDeclared := LinkVariableManager.M_RegisterModuleNode(...);
bDeclared := LinkVariableManager.M_RegisterModuleNode(...);
bDeclared := LinkVariableManager.M_RegisterModuleNode(...);
```

這段程式並不是依靠最後一次的 `bDeclared` 判斷整個 session 是否成功。

真正的錯誤狀態保存在 `FB_LinkVariableManager.Error` 與 `ErrorMessage` 中，並採用 first-error-wins：

1. 第一個失敗的 registration call 設定 `Error` 與 `ErrorMessage`。
2. 後續 registration calls 發現 `Error = TRUE` 後停止處理。
3. 後續呼叫不會覆蓋第一個錯誤原因。
4. `M_EndModuleRegistration()` 統一判斷整個 session 是否成功。

所以 `bDeclared` 只是接住各 helper 的 BOOL 回傳值；它不是累積結果，也不是最後的 commit flag。整個 registration session 的權威結果是 `M_EndModuleRegistration()` 的回傳值。

這種寫法可讓 adapter 保持線性的完整 I/O declaration 清單，同時把錯誤處理集中在 session 尾端。

## 7. 錯誤與清除行為

Begin／End 的概念類似 registration transaction，但它本身不是具有立即 rollback 能力的資料庫 transaction。

如果中間某個 node 已成功 materialize，後續 declaration 或 End 才失敗，已寫入的 Module node 可能暫時留在 table 中。完整 initialization 失敗時，外層 `FB_ModuleConfigurationManager` 會執行清除流程，移除：

- Module nodes；
- links；
- runtime-valid slots；
- reference binding context；
- adapter 內的 typed references。

Beckhoff hardware infrastructure nodes 則保留，供下一次 initialization attempt 使用。

## 8. 總結

- `M_BeginModuleRegistration`：指定「現在要宣告哪個 Module instance」，解析 root symbol 並保存 configuration context。
- `M_RegisterModuleNode`：從變數本身取得完整 symbol，驗證歸屬後將所有 declarations materialize。
- `M_EndModuleRegistration`：確認 configuration 要求的 nodes 全部已宣告，回傳完整 Module symbol，並結束 session。

這個 session interface 同時達成三個目的：減少 adapter 重複參數、集中完整性驗證，以及讓 `FB_LinkVariableManager` 維持與 concrete Module type 解耦。
