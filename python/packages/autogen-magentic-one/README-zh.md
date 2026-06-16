# Magentic-One 傳統套件說明

## 套件定位
- 本套件保留 AutoGen 0.4.4 之前的 Magentic-One 原始實作，現階段主要用於查閱歷史程式碼或協助舊版專案維護，官方新功能已整合進 `autogen-agentchat` 與 `autogen-ext`。[python/packages/autogen-magentic-one/README.md](python/packages/autogen-magentic-one/README.md)
- 官方文件指出新版 Magentic-One 以 AgentChat Team 形式提供，並建議使用者遷移至最新介面以獲得模組化與擴充性優勢。[https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html)

## 建議使用時機
- 若需比對舊版 orchestrator、Web/File Surfer 實作細節或撰寫遷移工具，可參考 README 提供的 v0.4.4 原始碼連結。[python/packages/autogen-magentic-one/README.md](python/packages/autogen-magentic-one/README.md)
- 新專案建議直接採用 `autogen-ext` 的 `MagenticOne` 組件，以獲得人類監督、Docker 執行器與最新安全建議的整合支援。[python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py#L22](python/packages/autogen-ext/src/autogen_ext/teams/magentic_one.py#L22)
