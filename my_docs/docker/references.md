# 官方文檔與參考資料

> **目的：** 彙整所有相關的官方文檔與學習資源

[← 返回主指南](./README.md)

---

## 🐳 Docker 官方資源

### 核心文檔
- [Docker 官方文檔首頁](https://docs.docker.com/)
- [Dockerfile 最佳實踐](https://docs.docker.com/build/building/best-practices/)
- [Docker Desktop macOS 安裝](https://docs.docker.com/desktop/install/mac-install/)
- [多平台建置](https://docs.docker.com/build/building/multi-platform/)

### 命令參考
- [docker build 命令](https://docs.docker.com/reference/cli/docker/image/build/)
- [docker run 命令](https://docs.docker.com/reference/cli/docker/container/run/)
- [docker image 命令](https://docs.docker.com/reference/cli/docker/image/)

### 進階主題
- [多階段建置](https://docs.docker.com/build/building/multi-stage/)
- [Docker Compose](https://docs.docker.com/compose/)
- [磁碟區掛載](https://docs.docker.com/engine/storage/bind-mounts/)
- [網路配置](https://docs.docker.com/engine/network/)

### 疑難排解
- [Docker Desktop 疑難排解](https://docs.docker.com/desktop/troubleshoot/overview/)
- [資源配置](https://docs.docker.com/desktop/settings-and-maintenance/settings/#resources)

---

## 🐍 Conda 官方資源

### 環境管理
- [Conda 官方文檔](https://docs.conda.io/)
- [管理環境](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)
- [分享環境](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#sharing-an-environment)
- [跨平台環境匯出](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#exporting-an-environment-file-across-platforms)

### 命令參考
- [conda env export](https://docs.conda.io/projects/conda/en/latest/commands/env/export.html)
- [conda env create](https://docs.conda.io/projects/conda/en/latest/commands/env/create.html)
- [conda activate](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#activating-an-environment)

### 進階工具
- [mamba 文檔](https://mamba.readthedocs.io/) - 快速的 Conda 替代品
- [conda-lock](https://conda.github.io/conda-lock/) - 確保完全可重現的環境

---

## 🤖 AutoGen 官方資源

### 核心文檔
- [AutoGen 官方文檔](https://microsoft.github.io/autogen/dev/)
- [Code Executors 指南](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html)
- [Docker Code Executor](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/docker-executor.html)

### 源碼參考
- [AutoGen GitHub 倉庫](https://github.com/microsoft/autogen)
- [DockerCommandLineCodeExecutor 源碼](https://github.com/microsoft/autogen/blob/main/python/packages/autogen-ext/src/autogen_ext/code_executors/docker/_docker_code_executor.py)

### 套件文檔
- [autogen-agentchat PyPI](https://pypi.org/project/autogen-agentchat/)
- [autogen-core PyPI](https://pypi.org/project/autogen-core/)
- [autogen-ext PyPI](https://pypi.org/project/autogen-ext/)

---

## 📦 相關專案

### Python 套件管理
- [pip 官方文檔](https://pip.pypa.io/)
- [pip freeze](https://pip.pypa.io/en/stable/cli/pip_freeze/)
- [pipreqs](https://github.com/bndr/pipreqs) - 自動生成 requirements.txt

### 系統依賴
- [Rtree 安裝指南](https://rtree.readthedocs.io/en/latest/install.html) - libspatialindex-dev
- [Poppler Utils](https://poppler.freedesktop.org/) - PDF 處理工具
- [PyObjC 文檔](https://pyobjc.readthedocs.io/) - macOS 專用套件

### 容器工具
- [miniforge 專案](https://github.com/conda-forge/miniforge) - ARM64 優化的 Conda
- [Docker BuildKit](https://docs.docker.com/build/buildkit/) - 進階建置引擎

---

## 🛡️ 安全與標準

- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker) - Docker 安全標準
- [Docker 安全最佳實踐](https://docs.docker.com/build/building/best-practices/#user)
- [OWASP Container Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

---

## 💻 macOS 系統資源

- [macOS 儲存管理](https://support.apple.com/zh-tw/guide/mac-help/welcome/mac)
- [Apple Silicon (ARM64) 說明](https://support.apple.com/zh-tw/116943)
- [Hard disk drive performance](https://en.wikipedia.org/wiki/Hard_disk_drive_performance_characteristics)

---

## 📖 學習資源

### Docker 學習路徑
1. [Docker 官方教學](https://docs.docker.com/get-started/)
2. [Dockerfile 最佳實踐](https://docs.docker.com/build/building/best-practices/)
3. [Docker Compose 入門](https://docs.docker.com/compose/gettingstarted/)

### Conda 學習路徑
1. [Conda 入門](https://docs.conda.io/projects/conda/en/latest/user-guide/getting-started.html)
2. [管理套件](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-pkgs.html)
3. [建立環境](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)

### AutoGen 學習路徑
1. [AutoGen 快速開始](https://microsoft.github.io/autogen/dev/user-guide/quickstart.html)
2. [建立 Agents](https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/design-patterns/intro.html)
3. [Code Executors](https://microsoft.github.io/autogen/dev/user-guide/extensions-user-guide/code-executors/index.html)

---

## 🔗 本專案相關文件

### 核心指南
- [主指南 (README.md)](./README.md)
- [系統要求](./system-requirements.md)
- [環境檔案準備](./environment-setup.md)
- [AutoGen 整合](./autogen-integration.md)
- [疑難排解](./troubleshooting.md)

### 其他專案文檔
- [AUTOGEN_USER_MANUAL.md](../AUTOGEN_USER_MANUAL.md) - AutoGen 使用手冊
- [DOCKER_BUILD_GUIDE_ERRATA.md](../DOCKER_BUILD_GUIDE_ERRATA.md) - 詳細勘誤報告
- [Dockerfile.surveyx.fixed](../Dockerfile.surveyx.fixed) - 修正版 Dockerfile

---

## 📬 社群資源

### 官方社群
- [AutoGen GitHub Discussions](https://github.com/microsoft/autogen/discussions)
- [Docker Community Forums](https://forums.docker.com/)
- [Conda Discourse](https://conda.discourse.group/)

### 問題回報
- [AutoGen Issues](https://github.com/microsoft/autogen/issues)
- [Docker Desktop Issues](https://github.com/docker/for-mac/issues)

---

## 📊 版本對應表

### 已測試環境組合

| 組件 | 版本 | 狀態 |
|------|------|------|
| macOS | 15.1.1 (Apple Silicon) | ✅ 測試通過 |
| Docker Desktop | 26.0+ | ✅ 測試通過 |
| Python | 3.11.13 | ✅ 測試通過 |
| AutoGen (agentchat) | 0.7.5 | ✅ 測試通過 |
| AutoGen (core) | 0.7.5 | ✅ 測試通過 |
| AutoGen (ext) | 0.7.5 | ✅ 測試通過 |
| Conda | 24.0+ | ✅ 測試通過 |

### 最低版本要求

| 組件 | 最低版本 |
|------|---------|
| Docker Desktop | 26.0 |
| Python | 3.10 |
| AutoGen | 0.7.5 |
| macOS | 12.0 (Monterey) |

---

**最後更新：** 2025-01-20  
**維護者：** AutoGen 開發團隊
