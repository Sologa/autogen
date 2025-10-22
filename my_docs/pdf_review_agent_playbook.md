# PDF 格式審查代理「落地指南」

> 本指南將 `my_docs/pdf_review_agent_spec.md` 內的設計轉化為可實作的步驟，並補充實際匯入路徑、程式片段與 Docker 沙箱要求。若需建置 Docker 環境，請先完成 `my_docs/DOCKER_BUILD_GUIDE.md`。

---

## 0. 先決條件

1. **Python 環境**：建議使用 Conda，確保已安裝 `fitz`（PyMuPDF）、`autogen-*` 套件、`pydantic`、`rich` 等必要函式庫。
2. **Docker 沙箱**：依 `my_docs/DOCKER_BUILD_GUIDE.md` 建置 `surveyx-conda-env:latest`，供 `DockerCommandLineCodeExecutor` 使用；建置完成的映像已備份為 `/Volumes/My Book/surveyx_data/docker_build/surveyx-env-latest.tar`，需時可用 `docker load -i "/Volumes/My Book/surveyx_data/docker_build/surveyx-env-latest.tar"` 還原。
3. **資料結構**：準備好 PDF 根目錄 `ROOT_PATH` 以及最終輸出資料夾（預設為 `$ROOT_PATH` 底下）。
4. **規則表**：建立 `rules.yaml`（對應 spec §4.1 的 TODO）。範例架構如下：
   ```yaml
   - id: CITATION_PLACEHOLDER
     description: "引用標記仍為 [?]"
     severity: "高"
     examples_ok:
       - "請參考 Fig. 1."
     examples_ng:
       - "[? ]"
   ```

---

## 1. 專案結構建議

```
project_root/
├─ pdf_tools/
│  ├─ __init__.py
│  └─ extractor.py        # 專業工具（3.1.2）
├─ tools/
│  ├─ shell.py            # 基礎工具（3.1.1）
│  └─ aggregator.py       # aggregate_findings
├─ agents/
│  ├─ specialist.py
│  └─ orchestrator.py
├─ main.py
├─ rules.yaml
└─ my_docs/...
```

---

## 2. 工具實作

### 2.1 PDF 專業工具 (`pdf_tools/extractor.py`)

```python
from __future__ import annotations

from dataclasses import dataclass
from pathlib import Path
from typing import List

import fitz  # PyMuPDF

from tools.shell import execute_shell

TMP_DIR = Path("tmp_pdf_assets")
TMP_DIR.mkdir(parents=True, exist_ok=True)


@dataclass
class LineSpan:
    line_no: str
    text: str


def get_pdf_page_count(pdf_path: str) -> int:
    """回傳 PDF 的總頁數。"""
    doc = fitz.open(pdf_path)
    try:
        return doc.page_count
    finally:
        doc.close()


def get_text_from_page(pdf_path: str, page_number: int) -> str:
    """
    透過 shell 命令呼叫 pdftotext 擷取指定頁面文字，並使用 nl 加上模擬行號。
    """
    command = f"pdftotext -f {page_number} -l {page_number} {pdf_path} - | nl -ba"
    result = execute_shell(command)
    if result["returncode"] != 0:
        raise RuntimeError(f"pdftotext failed: {result['stderr']}")
    return result["stdout"]


def get_text_spans(pdf_path: str, page_number: int) -> List[LineSpan]:
    """
    若需要更細緻的座標資訊，可改用 PyMuPDF 取得文字區塊並回傳 LineSpan。
    """
    doc = fitz.open(pdf_path)
    try:
        page = doc.load_page(page_number)
        spans: List[LineSpan] = []
        line_index = 0
        for block in page.get_text("dict")["blocks"]:
            for line in block.get("lines", []):
                line_index += 1
                text = "".join(span.get("text", "") for span in line.get("spans", []))
                spans.append(LineSpan(line_no=f"L{line_index}", text=text.strip()))
        return spans
    finally:
        doc.close()


def extract_images_from_page(pdf_path: str, page_number: int) -> List[str]:
    """
    匯出指定頁面的所有圖片至 TMP_DIR，回傳圖片檔案路徑列表。
    """
    doc = fitz.open(pdf_path)
    exported: List[str] = []
    try:
        page = doc.load_page(page_number)
        for index, (xref, *_rest) in enumerate(page.get_images(full=True)):
            image = doc.extract_image(xref)
            output_path = TMP_DIR / f"page_{page_number}_img_{index}.png"
            with output_path.open("wb") as fw:
                fw.write(image["image"])
            exported.append(str(output_path))
        return exported
    finally:
        doc.close()


def cleanup_temp_assets() -> None:
    """清除 TMP_DIR 下的所有暫存檔。"""
    for path in TMP_DIR.glob("*"):
        path.unlink(missing_ok=True)
```

> **引用來源**：工具組合對應規格 §3.1.2 的要求；PyMuPDF 介面參考 [https://pymupdf.readthedocs.io/en/latest/document.html#Document.page_count](https://pymupdf.readthedocs.io/en/latest/document.html#Document.page_count) 與 [https://pymupdf.readthedocs.io/en/latest/page.html#Page.get_images](https://pymupdf.readthedocs.io/en/latest/page.html#Page.get_images)。`pdftotext` 使用方式參照 [https://www.manpagez.com/man/1/pdftotext/](https://www.manpagez.com/man/1/pdftotext/)；依據 `docs/design/01 - Programming Model.md` 需以工具取得資料。

### 2.2 聚合工具 (`tools/aggregator.py`)

```python
import json
from pathlib import Path
from typing import List, Dict, Any


def aggregate_findings(directory: str) -> str:
    root = Path(directory)
    findings: List[Dict[str, Any]] = []
    for file in sorted(root.glob("p*.jsonl")):
        with file.open("r", encoding="utf-8") as fr:
            for line in fr:
                findings.append(json.loads(line))
    return json.dumps({"issues": findings}, ensure_ascii=False, indent=2)


def persist_final_report(directory: str, output_path: str) -> str:
    """
    聚合後將結果寫入 output_path，回傳輸出檔案的實際路徑。
    """
    payload = aggregate_findings(directory)
    path = Path(output_path)
    path.write_text(payload, encoding="utf-8")
    return str(path.resolve())
```

> **引用來源**：聚合與輸出流程對應規格 §3.1.1／§3.2.2；`persist_final_report` 以 Python 原生 I/O 落實 `$ROOT_PATH/final_report.json` 需求。[my_docs/pdf_review_agent_spec.md:31](my_docs/pdf_review_agent_spec.md#L31)・[my_docs/pdf_review_agent_spec.md:80](my_docs/pdf_review_agent_spec.md#L80)

### 2.3 Shell 執行工具 (`tools/shell.py`)

```python
import subprocess
from typing import Dict


def execute_shell(command: str) -> Dict[str, str]:
    """
    預設以 subprocess 直接執行指令；若需沙箱可改為調用 DockerCommandLineCodeExecutor。
    """
    proc = subprocess.run(
        command,
        shell=True,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True,
        check=False,
    )
    return {"stdout": proc.stdout, "stderr": proc.stderr, "returncode": proc.returncode}
```

若需以 Docker 沙箱包裝，可建立額外的輔助函式：

```python
from autogen_core import CancellationToken
from autogen_core.code_executor import CodeBlock
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor


async def execute_shell_with_docker(command: str, /, *, work_dir: str, image: str) -> Dict[str, str]:
    """
    以 DockerCommandLineCodeExecutor 執行 shell 指令；可直接交給 FunctionTool (支援 coroutine) 使用。
    """
    async with DockerCommandLineCodeExecutor(image=image, work_dir=work_dir) as executor:
        result = await executor.execute_code_blocks(
            [CodeBlock(code=command, language="bash")],
            cancellation_token=CancellationToken(),
        )
        return {"stdout": result.output, "stderr": "", "returncode": result.exit_code}
```

> **引用來源**：Docker 執行器生命週期詳見 [python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py:495](python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L495)；AutoGen 手冊列出的執行器清單於 [AUTOGEN_USER_MANUAL.md:397](AUTOGEN_USER_MANUAL.md#L397)。

---

## 3. Specialist Agent

### 3.1 工具註冊

利用 `autogen_core.tools.FunctionTool` 將上述函式包裝成 Agent 能使用的工具：

```python
from autogen_core.tools import FunctionTool
from pdf_tools.extractor import (
    cleanup_temp_assets,
    extract_images_from_page,
    get_text_from_page,
)

get_text_tool = FunctionTool.from_defaults(
    fn=get_text_from_page,
    name="get_text_from_page",
    description="取得 PDF 指定頁面的文字並附帶模擬行號 (pdftotext + nl)",
)
get_images_tool = FunctionTool.from_defaults(
    fn=extract_images_from_page,
    name="extract_images_from_page",
    description="將 PDF 指定頁面的圖片匯出為檔案並回傳路徑列表",
)
cleanup_tool = FunctionTool.from_defaults(
    fn=cleanup_temp_assets,
    name="cleanup_temp_assets",
    description="清除 tmp_pdf_assets 目錄下的暫存圖片",
)
```

### 3.2 Agent 定義 (`agents/specialist.py`)

```python
from typing import Sequence, Dict, Any

from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.messages import TextMessage
from autogen_core.models import ChatCompletionClient


def build_specialist(model_client: ChatCompletionClient, rules: Sequence[Dict[str, Any]]) -> AssistantAgent:
    """
    Specialist 只持有 PDF 專業工具，負責逐頁分析文字與視覺內容。
    """
    system_message = (
        "你是 PDF 排版稽核專家。請遵循規則表提供的條款，"
        "針對每一頁輸入生成 JSONL 格式的問題列表。必要時提取圖片並清除暫存。"
    )
    return AssistantAgent(
        name="pdf_specialist",
        model_client=model_client,
        tools=[get_text_tool, get_images_tool, cleanup_tool],
        system_message=system_message + f"\n規則表：{rules}",
        reflect_on_tool_use=True,
        model_client_stream=True,
    )
```

> **參考**：`python/packages/autogen-agentchat/src/autogen_agentchat/agents/_assistant_agent.py`（`AssistantAgent` 介面）。

---

## 4. Orchestrator Agent

### 4.1 Specialist 作為工具

```python
from autogen_agentchat.tools import AgentTool
from autogen_agentchat.agents import AssistantAgent
from autogen_core.tools import FunctionTool
from tools.shell import execute_shell
from tools.aggregator import aggregate_findings, persist_final_report
from pdf_tools.extractor import get_pdf_page_count
from agents.specialist import build_specialist


def build_orchestrator(
    model_client: ChatCompletionClient,
    specialist: AssistantAgent,
) -> AssistantAgent:
    specialist_tool = AgentTool(specialist, return_value_as_last_message=True)

    return AssistantAgent(
        name="pdf_orchestrator",
        model_client=model_client,
        tools=[
            FunctionTool.from_defaults(fn=get_pdf_page_count, name="get_pdf_page_count"),
            FunctionTool.from_defaults(fn=aggregate_findings, name="aggregate_findings"),
            FunctionTool.from_defaults(
                fn=persist_final_report,
                name="persist_final_report",
                description="將 $ROOT_PATH 下的 JSONL 聚合後寫入 final_report.json",
            ),
            FunctionTool.from_defaults(fn=execute_shell, name="execute_shell"),
            specialist_tool,
        ],
        system_message=(
            "你是總指揮，負責查找所有 PDF，逐頁呼叫 review_page，"
            "驗證 Specialist 回報後寫出 JSONL。全部頁面完成後，"
            "呼叫 persist_final_report 將結果寫入 final_report.json，並視需要要求 Specialist 清理暫存圖片。"
        ),
        max_tool_iterations=5,
        reflect_on_tool_use=False,
    )
```

> **說明**：`AgentTool` 會在主代理內執行另一個 Agent，必須將 Orchestrator 模型的 `parallel_tool_calls` 設為 `False` 以避免狀態衝突；詳見 [python/packages/autogen-agentchat/src/autogen_agentchat/tools/_agent.py:20](python/packages/autogen-agentchat/src/autogen_agentchat/tools/_agent.py#L20)。`FunctionTool` 由 `autogen_core.tools._function_tool` 提供，允許將 Python 函式暴露為工具供 AssistantAgent 使用。[python/packages/autogen-core/src/autogen_core/tools/_function_tool.py:15](python/packages/autogen-core/src/autogen_core/tools/_function_tool.py#L15)

---

## 5. Main Script (`main.py`)

```python
from pathlib import Path

from autogen_core import AgentId, SingleThreadedAgentRuntime
from autogen_core.model_context import BufferedChatCompletionContext
from autogen_ext.models.openai import OpenAIChatCompletionClient

from agents.specialist import build_specialist
from agents.orchestrator import build_orchestrator
from pdf_tools.extractor import cleanup_temp_assets

ROOT_PATH = Path("/Users/xjp/Desktop/.../error_detection")


def load_rules(path: Path):
    import yaml

    with path.open("r", encoding="utf-8") as fr:
        return yaml.safe_load(fr)


async def main() -> None:
    orchestrator_client = OpenAIChatCompletionClient(model="gpt-5-mini", parallel_tool_calls=False)
    orchestrator_client.context = BufferedChatCompletionContext(buffer_size=10)
    specialist_client = OpenAIChatCompletionClient(model="gpt-5-nano", parallel_tool_calls=True)

    rules = load_rules(Path("rules.yaml"))
    specialist = build_specialist(specialist_client, rules)
    orchestrator = build_orchestrator(orchestrator_client, specialist)

    runtime = SingleThreadedAgentRuntime()
    await specialist.register_instance(runtime, AgentId("pdf_specialist", "default"))
    await orchestrator.register_instance(runtime, AgentId("pdf_orchestrator", "default"))
    runtime.start()

    task = (
        f"根目錄位於 {ROOT_PATH}。"
        "1. 找出所有 pdf\n"
        "2. 逐頁呼叫 review_page 並將結果寫入 p{{page}}.jsonl\n"
        "3. 所有頁面完成後呼叫 persist_final_report(directory=str(ROOT_PATH), output_path=str(ROOT_PATH / 'final_report.json'))"
    )

    await orchestrator.run(task=task)
    await runtime.stop_when_idle()
    await orchestrator_client.close()
    await specialist_client.close()

    final_report = ROOT_PATH / "final_report.json"
    if not final_report.exists():
        raise FileNotFoundError(f"final_report.json 未被生成：{final_report}")
    cleanup_temp_assets()


if __name__ == "__main__":
    import asyncio

    asyncio.run(main())
```

> **依據**：Orchestrator 與 Specialist 的模型選擇 (`gpt-5-mini` / `gpt-5-nano`) 與並行設定對應規格 §3.2；模型代號可在 [python/packages/autogen-ext/src/autogen_ext/models/openai/_model_info.py:20](python/packages/autogen-ext/src/autogen_ext/models/openai/_model_info.py#L20) 查詢。`register_instance` 由 `BaseAgent` 提供，會在 Runtime 中設定訂閱與序列化器。[python/packages/autogen-core/src/autogen_core/_base_agent.py:198](python/packages/autogen-core/src/autogen_core/_base_agent.py#L198) `SingleThreadedAgentRuntime` 行為參照 [https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/framework/agent-and-agent-runtime.html](https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/framework/agent-and-agent-runtime.html)。

---

## 6. Docker 沙箱整合

若需對 `execute_shell` 進行沙箱化，可於專案初始化階段建立協程包裝：

```python
from functools import partial
from autogen_core.tools import FunctionTool
from tools.shell import execute_shell_with_docker

docker_shell_tool = FunctionTool.from_defaults(
    fn=partial(execute_shell_with_docker, work_dir=str(ROOT_PATH), image="surveyx-conda-env:latest"),
    name="execute_shell",
)
```

或於 Specialist 完成作業後呼叫 `cleanup_temp_assets()` 清除 Docker 掛載所生成的暫存檔。上述流程會在 `async with DockerCommandLineCodeExecutor(...)` 中自動啟動與關閉容器，符合規格要求的「開始→執行→捕捉→銷毀」生命週期。

---

## 7. 測試建議

1. **工具層級**：使用 pytest 撰寫 `test_extractor.py`、`test_aggregator.py`，對單頁輸出與聚合結果進行快照測試。
2. **Agent 層級**：利用 `autogen_ext.ui.RichConsole` 以串流方式觀察 Specialist 的輸出是否符合預期。
3. **端到端流程**：準備一份含已知格式錯誤的 PDF，驗證最終產出的 `p*.jsonl` 與 `final_report.json`。

---

## 8. 尚待補完的 TODO

| 編號 | 內容 |
| --- | --- |
| §3.1.2 | <span style="color:red">定義例外類別、行號演算法、暫存檔策略。</span> |
| §3.2.1 | <span style="color:red">補上規則表內容。</span> |
| §3.2.1 | <span style="color:red">確定模型型號、超參數與 Vision 支援策略。</span> |
| §4.2 | <span style="color:red">工具單元測試與錯誤結構。</span> |
| §4.3 | <span style="color:red">TaskResult.stop_reason 定義。</span> |
| §4.5 | <span style="color:red">重試機制與退避策略。</span> |
| §4.6 | <span style="color:red">測試資料集與 CI pipeline。</span> |
| §5   | <span style="color:red">最終報告版本控制與狀態保存策略。</span> |

---

完成以上步驟後，即可依據 `my_docs/pdf_review_agent_spec.md` 快速落實多代理 PDF 審查系統。若後續需要擴充 gRPC Worker Runtime、Human-in-the-loop 等進階功能，請依 spec 的 TODO 項目逐一補強。*** End Patch
