# FrameworkUnitTest

這個 TcUnit 專案為 `RobustSolutionFramework` 的 19 個正式 Function Block 各提供一個 Test Suite，另保留 Fake Axis 與 Service Execute substate 的支援 suite。

完整案例矩陣、fake/fixture 對照、library 安裝、模擬 NC 與結果判讀，請參閱 [framework-unit-tests.md](../docs/framework-unit-tests.md)。

快速選擇：

- 只跑 deterministic：保持 `GVL_TestConfig.RunNcAxisRuntimeTests = FALSE` 及 `RunConfigurationFileRuntimeTests = FALSE`。
- 加跑 JSON/ADS：部署 `TestData/Configuration` 至 `C:\ProgramData\RobustSolution\Tests\`，再將 `RunConfigurationFileRuntimeTests = TRUE`。
- 加跑模擬 NC：完成正式 Axis reference linking 後，將 `RunNcAxisRuntimeTests = TRUE`。

本專案透過 `[RobustSolution]` library placeholder 編譯。執行前必須先建置並安裝目前工作區的 Framework library；XML 靜態驗證不代表 XAE 編譯或 TcUnit Runtime 通過。
