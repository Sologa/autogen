# AutoGen Core 套件說明

## 套件定位
- AutoGen Core 提供事件驅動、分散式的多代理基礎設施，支援 Actor 模型與 CloudEvents 式的 Topic 路由，適合作為所有上層產品的 Runtime 核心。[python/packages/autogen-core/README.md](python/packages/autogen-core/README.md)
- 設計文件釐清 Topic、Agent 與 Service 拓撲，說明如何跨語言部署及 Worker 協定，是深入理解架構時的首要參考。[docs/design/01%20-%20Programming%20Model.md](docs/design/01%20-%20Programming%20Model.md) [docs/design/05%20-%20Services.md](docs/design/05%20-%20Services.md)

## 核心概念與類別
- `BaseAgent` 定義代理生命週期、訊息傳遞 API、訂閱登記與狀態儲存，並提供 `handles` 裝飾器讓子類宣告可處理的訊息型別。[python/packages/autogen-core/src/autogen_core/_base_agent.py#L1](python/packages/autogen-core/src/autogen_core/_base_agent.py#L1)
- `AgentRuntime` 介面抽象出訊息傳遞、Topic 發佈以及工廠註冊，所有 Runtime 實作都需遵守此協定以確保互通性。[python/packages/autogen-core/src/autogen_core/_agent_runtime.py#L1](python/packages/autogen-core/src/autogen_core/_agent_runtime.py#L1)
- `ComponentModel` 與 `ComponentLoader` 支援宣告式載入模型、代理、工具等元件，並透過 provider 字串與版本欄位確保序列化一致性。[python/packages/autogen-core/src/autogen_core/_component_config.py#L18](python/packages/autogen-core/src/autogen_core/_component_config.py#L18)
- `tool_agent` 模組提供 `ToolAgent`，可接受 LLM `FunctionCall` 訊息並轉換為實際工具執行與結果回傳，常用於構建可重用的工具代理。[python/packages/autogen-core/src/autogen_core/tool_agent/_tool_agent.py#L1](python/packages/autogen-core/src/autogen_core/tool_agent/_tool_agent.py#L1)
- `models` 命名空間定義 `ChatCompletionClient` 抽象介面、模型族群常數與 Token 計量，協助上層產品統一與外部模型服務互動的流程。[python/packages/autogen-core/src/autogen_core/models/_model_client.py#L1](python/packages/autogen-core/src/autogen_core/models/_model_client.py#L1)

## 事件模型與部署
- Topic/Agent ID 規格、訂閱匹配與 Runtime 互動細節記載於設計文件，有助於規劃多租戶或跨語言的部署策略。[docs/design/02%20-%20Topics.md](docs/design/02%20-%20Topics.md) [docs/design/03%20-%20Agent%20Worker%20Protocol.md](docs/design/03%20-%20Agent%20Worker%20Protocol.md)
- 服務拓撲包含 Gateway、Registry、Routing、AgentState 等元件，並支援 In-Process 及 Orleans 等分散式選項，對大規模場景可直接套用。[docs/design/05%20-%20Services.md](docs/design/05%20-%20Services.md)
- 官方文件提供 Core User Guide，涵蓋事件流、記憶體管理與程式範例，可與程式碼互相對照快速上手。[https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html](https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/index.html)

## 開發建議
- 撰寫自訂代理時，優先繼承 `BaseAgent` 或 `RoutedAgent`，並透過 `handles` 宣告序列化器，避免在 Runtime 中發生訊息無法反序列化的問題。[python/packages/autogen-core/src/autogen_core/_base_agent.py#L60](python/packages/autogen-core/src/autogen_core/_base_agent.py#L60)
- 若需匯出代理設定給 AutoGen Studio 或 YAML 團隊配置，務必實作 `_to_config` 與 `dump_component`，以符合 Component 序列化規範。[python/packages/autogen-core/src/autogen_core/_component_config.py#L107](python/packages/autogen-core/src/autogen_core/_component_config.py#L107)
- 進行跨語言部署時，請對照設計文件中 Worker Protocol，確保註冊與健康檢查機制符合預期。[docs/design/03%20-%20Agent%20Worker%20Protocol.md](docs/design/03%20-%20Agent%20Worker%20Protocol.md)
