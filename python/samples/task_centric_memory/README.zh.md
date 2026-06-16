# 任務導向記憶(Task-Centric Memory) 範例詳細說明

## 範例概述
- 本目錄收錄多個程式，展示如何結合 AutoGen Task-Centric Memory 實驗特性，讓代理從使用者建議、示範或自身經驗中快速學習，並評估記憶檢索的精確率／召回率。[README.md:1](README.md#L1)

## 互動範例
- `chat_with_teachable_agent.py`：建立帶有 `Teachability` 記憶的 `AssistantAgent`，在終端與使用者互動並於多輪對話間持續記住教導內容。[chat_with_teachable_agent.py#L8-L35](chat_with_teachable_agent.py#L8)

## 評估腳本
- `eval_retrieval.py`：使用 `MemoryController` 在 YAML 定義的任務／洞察資料上測量檢索精確率與召回率，並將結果輸出至頁面日誌。[eval_retrieval.py#L28-L84](eval_retrieval.py#L28)
- `eval_teachability.py`：透過 `Apprentice` 與 `Grader` 類別測試代理在接受教導前後的答題表現，驗證記憶是否成功存取與套用。[eval_teachability.py#L31-L85](eval_teachability.py#L31)
- `eval_learning_from_demonstration.py` 與 `eval_self_teaching.py` 採相同結構，分別模擬從示範與自我迭代中學習的流程。
- `utils.py` 提供建立 OpenAI ChatCompletionClient 與讀取 YAML 的輔助函式，供所有評估腳本共用。[utils.py#L10-L31](utils.py#L10)

## 資料與設定
- `configs/` 內的 YAML 檔定義測試案例、PageLogger 設定、模型參數與任務／洞察來源；`data_files/` 則存放對應的任務描述、示範與答案資料。[README.md:17](README.md#L17)

## 執行建議
- 依 README 安裝 `autogen-agentchat`、`autogen-ext[openai,task-centric-memory]` 等套件並設定 `OPENAI_API_KEY`，即可在目錄中執行對應腳本，例如 `python eval_teachability.py configs/teachability.yaml`。[README.md:33](README.md#L33)｜[README.md:112](README.md#L112)

## 延伸參考
- 建議閱讀 AutoGen AgentChat 官方文件，了解如何將記憶模組整合至自訂代理；此外，實驗性 Task-Centric Memory API 的更多細節可參考 `autogen-ext` 套件說明與官方部落格更新。[AutoGen AgentChat User Guide](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/index.html)
