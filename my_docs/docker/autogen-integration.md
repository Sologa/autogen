# AutoGen 整合使用指南

> **目的：** 在 AutoGen Agent 中正確使用 surveyx-env Docker image

[← 返回主指南](./README.md)

---

## ⚠️ 重要：API 變更說明

### AutoGen 0.7.5+ 的關鍵變更

**❌ 舊版 API（已無效）：**
```python
from autogen.agentchat.contrib.docker_command_line_code_executor import DockerCommandLineCodeExecutor

executor = DockerCommandLineCodeExecutor(
    image="surveyx-env:latest",
    command=["bash", "-c", "source activate surveyx && python {file_name}"]  # ❌ 無此參數
)
```

**✅ 新版 API（0.7.5+）：**
```python
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor

executor = DockerCommandLineCodeExecutor(
    image="surveyx-env:latest",
    work_dir="./workspace",
    init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",  # ✅ 使用 init_command
    timeout=120,
    auto_remove=True,
)
```

**API 變更對照表：**

| 參數 | 舊版 (< 0.7) | 新版 (0.7.5+) | 說明 |
|------|-------------|--------------|------|
| `command` | ✅ 存在 | ❌ 已移除 | 自訂執行命令 |
| `init_command` | ❌ 不存在 | ✅ 新增 | 每次執行前的初始化指令 |
| `work_dir` | 絕對路徑 | 相對/絕對 | 主機端工作目錄 |
| `auto_remove` | 預設 False | 預設 True | 執行後自動刪除容器 |

**參考來源：**
- [AutoGen 源碼：_docker_code_executor.py:81](https://github.com/microsoft/autogen/blob/main/python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L81)
- [AutoGen 源碼：_docker_code_executor.py:528](https://github.com/microsoft/autogen/blob/main/python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L528)

---

## 🚀 基本使用

### 最小範例

```python
import asyncio
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor
from autogen_core.code_executor import CodeBlock

async def main():
    # 建立執行器
    executor = DockerCommandLineCodeExecutor(
        image="surveyx-env:latest",
        work_dir="./workspace",
        init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",
    )
    
    # 執行程式碼
    code = """
import autogen_agentchat
print(f"AutoGen version: {autogen_agentchat.__version__}")
print("✅ Environment working!")
"""
    
    result = await executor.execute_code_blocks(
        code_blocks=[CodeBlock(language="python", code=code)]
    )
    
    print(result.output)
    
    # 清理
    await executor.stop()

asyncio.run(main())
```

---

## 💡 完整 Agent 範例

### 使用 AssistantAgent 和 CodeExecutorAgent

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent, CodeExecutorAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor

async def main():
    # 1. 建立模型客戶端
    model_client = OpenAIChatCompletionClient(
        model="gpt-4o",
        # API key 從環境變數 OPENAI_API_KEY 讀取
    )
    
    # 2. 建立 Docker 程式碼執行器
    executor = DockerCommandLineCodeExecutor(
        image="surveyx-env:latest",
        work_dir="./workspace",
        init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",
        timeout=120,
        auto_remove=True,
        stop_container=True,
    )
    
    # 3. 建立 Assistant Agent
    assistant = AssistantAgent(
        "assistant",
        model_client=model_client,
        system_message="""You are a helpful coding assistant.
        Write Python code to solve user requests.
        The code will run in a surveyx conda environment with AutoGen installed.""",
    )
    
    # 4. 建立 Code Executor Agent
    code_executor_agent = CodeExecutorAgent(
        "code_executor",
        code_executor=executor,
    )
    
    # 5. 建立團隊
    team = RoundRobinGroupChat(
        participants=[assistant, code_executor_agent],
        max_turns=10,
    )
    
    # 6. 執行任務
    task = """
    請撰寫 Python 程式完成以下任務：
    1. 匯入 autogen_agentchat 函式庫
    2. 顯示版本號碼
    3. 建立一個簡單的加法函數
    4. 測試這個函數（5 + 3）
    """
    
    result = await team.run(task=task)
    
    # 7. 顯示結果
    print("\n" + "="*60)
    print("執行結果：")
    print("="*60)
    for message in result.messages:
        print(f"\n[{message.source}]:")
        print(message.content)
    
    # 8. 清理
    await executor.stop()
    await model_client.close()

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 🔧 init_command 詳解

### 為什麼需要 init_command？

Docker 容器預設使用 `/bin/sh`，**不會自動啟動 conda 環境**。

**執行流程：**
```
容器啟動
    ↓
執行 ENTRYPOINT (/home/appuser/entrypoint.sh)
    ↓
AutoGen 呼叫容器執行程式碼
    ↓
執行 init_command (啟動 conda 環境)  ← 關鍵步驟
    ↓
執行實際的 Python 程式碼
    ↓
回傳結果
```

### 三種 init_command 方案

**方案 A：標準 conda activate（推薦）**
```python
init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx"
```
- ✅ 完整啟動 conda 環境
- ✅ 所有環境變數正確設定
- ⚠️ 每次執行耗時約 0.5 秒

**方案 B：直接指定 Python 路徑（最快）**
```python
init_command="export PATH=/opt/conda/envs/surveyx/bin:$PATH"
```
- ✅ 執行最快（幾乎無延遲）
- ❌ 某些環境變數可能未設定
- ⚠️ 不適用於需要完整環境的套件

**方案 C：使用 conda run（不推薦）**
```python
init_command="conda run -n surveyx"
```
- ❌ 執行較慢（約 1 秒）
- ⚠️ 可能有額外的 shell 層級問題

**參考來源：**
- [Conda 非互動式啟動](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#activating-an-environment)

---

## 📂 work_dir 掛載說明

### 掛載映射關係

```python
executor = DockerCommandLineCodeExecutor(
    work_dir="./my_project",  # 主機端路徑
    ...
)
```

**映射示意：**
```
主機端 (Host)                     容器端 (Container)
./my_project/                  →  /workspace/
./my_project/data/file.csv     →  /workspace/data/file.csv
./my_project/output/result.txt →  /workspace/output/result.txt
```

### 相對路徑 vs 絕對路徑

```python
# 相對路徑（相對於執行 Python 腳本的目錄）
work_dir="./workspace"

# 絕對路徑（明確指定）
work_dir="/Users/you/projects/my_project"
```

### 權限注意事項

容器內以 `appuser` 身份執行，需要：
- ✅ 主機端目錄有讀寫權限
- ⚠️ 如遇 `Permission denied`，執行：`chmod -R 755 ./workspace`

---

## 🧪 測試與驗證

### 測試掛載功能

```python
import asyncio
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor
from autogen_core.code_executor import CodeBlock

async def test_mount():
    executor = DockerCommandLineCodeExecutor(
        image="surveyx-env:latest",
        work_dir="./test_workspace",
        init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",
    )
    
    code = """
import os
print(f"當前目錄: {os.getcwd()}")
print(f"目錄內容: {os.listdir('.')}")

# 寫入測試檔案
with open('test_output.txt', 'w') as f:
    f.write('Hello from container!')
print("✅ 檔案寫入成功")
"""
    
    result = await executor.execute_code_blocks(
        code_blocks=[CodeBlock(language="python", code=code)]
    )
    print(result.output)
    
    await executor.stop()

# 執行測試
asyncio.run(test_mount())
```

**驗證：**
```bash
# 檢查生成的檔案
cat test_workspace/test_output.txt
# 應輸出: Hello from container!
```

---

## 📋 參數完整說明

### DockerCommandLineCodeExecutor 參數

| 參數 | 類型 | 預設值 | 說明 |
|------|------|-------|------|
| `image` | str | **必填** | Docker image 名稱 |
| `work_dir` | str | `"."` | 主機端工作目錄（掛載到 /workspace） |
| `init_command` | str | `None` | 每次執行前的初始化命令 |
| `timeout` | int | `60` | 執行超時（秒） |
| `auto_remove` | bool | `True` | 執行後自動刪除容器 |
| `stop_container` | bool | `True` | 停止時清理容器 |
| `container_name` | str | 自動生成 | 容器名稱（可選） |

**範例：完整參數**
```python
executor = DockerCommandLineCodeExecutor(
    image="surveyx-env:latest",
    work_dir="/Users/you/projects/my_project",
    init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",
    timeout=300,  # 5 分鐘
    auto_remove=True,
    stop_container=True,
    container_name="surveyx-executor-001",
)
```

---

## 🔍 除錯技巧

### 檢視容器執行日誌

```python
# 設定 auto_remove=False 保留容器
executor = DockerCommandLineCodeExecutor(
    image="surveyx-env:latest",
    work_dir="./workspace",
    init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",
    auto_remove=False,  # 保留容器以便除錯
)

# 執行程式碼後...
# 在終端機查看容器日誌
# docker ps -a  # 找到容器 ID
# docker logs <container_id>
```

### 手動測試 init_command

```bash
# 啟動容器
docker run -it --rm surveyx-env:latest bash

# 在容器內測試 init_command
$ source /opt/conda/etc/profile.d/conda.sh
$ conda activate surveyx
$ python -c "import autogen_agentchat; print(autogen_agentchat.__version__)"
```

---

## 📚 相關文檔

- [← 返回主指南](./README.md)
- [環境檔案準備](./environment-setup.md)
- [Dockerfile 詳解](./dockerfile-explained.md)
- [疑難排解](./troubleshooting.md)

**參考來源：**
- [AutoGen 官方文檔](https://microsoft.github.io/autogen/dev/)
- [Code Executors 指南](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html)
- [Docker Code Executor](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/docker-executor.html)
