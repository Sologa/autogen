# 環境檔案準備指南

> **目的：** 正確匯出 surveyx Conda 環境，避免常見陷阱

[← 返回主指南](./README.md)

---

## 🎯 核心概念

### 為什麼環境檔案如此重要？

Conda 環境檔案 (`environment.yml`) 是 Docker image 的**藍圖**。錯誤的環境檔案會導致：

- ❌ Docker 建置失敗
- ❌ Conda 環境建立錯誤  
- ❌ 依賴版本不一致
- ❌ 執行時缺少套件
- ❌ 跨平台相容性問題

### surveyx 環境的特殊性

surveyx 環境通常包含：
1. **Conda 套件** - Python、基礎函式庫
2. **pip 套件** - AutoGen、Docling、HuggingFace 等專案依賴
3. **系統依賴** - libspatialindex、poppler-utils 等

⚠️ **關鍵：** 必須同時處理 Conda 和 pip 依賴！

---

## 📋 三種匯出策略

根據您的需求選擇合適的策略：

### 策略 A：完整環境匯出（推薦新手）

**適用情況：** 您希望完全複製當前環境，包含所有隱式依賴

**優點：**
- ✅ 最完整，包含所有已安裝套件
- ✅ 包含 pip 部分（如果有）
- ✅ 一次性匯出

**缺點：**
- ❌ 包含平台特定的 build 字串
- ❌ 檔案較大
- ❌ 可能包含不必要的依賴

**步驟：**

```bash
# 1. 啟動環境
conda activate surveyx

# 2. 匯出環境（移除 build 字串和 prefix）
conda env export --no-builds | grep -v "^prefix:" > environment.yml

# 3. 驗證檔案
cat environment.yml
```

**生成的檔案範例：**
```yaml
name: surveyx
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.11.13
  - pip=25.2
  - numpy=1.24.3
  - pandas=2.0.3
  - pip:
      - autogen-agentchat==0.7.5
      - autogen-ext[openai]==0.7.5
      - docling==2.0.0
      # ... 其他 pip 套件
```

**參考來源：**
- [Conda env export 文檔](https://docs.conda.io/projects/conda/en/latest/commands/env/export.html)

---

### 策略 B：僅匯出明確安裝的套件（推薦進階使用者）

**適用情況：** 您希望建立精簡、跨平台的環境定義

**優點：**
- ✅ 檔案精簡，僅包含明確安裝的套件
- ✅ 跨平台相容性最佳
- ✅ 減少不必要的依賴

**缺點：**
- ❌ 可能遺漏隱式依賴
- ❌ **不包含 pip 安裝的套件**（需額外處理）

**步驟：**

```bash
# 1. 啟動環境
conda activate surveyx

# 2. 匯出明確安裝的 Conda 套件
conda env export --from-history --no-builds | grep -v "^prefix:" > environment.yml

# 3. ⚠️ 重要：額外匯出 pip 套件
pip freeze > requirements.txt
```

**⚠️ 關鍵步驟：** 因為 `--from-history` 不包含 pip 套件，必須額外執行 `pip freeze`

**在 Dockerfile 中使用：**

```dockerfile
# 複製環境檔案
COPY --chown=appuser:appuser environment.yml .
COPY --chown=appuser:appuser requirements.txt .

# 建立 Conda 環境
RUN mamba env create -f environment.yml && \
    mamba clean -afy && \
    conda init bash

# ⚠️ 安裝 pip 依賴
RUN /opt/conda/envs/surveyx/bin/pip install --no-cache-dir -r requirements.txt
```

**參考來源：**
- [Conda --from-history 文檔](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#exporting-an-environment-file-across-platforms)

---

### 策略 C：使用專案提供的環境檔案（推薦 SurveyX 使用者）

**適用情況：** 您的專案已提供官方環境快照（如 SurveyX）

**優點：**
- ✅ 與專案官方環境完全一致
- ✅ 已經過測試驗證
- ✅ 無需手動匯出

**缺點：**
- ❌ 可能包含本機路徑（需手動移除）
- ❌ 可能包含 macOS 專用套件（需處理）

**步驟（以 SurveyX 為例）：**

```bash
# 1. 複製專案提供的環境檔案
cp ../SurveyX/env/env-survey.yml ./environment.yml

# 2. ⚠️ 移除 prefix 行（如果存在）
sed -i '' '/^prefix:/d' environment.yml

# 3. 複製 pip 依賴清單
cp ../SurveyX/env/requirements-freeze.txt ./requirements.txt

# 4. ⚠️ 移除 macOS 專用套件（可選）
# 編輯 requirements.txt，移除或註解以下行：
# pyobjc-framework-Cocoa
# pyobjc-framework-Quartz
# pyobjc-core
```

**在 Dockerfile 中使用：**

```dockerfile
# 複製環境檔案
COPY --chown=appuser:appuser environment.yml .
COPY --chown=appuser:appuser requirements.txt .

# 建立 Conda 環境
RUN mamba env create -f environment.yml && \
    mamba clean -afy && \
    conda init bash

# 安裝 pip 依賴（過濾 macOS 專用套件）
RUN /opt/conda/envs/surveyx/bin/pip install --no-cache-dir \
    -r requirements.txt || true
```

**參考來源：**
- [SurveyX 環境文檔](../SurveyX/env/README.md)

---

## 🔧 pip 依賴處理

### 問題：為什麼需要特別處理 pip 套件？

Conda 和 pip 是**兩個獨立的套件管理器**：

- `conda env export` 會包含 pip 部分，**但僅限於完整匯出**
- `conda env export --from-history` **不會**包含 pip 套件
- surveyx 的核心依賴（AutoGen、Docling 等）**都是 pip 安裝的**

### 方案對比

| 方案 | Conda 套件 | pip 套件 | 複雜度 | 推薦度 |
|------|-----------|---------|--------|-------|
| **A. 完整匯出** | ✅ 完整 | ✅ 完整 | 🟢 簡單 | ⭐⭐⭐ |
| **B. 分開匯出** | ✅ 精簡 | ✅ 完整 | 🟡 中等 | ⭐⭐⭐⭐ |
| **C. 專案檔案** | ✅ 官方 | ✅ 官方 | 🟢 簡單 | ⭐⭐⭐⭐⭐ |

### 如何在 Dockerfile 中安裝 pip 依賴

**方式 1：使用 requirements.txt（推薦）**

```dockerfile
# 複製 pip 依賴清單
COPY --chown=appuser:appuser requirements.txt .

# 在 Conda 環境中安裝
RUN /opt/conda/envs/surveyx/bin/pip install --no-cache-dir -r requirements.txt
```

**方式 2：內嵌於 environment.yml**

```yaml
name: surveyx
channels:
  - conda-forge
dependencies:
  - python=3.11
  - pip=25.2
  - pip:
      - autogen-agentchat==0.7.5
      - autogen-ext[openai]==0.7.5
      - docling==2.0.0
```

此方式 `mamba env create` 會自動安裝 pip 部分。

---

## ⚠️ 常見陷阱與解決方案

### 陷阱 1: environment.yml 包含 prefix 行

**問題：**
```yaml
name: surveyx
channels:
  - conda-forge
dependencies:
  - python=3.11
prefix: /opt/anaconda3/envs/surveyx  # ❌ 問題行
```

**症狀：** Docker 建置時錯誤
```
CondaEnvironmentError: cannot locate environment at /opt/anaconda3/envs/surveyx
```

**原因：** `prefix:` 指向主機端路徑，在容器中無效

**解決方案：**
```bash
# 重新匯出並移除 prefix
conda env export --no-builds | grep -v "^prefix:" > environment.yml

# 或手動編輯移除該行
```

**參考來源：**
- [Conda 環境匯出文檔](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#exporting-an-environment-file)

---

### 陷阱 2: 包含平台特定的 build 字串

**問題：**
```yaml
dependencies:
  - python=3.11.13=h47qx0dl_1_cpython  # ❌ 包含 build 字串
```

**症狀：** 在不同架構上建置失敗

**解決方案：**
```bash
# 使用 --no-builds 參數
conda env export --no-builds | grep -v "^prefix:" > environment.yml
```

**正確格式：**
```yaml
dependencies:
  - python=3.11.13  # ✅ 僅版本號
```

**參考來源：**
- [Conda --no-builds 文檔](https://docs.conda.io/projects/conda/en/latest/commands/env/export.html)

---

### 陷阱 3: macOS 專用套件無法在 Linux 容器安裝

**問題：**
```txt
# requirements.txt
pyobjc-framework-Cocoa==10.1  # ❌ 僅 macOS
pyobjc-framework-Quartz==10.1  # ❌ 僅 macOS
```

**症狀：**
```
ERROR: Could not find a version that satisfies the requirement pyobjc-framework-Cocoa
```

**原因：** PyObjC 系列套件僅提供 macOS wheel，Linux 無法安裝

**解決方案 A：移除 macOS 專用套件**

```bash
# 編輯 requirements.txt，移除或註解
sed -i '' '/pyobjc/d' requirements.txt
```

**解決方案 B：條件化安裝（進階）**

```dockerfile
# 在 Dockerfile 中使用條件安裝
RUN /opt/conda/envs/surveyx/bin/pip install --no-cache-dir \
    -r requirements.txt 2>&1 | tee /tmp/pip-install.log || \
    (grep -v "pyobjc" requirements.txt > requirements-linux.txt && \
     /opt/conda/envs/surveyx/bin/pip install -r requirements-linux.txt)
```

**解決方案 C：使用 try-except（最簡單）**

```dockerfile
# 安裝時忽略錯誤（不推薦用於生產）
RUN /opt/conda/envs/surveyx/bin/pip install -r requirements.txt || true
```

**參考來源：**
- [PyObjC 官方文檔](https://pyobjc.readthedocs.io/)

---

### 陷阱 4: pip freeze 包含過多套件

**問題：** `pip freeze` 會匯出**所有** pip 安裝的套件，包含間接依賴

**症狀：** requirements.txt 超過 200 行，建置時間過長

**解決方案 A：使用 pipreqs（推薦）**

```bash
# 安裝 pipreqs
pip install pipreqs

# 分析專案實際使用的套件
pipreqs /path/to/your/project --force

# 這會生成精簡的 requirements.txt
```

**解決方案 B：手動篩選**

```bash
# 僅保留專案核心依賴
cat > requirements.txt << EOF
autogen-agentchat==0.7.5
autogen-ext[openai]==0.7.5
docling==2.0.0
# ... 其他明確需要的套件
EOF
```

**參考來源：**
- [pipreqs GitHub](https://github.com/bndr/pipreqs)

---

## 📝 完整工作流程範例

### 範例 1: 從零開始（策略 A）

```bash
# 步驟 1: 啟動環境
conda activate surveyx

# 步驟 2: 完整匯出（包含 pip）
conda env export --no-builds | grep -v "^prefix:" > environment.yml

# 步驟 3: 驗證檔案
cat environment.yml | head -20

# 步驟 4: 檢查是否包含 pip 部分
grep -A 5 "pip:" environment.yml

# 步驟 5: 建立 Dockerfile（見主指南）

# 步驟 6: 建置
docker build --platform linux/arm64 -t surveyx-env:latest .
```

### 範例 2: 精簡環境（策略 B）

```bash
# 步驟 1: 啟動環境
conda activate surveyx

# 步驟 2: 僅匯出明確安裝的 Conda 套件
conda env export --from-history --no-builds | grep -v "^prefix:" > environment.yml

# 步驟 3: 匯出 pip 套件
pip freeze > requirements.txt

# 步驟 4: 移除 macOS 專用套件（可選）
sed -i '' '/pyobjc/d' requirements.txt

# 步驟 5: 在 Dockerfile 中同時使用兩個檔案（見上文）

# 步驟 6: 建置
docker build --platform linux/arm64 -t surveyx-env:latest .
```

### 範例 3: 使用專案檔案（策略 C - SurveyX）

```bash
# 步驟 1: 複製專案環境檔案
cp ../SurveyX/env/env-survey.yml ./environment.yml
cp ../SurveyX/env/requirements-freeze.txt ./requirements.txt

# 步驟 2: 移除 prefix 行
sed -i '' '/^prefix:/d' environment.yml

# 步驟 3: 移除 macOS 專用套件
sed -i '' '/pyobjc/d' requirements.txt

# 步驟 4: 建置（使用兩階段 pip 安裝的 Dockerfile）
docker build --platform linux/arm64 -t surveyx-env:latest .
```

---

## ✅ 驗證檢查清單

建置前請確認：

### environment.yml 檢查

```bash
# ✅ 不包含 prefix 行
grep "^prefix:" environment.yml
# 應無輸出

# ✅ 不包含 build 字串
grep "=.*=.*=" environment.yml
# 應無輸出（或僅 pip 套件有 ==）

# ✅ 包含必要的套件
grep "python" environment.yml
grep "pip" environment.yml

# ✅ 如果使用策略 A，檢查是否有 pip 部分
grep -A 3 "pip:" environment.yml
```

### requirements.txt 檢查（如使用）

```bash
# ✅ 包含核心依賴
grep "autogen" requirements.txt

# ⚠️ 檢查是否包含 macOS 專用套件
grep "pyobjc" requirements.txt
# 如有輸出，考慮移除

# ✅ 檢查套件數量（建議 < 100）
wc -l requirements.txt
```

---

## 🎓 進階主題

### 使用 Conda-lock 確保完全可重現

```bash
# 安裝 conda-lock
conda install -c conda-forge conda-lock

# 生成 lock 檔案
conda-lock -f environment.yml -p linux-64

# 在 Dockerfile 中使用
# RUN mamba create --name surveyx --file conda-linux-64.lock
```

**參考來源：**
- [conda-lock 文檔](https://conda.github.io/conda-lock/)

### 處理私有套件

如果您的環境包含私有 PyPI 套件：

```dockerfile
# 在建置時傳入憑證
ARG PIP_INDEX_URL
ARG PIP_EXTRA_INDEX_URL

RUN --mount=type=secret,id=pip_conf \
    /opt/conda/envs/surveyx/bin/pip install -r requirements.txt
```

---

## 📚 相關文檔

- [← 返回主指南](./README.md)
- [系統要求](./system-requirements.md)
- [Dockerfile 詳解](./dockerfile-explained.md)
- [疑難排解](./troubleshooting.md)

**參考來源：**
- [Conda 環境管理](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)
- [跨平台環境分享](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#sharing-an-environment)
- [pip freeze 文檔](https://pip.pypa.io/en/stable/cli/pip_freeze/)
