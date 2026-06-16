# AutoGen Extensions 套件說明

## 套件定位
- Extensions 收錄官方維護的額外代理、模型客戶端、程式執行器與工具，提供 AutoGen 使用者快速擴充能力的範例與預設實作。[python/packages/autogen-ext/README.md](python/packages/autogen-ext/README.md)
- 官方 Extensions Guide 提供安裝 extras、常見代理與工具使用方式，可搭配程式碼作為開發手冊。[https://microsoft.github.io/autogen/stable/user-guide/extensions-user-guide/index.html](https://microsoft.github.io/autogen/stable/user-guide/extensions-user-guide/index.html)

## 模型客戶端
- `autogen_ext.models.openai._openai_client` 封裝 OpenAI/Azure OpenAI 非同步客戶端，內建模型族群辨識、串流事件記錄與 User-Agent 設定，並支援函式呼叫、JSON Schema 及多模態內容。[python/packages/autogen-ext/src/autogen_ext/models/openai/_openai_client.py#L1](python/packages/autogen-ext/src/autogen_ext/models/openai/_openai_client.py#L1)
- 透過 `AzureOpenAIClientConfiguration` 等設定模型，可簡化金鑰、部署名稱與 API 版本管理，避免在應用程式層重複撰寫樣板程式碼。[python/packages/autogen-ext/src/autogen_ext/models/openai/config](python/packages/autogen-ext/src/autogen_ext/models/openai/config)

## 程式執行與工具
- `DockerCommandLineCodeExecutor` 提供可重複使用的容器執行環境，支援函式註冊、暫存檔管理、GPU 掛載與初始化指令，預設採用 `python:3-slim` 映像檔。[python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L1](python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L1)
- `create_default_code_executor` 會依環境可用性選擇 Docker 或本地執行器，是 Magentic-One 等高階團隊的預設依賴。[python/packages/autogen-ext/src/autogen_ext/code_executors/__init__.py#L1](python/packages/autogen-ext/src/autogen_ext/code_executors/__init__.py#L1)

## 代理與團隊
- `agents/web_surfer/MultimodalWebSurfer` 融合 Playwright 瀏覽器操作與多模態模型，支援 SOM 標註、工具呼叫與下載管理，適用於網頁探索任務。[python/packages/autogen-ext/src/autogen_ext/agents/web_surfer/_multimodal_web_surfer.py#L1](python/packages/autogen-ext/src/autogen_ext/agents/web_surfer/_multimodal_web_surfer.py#L1)
- `agents/file_surfer.FileSurfer` 與 `agents.magentic_one.MagenticOneCoderAgent` 分別負責檔案瀏覽與程式撰寫，為 Magentic-One 团隊的核心成員。[python/packages/autogen-ext/src/autogen_ext/agents/file_surfer](python/packages/autogen-ext/src/autogen_ext/agents/file_surfer)
- `teams/magentic_one.MagenticOne` 將 Orchestrator、Web/File Surfer、Coder 與 Code Executor 整合為 AgentChat 團隊，並提供人類監督、程式執行審批等選項。[python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py#L22](python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py#L22) [https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html)

## 開發建議
- 若需擴充新的模型客戶端，可依 `_openai_client` 模式實作 `ChatCompletionClient` 介面，確保與 AgentChat 的串流與工具呼叫協定一致。[python/packages/autogen-ext/src/autogen_ext/models/openai/_openai_client.py#L201](python/packages/autogen-ext/src/autogen_ext/models/openai/_openai_client.py#L201)
- 自訂 Code Executor 時建議實作 `CodeExecutor` 介面並支援 context manager，以符合 Magentic-One 與 Studio 的自動資源管理流程。[python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L200](python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L200)
