# Magentic-One CLI 套件說明

## 套件定位
- `magentic-one-cli` 提供命令列介面，讓使用者以單行指令執行 Magentic-One 多代理系統，支援人類在迴圈 (HIL) 模式、Rich 介面與自訂模型設定。[python/packages/magentic-one-cli/README.md](python/packages/magentic-one-cli/README.md)

## 核心流程
- `_m1.py` 使用 `argparse` 處理任務字串、HIL 切換、Rich 渲染與 `--config` 選項，並提供 `--sample-config` 輸出範本，方便快速建立模型設定。[python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L31](python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L31)
- CLI 會讀取 YAML/JSON 設定檔建立 `ChatCompletionClient`，並透過 `UserInputManager`、`DockerCommandLineCodeExecutor` 與 `autogen_ext.teams.MagenticOne` 執行實際任務流程。[python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L112](python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L112)
- Rich 模式下會啟用 `autogen_ext.ui.RichConsole` 顯示代理輸出與事件，若未指定則改用純文字 Console。[python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L141](python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L141)

## 使用建議
- 建議在執行前準備 `config.yaml`，至少指定 `client.provider` 與模型名稱；若使用 Azure OpenAI，請依官方文件設定對應欄位。[python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L90](python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L90) [https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html](https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html)
- 若需串接人類審批，可自訂 `approval_func` 或 `input_func`，並利用 `--no-hil`、`--rich` 參數調整互動體驗。[python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L138](python/packages/magentic-one-cli/src/magentic_one_cli/_m1.py#L138)
