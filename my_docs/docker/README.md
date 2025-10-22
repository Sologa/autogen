# surveyx Docker Image 建置指南

> **目標：** 為 AutoGen Agent 建立基於 Conda 的 surveyx 沙箱執行環境  
> **平台：** Apple Silicon (ARM64) 專用  
> **AutoGen 版本：** 0.7.5+  
> **最後更新：** 2025-01-20

本指南提供建立 `surveyx-env` Docker image 的核心步驟與關鍵概念。

---

## 📚 文檔導航

### 核心文檔
- **本文檔 (README.md)** - 快速建置指南與核心概念
- [完整 Dockerfile 說明](./dockerfile-explained.md) - 逐行解釋與設計決策
- [AutoGen 整合使用](./autogen-integration.md) - 如何在 AutoGen 中使用此 image

### 環境準備
- [系統要求與配置](./system-requirements.md) - 硬體、軟體、記憶體、儲存規劃
- [環境檔案準備](./environment-setup.md) - Conda 環境匯出與 pip 依賴處理

### 進階主題
- [疑難排解](./troubleshooting.md) - 常見問題與解決方案
- [最佳實踐](./best-practices.md) - 安全性、性能優化、開發工作流
- [跨平台建置](./cross-platform.md) - AMD64/ARM64 多架構支援

### 參考資料
- [官方文檔連結](./references.md) - Docker、Conda、AutoGen 官方資源

---

## 🎯 核心概念

### 架構圖

```
┌─────────────────────────────────────────┐
│      Host Machine (macOS ARM64)         │
│  ┌───────────────────────────────────┐  │
│  │    AutoGen Application            │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │ DockerCommandLineCodeExecutor│  │  │
│  │  └─────────────┬───────────────┘  │  │
│  └────────────────┼───────────────────┘  │
│                   │ Docker API           │
│  ┌────────────────▼───────────────────┐  │
│  │   Docker Container                 │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │ Conda Env: surveyx           │  │  │
│  │  │  • Python 3.11               │  │  │
│  │  │  • autogen-agentchat         │  │  │
│  │  │  • autogen-ext[openai]       │  │  │
│  │  │  • 所有 pip 依賴             │  │  │
│  │  └──────────────────────────────┘  │  │
│  │  Base: miniforge3 (ARM64)         │  │
│  └────────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### 為什麼需要 Docker？

1. **隔離性** - Agent 執行的程式碼與主機系統完全隔離
2. **安全性** - 限制權限，避免惡意程式碼影響主機
3. **可重現性** - 確保在不同機器上環境一致
4. **易於部署** - 一次建置，隨處執行

---

## 🚀 快速開始（5 分鐘）

### 前置檢查

```bash
# 1. 確認系統架構（應顯示 arm64）
uname -m

# 2. 確認 Docker 運行中
docker ps

# 3. 確認 surveyx 環境存在
conda env list | grep surveyx
```

### 步驟 1: 準備環境檔案

```bash
# 啟動 surveyx 環境
conda activate surveyx

# 生成 Conda 環境檔案（移除平台特定資訊）
conda env export --no-builds | grep -v "^prefix:" > environment.yml

# ⚠️ 重要：如果您的環境包含大量 pip 套件，建議額外匯出
pip freeze > requirements.txt
```

**詳細說明與選項：** 見 [環境檔案準備](./environment-setup.md)

### 步驟 2: 建立 Dockerfile

在專案根目錄建立 `Dockerfile`：

```dockerfile
FROM --platform=linux/arm64 condaforge/miniforge3:latest

ENV PYTHONUNBUFFERED=1 \
    CONDA_ENV_NAME=surveyx

# 安裝系統依賴
RUN apt-get update && apt-get install -y --no-install-recommends \
    poppler-utils \
    libgl1-mesa-glx \
    libglib2.0-0 \
    libspatialindex-dev \
    gcc g++ make \
    curl wget git \
    && rm -rf /var/lib/apt/lists/*

# 建立非 root 使用者
RUN useradd --create-home --shell /bin/bash appuser
WORKDIR /home/appuser/app

# 複製環境檔案
COPY --chown=appuser:appuser environment.yml .

# 建立 Conda 環境
RUN mamba env create -f environment.yml && \
    mamba clean -afy && \
    conda init bash

# ⚠️ 如果有額外的 pip 依賴，解除註解以下行
# COPY --chown=appuser:appuser requirements.txt .
# RUN /opt/conda/envs/surveyx/bin/pip install -r requirements.txt

# 建立啟動腳本
RUN echo '#!/bin/bash' > /home/appuser/entrypoint.sh && \
    echo 'source /opt/conda/etc/profile.d/conda.sh' >> /home/appuser/entrypoint.sh && \
    echo 'conda activate ${CONDA_ENV_NAME}' >> /home/appuser/entrypoint.sh && \
    echo 'exec "$@"' >> /home/appuser/entrypoint.sh && \
    chmod +x /home/appuser/entrypoint.sh

USER appuser
WORKDIR /workspace

ENTRYPOINT ["/home/appuser/entrypoint.sh"]
CMD ["/bin/bash"]
```

**完整解釋與設計決策：** 見 [Dockerfile 詳解](./dockerfile-explained.md)

### 步驟 3: 建立 .dockerignore

```bash
cat > .dockerignore << 'EOF'
**/__pycache__
**/*.pyc
**/.venv
**/.git
**/.DS_Store
*.log
.env
**/*.csv
**/*.db
EOF
```

### 步驟 4: 建置 Image

```bash
# 基本建置（Apple Silicon）
docker build --platform linux/arm64 -t surveyx-env:latest .

# 預期時間：首次 5-15 分鐘，後續 1-3 分鐘（快取）
```

**進階選項與跨平台建置：** 見 [跨平台建置](./cross-platform.md)

### 步驟 5: 驗證

```bash
# 測試容器啟動
docker run --rm -it surveyx-env:latest

# 在容器內檢查環境
$ conda env list
$ python --version
$ pip list | grep autogen

# 測試 AutoGen 套件
$ python -c "import autogen_agentchat; print(autogen_agentchat.__version__)"

# 退出容器
$ exit
```

---

## 💻 在 AutoGen 中使用

### 基本使用範例

```python
from autogen_ext.code_executors.docker import DockerCommandLineCodeExecutor

# 建立 Docker 執行器
executor = DockerCommandLineCodeExecutor(
    image="surveyx-env:latest",
    work_dir="./workspace",
    init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx",
    timeout=120,
    auto_remove=True,
)

# 在 AutoGen Agent 中使用
# ... (見完整範例)
```

**完整整合指南與範例：** 見 [AutoGen 整合使用](./autogen-integration.md)

**⚠️ 重要：** AutoGen 0.7.5+ 使用 `init_command` 參數（不是 `command`）

---

## 🔧 常見問題速查

### 建置失敗

| 錯誤 | 原因 | 解決方案 |
|------|------|---------|
| `platform mismatch` | 未指定 ARM64 | 加上 `--platform linux/arm64` |
| `prefix: /opt/anaconda3` | environment.yml 含本機路徑 | 重新匯出並移除 prefix 行 |
| `libspatialindex not found` | 系統依賴缺失 | 檢查 Dockerfile apt-get 部分 |

**完整疑難排解：** 見 [疑難排解指南](./troubleshooting.md)

### 性能問題

- **建置太慢？** 檢查是否使用 SSD（HDD 可能需 20-40 分鐘）
- **記憶體不足？** 8GB 裝置需調整 Docker Memory limit 至 4GB
- **磁碟空間不足？** 建議預留 15GB 可用空間

**系統配置建議：** 見 [系統要求](./system-requirements.md)

---

## 📊 重要注意事項

### 平台限制

⚠️ **本 Dockerfile 針對 Apple Silicon (ARM64) 優化**

- ✅ **適用於：** M1/M2/M3 Mac
- ⚠️ **Intel Mac：** 需修改 `--platform` 參數為 `linux/amd64`
- ⚠️ **Linux/Windows：** 需根據架構調整

如需真正的跨平台支援，見 [跨平台建置指南](./cross-platform.md)

### pip 依賴處理

如果您的 surveyx 環境包含大量 pip 安裝的套件：

1. **方案 A：** `conda env export` 會自動包含 pip 部分
2. **方案 B：** 額外執行 `pip freeze > requirements.txt` 並在 Dockerfile 中安裝
3. **方案 C：** 使用 SurveyX 官方的 `env/requirements-freeze.txt`

**詳細處理方式：** 見 [環境檔案準備](./environment-setup.md#pip-依賴處理)

---

## 🎓 下一步

### 基礎使用者
1. ✅ 完成上述快速開始步驟
2. 📖 閱讀 [AutoGen 整合使用](./autogen-integration.md)
3. 🔍 遇到問題時查閱 [疑難排解](./troubleshooting.md)

### 進階使用者
1. 📚 深入了解 [Dockerfile 設計](./dockerfile-explained.md)
2. ⚡ 學習 [最佳實踐](./best-practices.md) 優化性能
3. 🌐 實作 [跨平台建置](./cross-platform.md)

---

## 📝 版本資訊

- **文檔版本：** 2.0
- **Dockerfile 版本：** 2.0
- **測試環境：** macOS 15.1.1 (Apple Silicon)
- **AutoGen 版本：** 0.7.5

---

## 🤝 貢獻與回饋

如發現問題或有改進建議，歡迎：
1. 回報 Issues
2. 提交 Pull Request
3. 分享使用經驗

**授權：** MIT License
