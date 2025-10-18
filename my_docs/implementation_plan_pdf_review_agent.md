# PDF 格式審查 Agent 系統實作計畫

這份文件旨在規劃一個由 Orchestrator Agent 指揮 Specialist Agent，逐頁閱讀 PDF 論文並找出格式問題的系統。

## 1. 總覽 (Executive Summary)

本計畫的目標是建立一個自動化系統，用以審查 PDF 文件的格式。系統將由兩個核心 Agent 組成：

1.  **Orchestrator Agent**: 負責管理整個審查流程。它會接收 PDF 文件，將任務拆解（逐頁），分派給 Specialist Agent，並在最後匯總所有結果。
2.  **Specialist Agent**: 負責執行單一頁面的審查。它會接收一頁的內容、頁碼以及格式規則清單，然後找出該頁面中所有違反規則的地方，回報問題的描述與大致行號。

最終產出會是一份完整的 Markdown 或 JSON 報告，列出在文件各頁中發現的所有格式問題。

## 2. 核心組件 (Core Components)

### a. PDF 處理器 (PDF Processor)

這不是一個 Agent，而是一系列關鍵的輔助工具（Tools/Functions）。PDF 是一種複雜的格式，文字和圖片都是由帶有座標的物件組成。

-   **推薦工具**: `PyMuPDF (fitz)`。這個 Python 函式庫可以精確地提取文字塊、圖片及其在頁面上的座標。

### b. Specialist Agent

-   **職責**: 深度分析單一頁面的內容，包括文字和圖片。
-   **工具 (Tools)**: 它需要一組可以自主呼叫的 Python 函數，例如 `get_text_from_page` 和 `extract_images_from_page`。
-   **輸出**: 一個結構化的清單，包含該頁發現的所有問題。

### c. Orchestrator Agent

-   **職責**: 總指揮，管理端到端的流程。
-   **流程**: 接收高層次任務，自主規劃，委派分頁任務給 Specialist，並在最後匯總報告。

---

## 3. 互動模式探討 (參考用)

以下是幾種可能的做法，其中模式 A 和 B 作為架構演進的參考。

### 模式 A: 程序化迴圈 (參考用)

此模式在一個傳統的 Python 腳本中使用 `for` 迴圈，在迴圈中呼叫 Agent。流程直觀，但較不靈活，未能完全發揮 Agent 的自主性。

### 模式 B: Agent 驅動的工作流 (參考用)

此模式中，Orchestrator Agent 接收一個包含所有頁面內容的大任務，然後自主決定如何呼叫 Specialist。這比模式 A 更符合 Agent 範式，但初始資料準備繁瑣，且受限於 LLM 的上下文長度。

---

## 4. 進階架構 (模式 C) - 最終選用架構

這套架構賦予 Agent 最高的自主性，解決了長文件處理和多模態分析的挑戰，是你最終決定採用的方案。

核心思想是：**不要「餵」(push) 資料給 Agent，而是給他們獲取資料的「工具」(tools)，讓他們自己去「拉」(pull)。**

### a. 新的原子化工具 (Atomic Tools)

我們為 Agent 提供一系列精細的工具：

1.  **`get_pdf_page_count(pdf_path: str) -> int`**
    -   **功能**: 接收 PDF 路徑，回傳總頁數。
    -   **目的**: 讓 Orchestrator 能自主探測任務規模，進行規劃。

2.  **`get_text_from_page(pdf_path: str, page_number: int) -> str`**
    -   **功能**: 接收 PDF 路徑和頁碼，回傳那一頁的、**已經過 `PyMuPDF` 處理並加上模擬行號的**文字。
    -   **目的**: 讓 Agent 能分析頁面中的文字內容。

3.  **`extract_images_from_page(pdf_path: str, page_number: int) -> list[str]`**
    -   **功能**: 接收 PDF 路徑和頁碼，找到該頁所有圖片，將其以原始品質儲存為獨立檔案（例如 `page_5_img_0.png`），並回傳一個包含所有圖片檔案路徑的清單。
    -   **目的**: 讓 Agent 能夠「看見」並分析頁面中的視覺內容。

### b. 更智能的 Orchestrator 工作流程

Orchestrator 的工作流程變得高度自主：

1.  **接收高層次任務**: > "請審查位於 `/path/to/paper.pdf` 的文件，找出問題並匯總報告。"
2.  **自主規劃 (Planning)**: Orchestrator 呼叫 `get_pdf_page_count` 得知總頁數，並形成逐頁審查的計畫。
3.  **自主執行 (Execution)**: Orchestrator 在內部迴圈中，為每一頁向 Specialist Agent 發出指令：> "去審查 `/path/to/paper.pdf` 的第 N 頁"。
4.  **匯總 (Aggregation)**: Orchestrator 收集所有 Specialist 的回報，並整理成最終報告。

### c. 更強大的 Specialist 工作流程

Specialist 在接收到 Orchestrator 的指令後（例如審查第 5 頁），其工作流程如下：

1.  **接收指令**: > "去審查 `/path/to/paper.pdf` 的第 5 頁"。
2.  **自主獲取資料**: 它知道需要文字和圖片才能完成任務，於是自主呼叫它所擁有的工具：
    -   呼叫 `get_text_from_page(pdf_path, 5)` 來獲取文字內容。
    -   呼叫 `extract_images_from_page(pdf_path, 5)` 來獲取圖片檔案列表，例如 `['.../page_5_img_0.png']`。
3.  **多模態分析 (Multi-modal Analysis)**:
    -   **文字分析**: 將文字內容和文字格式規則傳遞給 LLM 進行分析。
    -   **圖片分析**: 將提取出的圖片檔案（例如 `page_5_img_0.png`）和圖片格式規則（例如「圖片必須有標題」、「解析度不能過低」）傳遞給一個**具備視覺能力的多模態 LLM (如 Gemini Pro Vision, GPT-4V)** 進行分析。
4.  **回報結果**: 將文字和圖片的分析結果整合後，回報給 Orchestrator。

---

## 5. 處理視覺內容：為何提取圖片，而非螢幕截圖？

在討論如何讓 Agent 「看見」圖片時，我們探討了「螢幕截圖」和「直接提取」兩種方案。最終，我們選擇了「直接提取」，因為它在可靠性和品質上具有壓倒性優勢。

-   **螢幕截圖 (不推薦)**
    -   **不可靠**: Agent 無法控制你的電腦螢幕，它不知道 PDF 閱讀器是否開啟、是否在正確的頁面、是否被其他視窗遮擋。這使得該方法極度不穩定。
    -   **品質低劣**: 截圖是點陣圖，會嚴重損害 PDF 中原始向量圖或高解析度圖片的品質，影響分析的準確性。

-   **直接提取 (推薦)**
    -   **可靠**: 直接從 PDF 這個「源頭」讀取資料，保證了每次都能精確獲取到目標頁面的正確圖片，不受任何外部環境干擾。
    -   **品質保證**: 使用 `PyMuPDF` 等工具可以直接匯出圖片的原始數據，無論是 PNG, JPG 還是其他格式，都能保持 100% 的原始品質，為後續的視覺分析提供最佳輸入。

---

## 6. 潛在問題與解決方案

1.  **PDF 內容提取不準確**: 對於掃描版、雙欄或複雜表格的 PDF，`PyMuPDF` 的提取仍可能出錯。應優先處理數位原生、結構簡單的 PDF。
2.  **"行號" 不精確**: 模擬的行號僅供參考，在報告中應同時提供問題文字的上下文以幫助定位。
3.  **格式規則定義模糊**: 規則必須非常具體、無歧義（例如，提供正例和反例），以便 LLM 能準確判斷。
4.  **多模態模型的使用**: 圖片分析需要呼叫具備視覺能力的 LLM API，這會影響成本和延遲，並需要處理圖片資料的傳輸。
5.  **成本與效率**: 對每一頁的文字和圖片都進行 LLM 呼叫，處理長篇論文可能會非常耗時且昂貴。可考慮引入快取或使用更快的模型進行初審。

## 7. 完整實作步驟建議

1.  **環境設定**: `pip install pyautogen pymupdf`。
2.  **開發 `pdf_tools.py`**: 在此檔案中實作 `get_pdf_page_count`, `get_text_from_page`, `extract_images_from_page` 這三個核心工具函式。
3.  **開發 `specialist_agent.py`**:
    -   建立一個 `UserProxyAgent` 或 `AssistantAgent` 作為 Specialist。
    -   將 `pdf_tools.py` 中的三個函式註冊為它的工具。
    -   確保在分析圖片時，有邏輯可以呼叫多模態 LLM API。
4.  **開發 `orchestrator.py`**:
    -   建立一個 `AssistantAgent` 作為 Orchestrator。
    -   編寫清晰的系統訊息，指導它使用 `get_pdf_page_count` 進行規劃，並委派任務給 Specialist。
5.  **開發主腳本 `main.py`**:
    -   初始化 Orchestrator 和 Specialist Agent。
    -   建立一個包含這兩個 Agent 的 `GroupChat` 或使用 `initiate_chat` 讓它們直接對話。
    -   給予 Orchestrator 最初的高層次任務，啟動整個審查流程。
