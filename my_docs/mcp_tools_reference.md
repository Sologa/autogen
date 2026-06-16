# MCP 工具完整參考手冊

> 本文件記錄所有已配置的 MCP (Model Context Protocol) 工具及其使用方式
> 
> 最後更新：2025年10月31日

---

## 📋 目錄

1. [Time MCP](#1-time-mcp) - 時區轉換
2. [Sequential Thinking MCP](#2-sequential-thinking-mcp) - 逐步推理
3. [Fetch MCP](#3-fetch-mcp) - 網頁抓取
4. [GitKraken MCP](#4-gitkraken-mcp) - Git 與 GitHub/GitLab 操作
5. [Filesystem MCP](#5-filesystem-mcp) - 檔案系統操作
6. [Memory MCP](#6-memory-mcp) - 知識圖譜管理
7. [Obsidian MCP](#7-obsidian-mcp) - Obsidian 筆記管理
8. [Everything MCP](#8-everything-mcp) - MCP 協定測試工具

---

## 1. Time MCP

**伺服器名稱**：`mcp-server-time`  
**狀態**：✅ 永久啟用  
**用途**：處理時區轉換與時間查詢

### 可用工具

#### `mcp_time_get_current_time`
取得指定時區的當前時間。

**參數**：
- `timezone` (string, required): IANA 時區名稱（例如：'America/New_York', 'Asia/Taipei'）

**範例**：
```
取得台北時間：
mcp_time_get_current_time(timezone="Asia/Taipei")

取得紐約時間：
mcp_time_get_current_time(timezone="America/New_York")
```

#### `mcp_time_convert_time`
在不同時區之間轉換時間。

**參數**：
- `source_timezone` (string, required): 來源時區
- `time` (string, required): 時間（24小時制，格式：HH:MM）
- `target_timezone` (string, required): 目標時區

**範例**：
```
將台北時間 14:30 轉換為紐約時間：
mcp_time_convert_time(
  source_timezone="Asia/Taipei",
  time="14:30",
  target_timezone="America/New_York"
)
```

---

## 2. Sequential Thinking MCP

**伺服器名稱**：`@modelcontextprotocol/server-sequential-thinking`  
**狀態**：✅ 永久啟用  
**用途**：提供逐步推理能力，用於複雜問題分析

### 可用工具

#### `mcp_sequential-th_sequentialthinking`
進行結構化的逐步思考流程。

**參數**：
- `thought` (string, required): 當前思考步驟
- `nextThoughtNeeded` (boolean, required): 是否需要更多思考
- `thoughtNumber` (integer, required): 當前思考編號
- `totalThoughts` (integer, required): 預估總思考步驟數
- `isRevision` (boolean, optional): 是否為修正前述思考
- `revisesThought` (integer, optional): 正在修正的思考編號
- `branchFromThought` (integer, optional): 分支起點
- `branchId` (string, optional): 分支識別碼
- `needsMoreThoughts` (boolean, optional): 是否需要增加總步驟數

**適用場景**：
- 複雜問題的分解
- 多步驟規劃
- 需要回溯修正的推理過程

---

## 3. Fetch MCP

**伺服器名稱**：`mcp-server-fetch`  
**狀態**：✅ 永久啟用  
**用途**：抓取網頁內容並轉換為 Markdown

### 可用工具

#### `mcp_fetch_fetch`
抓取指定 URL 的內容。

**參數**：
- `url` (string, required): 要抓取的網址（必須是 http 或 https）
- `max_length` (integer, optional): 最大字元數（預設：5000，最大：1000000）
- `start_index` (integer, optional): 起始字元位置（預設：0）
- `raw` (boolean, optional): 是否返回原始 HTML（預設：false）

**範例**：
```
抓取網頁並轉換為 Markdown：
mcp_fetch_fetch(url="https://example.com")

抓取原始 HTML：
mcp_fetch_fetch(url="https://example.com", raw=true)

從第 1000 個字元開始抓取 2000 字元：
mcp_fetch_fetch(
  url="https://example.com",
  start_index=1000,
  max_length=2000
)
```

---

## 4. GitKraken MCP

**伺服器名稱**：`gitkraken`  
**狀態**：✅ 永久啟用  
**用途**：Git 操作與 GitHub/GitLab/Azure DevOps 整合

### 可用工具

#### Git 操作

##### `mcp_gitkraken_git_status`
顯示工作目錄狀態。

**參數**：
- `directory` (string, required): Git 倉庫路徑

##### `mcp_gitkraken_git_add_or_commit`
添加檔案到暫存區或提交變更。

**參數**：
- `directory` (string, required): Git 倉庫路徑
- `action` (enum, required): 'add' 或 'commit'
- `files` (array, optional): 檔案列表（省略則為全部）
- `message` (string, optional): 提交訊息（action='commit' 時必填）

##### `mcp_gitkraken_git_log_or_diff`
查看提交記錄或差異。

**參數**：
- `directory` (string, required): Git 倉庫路徑
- `action` (enum, required): 'log' 或 'diff'
- `commit` (string, optional): 用於 diff 的 commit（預設：HEAD）

##### `mcp_gitkraken_git_branch`
列出或建立分支。

**參數**：
- `directory` (string, required): Git 倉庫路徑
- `action` (enum, required): 'list' 或 'create'
- `branch_name` (string, optional): 分支名稱（create 時必填）

##### `mcp_gitkraken_git_checkout`
切換分支。

**參數**：
- `directory` (string, required): Git 倉庫路徑
- `branch` (string, required): 分支名稱

##### `mcp_gitkraken_git_push`
推送到遠端。

**參數**：
- `directory` (string, required): Git 倉庫路徑

##### `mcp_gitkraken_git_stash`
暫存當前變更。

**參數**：
- `directory` (string, required): Git 倉庫路徑
- `name` (string, optional): 暫存名稱

##### `mcp_gitkraken_git_blame`
顯示檔案每行的最後修改者。

**參數**：
- `directory` (string, required): Git 倉庫路徑
- `file` (string, required): 檔案路徑

##### `mcp_gitkraken_git_worktree`
管理 Git worktree。

**參數**：
- `directory` (string, required): Git 倉庫路徑
- `action` (enum, required): 'list' 或 'add'
- `path` (string, optional): worktree 路徑（add 時必填）
- `branch` (string, optional): 分支名稱（add 時可選）

#### GitHub/GitLab 整合

##### `mcp_gitkraken_pull_request_assigned_to_me`
搜尋與您相關的 PR（assignee、author 或 reviewer）。

**參數**：
- `provider` (enum, required): 'github', 'gitlab', 'bitbucket', 或 'azure'
- `is_closed` (boolean, optional): 是否搜尋已關閉的 PR
- `page` (integer, optional): 頁碼（預設：1）
- `repository_name` (string, optional): 倉庫名稱（Azure/Bitbucket 必填）
- `repository_organization` (string, optional): 組織名稱（Azure/Bitbucket 必填）
- `azure_project` (string, optional): Azure DevOps 專案名稱

##### `mcp_gitkraken_pull_request_get_detail`
取得 PR 詳細資訊。

**參數**：
- `provider` (enum, required): 提供者
- `pull_request_id` (string, required): PR ID
- `repository_name` (string, required): 倉庫名稱
- `repository_organization` (string, required): 組織名稱
- `pull_request_files` (boolean, optional): 是否包含檔案變更
- `azure_project` (string, optional): Azure DevOps 專案名稱

##### `mcp_gitkraken_pull_request_create`
建立新的 PR。

**參數**：
- `provider` (enum, required): 提供者
- `repository_name` (string, required): 倉庫名稱
- `repository_organization` (string, required): 組織名稱
- `title` (string, required): PR 標題
- `source_branch` (string, required): 來源分支
- `target_branch` (string, required): 目標分支
- `body` (string, optional): PR 描述
- `is_draft` (boolean, optional): 是否為草稿
- `azure_project` (string, optional): Azure DevOps 專案名稱

##### `mcp_gitkraken_pull_request_get_comments`
取得 PR 的所有評論。

**參數**：
- `provider` (enum, required): 提供者
- `pull_request_id` (string, required): PR ID
- `repository_name` (string, required): 倉庫名稱
- `repository_organization` (string, required): 組織名稱
- `azure_project` (string, optional): Azure DevOps 專案名稱

##### `mcp_gitkraken_pull_request_create_review`
建立 PR 審查。

**參數**：
- `provider` (enum, required): 提供者
- `pull_request_id` (string, required): PR ID
- `repository_name` (string, required): 倉庫名稱
- `repository_organization` (string, required): 組織名稱
- `review` (string, required): 審查評論
- `approve` (boolean, optional): 是否批准
- `azure_project` (string, optional): Azure DevOps 專案名稱

#### Issue 管理

##### `mcp_gitkraken_issues_assigned_to_me`
取得分配給您的 issue。

**參數**：
- `provider` (enum, required): 'github', 'gitlab', 'jira', 'azure', 或 'linear'
- `page` (integer, optional): 頁碼
- `repository_name` (string, optional): 倉庫名稱（GitHub/GitLab）
- `repository_organization` (string, optional): 組織名稱（GitHub/GitLab）
- `azure_organization` (string, optional): Azure DevOps 組織
- `azure_project` (string, optional): Azure DevOps 專案

##### `mcp_gitkraken_issues_get_detail`
取得 issue 詳細資訊。

**參數**：
- `provider` (enum, required): 提供者
- `issue_id` (string, required): Issue ID
- `repository_name` (string, optional): 倉庫名稱（GitHub/GitLab）
- `repository_organization` (string, optional): 組織名稱（GitHub/GitLab）
- `azure_organization` (string, optional): Azure DevOps 組織
- `azure_project` (string, optional): Azure DevOps 專案

##### `mcp_gitkraken_issues_add_comment`
在 issue 上新增評論。

**參數**：
- `provider` (enum, required): 提供者
- `issue_id` (string, required): Issue ID
- `comment` (string, required): 評論內容
- `repository_name` (string, optional): 倉庫名稱（GitHub/GitLab）
- `repository_organization` (string, optional): 組織名稱（GitHub/GitLab）
- `azure_organization` (string, optional): Azure DevOps 組織
- `azure_project` (string, optional): Azure DevOps 專案

#### 倉庫操作

##### `mcp_gitkraken_repository_get_file_content`
從倉庫取得檔案內容。

**參數**：
- `provider` (enum, required): 提供者
- `repository_name` (string, required): 倉庫名稱
- `repository_organization` (string, required): 組織名稱
- `ref` (string, required): 分支、標籤或 commit SHA
- `file_path` (string, required): 檔案路徑
- `azure_project` (string, optional): Azure DevOps 專案名稱

#### Workspace 管理

##### `mcp_gitkraken_gitkraken_workspace_list`
列出所有 GitKraken workspace。

**參數**：無

---

## 5. Filesystem MCP

**伺服器名稱**：`secure-filesystem-server`  
**狀態**：⚠️ 需要啟用 (`activate_filesystem_tools`)  
**用途**：安全的檔案系統操作

### 安全限制

僅能存取允許的目錄。使用 `list_allowed_directories` 查詢允許的路徑。

### 可用工具

#### 檔案讀取

##### `mcp_filesystem_read_text_file`
讀取文字檔案內容。

**參數**：
- `path` (string, required): 檔案絕對路徑
- `head` (number, optional): 僅讀取前 N 行
- `tail` (number, optional): 僅讀取後 N 行

**範例**：
```
讀取整個檔案：
mcp_filesystem_read_text_file(path="/path/to/file.txt")

讀取前 10 行：
mcp_filesystem_read_text_file(path="/path/to/file.txt", head=10)

讀取後 20 行：
mcp_filesystem_read_text_file(path="/path/to/file.txt", tail=20)
```

##### `mcp_filesystem_read_media_file`
讀取圖片或音訊檔案（返回 base64 編碼）。

**參數**：
- `path` (string, required): 檔案路徑

##### `mcp_filesystem_read_multiple_files`
同時讀取多個檔案。

**參數**：
- `paths` (array of strings, required): 檔案路徑列表

#### 檔案寫入

##### `mcp_filesystem_write_file`
建立或完全覆寫檔案。

**參數**：
- `path` (string, required): 檔案路徑
- `content` (string, required): 檔案內容

**警告**：會無警告覆寫現有檔案！

##### `mcp_filesystem_edit_file`
對文字檔案進行逐行編輯。

**參數**：
- `path` (string, required): 檔案路徑
- `edits` (array, required): 編輯操作列表
  - 每個編輯包含：
    - `oldText` (string): 要替換的文字（必須完全匹配）
    - `newText` (string): 新文字
- `dryRun` (boolean, optional): 預覽變更（顯示 git-style diff）

**範例**：
```
替換檔案中的文字：
mcp_filesystem_edit_file(
  path="/path/to/file.txt",
  edits=[
    {
      "oldText": "old version 1.0",
      "newText": "new version 2.0"
    }
  ]
)

預覽變更：
mcp_filesystem_edit_file(
  path="/path/to/file.txt",
  edits=[...],
  dryRun=true
)
```

#### 目錄操作

##### `mcp_filesystem_create_directory`
建立目錄（支援多層巢狀）。

**參數**：
- `path` (string, required): 目錄路徑

##### `mcp_filesystem_list_directory`
列出目錄內容。

**參數**：
- `path` (string, required): 目錄路徑

**輸出格式**：
- `[FILE]` 前綴表示檔案
- `[DIR]` 前綴表示目錄

##### `mcp_filesystem_list_directory_with_sizes`
列出目錄內容（含檔案大小）。

**參數**：
- `path` (string, required): 目錄路徑
- `sortBy` (enum, optional): 'name' 或 'size'（預設：'name'）

##### `mcp_filesystem_directory_tree`
取得目錄的遞迴樹狀結構（JSON 格式）。

**參數**：
- `path` (string, required): 目錄路徑

**輸出結構**：
```json
{
  "name": "folder",
  "type": "directory",
  "children": [
    {
      "name": "file.txt",
      "type": "file"
    },
    {
      "name": "subfolder",
      "type": "directory",
      "children": []
    }
  ]
}
```

#### 檔案管理

##### `mcp_filesystem_move_file`
移動或重新命名檔案。

**參數**：
- `source` (string, required): 來源路徑
- `destination` (string, required): 目標路徑

##### `mcp_filesystem_search_files`
遞迴搜尋檔案（不區分大小寫）。

**參數**：
- `path` (string, required): 起始搜尋路徑
- `pattern` (string, required): 搜尋模式
- `excludePatterns` (array, optional): 排除模式

**範例**：
```
搜尋所有 Python 檔案：
mcp_filesystem_search_files(
  path="/project",
  pattern="*.py"
)

搜尋特定名稱，排除測試檔案：
mcp_filesystem_search_files(
  path="/project",
  pattern="config",
  excludePatterns=["*test*", "*_test.py"]
)
```

##### `mcp_filesystem_get_file_info`
取得檔案或目錄的詳細資訊。

**參數**：
- `path` (string, required): 檔案路徑

**返回資訊**：
- 大小
- 建立時間
- 修改時間
- 權限
- 類型

##### `mcp_filesystem_list_allowed_directories`
列出允許存取的目錄。

**參數**：無

---

## 6. Memory MCP

**伺服器名稱**：`memory-server`  
**狀態**：⚠️ 需要啟用 (`activate_knowledge_graph_tools`)  
**用途**：管理知識圖譜（Knowledge Graph）

### 概念說明

Memory MCP 維護一個知識圖譜，包含：
- **實體（Entities）**：具名的節點，有類型和觀察記錄
- **關係（Relations）**：連接兩個實體的有向邊，有類型描述

### 可用工具

#### 實體管理

##### `mcp_memory_create_entities`
建立多個實體。

**參數**：
- `entities` (array, required): 實體列表
  - 每個實體包含：
    - `name` (string): 實體名稱
    - `entityType` (string): 實體類型
    - `observations` (array of strings): 觀察記錄

**範例**：
```
建立專案和成員實體：
mcp_memory_create_entities(
  entities=[
    {
      "name": "AutoGen Project",
      "entityType": "Project",
      "observations": [
        "多語言 AI 框架",
        "支援 Python 和 .NET",
        "由 Microsoft 維護"
      ]
    },
    {
      "name": "John Doe",
      "entityType": "Developer",
      "observations": [
        "專案主要貢獻者",
        "專精於 Python 開發"
      ]
    }
  ]
)
```

##### `mcp_memory_delete_entities`
刪除實體及相關關係。

**參數**：
- `entityNames` (array of strings, required): 要刪除的實體名稱

#### 關係管理

##### `mcp_memory_create_relations`
建立實體間的關係。

**參數**：
- `relations` (array, required): 關係列表
  - 每個關係包含：
    - `from` (string): 起始實體名稱
    - `to` (string): 目標實體名稱
    - `relationType` (string): 關係類型（使用主動語態）

**範例**：
```
建立「開發」關係：
mcp_memory_create_relations(
  relations=[
    {
      "from": "John Doe",
      "to": "AutoGen Project",
      "relationType": "develops"
    }
  ]
)
```

**注意**：關係類型應使用主動語態（如 "develops", "manages", "uses"）。

##### `mcp_memory_delete_relations`
刪除關係。

**參數**：
- `relations` (array, required): 要刪除的關係列表（格式同 create_relations）

#### 觀察記錄管理

##### `mcp_memory_add_observations`
為現有實體新增觀察記錄。

**參數**：
- `observations` (array, required): 觀察列表
  - 每個項目包含：
    - `entityName` (string): 實體名稱
    - `contents` (array of strings): 觀察內容

**範例**：
```
新增觀察記錄：
mcp_memory_add_observations(
  observations=[
    {
      "entityName": "AutoGen Project",
      "contents": [
        "2024年發布 0.4 版本",
        "支援事件驅動架構"
      ]
    }
  ]
)
```

##### `mcp_memory_delete_observations`
刪除特定觀察記錄。

**參數**：
- `deletions` (array, required): 刪除列表
  - 每個項目包含：
    - `entityName` (string): 實體名稱
    - `observations` (array of strings): 要刪除的觀察內容

#### 查詢操作

##### `mcp_memory_read_graph`
讀取整個知識圖譜。

**參數**：無

**返回格式**：
```json
{
  "entities": [
    {
      "name": "...",
      "entityType": "...",
      "observations": [...]
    }
  ],
  "relations": [
    {
      "from": "...",
      "to": "...",
      "relationType": "..."
    }
  ]
}
```

##### `mcp_memory_search_nodes`
搜尋節點（匹配實體名稱、類型或觀察內容）。

**參數**：
- `query` (string, required): 搜尋關鍵字

##### `mcp_memory_open_nodes`
根據名稱取得特定實體。

**參數**：
- `names` (array of strings, required): 實體名稱列表

---

## 7. Obsidian MCP

**伺服器名稱**：`mcp-obsidian`  
**狀態**：⚠️ 需要啟用 (`activate_obsidian_file_management_tools`)  
**用途**：管理 Obsidian 筆記庫

### 前提條件

需要配置 Obsidian vault 路徑。

### 可用工具

#### 檔案瀏覽

##### `mcp_obsidian_obsidian_list_files_in_vault`
列出 vault 根目錄的所有檔案。

**參數**：無

##### `mcp_obsidian_obsidian_list_files_in_dir`
列出特定目錄的檔案。

**參數**：
- `dirpath` (string, required): 目錄路徑（相對於 vault 根目錄）

**注意**：空目錄不會被返回。

#### 檔案讀取

##### `mcp_obsidian_obsidian_get_file_contents`
讀取單一檔案內容。

**參數**：
- `filepath` (string, required): 檔案路徑（相對於 vault 根目錄）

##### `mcp_obsidian_obsidian_batch_get_file_contents`
批次讀取多個檔案。

**參數**：
- `filepaths` (array of strings, required): 檔案路徑列表

**輸出**：所有檔案內容會串接，並以標題分隔。

#### 檔案修改

##### `mcp_obsidian_obsidian_append_content`
在檔案末尾追加內容（若檔案不存在則建立）。

**參數**：
- `filepath` (string, required): 檔案路徑
- `content` (string, required): 要追加的內容

##### `mcp_obsidian_obsidian_patch_content`
在特定位置插入內容。

**參數**：
- `filepath` (string, required): 檔案路徑
- `operation` (enum, required): 'append', 'prepend', 或 'replace'
- `target_type` (enum, required): 'heading', 'block', 或 'frontmatter'
- `target` (string, required): 目標識別符
  - heading: 標題路徑
  - block: 區塊引用
  - frontmatter: frontmatter 欄位名稱
- `content` (string, required): 要插入的內容

**範例**：
```
在標題下方追加內容：
mcp_obsidian_obsidian_patch_content(
  filepath="note.md",
  operation="append",
  target_type="heading",
  target="## Section 1",
  content="新增的段落內容"
)

修改 frontmatter 欄位：
mcp_obsidian_obsidian_patch_content(
  filepath="note.md",
  operation="replace",
  target_type="frontmatter",
  target="status",
  content="completed"
)
```

##### `mcp_obsidian_obsidian_delete_file`
刪除檔案或目錄。

**參數**：
- `filepath` (string, required): 檔案路徑
- `confirm` (boolean, required): 確認刪除（必須為 true）

#### 搜尋功能

##### `mcp_obsidian_obsidian_simple_search`
簡單文字搜尋。

**參數**：
- `query` (string, required): 搜尋關鍵字
- `context_length` (integer, optional): 上下文字元數（預設：100）

##### `mcp_obsidian_obsidian_complex_search`
使用 JsonLogic 進行複雜搜尋。

**參數**：
- `query` (object, required): JsonLogic 查詢物件

**JsonLogic 範例**：
```
搜尋所有 markdown 檔案：
{
  "glob": ["*.md", {"var": "path"}]
}

搜尋特定標籤：
{
  "in": ["#project", {"var": "tags"}]
}
```

**支援的運算子**：
- 標準 JsonLogic 運算子
- `glob`: 模式匹配
- `regexp`: 正則表達式匹配

#### Periodic Notes

##### `mcp_obsidian_obsidian_get_periodic_note`
取得當前的週期性筆記。

**參數**：
- `period` (enum, required): 'daily', 'weekly', 'monthly', 'quarterly', 或 'yearly'

##### `mcp_obsidian_obsidian_get_recent_periodic_notes`
取得最近的週期性筆記。

**參數**：
- `period` (enum, required): 週期類型
- `limit` (integer, optional): 最大數量（預設：5，最大：50）
- `include_content` (boolean, optional): 是否包含內容（預設：false）

#### 變更追蹤

##### `mcp_obsidian_obsidian_get_recent_changes`
取得最近修改的檔案。

**參數**：
- `limit` (integer, optional): 最大數量（預設：10，最大：100）
- `days` (integer, optional): 天數範圍（預設：90）

---

## 8. Everything MCP

**伺服器名稱**：`Everything Example Server`  
**狀態**：⚠️ 需要啟用 (`activate_mcp_tools_summary`)  
**用途**：MCP 協定功能測試與示範

### 特殊說明

這是一個測試伺服器，展示 MCP 協定的各種功能。不建議在生產環境使用。

### 可用工具

#### 基本工具

##### `mcp_everything_echo`
回傳輸入的訊息。

**參數**：
- `message` (string, required): 要回傳的訊息

##### `mcp_everything_add`
加法運算。

**參數**：
- `a` (number, required): 第一個數字
- `b` (number, required): 第二個數字

#### 進階功能

##### `mcp_everything_longRunningOperation`
模擬長時間執行的操作（含進度更新）。

**參數**：
- `duration` (number, optional): 持續時間（秒，預設：10）
- `steps` (number, optional): 步驟數（預設：5）

##### `mcp_everything_printEnv`
列印所有環境變數（用於除錯）。

**參數**：無

##### `mcp_everything_sampleLLM`
使用 MCP 的 sampling 功能呼叫 LLM。

**參數**：
- `prompt` (string, required): 提示詞
- `maxTokens` (number, optional): 最大 token 數（預設：100）

#### 資源功能

##### `mcp_everything_getTinyImage`
返回測試用的小圖片（MCP_TINY_IMAGE）。

**參數**：無

##### `mcp_everything_getResourceReference`
返回指定資源的引用。

**參數**：
- `resourceId` (number, required): 資源 ID（1-100）

**資源規則**：
- 偶數 ID (2, 4, 6...): 文字內容
- 奇數 ID (1, 3, 5...): 二進位資料（base64）

##### `mcp_everything_getResourceLinks`
返回多個資源連結。

**參數**：
- `count` (number, optional): 資源數量（1-10，預設：3）

#### Annotation 功能

##### `mcp_everything_annotatedMessage`
展示如何使用 annotations 提供內容的 metadata。

**參數**：
- `messageType` (enum, required): 'error', 'success', 或 'debug'
- `includeImage` (boolean, optional): 是否包含圖片（預設：false）

#### 結構化輸出

##### `mcp_everything_structuredContent`
返回結構化內容與 output schema（用於客戶端驗證）。

**參數**：
- `location` (string, required): 城市名稱或郵遞區號

**輸出 Schema**：
```json
{
  "temperature": number,
  "conditions": string,
  "humidity": number
}
```

#### Roots 與 Elicitation

##### `mcp_everything_listRoots`
列出客戶端提供的 MCP roots（展示 roots 協定功能）。

**參數**：無

##### `mcp_everything_startElicitation`
展示 Elicitation 功能（詢問使用者偏好：顏色、數字、寵物）。

**參數**：無

---

## 🔧 啟用指南

### 永久啟用的 MCP

以下 MCP 無需手動啟用，隨時可用：
- ✅ Time
- ✅ Sequential Thinking
- ✅ Fetch
- ✅ GitKraken

### 需要啟用的 MCP

以下 MCP 需要在每個新對話 thread 開始時啟用：

```bash
# 方法 1：透過 Copilot 指示檔自動啟用
# 在 .github/copilot-instructions.md 中加入啟用指令

# 方法 2：手動啟用（在對話中執行）
activate_filesystem_tools()
activate_knowledge_graph_tools()
activate_obsidian_file_management_tools()
activate_mcp_tools_summary()
```

### 檢查啟用狀態

如果收到以下錯誤訊息：
```
Tool is currently disabled by the user
```

表示該 MCP 需要啟用。請執行對應的 `activate_*` 函式。

---

## 📚 最佳實踐

### 1. 檔案操作

- 使用 `list_allowed_directories` 確認可存取路徑
- 寫入前使用 `read_text_file` 檢查現有內容
- 使用 `edit_file` 而非 `write_file` 進行部分修改
- 啟用 `dryRun` 預覽變更

### 2. Git 操作

- 使用 `git_status` 檢查工作目錄狀態
- 提交前使用 `git_diff` 查看變更
- 為提交訊息提供清晰描述

### 3. 知識圖譜

- 為實體提供描述性的觀察記錄
- 使用主動語態描述關係（"develops", "uses"）
- 定期使用 `read_graph` 檢視整體結構
- 使用 `search_nodes` 而非 `read_graph` 進行查詢

### 4. Obsidian 筆記

- 使用相對路徑（相對於 vault 根目錄）
- 善用 `patch_content` 進行精確的內容插入
- 利用 `periodic_notes` 功能管理日記型筆記

---

## 🐛 疑難排解

### 問題 1：工具被禁用

**錯誤訊息**：
```
Tool is currently disabled by the user
```

**解決方法**：
執行對應的 `activate_*` 函式：
- Filesystem → `activate_filesystem_tools()`
- Memory → `activate_knowledge_graph_tools()`
- Obsidian → `activate_obsidian_file_management_tools()`
- Everything → `activate_mcp_tools_summary()`

### 問題 2：路徑存取被拒

**錯誤訊息**：
```
Access denied / Path not in allowed directories
```

**解決方法**：
1. 執行 `mcp_filesystem_list_allowed_directories()` 查看允許的路徑
2. 確認您的路徑在允許範圍內
3. 如需新增允許路徑，修改 `mcp.json` 配置檔

### 問題 3：MCP 服務未啟動

**症狀**：
工具完全無法使用，即使執行 `activate_*` 也無效。

**解決方法**：
1. 開啟 `mcp.json` 檔案
2. 檢查對應 MCP 的狀態
3. 手動點擊「啟動」按鈕
4. 重新載入 VS Code 視窗

---

## 📞 參考資源

- **MCP 配置檔**：`~/Library/Application Support/Code/User/mcp.json` (macOS)
- **Copilot 指示檔**：`.github/copilot-instructions.md`
- **官方文件**：[Model Context Protocol](https://modelcontextprotocol.io/)

---

**最後更新**：2025年10月31日  
**版本**：1.0.0
