# GraphRAG 與 AgentChat 整合範例詳細說明

## 範例概述
- 此範例結合 GraphRAG 搜尋工具與 AutoGen AgentChat，透過單一 `AssistantAgent` 根據使用者查詢自動選擇「全域搜尋」或「局部搜尋」工具並串流回覆，示範如何在聊天體驗中引入圖形化檢索能力。[README.md:1](README.md#L1)

## 主要檔案與邏輯
- `app.py`：  
  - 啟動時檢查 `OPENAI_API_KEY`，建立輸入資料夾並於缺少範例文本時自動下載《福爾摩斯探案》。[app.py#L34-L52](app.py#L34)  
  - 以 `GlobalSearchTool.from_settings` 與 `LocalSearchTool.from_settings` 載入 GraphRAG 設定，並將兩者註冊為 `AssistantAgent` 工具，透過系統訊息要求代理僅負責挑選工具。[app.py#L55-L74](app.py#L55)
  - 使用 `Console(assistant_agent.run_stream(...))` 串流工具結果與模型輸出，示範查詢「車站站長對 Becher 醫生的說法」。[app.py#L77-L81](app.py#L77)
- `prompts/` 資料夾提供 GraphRAG 預設的提示模板，與 GraphRAG CLI 配套使用。

## 前置作業
- 需先完成 GraphRAG CLI 初始化、提示調整與索引建立流程，並於 `settings.yaml` 中配置模型與鑑別資訊，索引結果將儲存於 `output/` 目錄。[README.md:26](README.md#L26)
- 將 AutoGen 模型組態寫入 `model_config.yaml`，可從 `model_config_template.yaml` 複製範例後填入實際 API 參數。[README.md:54](README.md#L54)

## 執行步驟
- 安裝 `requirements.txt` 所列依賴、設定環境變數後執行 `python app.py`，程式會自動檢查索引檔並串流查詢結果。[README.md:64](README.md#L64)
- 若要測試不同問題，可編輯 `app.py` 內的 `query` 字串或改寫成自訂輸入流程。[app.py#L77-L80](app.py#L77)

## 延伸參考
- GraphRAG 設定與 YAML 參數說明可參考官方文件；AutoGen AgentChat 的工具調用模式詳載於官方指南，便於擴充更多檢索工具或整合其他代理。[GraphRAG 設定指南](https://microsoft.github.io/graphrag/config/yaml/)｜[AutoGen AgentChat User Guide](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)
