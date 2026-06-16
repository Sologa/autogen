# AutoGenBench 套件說明

## 核心定位
- AutoGenBench 旨在於受控環境反覆執行 AutoGen 任務，透過 Docker 或原生模式建立乾淨工作目錄並記錄完整實驗資料，以利基準測試與回歸分析。[python/packages/agbench/README.md](python/packages/agbench/README.md)
- 套件以 `agbench` 指令為對外入口，集中管理 `run`、`tabulate`、`lint` 與 `remove_missing` 等子命令的解析與說明字串。[python/packages/agbench/src/agbench/cli.py#L33](python/packages/agbench/src/agbench/cli.py#L33)

## 主要模組與流程
- `run_cmd.py` 內的 `run_scenarios` 會展開 JSONL/資料夾情境、建立結果資料夾、讀取 ENV 設定並選擇 Docker 或原生執行路徑，是批次執行的核心流程。[python/packages/agbench/src/agbench/run_cmd.py#L59](python/packages/agbench/src/agbench/run_cmd.py#L59)
- `expand_scenario` 會依模板複製檔案並執行字串替換，保證每次重跑都能回到同一初始狀態。[python/packages/agbench/src/agbench/run_cmd.py#L181](python/packages/agbench/src/agbench/run_cmd.py#L181)
- `tabulate_cmd.py` 掃描結果資料夾、套用自訂 scorer/timer，最後以 Pandas 與 Tabulate 整理成表格，亦支援 CSV/Excel 匯出。[python/packages/agbench/src/agbench/tabulate_cmd.py#L74](python/packages/agbench/src/agbench/tabulate_cmd.py#L74)
- `linter/` 子模組提供腳本檢查器與 YAML 解析，確保基準任務描述格式一致，減少長時運行前的配置錯誤。[python/packages/agbench/src/agbench/linter](python/packages/agbench/src/agbench/linter)

## 開發與測試建議
- 建議於本機先建立 `AUTOGEN_REPO_BASE` 與 `OAI_CONFIG_LIST`，以對應 README 所述的 Docker 掛載邏輯，避免容器找不到模型金鑰。[python/packages/agbench/README.md](python/packages/agbench/README.md)
- 若需追蹤特定 scenario，可啟用 `subsample` 或 `n_repeats` 參數，並檢查 `Results/<scenario>/<id>/<repeat>/console_log.txt` 作為快速診斷依據。[python/packages/agbench/src/agbench/run_cmd.py#L150](python/packages/agbench/src/agbench/run_cmd.py#L150)
- 透過 `tabulate` 匯出的摘要建議再比對原始 log，以符合官方文檔對重現性的要求。[https://microsoft.github.io/autogen/stable/index.html](https://microsoft.github.io/autogen/stable/index.html)

## 相關參考
- 官方文件：AutoGen Benchmarks 與使用指南（請參考 AutoGen 官方站點的 Benchmarks 專章）。[https://microsoft.github.io/autogen/stable/index.html](https://microsoft.github.io/autogen/stable/index.html)
- Repository 設計文件：`docs/design/05 - Services.md` 介紹的服務拓撲可作為大型基準測試部署參考。[docs/design/05%20-%20Services.md](docs/design/05%20-%20Services.md)
