# AutoGen Python Samples 快速導覽

本目錄收錄多種 AutoGen 範例，涵蓋 AgentChat、Core、分散式 runtime 與工具整合。以下整理各範例的中文詳細說明，協助快速定位程式碼與操作指引。

## AgentChat 系列
- [agentchat_azure_postgresql](agentchat_azure_postgresql/README.zh.md) – 連結 Azure PostgreSQL 多代理示例。
- [agentchat_chainlit](agentchat_chainlit/README.zh.md) – 使用 Chainlit 與 AgentChat 建立單代理、團隊與 UserProxy 互動。
- [agentchat_chess_game](agentchat_chess_game/README.zh.md) – 與 AI 對弈的棋局範例，示範工具使用與串流。
- [agentchat_dspy](agentchat_dspy/README.zh.md) – DSPy 整合範例預留狀態說明。
- [agentchat_fastapi](agentchat_fastapi/README.zh.md) – FastAPI + AgentChat 的單代理與多代理服務與狀態持久化。
- [agentchat_graphrag](agentchat_graphrag/README.zh.md) – 整合 GraphRAG 搜尋工具的聊天代理。
- [agentchat_streamlit](agentchat_streamlit/README.zh.md) – 利用 Streamlit 建立 AgentChat 聊天介面。

## AutoGen Core 與 Runtime
- [core_async_human_in_the_loop](core_async_human_in_the_loop/README.zh.md) – 非同步人類介入與狀態持久化範例。
- [core_chainlit](core_chainlit/README.zh.md) – Chainlit 與 Core runtime 的單代理與群聊展示。
- [core_chess_game](core_chess_game/README.zh.md) – 兩個核心代理透過工具代理進行棋局對弈。
- [core_distributed-group-chat](core_distributed-group-chat/README.zh.md) – gRPC 分散式多代理寫作／編輯協作。
- [core_grpc_worker_runtime](core_grpc_worker_runtime/README.zh.md) – gRPC worker runtime 發布訂閱、RPC 與級聯示例。
- [core_semantic_router](core_semantic_router/README.zh.md) – 依意圖路由至適合代理的分散式案例。
- [core_streaming_handoffs_fastapi](core_streaming_handoffs_fastapi/README.zh.md) – FastAPI 串流與多代理 handoff。
- [core_streaming_response_fastapi](core_streaming_response_fastapi/README.zh.md) – 最小化 FastAPI 串流聊天服務。
- [core_xlang_hello_python_agent](core_xlang_hello_python_agent/README.zh.md) – Python 代理與 .NET Agent Runtime 互通。

## 其他應用
- [gitty](gitty/README.zh.md) – AutoGen 驅動的 GitHub Issue/PR 回覆 CLI。
- [task_centric_memory](task_centric_memory/README.zh.md) – Task-Centric Memory 範例與評估腳本。
- pdf_review_agent – 保留原始英文 README，提供 PDF 審查範例。

> 若需英語原始說明，可參考各範例中的 `README.md`；此表列的 `README.zh.md` 則提供中文詳細導覽。
