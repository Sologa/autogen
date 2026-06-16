# pyautogen 套件說明

## 套件定位
- `pyautogen` 為 AutoGen 0.4.x 的代理套件 (proxy package)，安裝後會拉取最新的 `autogen-agentchat` 功能；若需舊版 0.2.x，需手動指定 `pyautogen~=0.2.0`。[python/packages/pyautogen/README.md](python/packages/pyautogen/README.md)
- 官方 Migration Guide 提供從 0.2 遷移到 AgentChat API 的步驟，包含核心差異與程式碼示例。[https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/migration-guide.html](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/migration-guide.html)

## 建議用法
- 新專案可直接 `pip install pyautogen` 取得最新 AgentChat 功能，與官方文檔保持同步。[https://microsoft.github.io/autogen/stable/index.html](https://microsoft.github.io/autogen/stable/index.html)
- 若需鎖定特定版本或 fork，可參考 README 提供的社群討論與套件權限說明，避免使用非官方維護的發佈來源。[python/packages/pyautogen/README.md](python/packages/pyautogen/README.md)
