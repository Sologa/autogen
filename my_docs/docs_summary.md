# docs 目錄詳細整理

## 檔案與目錄結構
- `docs/design/`：AutoGen 核心概念、事件模型、代理辨識與服務拓撲等設計文件。
- `docs/dotnet/`：DocFX 所需的 .NET 文件來源（Markdown、樣式、組態）與建站腳本。
- `docs/switcher.json`：對應官方網站版本切換器的名稱、版本與路徑設定。

## docs/design/

### `01 - Programming Model.md`
- AutoGen 以發佈/訂閱（publish-subscribe）模型驅動代理協作，事件採用 CloudEvents 規格標準化。
- CloudEvents 事件必備 `id`、`source`、`type` 三個 context attributes，`type` 建議使用反向 DNS 命名。
- 代理（Agent）透過事件處理器（event handlers）綁定特定 `type` 或類型前綴，處理器可更新狀態、調用模型、訪問記憶、觸發外部工具並發佈新事件。
- 可建立 orchestrator agent 以事件為基礎編排長流程；也可結合記憶、提示詞（prompts）、資料來源與 skills。
- 系統內建事件類型包含系統事件（啟停代理、廣播等），後續可擴充其他類別。
- AutoGen 提供具行為契約的基礎代理（例如 Python ChatAgents），用於在事件模型之上實現請求/回應模式。

### `02 - Topics.md`
- Topic 由 `TopicId = (type, source)` 定義，`type` 為靜態事件類型（符合 `^[\w\-\.\:\=]+\Z`），`source` 為動態來源 URI；對應 CloudEvents。
- Agent 透過 `AgentId = (type, key)` 辨識，`type` 需符合 `^[\w\-\.]+\Z`，`key` 建議使用 URI，以區分相同類型的實例。
- Subscription 以函式組成：matcher `TopicId -> bool` 決定是否收到訊息；mapper `TopicId -> AgentId` 判定要將訊息交付哪個代理。兩者需無副作用以利快取。
- 當 Topic 對應的 Agent 尚未啟動時，Runtime 會自動依據 Subscription 建立代理實例。
- 目前所有同 Channel 代理皆可收到訊息，各自判斷是否能處理（未來可能依效能調整）。
- 建議代理以 `{AgentType}:` 為前綴訂閱直接訊息通道，可自動獲得以下慣例 Topic：
  - `{AgentType}:`：一般直接訊息。
  - `{AgentType}:rpc_request={RequesterAgentType}`：RPC 請求，回覆時需指定請求者型別。
  - `{AgentType}:rpc_response={RequestId}`：RPC 回應。
  - `{AgentType}:error={RequestId}`：與請求對應的錯誤事件。

### `03 - Agent Worker Protocol.md`
- 系統由服務（service）與工作者（worker）程序構成；worker 連線至 service 並宣告可承載的 agent 名稱集合。
- Agent 實例以 `(namespace: str, name: str)` 標示，`namespace` 無系統語意，由應用自行詮釋；`name` 用於路由與 placement。
- Service 維護資料：
  - 可托管每個 agent name 的 worker 列表。
  - 活動中代理對應的 worker 目錄。
- Worker 生命周期：
  - 初始化：啟動後與 service 建立雙向通訊，並傳送 `RegisterAgentType(name: str)` 告知可承載代理。
  - 運作：接收 `Event(...)` 或 `RpcRequest(...)` 觸發代理實例化，於本地 catalog 建立 Agent；`Event` 無回覆，`RpcRequest` 需回傳 `RpcResponse`。
  - Worker 追蹤尚未完成的 RPC 請求（以 `RpcRequest.id` 索引），逾時需回報錯誤。
  - 終止：關閉連線時，service 解除註冊該 worker 與其代理。
- 尚有待決議的擴充：Worker metadata、唯一識別碼等。

### `04 - Agent and Topic ID Specs.md`
- Agent ID：
  - `type`：字串、反映工廠函式而非類別，僅能包含英數與底線、不可數字開頭。
  - `key`：字串、實例識別，允許 ASCII 32 到 126 之字元，用於區分相同類型的多個代理（例如 `default`、UUID）。
- Topic ID：
  - `type`：字串、表示訊息類型，允許英數、`:`、`=`、底線，禁止數字開頭與空白（如 `GitHub_Issues`）。
  - `source`：字串、Topic 的實體來源，允許 ASCII 32 至 126，常為 URI（如 `github.com/{repo}/issues/{issue}`）。

### `05 - Services.md`
- AutoGen 可部署為同進程、跨語言或分散式系統，所有事件皆以 CloudEvents 表示，可透過 gRPC 傳輸。
- 服務拓撲：
  - Worker：承載代理程式，作為 Gateway 客戶端。
  - Gateway：提供 RPC API、連接事件匯流排並管理訊息 session。
  - Registry：維護代理/Topic 訂閱清單，Roadmap 計畫提供查詢 API。
  - AgentState：代理持久化狀態。
  - Routing：依訂閱傳遞事件；Roadmap 亦包含訂閱管理 API。
  - 未來計畫：管理 API、排程（placement）、Discovery。
- 部署選項：
  - In-Memory：Python 與 .NET 均可於單進程內運行。
  - Python Service：Python Worker 對應內建訊息匯流排與註冊中心。
  - Microsoft Orleans：支援分散式部署、持久化儲存、多語言代理協作。
  - Roadmap：支援 dapr、Akka 等其他分散式系統。

### `readme.md`
- 直接指向官方線上文件入口：https://microsoft.github.io/autogen/dev/ 。

## docs/dotnet/

### `README.md`
- 建站需求：.NET 8.0+。
- 建置流程：於 `autogen/dotnet` 執行 DocFX 指令。

```bash
dotnet tool restore
dotnet tool run docfx ../docs/dotnet/docfx.json --serve
```

- 建置完成後瀏覽 `http://localhost:8080` 取得預覽站點。

### `docfx.json`
- `metadata`：掃描 `src/Microsoft.AutoGen/*` 多個子專案的 `.csproj`，生成 API `api` 目錄，預設不含私有成員、允許 Git 功能。
- `build`：
  - `content` 匯入 `api/**`, `core/**`, `toc.yml`, 站點根 Markdown。
  - `resource` 夾帶 `images/**`。
  - 輸出路徑 `_site`，套用 `default`, `modern`, `template` 三種樣板。
  - `globalMetadata` 設定站點標題、Logo、Favicon 與 Git 來源 (`https://github.com/microsoft/autogen.git`)。
  - 使用 `ExtractSearchIndex` 後處理器並開啟搜尋索引。

### `index.md`
- 首頁以卡片呈現 AutoGen .NET（Core）與 AgentChat（預告中）。
- Core 卡片提供 GitHub CI/ NuGet 套件徽章、適用案例（流程自動化、多代理協作、分散式應用、事件導向整合）。
- 建議先安裝 `Microsoft.AutoGen.Contracts` 與 `Microsoft.AutoGen.Core`，並列出其他可選套件（AgentHost、RuntimeGateway、Agents、Extensions）。
- 提供快速開始指令與「Get started」連結至 `core/index.md`。

### `toc.yml`
- 建立三個主選項：Core、API Reference、Python（外部連結）。

### `images/`
- `logo.svg` 與 `favicon.ico` 為網站視覺資產。

### `template/public/main.css`
- 調整 DocFX 頁面導覽、標題、卡片與淺/深色主題色票，並新增 `center`, `subheader`, `hero-title`, `wip-card` 等樣式以符合 AutoGen 風格。

### `template/public/main.js`
- 客製 DocFX icon links，於導覽列加入 GitHub 連結。

### `core/` 文件

#### `toc.yml`
- 章節順序：Overview、Installation、Tutorial、Differences from Python、Protobuf message types。

#### `index.md`
- 說明 .NET Core 與 Python 版本概念一致，建議先閱讀 Python 官方文件。
- 最少安裝 `Microsoft.AutoGen.Contracts`、`Microsoft.AutoGen.Core`，並指向 `python/samples` 範例。
- 提供建立 Agent 的 C# 範例：繼承 `BaseAgent`、實作 `IHandle<T>`。
- 演示 `AgentsAppBuilder` 組態與 `PublishMessageAsync` 啟動流程。
- 描述兩種 Runtime：
  - InProcess：同進程佇列。
  - Distributed（Microsoft Orleans + gRPC）：需加裝 `Microsoft.AutoGen.Core.Grpc`、後端使用 `Microsoft.AutoGen.RuntimeGateway`, `Microsoft.AutoGen.AgentHost`。
- 提供：
  - 後端執行指令 `dotnet run --project Microsoft.AutoGen.AgentHost`。
  - 將 Runtime 打包為 `dotnet tool` 的命令範例。
- 介紹 .NET Aspire 範例（`dotnet/samples/Hello/Hello.AppHost/Program.cs`）與 Python 代理協同示例，列出建置程式碼片段。
- 建議的 `appsettings.json` Logging 組態 JSON。
- 強調跨語言通訊需使用 CloudEvents/ gRPC，並引導至 `protobuf-message-types.md`。

#### `installation.md`
- 提供三種安裝方式：`.NET CLI`、`Package Manager Console`、`PackageReference`。
- 核心套件版本 `0.4.0-dev.1`，額外列出 AgentChat、Agents、Extensions 之 `0.4.0-dev-1`。
- 分散式與跨語言環境需加裝 `Microsoft.AutoGen.Core.Grpc`, `Microsoft.AutoGen.RuntimeGateway.Grpc`, `Microsoft.AutoGen.AgentHost`。

#### `tutorial.md`
- 以 `Modifier` 與 `Checker` 兩代理為例實作倒數工作流，對應 `dotnet/samples/GettingStarted` 程式碼段。
- 步驟：
  - 定義 `CountMessage` 與 `CountUpdate` 類別。
  - 建立繼承 `BaseAgent` 的代理並實作 `IHandle<T>` 處理函式。
  - 透過 `TypeSubscription("default")` 訂閱 Topic。
  - 在 `HandleAsync` 內透過 `PublishMessageAsync` 發佈更新。
  - 使用委派 `Func<int, int>` 將倒數邏輯注入建構式，以展示 DI/配置能力。
  - `Checker` 代理示範注入 `IHostApplicationLifetime` 以停止應用程式。
  - 構建 `AgentsAppBuilder`，註冊 Runtime、服務與代理，最後發送啟動訊息並等待結束。
- 結尾給出延伸練習建議（修改初始值、改為遞增、拆出輸出代理）。

#### `differences-from-python.md`
- 說明預設情況下 Agent 不會收到自己發佈的訊息；與 Python 一致。
- 在 `Microsoft.AutoGen.Core.InProcessRuntime` 中，可透過 `TopicSubscription` 的 `DeliverToSelf = true` 恢復自我接收行為。

#### `protobuf-message-types.md`
- 指出除 InProcess Runtime 外，其餘 Runtime 需使用 Protocol Buffers 定義訊息。
- 步驟：
  - 在 `.csproj` 引入 `Grpc.Tools` 套件。
  - 建立 `<Protobuf Include="messages.proto" GrpcServices="Client;Server" Link="messages.proto" />`。
  - 於 `.proto` 定義訊息（示例 `TextMessage`）。
  - 在 Agent 中引用生成的類別並實作 `IHandle<TextMessage>`。
- 強調後續可能加入轉換器或自訂序列化以放寬限制。

### 其他檔案
- `core/protobuf-message-types.md` 所引用的 `MessageContext`、`AgentId` 等型別來自 `Microsoft.AutoGen.Contracts` 與 `Microsoft.AutoGen.Core` 套件。
- `docs/dotnet/.gitignore` 排除 `_site/`, `api/`, `bin/`, `obj/` 等 DocFX 產物及建置資料夾。

## docs/switcher.json
- 列出官方文件站版本切換資訊：
  - `dev (main)` 指向 `/autogen/dev/`。
  - `0.7.5 (stable)` 為預設，對應 `/autogen/stable/`。
  - 包含 `0.7.4` 至 `0.2` 等歷史版本，每筆紀錄具 `name`、`version`、`url`。
- 可供 DocFX 網站生成版本下拉選單，協助使用者快速切換文件版本。

## 與官方資源關聯
- `docs/design/*` 與 `docs/dotnet/core/*` 內容對應至官方網站 https://microsoft.github.io/autogen/dev/ ，維護者可透過 `docfx.json` 重新建置以同步線上內容。
- `docs/switcher.json` 需與線上站台目錄結構一致，以確保版本導覽正常運作。
