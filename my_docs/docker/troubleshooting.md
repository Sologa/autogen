# 疑難排解指南

> **目的：** 快速解決常見的建置與執行問題

[← 返回主指南](./README.md)

---

## 🔴 建置階段問題

### 問題 1: 平台不匹配警告

**錯誤訊息：**
```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64)
```

**原因：** 未指定 `--platform` 參數，Docker 拉取了錯誤的架構映像

**解決方案：**
```bash
# Apple Silicon 明確指定 ARM64
docker build --platform linux/arm64 -t surveyx-env:latest .
```

**參考：** [Docker 多平台建置](https://docs.docker.com/build/building/multi-platform/)

---

### 問題 2: Conda 環境建立失敗

**錯誤訊息：**
```
CondaEnvironmentError: cannot locate environment at /opt/anaconda3/envs/surveyx
```

**原因：** `environment.yml` 包含 `prefix:` 行，指向主機端路徑

**解決方案：**
```bash
# 重新生成環境檔案，移除 prefix 行
conda env export --no-builds | grep -v "^prefix:" > environment.yml
```

**參考：** [環境檔案準備](./environment-setup.md#陷阱-1-environmentyml-包含-prefix-行)

---

### 問題 3: pip 套件安裝失敗 (macOS 專用套件)

**錯誤訊息：**
```
ERROR: Could not find a version that satisfies the requirement pyobjc-framework-Cocoa
```

**原因：** PyObjC 系列套件僅支援 macOS，無法在 Linux 容器中安裝

**解決方案：**
```bash
# 移除 macOS 專用套件
sed -i '' '/pyobjc/d' requirements.txt

# 或在 Dockerfile 中使用容錯安裝
RUN /opt/conda/envs/surveyx/bin/pip install -r requirements.txt || true
```

**參考：** [環境檔案準備](./environment-setup.md#陷阱-3-macos-專用套件無法在-linux-容器安裝)

---

### 問題 4: 系統依賴缺失

**錯誤訊息：**
```
E: Unable to locate package libspatialindex-dev
```

**原因：** apt repository 未更新或套件名稱錯誤

**解決方案：**
```dockerfile
# 確保執行 apt-get update
RUN apt-get update && apt-get install -y --no-install-recommends \
    libspatialindex-dev \
    && rm -rf /var/lib/apt/lists/*
```

---

### 問題 5: 建置記憶體不足

**症狀：** 建置過程中出現 `Killed` 或系統變慢

**解決方案：**
```bash
# 1. 調整 Docker Memory limit (8GB 主機設為 4GB)
# Docker Desktop > Settings > Resources > Memory

# 2. 使用 --memory 參數限制建置記憶體
docker build --memory=4g --platform linux/arm64 -t surveyx-env:latest .

# 3. 關閉其他應用程式釋放記憶體
```

**參考：** [系統要求](./system-requirements.md#記憶體ram)

---

## 🟡 執行階段問題

### 問題 6: conda activate 指令無效

**錯誤訊息：**
```
CommandNotFoundError: Your shell has not been properly configured to use 'conda activate'.
```

**原因：** `/bin/sh` 不支援 `conda activate`

**解決方案：**
```python
# ✅ 正確寫法
init_command="source /opt/conda/etc/profile.d/conda.sh && conda activate surveyx"

# ❌ 錯誤寫法（已廢棄）
init_command="source activate surveyx"
```

**參考：** [AutoGen 整合](./autogen-integration.md#init_command-詳解)

---

### 問題 7: 容器內無法訪問網路

**錯誤訊息：**
```
urllib.error.URLError: <urlopen error [Errno -3] Temporary failure in name resolution>
```

**原因：** Docker 網路配置問題

**解決方案：**
```bash
# 測試容器網路
docker run --rm surveyx-env:latest ping -c 3 google.com

# 如果失敗，重啟 Docker Desktop
```

---

### 問題 8: 權限被拒絕

**錯誤訊息：**
```
PermissionError: [Errno 13] Permission denied: '/workspace/output.txt'
```

**原因：** 主機端目錄權限不足

**解決方案：**
```bash
# 修改主機端目錄權限
chmod -R 755 ./workspace
```

---

## 🔵 性能問題

### 問題 9: 建置速度太慢

**症狀：** 建置時間超過 30 分鐘

**診斷：**
```bash
# 檢查儲存類型
diskutil info / | grep "Solid State"

# 檢查 Docker 資料位置
docker info | grep "Docker Root Dir"
```

**解決方案：**
- ✅ 使用內建 SSD（5-8 分鐘）
- ⚠️ 如使用外接 HDD，考慮搬移到 SSD
- 參考：[系統要求 - 儲存空間](./system-requirements.md#儲存類型與性能)

---

## 🟢 除錯技巧

### 進入容器檢查

```bash
# 啟動互動式 shell
docker run --rm -it surveyx-env:latest bash

# 在容器內檢查
$ conda env list
$ python --version
$ pip list | grep autogen
$ env | grep CONDA
```

### 查看建置歷史

```bash
# 查看每層的大小
docker image history surveyx-env:latest

# 檢查最大的層
docker image history surveyx-env:latest --no-trunc
```

### 查看容器日誌

```bash
# 執行並保留容器
docker run --name test-surveyx surveyx-env:latest python -c "print('test')"

# 查看日誌
docker logs test-surveyx

# 清理
docker rm test-surveyx
```

---

## 📋 快速診斷檢查表

執行以下指令快速診斷問題：

```bash
# 1. 系統架構
uname -m

# 2. Docker 版本
docker --version

# 3. Docker 運行狀態
docker ps

# 4. Docker 資源使用
docker system df

# 5. Docker 記憶體限制
docker info | grep "Total Memory"

# 6. 磁碟可用空間
df -h /

# 7. Conda 環境存在
conda env list | grep surveyx

# 8. environment.yml 檢查
grep "^prefix:" environment.yml  # 應無輸出
```

---

## 📚 相關文檔

- [← 返回主指南](./README.md)
- [系統要求](./system-requirements.md)
- [環境檔案準備](./environment-setup.md)
- [AutoGen 整合](./autogen-integration.md)

**官方資源：**
- [Docker 疑難排解](https://docs.docker.com/desktop/troubleshoot/overview/)
- [Conda 常見問題](https://docs.conda.io/projects/conda/en/latest/user-guide/troubleshooting.html)
