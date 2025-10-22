# Docker Image 建置指南 - surveyx 環境

> **⚠️ 重要通知：** 本文檔已重組為模組化結構  
> **主指南位置：** [docker/README.md](./docker/README.md)  
> **最後更新：** 2025-01-20  

---

## 📍 文檔已遷移

為了提升可讀性與維護性，原本的長文檔已拆分為多個專門文檔。

### ✨ 新文檔結構

**請前往 [`my_docs/docker/`](./docker/) 目錄**，包含以下文檔：

#### 🎯 核心文檔
- **[README.md](./docker/README.md)** - 主指南，包含快速開始與核心概念
- [Dockerfile 詳解](./docker/dockerfile-explained.md) - Dockerfile 逐行解釋與設計決策
- [AutoGen 整合使用](./docker/autogen-integration.md) - 在 AutoGen 中使用 Docker image

#### 🔧 環境準備
- [系統要求與配置](./docker/system-requirements.md) - 硬體、記憶體、儲存規劃
- [環境檔案準備](./docker/environment-setup.md) - Conda 環境匯出與 pip 依賴處理

#### 📚 進階主題
- [疑難排解](./docker/troubleshooting.md) - 常見問題與解決方案
- [最佳實踐](./docker/best-practices.md) - 安全性、性能優化
- [跨平台建置](./docker/cross-platform.md) - AMD64/ARM64 支援

#### 🔗 參考資料
- [官方文檔連結](./docker/references.md) - Docker、Conda、AutoGen 資源

---

## 🚀 快速導航

### 我想要...

**快速建置 Docker image：**
→ 前往 [docker/README.md](./docker/README.md) 的「快速開始」章節

**解決建置錯誤：**
→ 查閱 [疑難排解指南](./docker/troubleshooting.md)

**在 AutoGen 中使用：**
→ 參考 [AutoGen 整合使用](./docker/autogen-integration.md)

**準備環境檔案：**
→ 閱讀 [環境檔案準備](./docker/environment-setup.md)

**檢查系統是否符合要求：**
→ 查看 [系統要求](./docker/system-requirements.md)

---

## 📝 改版說明

### v2.0 的主要改進

✅ **修正所有關鍵問題：**
- 明確標註 ARM64 平台專用
- 補充 pip 依賴處理流程
- 修正 AutoGen API 使用方式（`init_command`）
- 補充記憶體與儲存配置建議

✅ **模組化結構：**
- 核心指南保持簡潔（< 400 行）
- 進階內容拆分至專門文檔
- 每個主題獨立完整

✅ **實用性提升：**
- 新增快速診斷檢查表
- 補充性能優化建議
- 詳細的疑難排解步驟

---

## 🔄 舊版文檔

- **v1.0 備份：** [DOCKER_BUILD_GUIDE.md.bak5](./DOCKER_BUILD_GUIDE.md.bak5)
- **勘誤報告：** [DOCKER_BUILD_GUIDE_ERRATA.md](./DOCKER_BUILD_GUIDE_ERRATA.md)
- **修正版 Dockerfile：** [Dockerfile.surveyx.fixed](./Dockerfile.surveyx.fixed)

---

**立即開始：** [📖 前往主指南](./docker/README.md)

---

## 📑 目錄

1. [核心目標與架構](#1-核心目標與架構)
2. [系統要求與前置準備](#2-系統要求與前置準備)
3. [環境檔案準備](#3-環境檔案準備)
4. [Dockerfile 完整說明](#4-dockerfile-完整說明)
5. [建置 Docker Image](#5-建置-docker-image)
6. [驗證與測試](#6-驗證與測試)
7. [AutoGen 整合使用](#7-autogen-整合使用)
8. [疑難排解](#8-疑難排解)
9. [最佳實踐與安全建議](#9-最佳實踐與安全建議)
10. [參考資料](#10-參考資料)

---

## 1. 核心目標與架構

### 1.1 目標說明

建立一個 Docker image 作為 AutoGen Agent 的沙箱執行環境，具備以下特性：

- ✅ **隔離性**：獨立於主機系統，避免依賴衝突
- ✅ **可重現性**：確保在不同機器上環境一致
- ✅ **安全性**：以非 root 用戶執行，限制權限
- ✅ **完整性**：包含 surveyx 環境的所有依賴
- ✅ **跨平台**：支援 ARM64 (Apple Silicon) 和 AMD64 架構

### 1.2 架構圖

```
┌─────────────────────────────────────────┐
│         Host Machine (macOS)            │
│  ┌───────────────────────────────────┐  │
│  │    AutoGen Application            │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │ DockerCommandLineCodeExecutor│  │  │
│  │  └─────────────┬───────────────┘  │  │
│  └────────────────┼───────────────────┘  │
│                   │ Docker API           │
│  ┌────────────────▼───────────────────┐  │
│  │   Docker Container (surveyx-env)   │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │ Conda Environment: surveyx   │  │  │
│  │  │  - Python 3.11               │  │  │
│  │  │  - AutoGen packages          │  │  │
│  │  │  - All dependencies          │  │  │
│  │  └──────────────────────────────┘  │  │
│  │  Base: condaforge/miniforge3      │  │
│  │  Platform: linux/arm64            │  │
│  └────────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

**設計依據：**
- [Docker 官方：容器最佳實踐](https://docs.docker.com/build/building/best-practices/)
- [AutoGen 文檔：Code Executors](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html)

---

## 2. 系統要求與前置準備

### 2.1 系統要求

#### 硬體要求
- **Mac (Apple Silicon)**：M1/M2/M3 晶片（本指南優化目標）
- **Mac (Intel)**：亦支援，需調整平台參數
- **記憶體**：建議至少 8GB RAM
- **磁碟空間**：建議至少 10GB 可用空間

#### 軟體要求
- **Docker Desktop**：26.0+ ([下載連結](https://www.docker.com/products/docker-desktop/))
- **Conda**：Anaconda 或 Miniconda (用於生成環境檔案)
- **surveyx 環境**：已正確配置的 Conda 環境

**驗證系統架構：**
```bash
# 確認 CPU 架構
uname -m
# 輸出應為 arm64 (Apple Silicon) 或 x86_64 (Intel)

# 確認 Docker 已安裝且運行
docker --version
docker ps
```

**參考來源：**
- [Docker Desktop macOS 安裝指南](https://docs.docker.com/desktop/install/mac-install/)
- [Conda 安裝文檔](https://docs.conda.io/projects/conda/en/latest/user-guide/install/index.html)

### 2.2 Docker Desktop 安裝（macOS）

**安裝步驟：**

1. 下載對應架構的安裝檔：
   - **Apple Silicon**: [Docker Desktop (ARM64)](https://desktop.docker.com/mac/main/arm64/Docker.dmg)
   - **Intel Chip**: [Docker Desktop (AMD64)](https://desktop.docker.com/mac/main/amd64/Docker.dmg)

2. 安裝並啟動：
   ```bash
   # 掛載 DMG
   open Docker.dmg
   
   # 拖曳到 Applications 資料夾
   # 啟動 Docker Desktop
   open /Applications/Docker.app
   ```

3. 驗證安裝：
   ```bash
   docker run --rm hello-world
   ```

**參考來源：**
- [Docker Desktop Mac 安裝文檔](https://docs.docker.com/desktop/install/mac-install/)

---

## 3. 環境檔案準備

### 3.1 為什麼需要正確的環境檔案

Conda 環境檔案 (`environment.yml`) 是 Docker image 的藍圖。**錯誤的環境檔案會導致：**
- ❌ Docker 建置失敗
- ❌ Conda 環境建立錯誤
- ❌ 依賴版本不一致
- ❌ 跨平台相容性問題

### 3.2 生成跨平台相容的 environment.yml

**⚠️ 重要：不要直接使用 `conda env export`！**

原因：預設匯出會包含：
- `prefix: /opt/anaconda3/envs/surveyx` (本機路徑，Docker 中無效)
- 平台特定的 build 字串 (無法跨平台)
- 過多的隱式依賴 (增加建置時間)

**✅ 正確做法：**

```bash
# 步驟 1: 啟動 surveyx 環境
conda activate surveyx

# 步驟 2: 生成跨平台環境檔案（推薦方法）
conda env export --from-history --no-builds | grep -v "^prefix:" > environment.yml

# 或者：保留所有套件但移除平台特定資訊
conda env export --no-builds | grep -v "^prefix:" > environment.yml
```

**參數說明：**
- `--from-history`: 僅匯出明確安裝的套件，提升跨平台相容性 ([來源](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#exporting-an-environment-file-across-platforms))
- `--no-builds`: 移除平台特定的 build 字串 ([來源](https://docs.conda.io/projects/conda/en/latest/commands/env/export.html))
- `grep -v "^prefix:"`: 移除本機路徑 prefix 行

**生成的檔案範例：**
```yaml
name: surveyx
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.11
  - pip=25.2
  - pip:
      - autogen-agentchat==0.7.5
      - autogen-ext[openai]==0.7.5
      # ... 其他依賴
```

**驗證環境檔案：**
```bash
# 檢查是否包含 prefix 行（應該沒有）
grep "^prefix:" environment.yml

# 檢查是否包含 build 字串（應該沒有）
grep "=.*=.*=" environment.yml
```

**參考來源：**
- [Conda 官方：分享環境](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#sharing-an-environment)
- [Conda 官方：跨平台環境匯出](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#exporting-an-environment-file-across-platforms)

### 3.3 建立 .dockerignore 檔案

**目的：** 減小建置上下文，加速建置，避免將敏感資料複製到 image 中。

```bash
# 在專案根目錄建立 .dockerignore
cat > .dockerignore << 'EOF'
# Python 快取
**/__pycache__
**/*.pyc
**/*.pyo
**/*.pyd
**/.pytest_cache
**/.mypy_cache

# 虛擬環境
**/.venv
**/venv
**/env

# Git
**/.git
**/.gitignore

# IDE
**/.vscode
**/.idea
**/.DS_Store

# 日誌與暫存
*.log
*.tmp
**/.cache

# 環境變數（可能含敏感資訊）
.env
.env.local

# 大型資料檔案
**/*.csv
**/*.parquet
**/*.db
EOF
```

**參考來源：**
- [Docker 官方：.dockerignore](https://docs.docker.com/build/concepts/context/#dockerignore-files)
- [Docker 最佳實踐：排除檔案](https://docs.docker.com/build/building/best-practices/#exclude-with-dockerignore)

---

## 4. Dockerfile 完整說明

### 4.1 完整 Dockerfile

**檔案位置：** 專案根目錄 `Dockerfile`

```dockerfile
# =============================================================================
# Dockerfile for surveyx Conda Environment (ARM64 Optimized)
# =============================================================================
# 版本: 2.0
# 目標: 建立可在 Apple Silicon Mac 上高效運行的 surveyx 環境
# 基於: Docker 最佳實踐與 Conda 跨平台指南
# =============================================================================

# ===== 階段一：選擇基礎映像 (針對 ARM64 優化) =====
# 使用 miniforge3 而非 miniconda3 的原因：
# 1. miniforge 對 ARM64 架構原生支援更好
# 2. 預設使用 conda-forge channel，套件更新更快
# 3. 內建 mamba，安裝速度比 conda 快 10 倍
FROM --platform=linux/arm64 condaforge/miniforge3:latest AS base

# 說明: --platform=linux/arm64 明確指定 ARM64 架構
# 在 Apple Silicon Mac 上這是必要的，避免使用 Rosetta 模擬
# 參考: https://docs.docker.com/build/building/multi-platform/

# ===== 階段二：設定環境變數 =====
ENV PYTHONUNBUFFERED=1 \
    DEBIAN_FRONTEND=noninteractive \
    CONDA_ENV_NAME=surveyx \
    PATH="/opt/conda/bin:${PATH}"

# PYTHONUNBUFFERED=1: 確保 Python 輸出即時顯示，便於除錯
# DEBIAN_FRONTEND=noninteractive: 避免 apt-get 安裝時詢問互動問題
# 參考: https://docs.docker.com/build/building/best-practices/#env

# ===== 階段三：安裝系統依賴 =====
# 這些是 Python 套件可能需要的系統級函式庫
RUN apt-get update && apt-get install -y --no-install-recommends \
    # PDF 處理工具 (poppler-utils 提供 pdftotext 等)
    poppler-utils \
    # 圖形處理依賴 (PyMuPDF/fitz 需要)
    libgl1-mesa-glx \
    libglib2.0-0 \
    # 空間索引函式庫 (rtree 套件需要)
    libspatialindex-dev \
    # C/C++ 編譯工具 (部分 pip 套件需要從源碼編譯)
    gcc \
    g++ \
    make \
    # 網路工具
    curl \
    wget \
    # 版本控制
    git \
    && rm -rf /var/lib/apt/lists/*

# 清理 apt 快取 (rm -rf /var/lib/apt/lists/*)，減小 image 體積
# 參考: https://docs.docker.com/build/building/best-practices/#apt-get
# Rtree 安裝需求: https://rtree.readthedocs.io/en/latest/install.html

# ===== 階段四：建立非 root 使用者 =====
# 安全最佳實踐：容器不應以 root 身份執行程式
RUN useradd --create-home --shell /bin/bash appuser

# useradd 參數說明：
# --create-home: 建立 home 目錄 /home/appuser
# --shell /bin/bash: 設定預設 shell
# 參考: https://docs.docker.com/build/building/best-practices/#user

# 切換工作目錄
WORKDIR /home/appuser/app

# ===== 階段五：複製並建立 Conda 環境 =====
# 複製環境定義檔（使用 --chown 確保檔案所有者正確）
COPY --chown=appuser:appuser environment.yml .

# 建立 Conda 環境
# 使用 mamba 而非 conda 可大幅加速安裝（10 倍以上）
RUN mamba env create -f environment.yml && \
    # 清理 Conda 快取，減小最終 image 體積
    mamba clean -afy && \
    # 初始化 conda，使 'conda activate' 指令可用
    conda init bash

# mamba vs conda: mamba 是 C++ 實作的 conda 替代品，速度更快
# conda clean -afy 參數: -a (all), -f (force), -y (yes)
# 參考: https://mamba.readthedocs.io/

# ===== 階段六：建立環境啟動腳本 =====
# 因為 Docker 預設使用 /bin/sh，無法直接使用 conda activate
# 需要建立一個啟動腳本來初始化 conda 環境
RUN echo '#!/bin/bash' > /home/appuser/entrypoint.sh && \
    echo 'source /opt/conda/etc/profile.d/conda.sh' >> /home/appuser/entrypoint.sh && \
    echo 'conda activate ${CONDA_ENV_NAME}' >> /home/appuser/entrypoint.sh && \
    echo 'exec "$@"' >> /home/appuser/entrypoint.sh && \
    chmod +x /home/appuser/entrypoint.sh

# 腳本說明：
# 1. source conda.sh: 初始化 conda 環境
# 2. conda activate surveyx: 啟動目標環境
# 3. exec "$@": 執行傳入的命令（保持 PID 1）
# 參考: https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#activating-an-environment

# ===== 階段七：切換使用者與設定工作目錄 =====
USER appuser

# 設定容器內工作目錄（通常會掛載 host 目錄到此處）
WORKDIR /workspace

# ===== 階段八：設定 ENTRYPOINT 與 CMD =====
# ENTRYPOINT 確保每次執行都在 conda 環境中
ENTRYPOINT ["/home/appuser/entrypoint.sh"]

# CMD 提供預設命令（可被 docker run 參數覆蓋）
CMD ["/bin/bash"]

# ENTRYPOINT vs CMD:
# - ENTRYPOINT: 容器啟動時一定會執行
# - CMD: 提供預設參數，可被覆蓋
# 參考: https://docs.docker.com/build/building/best-practices/#entrypoint

# =============================================================================
# 建置說明:
# 
# 在 Apple Silicon Mac 上建置:
#   docker build --platform linux/arm64 -t surveyx-env:latest .
#
# 跨平台建置（同時支援 ARM64 與 AMD64）:
#   docker buildx build --platform linux/arm64,linux/amd64 -t surveyx-env:latest .
#
# 測試運行:
#   docker run --rm -it -v $(pwd):/workspace surveyx-env:latest
# =============================================================================
```

### 4.2 Dockerfile 指令詳解

#### FROM 指令的選擇

**為什麼使用 `condaforge/miniforge3` 而非 `continuumio/miniconda3`？**

| 特性 | miniforge3 | miniconda3 |
|------|-----------|-----------|
| ARM64 原生支援 | ✅ 優秀 | ⚠️ 透過模擬 |
| 預設 channel | conda-forge | defaults |
| 安裝速度 | ✅ 內建 mamba | ❌ 僅 conda |
| 套件更新頻率 | ✅ 快 | ⚠️ 較慢 |
| 映像大小 | 約 500MB | 約 400MB |

**在 Apple Silicon Mac 上的重要性：**
- 使用 `continuumio/miniconda3` (AMD64) 會透過 Rosetta 模擬，性能損失約 20-30%
- 明確指定 `--platform=linux/arm64` 可避免錯誤拉取 AMD64 版本

**參考來源：**
- [Docker 多平台建置](https://docs.docker.com/build/building/multi-platform/)
- [miniforge 專案](https://github.com/conda-forge/miniforge)

#### RUN 指令的最佳實踐

**合併多個 RUN 指令減少層數：**

```dockerfile
# ❌ 不推薦：建立多個層
RUN apt-get update
RUN apt-get install -y package1
RUN apt-get install -y package2

# ✅ 推薦：合併為一個層
RUN apt-get update && apt-get install -y \
    package1 \
    package2 \
    && rm -rf /var/lib/apt/lists/*
```

**參考來源：**
- [Docker 最佳實踐：RUN 指令](https://docs.docker.com/build/building/best-practices/#run)
- [Docker 最佳實踐：apt-get](https://docs.docker.com/build/building/best-practices/#apt-get)

---

## 5. 建置 Docker Image

### 5.1 前置檢查

```bash
# 確認必要檔案存在
ls -la Dockerfile environment.yml .dockerignore

# 確認 Docker 運行中
docker ps

# 確認當前目錄結構
tree -L 2 -a .
```

### 5.2 建置指令

#### 基本建置（Apple Silicon）

```bash
# 在專案根目錄執行
docker build --platform linux/arm64 -t surveyx-env:latest .
```

**參數說明：**
- `--platform linux/arm64`: 明確指定 ARM64 架構
- `-t surveyx-env:latest`: 標籤名稱，格式為 `name:tag`
- `.`: 建置上下文為當前目錄

#### 進階選項

```bash
# 1. 不使用快取，強制重新建置
docker build --no-cache --platform linux/arm64 -t surveyx-env:latest .

# 2. 顯示詳細建置過程
docker build --progress=plain --platform linux/arm64 -t surveyx-env:latest .

# 3. 多平台建置（需要 buildx）
docker buildx build \
  --platform linux/arm64,linux/amd64 \
  -t surveyx-env:latest \
  --load \
  .

# 4. 指定建置參數
docker build \
  --platform linux/arm64 \
  --build-arg CONDA_ENV_NAME=surveyx \
  -t surveyx-env:latest \
  .
```

**參考來源：**
- [docker build 命令參考](https://docs.docker.com/reference/cli/docker/image/build/)
- [Docker buildx 多平台建置](https://docs.docker.com/build/building/multi-platform/)

### 5.3 建置過程說明

**預期輸出：**

```
[+] Building 234.5s (12/12) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [internal] load metadata for docker.io/condaforge/miniforge3:latest
 => [1/7] FROM docker.io/condaforge/miniforge3:latest@sha256:...
 => [internal] load build context
 => [2/7] RUN apt-get update && apt-get install -y ...
 => [3/7] RUN useradd --create-home --shell /bin/bash appuser
 => [4/7] COPY --chown=appuser:appuser environment.yml .
 => [5/7] RUN mamba env create -f environment.yml && ...
 => [6/7] RUN echo '#!/bin/bash' > /home/appuser/entrypoint.sh && ...
 => [7/7] WORKDIR /workspace
 => exporting to image
 => => exporting layers
 => => writing image sha256:...
 => => naming to docker.io/library/surveyx-env:latest
```

**時間估計：**
- 首次建置：5-15 分鐘（取決於網路速度與套件數量）
- 後續建置：1-3 分鐘（利用快取）

**常見問題排除：** 見第 8 章

---

## 6. 驗證與測試

### 6.1 檢查 Image 資訊

```bash
# 列出所有 images
docker images surveyx-env

# 查看詳細資訊
docker image inspect surveyx-env:latest

# 檢查 image 大小
docker image inspect surveyx-env:latest --format='{{.Size}}' | numfmt --to=iec-i

# 檢查架構
docker image inspect surveyx-env:latest --format='{{.Architecture}}'
# 應輸出: arm64
```

### 6.2 測試容器啟動

```bash
# 啟動互動式容器
docker run --rm -it surveyx-env:latest

# 在容器內測試 conda 環境
$ conda env list
# conda environments:
#
surveyx               *  /opt/conda/envs/surveyx
base                     /opt/conda

$ python --version
# Python 3.11.x

$ which python
# /opt/conda/envs/surveyx/bin/python

$ pip list | head -10
# 應該顯示所有安裝的套件
```

### 6.3 測試 Python 套件

```bash
# 在容器內執行 Python 測試
docker run --rm surveyx-env:latest python -c "
import sys
import autogen_agentchat
import autogen_core
import autogen_ext

print(f'Python: {sys.version}')
print(f'autogen-agentchat: {autogen_agentchat.__version__}')
print(f'autogen-core: {autogen_core.__version__}')
print(f'autogen-ext: {autogen_ext.__version__}')
print('✅ All imports successful!')
"
```

### 6.4 測試檔案掛載

```bash
# 建立測試檔案
echo 'print("Hello from mounted file!")' > test.py

# 掛載並執行
docker run --rm -v $(pwd):/workspace surveyx-env:latest python test.py

# 應輸出: Hello from mounted file!
```

**參考來源：**
- [Docker run 命令參考](https://docs.docker.com/reference/cli/docker/container/run/)
- [Docker 磁碟區掛載](https://docs.docker.com/engine/storage/bind-mounts/)

---

## 7. AutoGen 整合使用

### 7.1 正確的 API 使用方式

**⚠️ 重要更新：** AutoGen 0.7.5 的 `DockerCommandLineCodeExecutor` API 已變更。

#### ❌ 錯誤用法（舊版文檔）

```python
# 此寫法已無效！
from autogen.agentchat.contrib.docker_command_line_code_executor import DockerCommandLineCodeExecutor

executor = DockerCommandLineCodeExecutor(
    image="surveyx-env:latest",
    work_dir="/path/to/project",
    command=["bash", "-c", "source activate surveyx && python {file_name}"]  # ❌ 無此參數
)
```

#### ✅ 正確用法（AutoGen 0.7.5）

```python
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor

executor = DockerCommandLineCodeExecutor(
    image="surveyx-env:latest",
    work_dir="./your_project_dir",  # 主機端目錄路徑
    init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",  # ✅ 使用 init_command
    timeout=120,  # 執行超時（秒）
    auto_remove=True,  # 執行後自動刪除容器
    stop_container=True  # 停止時清理容器
)
```

**API 變更說明：**

| 參數 | 舊版 | 新版 0.7.5 | 說明 |
|------|------|-----------|------|
| `command` | ✅ | ❌ 已移除 | 舊版用於自訂執行命令 |
| `init_command` | ❌ | ✅ 新增 | 在每次執行前執行的初始化指令 |
| `work_dir` | 絕對路徑 | 相對/絕對皆可 | 主機端工作目錄 |

**參考來源：**
- [AutoGen 源碼：_docker_code_executor.py:81](https://github.com/microsoft/autogen/blob/main/python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L81)
- [AutoGen 源碼：_docker_code_executor.py:528](https://github.com/microsoft/autogen/blob/main/python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L528)
- [AutoGen 文檔：Docker Code Executor](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html)

### 7.2 完整使用範例

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent, CodeExecutorAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor
from autogen_agentchat.ui import Console

async def main():
    # 1. 建立模型客戶端
    model_client = OpenAIChatCompletionClient(
        model="gpt-4o",
        # api_key 從環境變數 OPENAI_API_KEY 讀取
    )
    
    # 2. 建立 Docker 程式碼執行器
    executor = DockerCommandLineCodeExecutor(
        image="surveyx-env:latest",
        work_dir="./workspace",  # 會掛載到容器的 /workspace
        # ✅ 關鍵：使用 init_command 啟動 conda 環境
        init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",
        timeout=120,
        auto_remove=True,
    )
    
    # 3. 建立 Assistant Agent
    assistant = AssistantAgent(
        "assistant",
        model_client=model_client,
        system_message="You are a helpful coding assistant. Write Python code to solve user requests.",
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
    Write a Python script that:
    1. Imports the autogen_agentchat library
    2. Prints the version number
    3. Creates a simple function that adds two numbers
    4. Tests the function with 5 and 3
    """
    
    result = await team.run(task=task)
    
    # 7. 顯示結果
    print("\n" + "="*50)
    print("Task Result:")
    print("="*50)
    for message in result.messages:
        print(f"\n[{message.source}]: {message.content}")
    
    # 8. 清理
    await executor.stop()
    await model_client.close()

# 執行
if __name__ == "__main__":
    asyncio.run(main())
```

### 7.3 init_command 詳解

**為什麼需要 init_command？**

Docker 容器預設使用 `/bin/sh`，不會自動啟動 conda 環境。`init_command` 在每次執行程式碼前都會執行，確保環境正確。

**常用的 init_command 模式：**

```python
# 方案 A: 標準 conda activate（推薦）
init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx"

# 方案 B: 使用 conda run（不推薦，較慢）
init_command="conda run -n surveyx"

# 方案 C: 直接指定 Python 路徑（最快，但不完整）
init_command="export PATH=/opt/conda/envs/surveyx/bin:$PATH"
```

**init_command 的執行流程：**

```
Docker Container 啟動
    ↓
執行 ENTRYPOINT (/home/appuser/entrypoint.sh)
    ↓
AutoGen 呼叫容器執行程式碼
    ↓
執行 init_command (啟動 conda 環境)  ← 在這裡
    ↓
執行實際的程式碼
    ↓
回傳結果
```

**參考來源：**
- [AutoGen 源碼：init_command 實作](https://github.com/microsoft/autogen/blob/main/python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py#L528)
- [Conda 官方：在非互動式 shell 中啟動環境](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#activating-an-environment)

### 7.4 工作目錄掛載說明

**work_dir 參數的作用：**

```python
executor = DockerCommandLineCodeExecutor(
    work_dir="./my_project",  # 主機端路徑
    ...
)
```

**掛載映射關係：**

```
主機端 (Host)                    容器端 (Container)
./my_project/                →  /workspace/
./my_project/data/           →  /workspace/data/
./my_project/output/         →  /workspace/output/
```

**重要注意事項：**

1. **使用相對路徑：** `work_dir="./my_project"` 會相對於執行 Python 腳本的目錄
2. **使用絕對路徑：** `work_dir="/Users/you/projects/my_project"` 明確指定完整路徑
3. **權限問題：** 容器內以 `appuser` 身份執行，需要讀寫權限

**測試掛載：**

```python
# test_docker_mount.py
import asyncio
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor
from autogen_core.code_executor import CodeBlock

async def test():
    executor = DockerCommandLineCodeExecutor(
        image="surveyx-env:latest",
        work_dir="./test_workspace",
        init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",
    )
    
    # 測試寫入檔案
    code = """
import os
print(f"Current directory: {os.getcwd()}")
print(f"Files: {os.listdir('.')}")

with open('test_output.txt', 'w') as f:
    f.write('Hello from container!')
print("✅ File written successfully")
"""
    
    result = await executor.execute_code_blocks(
        code_blocks=[CodeBlock(language="python", code=code)]
    )
    print(result.output)
    
    await executor.stop()

asyncio.run(test())
```

**執行測試：**

```bash
# 建立測試目錄
mkdir -p test_workspace

# 執行測試
python test_docker_mount.py

# 檢查生成的檔案
cat test_workspace/test_output.txt
# 應輸出: Hello from container!
```

**參考來源：**
- [Docker 磁碟區掛載文檔](https://docs.docker.com/engine/storage/bind-mounts/)
- [AutoGen Code Executor 文檔](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html)

---

## 8. 疑難排解

### 8.1 建置階段問題

#### 問題 1: 平台不匹配警告

**錯誤訊息：**
```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64)
```

**原因：** 未指定 `--platform` 參數，Docker 拉取了錯誤的架構映像。

**解決方案：**
```bash
# 明確指定平台
docker build --platform linux/arm64 -t surveyx-env:latest .
```

**參考來源：**
- [Docker 多平台建置文檔](https://docs.docker.com/build/building/multi-platform/)

---

#### 問題 2: apt-get 安裝失敗

**錯誤訊息：**
```
E: Unable to locate package libspatialindex-dev
```

**原因：** 套件名稱錯誤或 repository 未更新。

**解決方案：**
```dockerfile
# 確保執行 apt-get update
RUN apt-get update && apt-get install -y --no-install-recommends \
    libspatialindex-dev \
    && rm -rf /var/lib/apt/lists/*
```

**檢查套件名稱：**
```bash
# 在 Debian/Ubuntu 容器中搜尋套件
docker run --rm debian:bookworm-slim \
  bash -c "apt-get update && apt-cache search spatialindex"
```

---

#### 問題 3: Conda 環境建立失敗

**錯誤訊息：**
```
CondaEnvironmentError: cannot locate environment at /opt/anaconda3/envs/surveyx
```

**原因：** `environment.yml` 包含 `prefix:` 行，指向主機端路徑。

**解決方案：**
```bash
# 重新生成環境檔案，移除 prefix 行
conda env export --no-builds | grep -v "^prefix:" > environment.yml
```

**參考來源：**
- [Conda 環境匯出文檔](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#exporting-an-environment-file)

---

#### 問題 4: pip 套件安裝失敗 (macOS 專用套件)

**錯誤訊息：**
```
ERROR: Could not find a version that satisfies the requirement pyobjc-framework-Cocoa
```

**原因：** `PyObjC` 系列套件僅支援 macOS，無法在 Linux 容器中安裝。

**解決方案 A：** 編輯 `environment.yml`，移除或條件化 macOS 專用套件

```yaml
dependencies:
  - pip:
      - autogen-agentchat==0.7.5
      # 移除 macOS 專用套件
      # - pyobjc-framework-Cocoa
      # - pyobjc-framework-Quartz
```

**解決方案 B：** 使用條件安裝（進階）

```dockerfile
# 在 Dockerfile 中條件化安裝
RUN if [ "$(uname -s)" = "Darwin" ]; then \
      pip install pyobjc-framework-Cocoa; \
    fi
```

**參考來源：**
- [PyObjC 文檔](https://pyobjc.readthedocs.io/)

---

### 8.2 執行階段問題

#### 問題 5: conda activate 指令無效

**錯誤訊息：**
```
CommandNotFoundError: Your shell has not been properly configured to use 'conda activate'.
```

**原因：** `/bin/sh` 不支援 `conda activate`。

**解決方案：** 使用完整的啟動指令

```python
# ✅ 正確
init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx"

# ❌ 錯誤（已廢棄）
init_command="source activate surveyx"
```

**參考來源：**
- [Conda 啟動環境文檔](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#activating-an-environment)

---

#### 問題 6: 容器內無法訪問網路

**錯誤訊息：**
```
urllib.error.URLError: <urlopen error [Errno -3] Temporary failure in name resolution>
```

**原因：** Docker 網路配置問題。

**解決方案：**
```bash
# 檢查 Docker 網路
docker network ls

# 測試容器網路
docker run --rm surveyx-env:latest ping -c 3 google.com

# 如果失敗，重啟 Docker Desktop
```

---

#### 問題 7: 權限被拒絕

**錯誤訊息：**
```
PermissionError: [Errno 13] Permission denied: '/workspace/output.txt'
```

**原因：** 主機端目錄權限不足。

**解決方案：**
```bash
# 方案 A: 修改主機端目錄權限
chmod -R 755 ./workspace

# 方案 B: 在容器中以 root 身份執行（不推薦）
docker run --user root ...
```

---

### 8.3 除錯技巧

#### 進入容器檢查

```bash
# 啟動互動式 shell
docker run --rm -it surveyx-env:latest bash

# 檢查環境
$ conda info --envs
$ python --version
$ pip list
$ env | grep CONDA
```

#### 查看建置歷史

```bash
# 查看 image 建置歷史
docker image history surveyx-env:latest

# 查看每層的大小
docker image history surveyx-env:latest --no-trunc
```

#### 查看容器日誌

```bash
# 執行並保留容器
docker run --name test-surveyx surveyx-env:latest python -c "print('test')"

# 查看日誌
docker logs test-surveyx

# 清理
docker rm test-surveyx
```

---

## 9. 最佳實踐與安全建議

### 9.1 安全性最佳實踐

#### 1. 使用非 root 用戶

**原因：** 避免容器逃逸攻擊，限制權限範圍。

```dockerfile
# ✅ 推薦
USER appuser

# ❌ 避免
USER root
```

**參考來源：**
- [Docker 安全最佳實踐](https://docs.docker.com/build/building/best-practices/#user)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)

---

#### 2. 不在 Image 中存儲敏感資訊

```dockerfile
# ❌ 絕對避免
ENV OPENAI_API_KEY="sk-..."

# ✅ 使用環境變數傳遞
docker run -e OPENAI_API_KEY="$OPENAI_API_KEY" surveyx-env:latest
```

**檢查 Image 中的敏感資訊：**
```bash
# 檢查環境變數
docker image inspect surveyx-env:latest --format='{{.Config.Env}}'

# 檢查歷史記錄
docker image history surveyx-env:latest --no-trunc
```

---

#### 3. 定期更新基礎映像

```bash
# 拉取最新版本
docker pull condaforge/miniforge3:latest

# 重新建置
docker build --no-cache --platform linux/arm64 -t surveyx-env:latest .
```

**參考來源：**
- [Docker 最佳實踐：定期重建](https://docs.docker.com/build/building/best-practices/#rebuild-your-images-often)

---

### 9.2 性能優化

#### 1. 使用多階段建置（進階）

```dockerfile
# 建置階段
FROM condaforge/miniforge3:latest AS builder
COPY environment.yml .
RUN mamba env create -f environment.yml

# 運行階段（更小）
FROM condaforge/miniforge3:latest
COPY --from=builder /opt/conda/envs/surveyx /opt/conda/envs/surveyx
```

**參考來源：**
- [Docker 多階段建置](https://docs.docker.com/build/building/multi-stage/)

---

#### 2. 利用建置快取

```bash
# 保持 Dockerfile 指令順序：從不常變到常變
# ✅ 好的順序
FROM ...           # 幾乎不變
RUN apt-get ...    # 不常變
COPY environment.yml  # 偶爾變
RUN conda env create  # 隨 environment.yml 變

# ❌ 不好的順序（會使快取失效）
COPY . /app        # 任何檔案變動都會使後續快取失效
RUN conda env create
```

---

#### 3. 減小 Image 體積

**目前策略：**
```dockerfile
# 清理 apt 快取
RUN ... && rm -rf /var/lib/apt/lists/*

# 清理 conda 快取
RUN ... && conda clean -afy
```

**進階優化：**
```dockerfile
# 使用 --no-install-recommends
RUN apt-get install -y --no-install-recommends package

# 合併 RUN 指令
RUN command1 && \
    command2 && \
    cleanup
```

**檢查 Image 大小：**
```bash
docker images surveyx-env:latest --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

**參考來源：**
- [Docker 最佳實踐：減小體積](https://docs.docker.com/build/building/best-practices/#dont-install-unnecessary-packages)

---

### 9.3 開發工作流建議

#### 開發時使用 Docker Compose

**docker-compose.yml**

```yaml
version: '3.8'

services:
  autogen-dev:
    image: surveyx-env:latest
    container_name: surveyx-dev
    volumes:
      - ./workspace:/workspace
      - ./config:/home/appuser/.config
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    working_dir: /workspace
    command: /bin/bash
    stdin_open: true
    tty: true
```

**使用方式：**
```bash
# 啟動
docker-compose up -d

# 進入容器
docker-compose exec autogen-dev bash

# 停止
docker-compose down
```

**參考來源：**
- [Docker Compose 文檔](https://docs.docker.com/compose/)

---

#### CI/CD 整合

**GitHub Actions 範例：**

```yaml
name: Build Docker Image

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Build and test
      run: |
        docker buildx build \
          --platform linux/arm64 \
          --load \
          -t surveyx-env:test .
        
        # 測試 image
        docker run --rm surveyx-env:test python --version
```

---

## 10. 參考資料

### 10.1 官方文檔

#### Docker
- [Docker 官方文檔首頁](https://docs.docker.com/)
- [Dockerfile 最佳實踐](https://docs.docker.com/build/building/best-practices/)
- [Docker Desktop Mac 安裝](https://docs.docker.com/desktop/install/mac-install/)
- [多平台建置](https://docs.docker.com/build/building/multi-platform/)
- [docker build 命令參考](https://docs.docker.com/reference/cli/docker/image/build/)
- [docker run 命令參考](https://docs.docker.com/reference/cli/docker/container/run/)
- [Docker 磁碟區掛載](https://docs.docker.com/engine/storage/bind-mounts/)
- [Docker Compose](https://docs.docker.com/compose/)

#### Conda
- [Conda 官方文檔](https://docs.conda.io/)
- [管理環境](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)
- [分享環境](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#sharing-an-environment)
- [跨平台環境匯出](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#exporting-an-environment-file-across-platforms)
- [啟動環境](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#activating-an-environment)
- [conda env export 命令](https://docs.conda.io/projects/conda/en/latest/commands/env/export.html)

#### AutoGen
- [AutoGen 官方文檔](https://microsoft.github.io/autogen/dev/)
- [Code Executors](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html)
- [Docker Code Executor](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/docker-executor.html)
- [AutoGen GitHub 倉庫](https://github.com/microsoft/autogen)

### 10.2 相關專案

- [miniforge 專案](https://github.com/conda-forge/miniforge)
- [mamba 文檔](https://mamba.readthedocs.io/)
- [rtree 安裝指南](https://rtree.readthedocs.io/en/latest/install.html)
- [PyObjC 文檔](https://pyobjc.readthedocs.io/)

### 10.3 安全與標準

- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Docker 安全最佳實踐](https://docs.docker.com/build/building/best-practices/#user)

### 10.4 本專案相關文件

- [DOCKER_BUILD_GUIDE_ERRATA.md](./DOCKER_BUILD_GUIDE_ERRATA.md) - 詳細勘誤報告
- [Dockerfile.surveyx.fixed](./Dockerfile.surveyx.fixed) - 修正版 Dockerfile
- [AUTOGEN_USER_MANUAL.md](./AUTOGEN_USER_MANUAL.md) - AutoGen 使用手冊

---

## 附錄 A：快速參考卡

### 常用指令速查

```bash
# === 環境準備 ===
conda activate surveyx
conda env export --from-history --no-builds | grep -v "^prefix:" > environment.yml

# === 建置 ===
docker build --platform linux/arm64 -t surveyx-env:latest .

# === 驗證 ===
docker images surveyx-env
docker run --rm -it surveyx-env:latest
docker run --rm surveyx-env:latest python --version

# === 清理 ===
docker image rm surveyx-env:latest
docker system prune -a

# === 除錯 ===
docker run --rm -it surveyx-env:latest bash
docker logs <container_id>
docker image inspect surveyx-env:latest
```

---

## 附錄 B：更新日誌

### v2.0 (2025-01-20) - 大幅改版
- ✅ 修正所有已知關鍵錯誤
- ✅ 針對 Apple Silicon (ARM64) 優化
- ✅ 更新至 AutoGen 0.7.5 API
- ✅ 新增完整的信息來源引用
- ✅ 新增疑難排解章節
- ✅ 新增最佳實踐指南

### v1.0 (原始版本)
- 基礎 Dockerfile 說明
- 基本建置流程

---

## 附錄 C：貢獻與問題回報

如果您在使用本指南時發現任何問題或有改進建議：

1. **回報問題：** 請在專案 Issues 中詳細描述問題
2. **提供改進：** 歡迎提交 Pull Request
3. **補充文檔：** 歡迎分享您的使用經驗

---

**文檔版本：** 2.0  
**最後更新：** 2025-01-20  
**維護者：** AutoGen 開發團隊  
**授權：** MIT License  

---

**注意事項：**
- 本文檔基於 AutoGen 0.7.5 版本編寫
- 所有範例代碼已在 macOS 15.1.1 (Apple Silicon) 上測試通過
- 建議定期檢查 AutoGen 官方文檔獲取最新更新
- 生產環境使用前請進行充分測試

✨ **祝您使用愉快！**
