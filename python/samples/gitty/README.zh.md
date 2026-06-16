# Gitty CLI 詳細說明

## 工具概述
- Gitty 透過 AutoGen AgentChat 協助維護者為 GitHub Issue/PR 產生精煉回覆，依賴 GitHub CLI 取得資料、OpenAI 模型生成內容，並支援本地 `.gitty` 設定與資料庫快取。[README.md:1](README.md#L1)｜[README.md:14](README.md#L14)

## 核心流程
- 入口 (`src/gitty/__main__.py`)：  
  - 解析指令（`issue`、`pr`、`fetch`、`local`、`global`）並檢查 `gh` CLI 與 `OPENAI_API_KEY` 是否可用。[src/gitty/__main__.py#L42-L71](src/gitty/__main__.py#L42)  
  - `issue` 指令會自動偵測目前倉庫、取得 Issue 編號後呼叫 `run_gitty` 實際執行；`fetch` 則更新本地資料庫與向量索引。[src/gitty/__main__.py#L73-L111](src/gitty/__main__.py#L73)
- 代理協作 (`src/gitty/_gitty.py`)：  
  - 建立 `AssistantAgent`，載入全域與在地設定指令，並附加工具 `get_github_issue_content` 與 `generate_issue_tdlr`。[src/gitty/_gitty.py#L85-L102](src/gitty/_gitty.py#L85)  
  - 對 Issue 內容進行多階段分析：抓取討論、萃取被提及或相似的 Issue、要求模型整理重點，再根據使用者回饋迭代回覆草稿。[src/gitty/_gitty.py#L104-L168](src/gitty/_gitty.py#L104)
- 設定與樣式 (`src/gitty/_config.py`)：提供 Rich 顏色主題，並確保 `.gitty` 目錄位於倉庫根目錄以儲存設定與資料。[src/gitty/_config.py#L8-L34](src/gitty/_config.py#L8)
- GitHub 資料處理 (`src/gitty/_github.py`)：透過 `gh` CLI 取得 Issue 內容、檢索提及的 Issue、建構 Chroma DB 搜尋相似議題，並提供 TLDR 生成工具。[src/gitty/_github.py#L12-L66](src/gitty/_github.py#L12)
- 本地快取 (`src/gitty/_db.py`)：以 SQLite 儲存 Issue 摘要與內容，並使用 Chroma 建立向量索引，方便快速搜尋相關議題。[src/gitty/_db.py#L71-L170](src/gitty/_db.py#L71)

## 使用建議
- 首次使用請執行 `gitty fetch` 更新資料庫，再透過 `gitty issue <number>` 產生回覆；若需要自訂指令，可編輯全域 `~/.gitty/config` 或 repo `.gitty/config` 並於 CLI 中選擇 `global` 或 `local` 指令開啟檔案。[src/gitty/__main__.py#L95-L120](src/gitty/__main__.py#L95)
- 產生回覆後可依終端提示提供 `y` 或文字回饋，引導代理再次修正草稿直到滿意為止。[src/gitty/_gitty.py#L147-L168](src/gitty/_gitty.py#L147)

## 延伸參考
- 若欲擴充更多工具或團隊協作流程，可參考 AutoGen AgentChat 官方指南中關於工具、隊伍與串流訊息的章節，並將其引入 `run_gitty` 的行為流程。[AutoGen AgentChat User Guide](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)
