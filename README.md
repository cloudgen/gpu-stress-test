# B300 GPU 伺服器基本驗證與壓測前置腳本

適用環境：**Ubuntu + NVIDIA GPU**。  
此腳本會安裝 NVIDIA 驅動（提供 `nvidia-smi`）、NVIDIA DCGM（提供 `dcgmi`）與 InfiniBand／RDMA 診斷工具（提供 `ibstat`），並執行基本驗證。

> **重要**：安裝或更換 NVIDIA driver 後通常需要重新開機；腳本會偵測此情況並提示重新開機後再執行一次。

## 完整腳本

```bash
#!/usr/bin/env bash
# =========================================================
# B300 GPU 伺服器基本驗證與壓測前置腳本
# 適用：Ubuntu + NVIDIA GPU
# 功能：
#   1. 安裝 NVIDIA driver（提供 nvidia-smi）
#   2. 安裝 NVIDIA DCGM（提供 dcgmi）
#   3. 安裝 InfiniBand/RDMA 診斷工具（提供 ibstat）
#   4. 執行 GPU、NVLink、IB/RoCE 基本驗證
# =========================================================

set -euo pipefail

# ===== 可調整參數 =====
INSTALL_DRIVER="auto"       # auto / yes / no
INSTALL_DCGM="yes"
INSTALL_IB_TOOLS="yes"
DCGM_DIAG_LEVEL="3"         # 1=快速, 2=標準, 3=深度
IB_PORT="1"                 # InfiniBand 埠號，通常為 1

echo "=========================================="
echo " B300 GPU 基本驗證與環境準備"
echo " 開始時間：$(date)"
echo "=========================================="

# ===== 0. 確認 Ubuntu =====
if ! grep -qi ubuntu /etc/os-release; then
    echo "警告：此腳本主要針對 Ubuntu 設計。"
fi

# ===== 1. 更新套件庫 =====
echo "更新 apt 套件庫..."
sudo apt update

# ===== 2. 安裝基礎工具 =====
echo "安裝基礎工具..."
sudo apt install -y \
    build-essential \
    git \
    curl \
    wget \
    pciutils \
    lsof \
    numactl

# ===== 3. 安裝 NVIDIA Driver / nvidia-smi =====
install_nvidia_driver() {
    echo "檢查 NVIDIA driver..."

    if command -v nvidia-smi >/dev/null 2>&1; then
        echo "已偵測到 nvidia-smi："
        command -v nvidia-smi
        nvidia-smi --query-gpu=name,driver_version,pci.bus_id --format=csv
        return 0
    fi

    if [ "$INSTALL_DRIVER" = "no" ]; then
        echo "跳過 NVIDIA driver 安裝。"
        return 1
    fi

    echo "未偵測到 nvidia-smi，準備安裝 NVIDIA driver..."

    sudo apt install -y ubuntu-drivers-common

    echo "可用的 NVIDIA driver："
    sudo ubuntu-drivers list || true

    echo "安裝 NVIDIA 建議驅動..."
    sudo ubuntu-drivers install --gpgpu

    echo "NVIDIA driver 安裝完成。"
    echo "請重新開機後再次執行此腳本，以完成驗證。"
    echo "重新開機指令：sudo reboot"
    exit 0
}

install_nvidia_driver || true

# ===== 4. 安裝 DCGM / dcgmi =====
install_dcgm() {
    if [ "$INSTALL_DCGM" != "yes" ]; then
        echo "跳過 DCGM 安裝。"
        return 0
    fi

    if command -v dcgmi >/dev/null 2>&1; then
        echo "已偵測到 dcgmi：$(command -v dcgmi)"
        return 0
    fi

    echo "安裝 NVIDIA DCGM..."
    sudo apt install -y datacenter-gpu-manager

    echo "啟動 DCGM 服務..."
    sudo systemctl enable --now dcgm || true
}

install_dcgm

# ===== 5. 安裝 InfiniBand / RDMA 診斷工具 =====
install_ib_tools() {
    if [ "$INSTALL_IB_TOOLS" != "yes" ]; then
        echo "跳過 InfiniBand 工具安裝。"
        return 0
    fi

    if command -v ibstat >/dev/null 2>&1; then
        echo "已偵測到 ibstat：$(command -v ibstat)"
        return 0
    fi

    echo "安裝 InfiniBand/RDMA 診斷工具..."
    sudo apt install -y \
        infiniband-diags \
        rdma-core \
        librdmacm-dev \
        libibverbs-dev \
        ibverbs-utils \
        perftest
}

install_ib_tools

# ===== 6. 基本驗證：nvidia-smi =====
echo "=========================================="
echo " 基本驗證 1：nvidia-smi"
echo "=========================================="

if ! command -v nvidia-smi >/dev/null 2>&1; then
    echo "錯誤：nvidia-smi 仍無法使用，請確認 NVIDIA driver 已正確安裝並重新開機。"
    exit 1
fi

nvidia-smi

echo "GPU 摘要："
nvidia-smi --query-gpu=index,name,driver_version,pci.bus_id,memory.total,temperature.gpu,power.draw --format=csv

# ===== 7. 基本驗證：DCGM 診斷 =====
echo "=========================================="
echo " 基本驗證 2：dcgmi diag Level ${DCGM_DIAG_LEVEL}"
echo "=========================================="

if command -v dcgmi >/dev/null 2>&1; then
    echo "執行 DCGM Level ${DCGM_DIAG_LEVEL} 診斷..."
    dcgmi diag -r "${DCGM_DIAG_LEVEL}"
else
    echo "警告：找不到 dcgmi，略過 DCGM 診斷。"
fi

# ===== 8. 基本驗證：NVLink =====
echo "=========================================="
echo " 基本驗證 3：NVLink 狀態"
echo "=========================================="

echo "NVLink 狀態："
nvidia-smi nvlink -s || echo "此系統可能不支援 NVLink，或目前沒有 NVLink 連接。"

echo "NVLink 拓撲："
nvidia-smi topo -m || true

# ===== 9. 基本驗證：InfiniBand / RoCE =====
echo "=========================================="
echo " 基本驗證 4：ibstat 與 RDMA 裝置"
echo "=========================================="

if command -v ibstat >/dev/null 2>&1; then
    echo "InfiniBand 裝置狀態："
    ibstat

    echo "InfiniBand 埠 ${IB_PORT} 狀態："
    ibstat -p "$IB_PORT" || true
else
    echo "警告：找不到 ibstat，請確認已安裝 infiniband-diags。"
fi

echo "RDMA 裝置列表："
ibv_devices || true

echo "RDMA 裝置詳細資訊："
ibv_devinfo || true

# ===== 10. 基本驗證：PCIe GPU 裝置 =====
echo "=========================================="
echo " 基本驗證 5：PCIe GPU 裝置"
echo "=========================================="

echo "NVIDIA GPU PCIe 裝置："
lspci | grep -Ei 'nvidia|3d controller' || true

# ===== 11. 基本驗證：核心與驅動日誌 =====
echo "=========================================="
echo " 基本驗證 6：驅動與系統日誌"
echo "=========================================="

echo "NVIDIA kernel module："
lsmod | grep -E '^nvidia' || true

echo "最近的 NVIDIA 相關核心訊息："
sudo dmesg | grep -Ei 'nvidia|nvlink|xid|ecc' | tail -n 100 || true

echo "=========================================="
echo " 基本驗證完成：$(date)"
echo "=========================================="
echo ""
echo "後續建議："
echo "1. 若 dcgmi diag 通過，可開始 GPU Burn 壓力測試。"
echo "2. 若出現 Xid、ECC、NVLink 或 IB 錯誤，請先停止壓測並保存日誌。"
echo "3. 壓測期間建議持續監控："
echo "   nvidia-smi dmon -s pucvmet -d 5"
echo "   dcgmi dmon -e 100,101,102,203,204 -d 5000"
```

## 儲存與執行

將內容儲存為：

```bash
b300_basic_validation.sh
```

執行：

```bash
chmod +x b300_basic_validation.sh
./b300_basic_validation.sh
```

## 可調整參數

| 參數 | 預設值 | 說明 |
|---|---|---|
| `INSTALL_DRIVER` | `auto` | 若已有 `nvidia-smi` 則不重裝；設為 `no` 可完全跳過驅動安裝 |
| `INSTALL_DCGM` | `yes` | 是否安裝 NVIDIA DCGM 與 `dcgmi` |
| `INSTALL_IB_TOOLS` | `yes` | 是否安裝 InfiniBand／RDMA 診斷工具 |
| `DCGM_DIAG_LEVEL` | `3` | DCGM 診斷等級；`1` 快速、`2` 標準、`3` 深度 |
| `IB_PORT` | `1` | InfiniBand 埠號；多埠或多功能卡時依 `ibstat` 輸出調整 |

## 基本驗證項目

| 驗證項目 | 工具 | 檢查內容 |
|---|---|---|
| NVIDIA driver | `nvidia-smi` | GPU 是否可見、driver 版本、溫度、功耗、記憶體 |
| GPU 健康 | `dcgmi diag` | GPU、HBM、NVLink 等硬體健康狀態 |
| GPU 互連 | `nvidia-smi nvlink` | NVLink 鏈路與拓撲 |
| 高速網路 | `ibstat`、`ibv_devinfo` | IB／RoCE HCA 狀態、埠狀態、速率與 RDMA 裝置 |
| 系統層 | `lspci`、`dmesg` | GPU PCIe 裝置、kernel module、Xid／ECC／NVLink 異常 |

## 壓測期間監控

另開終端機執行：

```bash
nvidia-smi dmon -s pucvmet -d 5
```

```bash
dcgmi dmon -e 100,101,102,203,204 -d 5000
```

重點觀察：

- GPU 溫度是否穩定，沒有持續上升或異常降頻。
- 功耗、風扇轉速與液冷進出口溫差是否在原廠規範內。
- 是否出現 Xid 錯誤、ECC 錯誤、GPU 掉卡、驅動崩潰或程序異常退出。
- 測試完成後再次執行 `nvidia-smi`，確認所有 GPU 仍可正常辨識且無錯誤狀態。

## 壓測
以下是一份適用於 **Ubuntu + NVIDIA GPU** 的單一腳本，會安裝編譯相依套件、下載 GPU Burn、編譯，並執行壓力測試。腳本預設測試 **1 小時**；請先確認驅動、CUDA toolkit 與散熱／供電條件正常。

```bash
#!/usr/bin/env bash
# B300 GPU 壓力測試腳本
# 適用環境：Ubuntu + NVIDIA GPU + 已安裝 NVIDIA driver 與 CUDA toolkit
# 用途：下載、編譯並執行 GPU Burn 壓力測試

set -euo pipefail

# ===== 可調整參數 =====
TEST_SECONDS=3600          # 壓測時間（秒）；3600 = 1 小時
GPU_MEMORY_PERCENT="90%"   # 使用 GPU 記憶體比例
GPU_INDEX=""               # 留空表示測試所有 GPU；例如 "0,1" 只測 GPU 0 與 1
INSTALL_DIR="$HOME/gpu-burn"

echo "=========================================="
echo " 開始 B300 GPU 壓力測試"
echo " 測試時間：${TEST_SECONDS} 秒"
echo "=========================================="

# ===== 1. 檢查 NVIDIA 驅動與 GPU =====
if ! command -v nvidia-smi >/dev/null 2>&1; then
    echo "錯誤：找不到 nvidia-smi。請先安裝 NVIDIA driver。"
    exit 1
fi

echo "目前 GPU 狀態："
nvidia-smi

# ===== 2. 安裝必要套件 =====
echo "安裝編譯所需套件..."
sudo apt update
sudo apt install -y \
    build-essential \
    git \
    curl \
    wget \
    pciutils

# ===== 3. 下載 GPU Burn =====
if [ -d "$INSTALL_DIR" ]; then
    echo "已存在 $INSTALL_DIR，更新原始碼..."
    cd "$INSTALL_DIR"
    git pull
else
    echo "下載 GPU Burn..."
    git clone https://github.com/wilicc/gpu-burn.git "$INSTALL_DIR"
    cd "$INSTALL_DIR"
fi

# ===== 4. 編譯 GPU Burn =====
echo "編譯 GPU Burn..."
make clean || true
make

# ===== 5. 執行壓力測試 =====
echo "開始壓力測試..."

if [ -n "$GPU_INDEX" ]; then
    echo "僅測試 GPU index：$GPU_INDEX"
    ./gpu_burn -i "$GPU_INDEX" -m "$GPU_MEMORY_PERCENT" "$TEST_SECONDS"
else
    echo "測試所有可見 GPU..."
    ./gpu_burn -m "$GPU_MEMORY_PERCENT" "$TEST_SECONDS"
fi

# ===== 6. 測試後檢查 =====
echo "壓力測試完成，檢查 GPU 狀態..."
nvidia-smi

echo "=========================================="
echo " 壓力測試結束"
echo " 請檢查：溫度、功耗、ECC、Xid 錯誤與降頻"
echo "=========================================="
```

## 使用方式

將上述內容儲存為 `b300_gpu_burn.sh`，然後執行：

```bash
chmod +x b300_gpu_burn.sh
./b300_gpu_burn.sh
```

## 參數說明

- `TEST_SECONDS=3600`：壓測 1 小時；若要 3 小時，改成 `10800`。
- `GPU_MEMORY_PERCENT="90%"`：GPU Burn 預設使用約 90% 可用 GPU 記憶體；若要較保守測試，可改成 `"50%"`。
- `GPU_INDEX=""`：預設測試所有 GPU；例如只想測 GPU 0 與 GPU 1，改成 `GPU_INDEX="0,1"`。
- 腳本會先檢查 `nvidia-smi`，確認 NVIDIA 驅動可用後才繼續執行。
- GPU Burn 是多 GPU CUDA 壓力測試工具，可對 NVIDIA GPU 進行高負載運算測試。 [github](https://github.com/wilicc/gpu-burn)

## 建議的驗收觀察

測試期間另開一個終端機，持續觀察：

```bash
nvidia-smi dmon -s pucvmet -d 5
```

重點檢查：

- GPU 溫度是否穩定，沒有持續上升或異常降頻。
- 功耗、風扇轉速與液冷進出口溫差是否在原廠規範內。
- 是否出現 Xid 錯誤、ECC 錯誤、GPU 掉卡、驅動崩潰或程序異常退出。
- 測試完成後再次執行 `nvidia-smi`，確認所有 GPU 仍可正常辨識且無錯誤狀態。

## 安全提醒
GPU Burn 與 DCGM Level 3 診斷會使 GPU 進入高負載、高溫與高功耗狀態。B300 屬於高功率資料中心 GPU，正式壓測前應確認：

- 機櫃供電與斷路器容量。
- 散熱或液冷系統正常。
- 液冷洩漏偵測與緊急停機程序。
- BMC／SEL、kernel log 與 DCGM 告警已啟用。
- 若出現溫度異常、液冷流量異常、供電告警或反覆 Xid，立即停止測試並保存日誌。
