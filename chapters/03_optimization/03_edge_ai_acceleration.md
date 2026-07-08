# 第三章 優化 Optimization

## 3. 邊緣運算優化 Edge AI Acceleration

在現代機器人感知系統中，深度學習模型（例如 YOLO 障礙物偵測、RT-DETR、或語意分割演算法）已成為不可或缺的核心組件。
然而，在機器人搭載的邊緣運算主機（Edge Devices）上，算力與功耗（Power Consumption）通常受到嚴格限制。
為了讓 AI 模型在不消耗過多電量與 CPU 資源的前提下，能夠高幀率、低延遲地運行，開發團隊必須針對底層硬體晶片架構進行專門的推理加速優化。

本節將深入探討目前機器人產業最常見的兩大晶片系統陣營──**NVIDIA Jetson** 與 **Intel x86** 下的加速實務。

---

### 3.1 NVIDIA 體系：ROS 2 + Jetson + TensorRT 加速

NVIDIA Jetson 平台（如 Jetson Orin Nano, Orin NX, AGX Orin）是目前高階 AI 機器人的運算主力。其核心加速技術為 **TensorRT**。

#### 3.1.1 TensorRT 的加速原理
TensorRT 是一個高效能的深度學習推理（Inference）優化器和運行時（Runtime）庫。當我們在 PC 端訓練好模型（如 PyTorch 的 `.pt` 格式）並導出為通用 **ONNX** 格式後，將其送入 Jetson 板載的 TensorRT 編譯引擎，會進行以下底層優化：
*   **算子融合 (Layer & Tensor Fusion)**：將神經網路中的多個層（例如 Convolution, Bias, Relu）合併為一個單一算子，減少記憶體讀寫（I/O）與 GPU 核心切換的開銷。
*   **量化 (Precision Calibration)**：將原本的 FP32（32位元浮點數）模型，安全地量化為 **FP16**（半精度浮點數）或 **INT8**（8位元整數）。這能在模型精度幾乎不下降的前提下，使推論速度提升數倍，並大幅降低 GPU 記憶體佔用。
*   **核心自動調優 (Kernel Auto-Tuning)**：根據當前 Jetson 晶片的具體硬體規格（如 CUDA 核心與張量核心 Tensor Cores 數量），自動挑選最適合的底層矩陣乘法演算法。

#### 3.1.2 ROS 2 整合實務：NVIDIA Isaac ROS
為了讓開發者免於手動撰寫複雜的 C++ TensorRT 介面，NVIDIA 官方推出了針對 ROS 2 優化的 **Isaac ROS** 開源套件：
*   **Isaac ROS DNN Inference**：
    內建對 TensorRT 的支持。開發者只需將 ONNX 模型放入指定路徑，設定對應的設定檔，此節點便會自動將其編譯為 TensorRT `.engine` 格式，並高效率地訂閱 ROS 2 的 `sensor_msgs/msg/Image` 話題進行實時推論。
*   **零拷貝硬體加速 (NITROS)**：
    利用 NVIDIA Hardware Acceleration 技術（NITROS），相機捕獲的影像直接存放在 GPU 顯存（CUDA Memory）中，影像裁剪、縮放、預處理與 TensorRT 推論**全程在 GPU 內完成，無需拷貝回 CPU 記憶體**。這能釋放極大量的 CPU 資源，讓大腦可以專注於路徑規劃與控制邏輯。

---

### 3.2 Intel 體系：ROS 2 + Intel CPU / x86 + OpenVINO 加速

對於許多採用標準工業 PC、搭載 Intel 處理器（如 Core i5/i7/i9）或結合 Intel RealSense 深度相機的移動機器人，其 AI 加速的黃金鑰匙則是 **OpenVINO Toolkit**。

#### 3.2.1 OpenVINO 的加速原理
OpenVINO 是 Intel 推出的跨平台深度學習部署套件，旨在將 AI 模型完美優化並無縫運行在 Intel 的多種硬體（CPU、整合顯示卡 iGPU、獨立顯示卡 dGPU 以及 NPU / VPU 晶片）上：
*   **模型優化器 (Model Optimizer)**：
    將 PyTorch, TensorFlow 或 ONNX 模型轉換為 OpenVINO 的中間表示格式（IR 格式，包含 `.xml` 結構檔與 `.bin` 權重檔），優化神經網路的計算圖（Computational Graph）。
*   **硬體異構執行 (Heterogeneous Execution)**：
    支援「CPU + iGPU」協同工作。例如，可以設定將輕量級的影像預處理放在 CPU 執行，而將 YOLO 模型的核心卷積矩陣計算指派給 Intel CPU 內置的強大內顯（Intel Iris Xe Graphics）執行。這能在不增加額外獨立顯卡成本的前提下，壓榨出硬體的極限算力。
*   **向量化指令集優化**：
    在 Intel CPU 上，OpenVINO 會底層啟用 **AVX-512** 或 **AMX (Advanced Matrix Extensions)** 等高級向量化與矩陣加速指令集，使傳統 CPU 推論深度學習的速度產生質的飛躍。

#### 3.2.2 ROS 2 整合實務：OpenVINO ROS
在 ROS 2 開發中，社群與 Intel 官方維護了 `openvino_node` 相關套件：
*   **影像資料流無縫對接**：
    OpenVINO ROS 節點訂閱標準的影像話題（如來自 `realsense_ros` 的 `/camera/color/image_raw`），在內部將 ROS 影像轉化為 OpenVINO Tensor 進行極速推論，並發布自定義的偵測話題（如包含障礙物 2D 座標與類別名稱的自定義 Message）。
*   **結合深度圖進行 3D 空間定位**：
    將 OpenVINO 偵測出的 YOLO 2D 邊界框（Bounding Box），與 RealSense 輸出的空間對齊深度圖（Depth Image）進行疊加計算，即可直接計算出目標障礙物相對於機器人 `base_link` 座標系的三維物理位置（x, y, z），完美解決避障系統的目標物定位需求。
