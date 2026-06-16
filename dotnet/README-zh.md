# AutoGen for .NET 中文導覽

## 專案總覽
- `AutoGen.sln` 彙整了整個 .NET 解決方案，包含相容 AutoGen 0.2 的 `AutoGen.*` 套件與採取事件驅動模型的 `Microsoft.AutoGen.*` 新套件；新套件 API 仍在調整，舊套件將逐步被移植並淘汰。[README.md](README.md)
- 專案預設使用 .NET SDK 9.0.100，並採 `latestFeature` 向前滾動策略，確保本機與 CI 環境一致。[global.json](global.json)
- 全域建置屬性啟用 `netstandard2.0` 與 `net8.0` 雙目標、啟用可為 null 參考型別、強制警告視為錯誤並設定簽章金鑰，亦針對測試專案統一套件與測試資源連結。[Directory.Build.props](Directory.Build.props)
- 套件版本與共用相依性透過集中管理設定於 `Directory.Packages.props`，涵蓋 OpenAI、Azure AI Inference、Semantic Kernel、gRPC 等依賴版本。[Directory.Packages.props](Directory.Packages.props)
- `NuGet.config` 僅啟用官方 NuGet feed，建議如需夜間版可依安裝文件加掛其他來源。[NuGet.config](NuGet.config)｜[website/articles/Installation.md](website/articles/Installation.md)

## 程式庫分層
### AutoGen.\* 套件
- `AutoGen.Core`, `AutoGen.OpenAI`, `AutoGen.Mistral`, `AutoGen.Ollama`, `AutoGen.Anthropic`, `AutoGen.AzureAIInference`, `AutoGen.Gemini`, `AutoGen.SemanticKernel`, `AutoGen.LMStudio`, `AutoGen.SourceGenerator`, `AutoGen.DotnetInteractive`, `AutoGen.WebAPI` 等專案提供對話式代理、函式呼叫、各家模型整合與 .NET Interactive 執行環境，使用者可依需求選擇單一或多個封裝。[website/articles/Installation.md](website/articles/Installation.md)
- `AutoGen.SourceGenerator` 透過 `Function` 屬性自動產生函式定義與呼叫包裝器，適合建立型別安全的函式呼叫表述。[src/AutoGen.SourceGenerator/README.md](src/AutoGen.SourceGenerator/README.md)
- `AutoGen.LMStudio` 提供連接本機 LM Studio OpenAI 相容 API 的代理與設定範例。[src/AutoGen.LMStudio/README.md](src/AutoGen.LMStudio/README.md)
- `AutoGen.OpenAI` 中的 `OpenAIChatAgent` 示範如何包裝 `OpenAIClient`、支援串流回應與系統提示插入，作為其他代理的整合範本。[src/AutoGen.OpenAI/Agent/OpenAIChatAgent.cs](src/AutoGen.OpenAI/Agent/OpenAIChatAgent.cs)

### Microsoft.AutoGen.\* 套件
- `Microsoft.AutoGen.Core` 提供事件驅動的代理基底類別、Runtime 與訂閱機制；`BaseAgent` 透過 `IHandle<>` 介面反射註冊處理器，並可向 Runtime 發佈或傳送訊息。[src/Microsoft.AutoGen/Core/BaseAgent.cs](src/Microsoft.AutoGen/Core/BaseAgent.cs)
- `Microsoft.AutoGen.AgentHost`, `Microsoft.AutoGen.AgentChat`, `Microsoft.AutoGen.RuntimeGateway.Grpc`、`Microsoft.AutoGen.Core.Grpc` 等專案擴充事件分發、gRPC Gateway 與聊天協調，支援分散式部署與語意訂閱。[src/Microsoft.AutoGen](src/Microsoft.AutoGen/readme.md)
- `Microsoft.AutoGen.Extensions`、`Microsoft.AutoGen.Agents` 等子專案補充常用代理與擴充點，搭配 `Microsoft.Extensions` 系列套件整合日誌與依賴注入。[Directory.Packages.props](Directory.Packages.props)

## 主要目錄與資源
- `eng/`：收錄共用版本資訊、簽章設定與金鑰。`MetaInfo.props` 定義 `AutoGen.*` 與新套件各自的版本前綴與封裝中繼資料。[eng/MetaInfo.props](eng/MetaInfo.props)｜[eng/Sign.props](eng/Sign.props)
- `nuget/`：包含 NuGet 套件圖示、說明文件與 `nuget-package.props`，用於一致化封裝後設資料。[nuget/README.md](nuget/README.md)
- `resource/`：提供測試使用的共用資源，於測試專案建置時透過 `Directory.Build.props` 自動連結。[Directory.Build.props](Directory.Build.props)
- `samples/`：示範各種場景。
  - `samples/Hello` 展示以 .NET Aspire AppHost 啟動事件驅動代理、處理自訂 Topic 與 CloudEvents 串流。[samples/Hello/README.md](samples/Hello/README.md)｜[samples/Hello/HelloAgent/README.md](samples/Hello/HelloAgent/README.md)
  - `samples/GettingStarted` 與 `samples/GettingStartedGrpc` 提供最小化的本機與 gRPC 事件處理流程程式碼，可供理解訂閱與訊息發佈模型。[samples/GettingStarted/Program.cs](samples/GettingStarted/Program.cs)｜[samples/GettingStartedGrpc/Program.cs](samples/GettingStartedGrpc/Program.cs)
  - `samples/AgentChat` 底下以多個範例展示 `AutoGen.*` 封裝的單代理、群組對話、函式呼叫、LMStudio、Semantic Kernel 等整合。[samples/AgentChat/AutoGen.Basic.Sample](samples/AgentChat/AutoGen.Basic.Sample)
  - `samples/dev-team` 說明如何以事件驅動代理協調 GitHub 開發流程，包含產品經理、開發領隊與開發代理的互動鏈結。[samples/dev-team/README.md](samples/dev-team/README.md)
- `test/`：依套件區分的 xUnit 測試專案，涵蓋功能測試、整合測試與 AOT 相容性檢查，並共用 `AutoGen.Test.Share`、`Microsoft.AutoGen.Tests.Shared` 等測試工具。[test](test)
- `website/`：DocFX 網站原始碼與內容，包括安裝指南、代理教學、函式呼叫與群聊教學等文章，可透過 docfx 建置對外文件。[website/README.md](website/README.md)｜[website/articles/Installation.md](website/articles/Installation.md)
- `PACKAGING.md`：說明 `dotnet restore/build/pack` 流程與如何讓新專案納入 NuGet 封裝清單。[PACKAGING.md](PACKAGING.md)
- `dotnet-install.sh`：提供在 CI 或本機安裝 .NET SDK 的便利腳本。[dotnet-install.sh](dotnet-install.sh)

## 開發工作流程
- **還原與建置**：於 `dotnet/` 目錄執行 `dotnet restore` 與 `dotnet build --configuration Release --no-restore`，即可依預設多目標設定進行編譯。[PACKAGING.md](PACKAGING.md)
- **打包 NuGet**：完成建置後使用 `dotnet pack --configuration Release --no-build` 產生 `.nupkg` 與 `.snupkg`，輸出位置位於 `./artifacts/package/release`；若新增專案需在 `.csproj` 匯入 `nuget-package.props` 才會加入封裝流程。[PACKAGING.md](PACKAGING.md)
- **版本管理**：版本號由 `eng/MetaInfo.props` 控制，專案名稱以 `AutoGen.` 開頭者將自動套用 AutoGen 0.2 相容版號，其餘專案使用核心版號。[eng/MetaInfo.props](eng/MetaInfo.props)｜[Directory.Build.props](Directory.Build.props)
- **測試**：測試專案自動引用 xUnit、FluentAssertions 等套件並帶入 `resource/` 測試資料，可直接以 `dotnet test` 執行；測試組態在 `Directory.Build.props` 內標註 `IsTestProject` 條件設定。[Directory.Build.props](Directory.Build.props)
- **文件建置**：先執行 `dotnet tool restore` 安裝 DocFX，再透過 `dotnet tool run docfx website/docfx.json --serve` 啟動本地文件伺服器 (`http://localhost:8080`) 進行預覽。[website/README.md](website/README.md)

## 延伸資源
- 官方 AutoGen for .NET 文件提供安裝步驟、教學與範例連結，可搭配倉庫內容交叉參考。[AutoGen for .NET 官方網站](https://microsoft.github.io/autogen-for-net/)
