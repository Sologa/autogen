# Component Schema Generator 套件說明

## 套件定位
- `component-schema-gen` 提供命令列工具 `gen-component-schema`，可輸出 AutoGen 組件（模型、Token Provider 等）的 JSON Schema，協助驗證 YAML/JSON 組態與產生文件。[python/packages/component-schema-gen/README.md](python/packages/component-schema-gen/README.md)

## 運作方式
- `__main__.py` 讀取 AutoGen Core 的 `ComponentModel` 定義與 `WELL_KNOWN_PROVIDERS` 對照表，為每個組件生成帶 `$defs` 的 Schema 並展開各 Provider 的別名。[python/packages/component-schema-gen/src/component_schema_gen/__main__.py#L1](python/packages/component-schema-gen/src/component_schema_gen/__main__.py#L1)
- `build_specific_component_schema` 將元件的 `component_config_schema` 合併到 `ComponentModel`，指定 `config` 引用與 `provider` 常數，確保輸出 Schema 能與宣告式配置相容。[python/packages/component-schema-gen/src/component_schema_gen/__main__.py#L21](python/packages/component-schema-gen/src/component_schema_gen/__main__.py#L21)
- 預設支援 OpenAI/Azure OpenAI 模型與 AzureTokenProvider；若需擴充其他 Component，可依樣式在 `main()` 中呼叫 `add_type`。[python/packages/component-schema-gen/src/component_schema_gen/__main__.py#L97](python/packages/component-schema-gen/src/component_schema_gen/__main__.py#L97)

## 使用建議
- 生成的 JSON Schema 可直接掛入 CI 驗證 YAML/JSON Team 配置，也可提供 AutoGen Studio 或自訂工具解析。
- 若專案新增自訂組件，建議同步更新此工具，避免外部用戶載入新 Provider 時缺乏 Schema 參考。
