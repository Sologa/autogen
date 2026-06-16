# AutoGen Test Utils 套件說明

## 套件定位
- 套件提供針對 OpenTelemetry 的測試輔助工具，協助在單元測試或整合測試中檢驗 AutoGen 產生的遙測事件。[python/packages/autogen-test-utils/README.md](python/packages/autogen-test-utils/README.md)

## 核心元件
- `MyTestExporter` 實作 `SpanExporter`，將匯出的 spans 緩存在記憶體並提供清除/讀取方法，方便在測試中比對期望的事件順序與內容。[python/packages/autogen-test-utils/src/autogen_test_utils/telemetry_test_utils.py#L7](python/packages/autogen-test-utils/src/autogen_test_utils/telemetry_test_utils.py#L7)
- `get_test_tracer_provider` 建立帶有 `SimpleSpanProcessor` 的 `TracerProvider`，可快速在測試環境中註冊並攔截 AutoGen 代理所發出的遙測資料。[python/packages/autogen-test-utils/src/autogen_test_utils/telemetry_test_utils.py#L27](python/packages/autogen-test-utils/src/autogen_test_utils/telemetry_test_utils.py#L27)

## 使用建議
- 建議在測試開始時建立 exporter 並注入至 AutoGen 的追蹤設定，測試結束後呼叫 `clear` 清理狀態，確保測試互不干擾。
- 若需對照官方遙測事件格式，可參考 AutoGen Core 對 LLM 事件的定義，並在測試中檢查 `name`、`attributes` 等欄位是否符合預期。[python/packages/autogen-core/src/autogen_core/logging](python/packages/autogen-core/src/autogen_core/logging)
