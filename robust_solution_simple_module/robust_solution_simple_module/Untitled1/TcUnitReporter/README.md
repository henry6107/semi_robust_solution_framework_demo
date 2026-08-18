# TcUnit JSON Reporter

`FB_TcUnitJsonReporter` waits for TcUnit 1.3.2 results, serializes one completed run, and publishes a versioned JSON report without changing the test outcome.

## PLC usage

```iecst
VAR
    Reporter : FB_TcUnitJsonReporter;
END_VAR

TcUnit.RUN();
Reporter();
```

The default `ReportFilePath` is `test-report.json`, resolved relative to `PATH_BOOTPATH`. Configure a local absolute path once during initialization when required:

```iecst
Reporter.ReportFilePath := 'D:\TcUnitReports\gc-test-report.json';
```

Absolute drive paths use `PATH_GENERIC`. Empty paths, UNC paths, traversal, unsupported characters, non-JSON extensions, and paths longer than 250 characters are rejected. Directories are not created automatically.

Monitor `Busy`, `Done`, `Error`, `ErrorCode`, and `NativeErrorId` for publication status. A publication failure is latched and logged once through ADS, but it does not alter TcUnit pass/fail results.

## JSON contract

The root fields are `schemaVersion`, `generator`, `generatedAtUtc`, `durationSeconds`, and `suites`. Schema version `1.0.0` stores suite/test details only; aggregate counts are intentionally derived by the Viewer. Failed tests contain TcUnit's stored first failure type and message, while successful and skipped tests use `failure: null`.

## Offline Viewer

Open `Viewer/index.html` directly in Edge or Chrome, then drag `test-report.json` onto the page or use **開啟報告**. The Viewer has no CDN or server dependency. It validates schema major version 1, renders report strings with DOM text nodes, supports search and status filters, and prints the current filtered view for browser PDF export.

The files under `Tests/Fixtures` cover pass, mixed status, empty, unsupported-version, malformed, long-name, UTF-8, and hostile-HTML input scenarios.
