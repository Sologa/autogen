# AutoGen Studio 套件說明

## 套件定位
- AutoGen Studio 提供圖形化介面與 API，用於快速原型設計、測試與觀察多代理工作流程，並內建 Studio Lite 以支援輕量級實驗。[python/packages/autogen-studio/README.md](python/packages/autogen-studio/README.md)
- 官方文件（0.2 版）說明安裝流程、資料庫選項與 Studio Lite 操作，建議搭配程式碼查閱。[https://microsoft.github.io/autogen/0.2/docs/autogen-studio/getting-started](https://microsoft.github.io/autogen/0.2/docs/autogen-studio/getting-started)

## 後端組件
- `DatabaseManager` 使用 SQLModel 管理資料庫連線、遷移初始化與重建流程，支援 SQLite 特殊設定及 JSON 序列化，保證 UI 與 API 的狀態一致。[python/packages/autogen-studio/autogenstudio/database/db_manager.py#L16](python/packages/autogen-studio/autogenstudio/database/db_manager.py#L16)
- `TeamManager` 能從 JSON/YAML/ComponentModel 建立 AgentChat 團隊，並串流輸出代理事件、LLM 呼叫紀錄，支援人類輸入與環境變數載入，為 Studio 執行核心。[python/packages/autogen-studio/autogenstudio/teammanager/teammanager.py#L28](python/packages/autogen-studio/autogenstudio/teammanager/teammanager.py#L28)
- `LiteStudio` 封裝 Lite 模式的配置檔處理、暫存檔建立、Uvicorn 啟動與瀏覽器自動開啟，方便在程式中快速啟服務。[python/packages/autogen-studio/autogenstudio/lite/studio.py#L1](python/packages/autogen-studio/autogenstudio/lite/studio.py#L1)
- `web/app.py` 建立 FastAPI 應用、註冊 CORS、認證中介層與多個 REST/WebSocket 路由，並透過 lifespan 管理資源初始化與收尾。[python/packages/autogen-studio/autogenstudio/web/app.py#L1](python/packages/autogen-studio/autogenstudio/web/app.py#L1)

## 典型工作流程
- 建議先呼叫 `DatabaseManager.initialize_database` 檢查資料表與遷移狀態，確保 UI 載入時不會遇到 schema 不一致。[python/packages/autogen-studio/autogenstudio/database/db_manager.py#L63](python/packages/autogen-studio/autogenstudio/database/db_manager.py#L63)
- 使用 Studio Lite 時，可傳入 `ComponentModel` 物件或 JSON 檔案，Lite 會轉為暫存檔並掛載到伺服器環境變數供前端讀取。[python/packages/autogen-studio/autogenstudio/lite/studio.py#L43](python/packages/autogen-studio/autogenstudio/lite/studio.py#L43)
- 在執行團隊期間，可透過 `TeamManager.run_stream` 取得 `LLMCallEvent` 等事件，與官方設計文件的追蹤流程相呼應，有助於除錯與分析。[python/packages/autogen-studio/autogenstudio/teammanager/teammanager.py#L109](python/packages/autogen-studio/autogenstudio/teammanager/teammanager.py#L109) [docs/design/01%20-%20Programming%20Model.md](docs/design/01%20-%20Programming%20Model.md)

## 開發建議
- 若需擴充自訂 API，應在 `web/app.py` 中新增 FastAPI router，並遵守既有的認證與 CORS 設定，避免破壞 Studio 前端的跨域請求。[python/packages/autogen-studio/autogenstudio/web/app.py#L38](python/packages/autogen-studio/autogenstudio/web/app.py#L38)
- 自訂資料表時請同步更新 `SchemaManager` 遷移腳本，以免破壞 `auto_upgrade` 流程；大規模變更建議透過 Alembic 指令產生遷移檔。[python/packages/autogen-studio/autogenstudio/database/db_manager.py#L56](python/packages/autogen-studio/autogenstudio/database/db_manager.py#L56)
