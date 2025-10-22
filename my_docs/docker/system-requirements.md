# 系統要求與配置

> **目的：** 確保您的系統滿足建置與運行 surveyx Docker image 的需求

[← 返回主指南](./README.md)

---

## 📋 硬體要求

### 處理器（CPU）

| 架構 | 狀態 | 說明 |
|------|------|------|
| **Apple Silicon (M1/M2/M3)** | ✅ **推薦** | 本指南優化目標，原生 ARM64 支援 |
| **Intel Mac (x86_64)** | ⚠️ 支援 | 需修改 Dockerfile `--platform` 參數 |
| **其他 ARM64** | ⚠️ 未測試 | 理論上支援，需自行驗證 |

**驗證指令：**
```bash
uname -m
# Apple Silicon 應輸出: arm64
# Intel 應輸出: x86_64
```

### 記憶體（RAM）

#### 建議配置

| 主機記憶體 | Docker Memory Limit | 建置狀態 | 說明 |
|-----------|---------------------|---------|------|
| **16GB+** | 8GB (預設 50%) | ✅ 優秀 | 無需調整，建置流暢 |
| **8GB** | **4GB (需手動設定)** | ⚠️ 可用 | **必須調整**，否則可能 OOM |
| **< 8GB** | - | ❌ 不建議 | 可能導致建置失敗或系統凍結 |

#### Docker Desktop 預設行為

⚠️ **重要：** Docker Desktop 在 macOS 上預設使用主機 **50% 記憶體**

- 建置 surveyx-env 峰值約需 **4-6GB**
- 8GB 主機在預設配置下可能記憶體不足

#### 如何調整記憶體限制

**方式 1：圖形界面**
```
Docker Desktop > Settings > Resources > Memory
設定為 4GB (8GB 主機) 或 6GB (16GB 主機)
```

**方式 2：命令列（進階）**
```bash
# 編輯 Docker daemon 設定
# macOS: ~/Library/Group Containers/group.com.docker/settings.json
```

#### 記憶體問題診斷

**症狀：**
- 建置過程出現 `Killed`
- 錯誤訊息：`Out of memory` 或 `OOMKilled`
- 系統變慢或凍結

**解決方案：**
1. 增加 Docker Memory limit
2. 關閉其他應用程式釋放記憶體
3. 分階段建置（使用多階段 Dockerfile）

**參考來源：**
- [Docker Desktop 資源設定](https://docs.docker.com/desktop/settings-and-maintenance/settings/#resources)

---

## 💾 儲存空間

### 容量需求

| 項目 | 空間需求 | 說明 |
|------|---------|------|
| **Docker Image** | 3-5GB | surveyx-env 最終大小 |
| **Conda 快取** | 1-2GB | 建置過程暫存 |
| **建置暫存** | 3-5GB | 中間層與下載檔案 |
| **額外緩衝** | 3-5GB | 系統運作與其他容器 |
| **建議總計** | **15GB+** | 安全餘裕 |

### 儲存類型與性能

#### 建置時間比較

| 儲存類型 | 讀寫速度 | 建置時間 | 適用場景 |
|---------|---------|---------|---------|
| **內建 SSD** | 2000+ MB/s | **5-8 分鐘** | ✅ 日常開發（強烈推薦） |
| **外接 SSD (USB 3.1)** | 500-1000 MB/s | 8-12 分鐘 | ⚠️ 空間不足時 |
| **外接 SSD (USB 3.0)** | 200-400 MB/s | 12-20 分鐘 | ⚠️ 空間不足時 |
| **外接 HDD (7200 RPM)** | 100-150 MB/s | 20-30 分鐘 | ❌ 不推薦 |
| **外接 HDD (5400 RPM)** | 50-100 MB/s | **30-60 分鐘** | ❌ 僅儲存用途 |

⚠️ **警告：** 使用外接 HDD 會使建置時間增加 **4-10 倍**

#### 如何搬移 Docker 資料目錄

**適用情況：** 主機內建 SSD 空間不足，需使用外接硬碟

**步驟（Docker Desktop for Mac）：**

1. **停止 Docker Desktop**
   ```bash
   # 完全退出 Docker Desktop
   ```

2. **修改磁碟映像位置**
   ```
   Docker Desktop > Settings > Resources > Disk image location
   
   點擊 "Browse" 選擇外接硬碟路徑  
   例如：`/Volumes/My Book/surveyx_data/`（注意路徑含空白時需使用引號）
   ```

3. **重新啟動 Docker Desktop**
   - Docker 會自動遷移資料（可能需 5-15 分鐘）
   - 遷移完成後可刪除舊的磁碟映像檔案

**注意事項：**
- ✅ **使用外接 SSD：** 性能損失約 20-50%
- ❌ **使用外接 HDD：** 性能損失約 70-90%，**不建議**用於開發

**參考來源：**
- [Docker Desktop 磁碟映像管理](https://docs.docker.com/desktop/settings-and-maintenance/settings/#resources)
- [Hard disk drive performance](https://en.wikipedia.org/wiki/Hard_disk_drive_performance_characteristics)

---

## 🖥️ 軟體要求

### Docker Desktop

**版本要求：** 26.0+（建議最新版本）

**下載連結：**
- **Apple Silicon**: [Docker Desktop (ARM64)](https://desktop.docker.com/mac/main/arm64/Docker.dmg)
- **Intel Chip**: [Docker Desktop (AMD64)](https://desktop.docker.com/mac/main/amd64/Docker.dmg)

**安裝驗證：**
```bash
# 檢查版本
docker --version
# 應輸出: Docker version 26.0+ 或更高

# 測試 Docker 運行
docker run --rm hello-world
# 應成功拉取並執行 hello-world 映像
```

**安裝步驟：**
1. 下載對應架構的 DMG 檔案
2. 掛載 DMG：`open Docker.dmg`
3. 拖曳 Docker.app 到 Applications 資料夾
4. 啟動：`open /Applications/Docker.app`
5. 等待 Docker daemon 啟動（狀態列圖示變綠）

**參考來源：**
- [Docker Desktop Mac 安裝文檔](https://docs.docker.com/desktop/install/mac-install/)

### Conda

**用途：** 用於匯出 surveyx 環境定義檔案

**版本要求：** Anaconda 或 Miniconda（任意版本）

**驗證指令：**
```bash
# 檢查 Conda 版本
conda --version

# 確認 surveyx 環境存在
conda env list | grep surveyx
```

**如果尚未安裝 surveyx 環境：**
請參考您的專案文檔建立環境。

**參考來源：**
- [Conda 安裝文檔](https://docs.conda.io/projects/conda/en/latest/user-guide/install/index.html)

---

## ⚙️ Docker Desktop 配置建議

### 資源配置總覽

**針對不同主機記憶體的建議配置：**

#### 16GB+ RAM 主機（推薦）
```
CPUs: 4-6 核心
Memory: 8GB (預設 50%)
Swap: 1GB
Disk image size: 64GB（自動擴展）
```

#### 8GB RAM 主機（最低要求）
```
CPUs: 2-4 核心
Memory: 4GB (⚠️ 必須手動設定)
Swap: 512MB
Disk image size: 64GB（自動擴展）
```

### 檔案共享設定

**預設行為：** macOS 下 Docker Desktop 會自動掛載：
- `/Users`
- `/Volumes`
- `/private`
- `/tmp`

**無需額外設定**，除非您的專案在其他路徑。

---

## 🔍 系統檢查清單

建置前請確認以下項目：

### 必要檢查 ✅

```bash
# 1. 確認系統架構
uname -m
# ✅ 應輸出: arm64 (Apple Silicon)

# 2. 確認 Docker 運行
docker ps
# ✅ 應成功顯示容器列表（可為空）

# 3. 確認 Docker 版本
docker --version
# ✅ 應為 26.0 或更高

# 4. 確認 Conda 環境
conda env list | grep surveyx
# ✅ 應顯示 surveyx 環境路徑

# 5. 確認可用磁碟空間
df -h /
# ✅ 應有 15GB+ 可用空間
```

### 建議檢查 ⚠️

```bash
# 6. 檢查 Docker 記憶體限制
docker info | grep "Total Memory"
# ⚠️ 8GB 主機應為 4GB 左右

# 7. 檢查 Docker 磁碟使用
docker system df
# ⚠️ 確認有足夠空間

# 8. 測試網路連接
ping -c 3 docker.io
# ⚠️ 確認可連接 Docker Hub
```

---

## 🚨 常見配置問題

### 問題 1: Docker Desktop 無法啟動

**症狀：** 狀態列圖示持續顯示「Docker Desktop is starting...」

**可能原因：**
- 權限問題
- 舊版本殘留檔案
- 系統資源不足

**解決方案：**
```bash
# 1. 完全重置 Docker Desktop
# Docker Desktop > Troubleshoot > Reset to factory defaults

# 2. 重新安裝 Docker Desktop
# 刪除應用程式並重新下載安裝

# 3. 檢查系統記憶體是否足夠
```

### 問題 2: 建置時記憶體不足

**症狀：** 建置過程中出現 `Killed` 或系統變慢

**解決方案：**
1. 降低 Docker Memory limit（8GB 主機設為 4GB）
2. 關閉其他應用程式
3. 使用 `--memory` 參數限制建置記憶體：
   ```bash
   docker build --memory=4g --platform linux/arm64 -t surveyx-env:latest .
   ```

### 問題 3: 磁碟空間不足

**症狀：** `no space left on device`

**解決方案：**
```bash
# 1. 清理未使用的 Docker 資源
docker system prune -a

# 2. 搬移 Docker 資料到外接硬碟（見上文）

# 3. 增加 Docker 磁碟映像大小
# Docker Desktop > Settings > Resources > Disk image size
```

---

## 📊 推薦配置總結

### 最佳配置（開發環境）
- **CPU:** Apple Silicon M1/M2/M3
- **RAM:** 16GB+
- **儲存:** 內建 SSD，15GB+ 可用空間
- **Docker Memory:** 8GB (預設)
- **網路:** 穩定網路連接（建置時下載依賴）

### 最低可用配置
- **CPU:** Apple Silicon M1 或 Intel i5
- **RAM:** 8GB（需手動調整 Docker Memory 至 4GB）
- **儲存:** 內建 SSD 或外接 SSD，15GB+ 可用空間
- **Docker Memory:** 4GB（手動設定）
- **網路:** 穩定網路連接

### 不建議配置 ❌
- RAM < 8GB
- 使用外接 HDD 作為 Docker 資料目錄
- 磁碟可用空間 < 10GB

---

## 📚 相關文檔

- [← 返回主指南](./README.md)
- [環境檔案準備 →](./environment-setup.md)
- [疑難排解](./troubleshooting.md)

**參考來源：**
- [Docker Desktop 資源配置](https://docs.docker.com/desktop/settings-and-maintenance/settings/#resources)
- [Docker daemon 配置](https://docs.docker.com/config/daemon/)
- [macOS 儲存管理](https://support.apple.com/zh-tw/guide/mac-help/welcome/mac)
