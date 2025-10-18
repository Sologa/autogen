# 指令與格式問題紀錄

## 指令紀錄
| 序號 | 指令 | 用途 | 結果與備註 |
| --- | --- | --- | --- |
| 1 | `ls` | 確認目前工作目錄內容 | 成功，確認僅有 `survey.pdf` |
| 2 | `pdftotext -f 5 -l 5 survey.pdf -` | 擷取第 5 頁文字 | 成功，取得文字內容以檢查格式 |
| 3 | `texdoc -l latex` | 查詢 LaTeX 文件以比對引用顯示行為 | 命令執行成功但進入互動式介面，未進一步開啟文件 |
| 4 | `echo done` | 結束互動式 `texdoc` 流程 | 成功 |
| 5 | `texdoc -l lshort` | 查詢 LaTeX short guide 以了解引用格式 | 命令成功但未進一步開啟文件 |
| 6 | `echo ok` | 結束互動式 `texdoc` 流程 | 成功 |
| 7 | `pdftotext /usr/local/texlive/2025/texmf-dist/doc/latex/lshort-english/lshort.pdf - \| rg "\\[\\?\\]" -n` | 嘗試在 LaTeX 官方文件中搜尋 `[?]` 範例 | 失敗，sandbox 阻擋對該路徑讀取 |
| 8 | `pdftotext -f 31 -l 31 survey.pdf -` | 擷取第 31 頁文字 | 成功 |
| 9 | `texdoc --machine latex` | 嘗試直接開啟 LaTeX 文件（非互動模式） | 失敗，sandbox 禁止執行 `open` |
| 10 | `pdftotext -f 57 -l 57 survey.pdf -` | 擷取第 57 頁文字 | 成功 |
| 11 | `texdoc hyphenation` | 查詢斷詞文件以佐證 Unicode 減號處理 | 失敗，sandbox 禁止開啟文件 |
| 12 | `pdftotext -f 63 -l 63 survey.pdf -` | 擷取第 63 頁文字 | 成功 |
| 13 | `pdftoppm -f 63 -l 63 -png survey.pdf page_63` | 產生第 63 頁 PNG 圖以檢視圖像、視覺異常 | 成功，輸出 `page_63-63.png` |
| 14 | （人工檢視 `page_63-63.png`） | 檢查圖 1 的節點顯示狀況 | 觀察到下半部節點全黑、紅框空白等異常 |
| 15 | `rm page_63-63.png` | 清除臨時輸出檔案 | 成功 |

## 格式問題彙整
- `survey.pdf:5`：多處 citation 顯示為 `[?]`、`[? ? ?]`，顯示參考文獻未解析或缺少 `bibtex`/`biber` 步驟。
- `survey.pdf:31`：保留 `[? ? …]` citation 佔位符，且圖表引用出現 `??tree_figure_6...`，顯示 `\ref{}` 未解析。
- `survey.pdf:57`：
  - citation 持續以 `[?]`、`[???]` 顯示；
  - 文字內頻繁出現 `text − to − token`、`barge− in` 等帶有 Unicode 減號且缺少空格的字串，導致斷詞與間距異常；
  - 句子如 `tokensviatext` 顯示空格遺失。
- `survey.pdf:63`：
  - 原應為 13.4 小節標題的位置顯示 `?? Bandwidth and edge–server trade-offs`，章節編號未成功解析；
  - 章節列表中缺少 `§13.4`，造成編號跳號；
  - 頁尾僅顯示 `Figure 1: chapter structure` 文字，未看到圖像或對應出處；
  - 視覺檢視圖 1 時，下半部節點全部呈現黑色無文字、帶紅框的節點內容空白，顯示圖像載入或渲染失敗，左右留白也因此失衡。

## 問題定位步驟補充
- 文字型格式問題：以 `pdftotext` 擷取指定頁面文字，逐行檢查引用與符號（對應指令序號 2、8、10、12）。
- 視覺型格式問題：先使用 `pdftoppm` 轉出第 63 頁 PNG，再人工檢視圖像內容確認圖 1 節點顯示異常（對應指令序號 13–15）。

## 待補驗（外部資料）
- `texdoc` 相關查詢因 sandbox 權限被拒，未能檢視官方文件；若需進一步佐證引用或 Unicode 減號處理，建議於具備權限的環境重新執行。
