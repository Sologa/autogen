# AgentChat 西洋棋對弈範例詳細說明

## 範例概述
- 範例展示如何運用 `AssistantAgent` 與 `ChatCompletionClient` 進行棋局推理，支援 AI vs AI 及 AI vs 人類兩種模式，並示範限制對話記憶與串流輸出的使用方式。[README.md:1](README.md#L1)

## 主要檔案與邏輯
- `main.py`：建立 `AssistantAgent` 時設定 `BufferedChatCompletionContext(buffer_size=10)`，限制模型僅保留最近 10 則訊息以控制上下文大小。[main.py#L14-L22](main.py#L14)
- `get_ai_prompt` 與 `extract_move`：提示中要求模型輸出 `<move>...</move>` 格式，並以 `chess.Move.from_uci` 驗證合法性；若模型回傳無效走法則重新提問或改以隨機走法應對。[main.py#L32-L98](main.py#L32)
- `main()`：讀取 `model_config.yaml` 建立模型客戶端，依 `--human` 參數決定是否詢問使用者輸入，並在每步後印出棋盤狀態與最終結果。[main.py#L101-L131](main.py#L101)

## 安裝與執行
- 需安裝 `chess`、`autogen-agentchat`、`pyyaml` 以及 `autogen-ext[openai]`（或對應模型供應商擴充），再自訂 `model_config.yaml`。[README.md:7](README.md#L7)
- 於範例目錄執行 `python main.py` 進行 AI 對弈；若欲與 AI 對戰，可加上 `--human` 參數後由終端輸入合法 UCI 棋步。[README.md:71](README.md#L71)

## 延伸參考
- AutoGen AgentChat 模型設定與工具整合說明，可參考官方「Models」章節了解其他模型或 API 端點配置方式。[Models Documentation](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/models.html)
